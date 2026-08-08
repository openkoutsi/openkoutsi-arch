# Data & storage model

openkoutsi stores everything in **SQLite** (WAL mode). Storage is a two-tier layout: one shared
**registry database**, **one database per user**, and a small dedicated
**LLM-usage database**.

## Two tiers

```mermaid
flowchart TD
    subgraph Registry["Registry DB — data/registry.db (shared)"]
        U["users (roles + consent)"]
        I["invitations (instance-wide)"]
        PR["password_reset_tokens"]
        PAT["personal_access_tokens"]
        PC["provider_connections (encrypted tokens)"]
        IS["instance_settings (single row)"]
        EN["llm_entitlements (per-user LLM access)"]
    end
    subgraph Usage["LLM-usage DB — data/llm_usage.db (separate)"]
        LU["llm_usage (instance-paid calls only)"]
    end
    subgraph PerUser["Per-user DB — data/users/&#123;id&#125;/user.db"]
        Ath["athlete (one per DB)"]
        Act["activities + sources + streams + intervals"]
        Best["power / distance bests"]
        Goals["goals"]
        Plans["plans + planned workouts"]
        Work["workout definitions"]
        Msg["message inbox"]
    end
    subgraph Files["Filesystem"]
        Fit["encrypted FIT files (per user)"]
        Av["avatars (per user)"]
    end
    U -->|owns| PerUser
    U --> PC
    U --> EN
    U --> PAT
    U -.records.-> LU
    PerUser --> Files
```

### Registry DB (`data/registry.db`)

Shared, instance-wide tables:

