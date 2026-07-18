# Backend

The backend ([`openkoutsi-backend`](https://github.com/openkoutsi/openkoutsi-backend)) is a
FastAPI application backed by a pure-Python domain library. It owns all storage and runs the
background tasks that pull data in from Strava and Wahoo.

## Layering

```mermaid
flowchart TD
    R["API routers<br/>backend/app/api/*"] --> S["Services<br/>backend/app/services/*"]
    S --> Core["Core library<br/>openkoutsi/*"]
    R --> Schemas["Pydantic schemas<br/>backend/app/schemas/*"]
    S --> Models["ORM models<br/>backend/app/models/*"]
    Models --> DB[("SQLite<br/>registry + per-user")]
```

- **Routers** (`backend/app/api/*`) — one module per resource (athlete, activities, metrics,
  goals, plans, workouts, integrations, messages, admin, …). They handle HTTP, validate input
  via Pydantic **schemas** (`backend/app/schemas/*`), resolve the authenticated user, and
  delegate to services. Mounted under `/api` by `backend/main.py`.
- **Services** (`backend/app/services/*`) — application logic: the provider-sync pipeline,
  `metrics_engine` (fitness/fatigue/form), the [LLM features](llm.md) (`llm_activity_analyzer`,
  `llm_plan_generator`, `llm_workout_generator`, `llm_training_status_analyzer`, and the shared
  `llm_client`), the `activity_workout_matcher`, `plan_adherence` (deterministic plan-adherence
  scoring), `pr_detection`, and `notifications`.
- **Core library** (`openkoutsi/`) — framework-agnostic domain code with no FastAPI or DB
  imports: `fit`/`fit_processing` (FIT decoding), `training_math` (training load, weighted power,
  power/distance bests), `categorization` (Coggan zone classification), `plan_builder`,
  `workout_schema`, and the `workout_formats/` exporters (Zwift `.zwo`, FIT workout, Wahoo plan).
- **ORM models** (`backend/app/models/*`) — SQLAlchemy 2 async models split across the registry
  DB and the per-user DB (see [Data & storage model](data-model.md)).

## Application lifecycle

`backend/main.py` builds the FastAPI app, installs middleware (CORS scoped to the frontend
origin, a security-headers middleware, and a rate limiter), and includes every router. On
startup its lifespan handler initializes the registry database and the separate
[LLM-usage database](data-model.md) (see the [subscription gate & usage tracking](llm.md#subscription-gating-usage-tracking-issue-9)),
then launches the two **background pollers** as asyncio tasks:

```python
strava_poller = asyncio.create_task(strava_bridge_poller())
wahoo_poller  = asyncio.create_task(wahoo_bridge_poller())
```

Each poller loops every 60 seconds, fetches pending events from its bridge, processes them, and
claims them. See [Integrations](../integrations/index.md).

## The provider sync pipeline

`backend/app/services/provider_sync.py` is a **single generic pipeline** used by every provider
(Strava, Wahoo, and manual uploads). The provider-specific clients are looked up from a registry
and only differ in how they list activities and fetch data; the dedup, storage, and metrics
logic is shared.

### One workout, many sources

A real-world workout is modelled as **one `Activity`** with **one `ActivitySource` per
provider** that observed it. If you ride with a Wahoo head unit and also have Strava connected,
the same ride produces one `Activity` and two `ActivitySource` rows.

To decide which source's data populates the `Activity`'s metrics, sources are ranked by
**priority** (lower wins):

| Priority | Source |
|---|---|
| 1 | `upload` — manual FIT upload |
| 2 | `wahoo` — cloud sync **with** a FIT file |
| 3 | `strava` — Strava API (stream-based) |
| 4 | `wahoo` — cloud sync **without** a FIT file |
| 5 | `manual` — manually entered activity |

When a new source arrives with higher priority than the one currently populating the activity,
the metrics, streams, intervals and bests are deleted and re-derived from the better source.

### Find-or-create and deduplication

For each incoming activity the pipeline:

1. **Skips** it if this exact `(provider, external_id)` already has an `ActivitySource`.
2. Otherwise looks for an existing `Activity` within a **±5-minute window** of the start time.
   If found (and it doesn't already carry a source from this same provider), it **attaches a new
   source** to that activity rather than creating a duplicate.
3. Otherwise **creates** a new `Activity` + `ActivitySource`.

```mermaid
flowchart TD
    A["Incoming activity"] --> B{"Same (provider,<br/>external_id) seen?"}
    B -- yes --> Skip["Skip"]
    B -- no --> C{"Existing Activity<br/>within ±5 min?"}
    C -- yes --> D["Attach new source;<br/>repopulate if higher priority"]
    C -- no --> E["Create Activity<br/>+ ActivitySource, then populate"]
```

Because two providers can deliver the same ride almost simultaneously (a Wahoo webhook and a
Strava sync firing within milliseconds), the dedup-window query and the create/attach step run
under a **per-user activity lock**, and the new row is **committed before the lock is released**.
This prevents two concurrent syncs from each seeing an empty window and creating duplicates.

### Data population

Once a source is attached, the pipeline fills in the activity:

- **FIT-first** (Wahoo and any FIT-capable provider): download the FIT, store it
  **encrypted on disk** under the user's directory, parse it with the core library, and compute
  weighted power, training load, intensity, zone/category, power/distance bests, streams,
  intervals, and a frozen per-activity `zone_times` snapshot (time-in-zone for power + HR,
  using the athlete's zones at processing time — see [data model](data-model.md)).
- **Stream-based fallback** (Strava): pull the activity streams from the API and compute the
  same metrics from those samples.

OAuth tokens are refreshed transparently before expiry (`ensure_fresh_token`), with
provider-specific lookahead windows — see the per-provider pages.

## Deterministic metrics: fitness and plan adherence

Two families of scores are computed by **plain arithmetic** in the core library and orchestrated
by services — never by the LLM, so they are always-on and **not gated behind the LLM
subscription**. The LLM daily training status may *reference* the numbers, but is not their source.

### Fitness / Fatigue / Form

`metrics_engine` applies the Banister impulse-response model (`openkoutsi.fatigue_metrics`) over
daily training Load, persisting a `daily_metrics` snapshot per day. `catch_up_metrics` backfills
missing days and repairs rows made stale by deleted activities.

### Plan adherence scoring

Sits beside `activity_workout_matcher` and `metrics_engine`:

- **Pure math** — `openkoutsi/plan_adherence.py` (no FastAPI/DB imports).
- **Orchestration** — `backend/app/services/plan_adherence.py` (reads workouts + linked
  activities, rolls up the plan score, persists the snapshot).

Two scores are produced:

1. **Per-workout match score (0–100)** — how well the linked activity/activities fulfilled one
   planned workout. Since a workout may be satisfied by **several** linked activities, scoring
   operates on the **summed** actuals.
   - *Cycling* — graded on Load + duration with a symmetric per-dimension deviation
     `dim_score = clamp(1 − |actual − target| / target, 0, 1)` (on target → 1.0; 20% off either way
     → 0.8; ≥100% over or 0 → 0.0), blended `score = 100 × (0.70 × load_score + 0.30 ×
     duration_score)` when both targets exist, else whichever exists, else completion-only.
   - *Supplemental (non-cycling)* — done/missed (100 if ≥1 activity linked, else a miss).
   - A **completed** workout is floored at `COMPLETED_MIN_SCORE = 50` — a session that was
     actually done, however far off target (e.g. an 85-min endurance ride ridden as a 4-hour
     Z1/Z2 spin), never scores as low as an outright miss. The floor is applied at the service
     layer (`workout_match_score`); the pure per-dimension math stays unfloored. Missed and
     skipped workouts are unaffected.
2. **Plan adherence score (0–100)** — a Load-weighted roll-up over the *elapsed* portion of the
   plan: `adherence = 100 × Σ(weight_i × score_i / 100) / Σ(weight_i)`. Weight is `target_load`
   for cycling (fallback `duration_min`); supplemental workouts get a flat weight
   `0.75 × mean(cycling target_load)` (fallback `30`). Workout states:
   - **Completed** → the match score at full weight.
   - **Skipped** → score 0 at partial weight `(1 − f) × weight`, where the skip reason maps to a
     fixed forgiveness factor `f` (illness/injury `0.90`, fatigue `0.60`, travel `0.50`, weather
     `0.40`; unrecognized/free-form/none `0.10`) — a plain lookup, no free-text parsing.
   - **Missed** (past, empty, not a rest day) → score 0 at full weight.
   - **Rest day / future** → excluded. **Today** → scored provisionally if an activity is linked,
     otherwise held in grace until the day rolls over.

The 60% auto-match threshold in `activity_workout_matcher._matches` and the deviation used by the
score both draw on the shared `meets_threshold` comparison in `openkoutsi.plan_adherence`, so the
matcher gate and the score cannot drift apart.

Scores are surfaced on `PlannedWorkoutResponse.match_score` and `TrainingPlanResponse`
(`adherence_score` + summary), and persisted as a **`plan_adherence_daily`** snapshot per active
plan per day for charting via `GET /plans/{id}/adherence`. `catch_up_adherence` (mirroring
`catch_up_metrics`) recomputes every day in `[start_date, today]` for each active plan and
rewrites any snapshot that is missing **or stale** — self-healing stored days invalidated by
retroactive changes (a link/unlink to an old workout, an edited past workout, a formula change),
while leaving matching days untouched so the pass stays idempotent. It reads with
`populate_existing` so it always scores against current DB state. Recompute piggybacks the daily
first-read hook and the manual/webhook ingest paths — the Strava/Wahoo syncs now also auto-link
ingested activities to planned workouts so adherence does not silently under-count.