- **`users`** — credentials, **`roles`**, and consent fields.
- **`invitations`** — instance-wide invite tokens.
- **`password_reset_tokens`**.
- **`personal_access_tokens`** — long-lived, scoped credentials a user issues to their own
  tooling. Only `sha256(secret)` is stored (unique), alongside the user-written `name`, a
  JSON-encoded `scopes` list, `expires_at`, a coarsely-written `last_used_at`, `revoked_at` and
  `last_expiry_notice`. `user_id` is indexed and cascades on delete. **Expired and revoked rows
  are retained rather than pruned** — the audit log stores token ids, and a retained hash keeps a
  presented-but-revoked token distinguishable from an unknown one. See
  [Auth](auth.md#personal-access-tokens).
- **`provider_connections`** — a user's Strava/Wahoo OAuth connection. Access and refresh tokens
  are stored with an `EncryptedString` column type. A connection belongs to the **user globally**
  (one connect per provider, enforced by a `(user_id, provider)` unique constraint).
- **`instance_settings`** — a single-row table holding instance-wide configuration. Its LLM
  config is entirely the **`llm_models`** JSON column: a list of selectable presets (`name`,
  `label`, `base_url`, `model`, `api_key_enc`, `headers`, `body`) whose **first entry is the
  instance default**. Per-preset API keys are encrypted with `encrypt_instance_secret`. There is
  no instance single-config or global-headers column, and no env-var fallback (see the
  [LLM architecture](llm.md)). The single boolean **`llm_requires_subscription`** (default
  false) is the opt-in [LLM subscription gate](llm.md); until an admin flips it, LLM features
  work as before. **`allow_personal_access_tokens`** (default **true**, unlike the other gates —
  it preserves no prior behaviour) is the self-hoster's kill switch, checked at *authentication*
  so flipping it off stops tokens already issued.
- **`llm_entitlements`** — a per-user "LLM access" entitlement (one row per user, `user_id`
  unique). A table rather than a role because it carries expiry, provenance and audit fields
  (`status`, `source`, `granted_by_user_id`, `starts_at`, `expires_at`, `external_ref`, `notes`)
  and is an idempotent upsert target for the future payment handler. Roles keep meaning
  *permissions*; entitlements mean *commercial state*. Entitled predicate:
  `status = active AND starts_at <= now AND (expires_at IS NULL OR expires_at > now)`.

### LLM-usage DB (`data/llm_usage.db`, path configurable via `LLM_USAGE_DB`)

A **separate** database — its own SQLAlchemy `Base`, engine, sessionmaker and Alembic chain
(head `001`) — holding one append-only row per **instance-paid** LLM call in a single
**`llm_usage`** table (`user_id`, `created_at`, `feature`, `provider`, `model`,
`prompt_tokens`, `completion_tokens`, `total_tokens`, `key_source`, `duration_ms`; indexed on
`(user_id, created_at)`). It is kept apart from the registry DB so its high-volume, unbounded
rows can be pruned/rotated independently, and it carries no registry foreign keys (a
user-deletion sweep is a plain `DELETE … WHERE user_id = ?`). Input and output tokens are stored
**separately** — providers price them differently. **BYOK calls are never recorded**: the hoster
pays nothing for them, so every row is instance-paid (there is no `byok` column). See the
[LLM architecture](llm.md).

### Per-user DB (`data/users/{user_id}/user.db`)

Everything a single athlete owns — **one athlete per database**:

- The **athlete** profile (FTP, zones, and `app_settings`). `app_settings` holds the user's
  **BYOK** LLM config: `llm_base_url`, `llm_model`, and `llm_api_key_enc` (encrypted per-user with
  `encrypt_secret(key, user_id)` and never serialized back — reads expose a derived
  `llm_api_key_set` boolean). A non-empty `llm_base_url` means BYOK is active and only the user's
  own config is used (the [no-mixing rule](llm.md#resolving-one-request)).
- All **activities** with their `ActivitySource`, `ActivityStream`, `ActivityInterval`, and
  `ActivityPowerBest` / `ActivityDistanceBest` rows. Each activity also carries a
  `zone_times` snapshot (JSON — `{"hr": {"Z1": secs, …}, "power": {…}}`) accumulated from its
  per-second streams **at processing time, using the athlete's zones as they were then**. The
  snapshot is **frozen** once written: editing zones later only changes future activities, so
  historical weekly zone distributions stay stable. Legacy activities without a snapshot are
  backfilled on first read (using current zones) and then frozen. Note the snapshot only holds
  the zones a ride actually reached, so a missing key means "never went there", not "no such
  zone". The athlete's own `hr_zones` / `power_zones` are fixed-length — **five** and **seven**
  entries, contiguous, and with **API-owned names**: the write path overwrites whatever the
  athlete typed with the canonical labels, because the snapshot is keyed by name and the
  three-band mapping recovers a zone's position from it. Snapshots frozen before that rule
  are left as they are; they are mapped proportionally when their names can be placed, and
  dropped from the aggregate when they can't.

    Activities also carry **aerobic response metrics** (issue #37). Two of them are *not*
    stored: `efficiency_factor` (weighted power / avg HR) and `variability_index` (weighted /
    avg power) are pure ratios of columns already on the row, so they are **derived on read**
    in the response schema — the same reasoning as the per-workout match score above. Nothing
    can drift out of sync with its operands, and activities processed long before the feature
    existed carry both without a reprocess. Stored are the ones that need the streams or a
    point-in-time fit: `decoupling_pct` with a companion `decoupling_reason` (exactly one is
    set — a decoupling figure over a short or interval ride is noise, so the gate stores a
    stable reason code instead of a number), plus a `cp_w` / `w_prime_j` / `cp_fit_points`
    snapshot. `cp_fit_points` records how many duration bests that fit had available: a
    provider backlog import walks newest-first, so an old ride can be processed while almost
    nothing on or before its date exists yet, and this keeps those rides findable for a future
    re-fit rather than indistinguishable from a well-supported one. That
    snapshot is fit from the athlete's rank-1 power bests **as of the activity's own date**,
    not all-time, and is **frozen** for the same reason `zone_times` is: a ride's W′ story
    shouldn't silently change as the athlete's power curve moves. W′ balance itself is an
    `ActivityStream` row (`stream_type="w_bal"`, joules per second), following the `torque`
    precedent including its reprocess backfill. When CP can't be fit, both columns stay NULL
    and no stream is written — no W′ is invented.
- **goals**, training **plans** (with planned workouts), and standalone **workout** definitions.
  Each goal also carries on-demand AI-guidance columns (`guidance`, `guidance_verdict`,
  `guidance_status`, `guidance_updated_at`) — the streamed coach prose, its parsed
  `realistic`/`ambitious`/`unrealistic` verdict, and the pending/done/error state with a
  timestamp for pending-timeout recovery (mirroring the athlete's `training_status*` columns).
  A planned workout can be satisfied by **several** activities via the
  `planned_workout_activities` join table.
- Deterministic **daily snapshots**: `daily_metrics` (fitness/fatigue/form + daily Load) and
  `plan_adherence_daily`. The latter, keyed by `(athlete_id, plan_id, date)`, stores the plan's
  "so far" adherence `score` (Float, nullable) plus denormalized counts (`completed`, `missed`,
  `skipped`, `pending`) — one snapshot per active plan per day, same shape/pattern as
  `daily_metrics`. The per-workout match score is **derived on read**, not stored.
- Earned **achievement tiers** in `achievement_unlocks`, keyed by
  `(athlete_id, achievement_id, tier)`. Derived state like the snapshots above, with two
  differences worth noting. First, the **catalogue lives in code** (`openkoutsi.achievements`), not
  in rows — `achievement_id` is a stable machine key whose display name is an i18n string in the
  web app, so no user-facing prose is stored and adding an achievement needs no migration. Second,
  the reconcile pass **deletes as well as inserts**: unlocks are a pure function of the athlete's
  current data, so removing the activity that earned a tier revokes it rather than leaving a badge
  the history no longer supports. The two timestamps do different jobs — `achieved_on` is derived
  from the history (back-filling an old ride moves it *earlier*, never to today), while
  `created_at` is wall-clock and only drives the "new" marker and the inbox notification.
- The user's **message inbox**.

The schema is created idempotently, so an existing message-only DB simply gains the training
tables on first initialization.

## Encryption

Sensitive data is encrypted at rest and **re-keyed per user**:

- **Provider tokens** — `EncryptedString` columns in `provider_connections`.
- **FIT files** — written to the user's directory and encrypted on disk, derived from the
  user's key (`info="user-key:{user_id}"`).

Because keys are scoped to `user_id`, a user's data is cryptographically isolated even though all
users share one instance.

## Migrations

Schema changes are managed with **Alembic**. There are three migration environments: one for the
registry DB, one for the per-user DB schema (applied to each user database), and one for the
separate LLM-usage DB.

For how the storage model reached this two-tier layout, see [Version history](../version-history.md).
