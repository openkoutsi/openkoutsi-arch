# API v2 contract

The public API is a REST interface served under the **`/api/v2`** base path and secured by a
JWT bearer token. This page describes the **v2 conventions** of the current architecture — a
contract that is consistent and predictable across every resource.

!!! note "Canonical spec"
    The authoritative, generated contract is `openapi.json` in the
    [`openkoutsi-backend`](https://github.com/openkoutsi/openkoutsi-backend) repository. This
    page documents the *conventions* that shape it, not every path.

## Tags

Operations are grouped by tag: `auth`, `athlete`, `activities`, `metrics`, `goals`, `plans`,
`workouts`, `achievements`, `integrations`, `messages`, `bikes`, `courses`, `admin`, and
`public`.

## Conventions

### 1. One pagination envelope for every collection

Every collection returns the same shared **`Page`** envelope — `items`, `total`, `page`,
`page_size` — composed into a typed page via `allOf` (`ActivityPage`, `GoalPage`, …). Pagination
parameters (`page`, `page_size`) are identical everywhere.

```jsonc
// GET /activities?page=1&page_size=50  → ActivityPage
{ "items": [ /* Activity */ ], "total": 128, "page": 1, "page_size": 50 }
```

### 2. Token-scoped paths (no team slug)

There is no `/teams/{slug}` prefix. Scope comes entirely from the bearer token. Team-admin
operations live under `/team/...` only where an admin acts on the instance, resolved from auth —
not from a path slug. See [Auth, roles & onboarding](../architecture/auth.md).

### 3. No trailing slashes

Collection roots have no trailing slash (`/activities`, `/goals`, `/plans`, …).

### 4. `GET` + `PATCH` only on `/athlete`

The athlete resource exposes `GET` and `PATCH` only.

### 5. A single `/messages` resource with a scope filter

One messages resource serves both users and admins. Admin visibility is a query parameter, not a
separate route tree:

```
GET /messages?scope=self            # default
GET /messages?scope=team|all        # admin only
```

### 6. Consolidated analytics under `/metrics`

All read-only analytics live under `/metrics`:

| Endpoint | Returns |
|---|---|
| `GET /metrics/bests/{kind}` (`distance` \| `power`) | Best efforts for the given kind |
| `GET /metrics/ftp` | Current FTP estimate |
| `GET /metrics/ftp/history` | FTP history |
| `GET /metrics/fitness`, `/metrics/fitness/current` | fitness/fatigue/form series and current values |
| `GET /metrics/efficiency` | Aerobic efficiency trend over steady endurance rides (derived on read) |
| `GET /metrics/zones/{activity_id}` | Frozen per-activity time-in-zone snapshot (power + HR) |
| `GET /metrics/zones/weekly` | Accumulated time-in-zone per ISO week over a period (power + HR) |
| `GET /metrics/intensity-distribution` | Three-band distribution and its shape over a block (derived on read) |

!!! note
    `/metrics/zones/weekly` is declared before `/metrics/zones/{activity_id}` so the literal
    `weekly` segment isn't captured as an activity id.

`/metrics/intensity-distribution` takes the usual `start` / `end` / `days` triple, but unlike its
siblings an unspecified window is **not** all of history — it defaults to 84 days, because a
distribution over every ride ever recorded answers nothing. `method=time|session` picks the
counting unit and `basis=power|hr` the zone set (`basis` is ignored, and echoed as `null`, for
`method=session`). Both are echoed back: the two methods legitimately disagree, so a response
that didn't state which one produced it would be unusable. The response also carries coverage
counts and a `zone_definitions_changed` flag, so a caller can tell a well-founded distribution
from one drawn from six rides across two different FTPs.

### 7. Provider-agnostic push actions

Pushing a workout or plan to a head-unit provider is generic — the provider is a path parameter,
not baked into the URL or the schema:

```
POST /workouts/{workout_id}/push/{provider}
POST /plans/{plan_id}/push-upcoming/{provider}
```

Both take a `ProviderPushRequest` and return a `ProviderPushResponse`. This mirrors the existing
`/integrations/{provider}/…` design and lets new providers slot in without new endpoints.

### 8. Consolidated avatar endpoints

Avatars reduce to an owner-scoped pair plus a public read:

| Operation | Path |
|---|---|
| Upload / replace own avatar | `PUT /athlete/avatar` |
| Delete own avatar | `DELETE /athlete/avatar` |
| Get a user's avatar (auth) | `GET /athletes/{athlete_id}/avatar` |
| Get a public avatar (no auth) | `GET /public/athletes/{athlete_id}/avatar` |

### 9. Explicit operation identifiers and named models

Every operation sets an explicit `operationId` and human `summary` (e.g. `listActivities`,
`pushWorkoutToProvider`), and responses use **named** schemas. This keeps FastAPI's
auto-generated function names and inline array titles out of the public contract.

### 10. Async AI actions (trigger + poll)

Long-running LLM work is exposed as a `POST` that returns `202` immediately and a `GET` that
reports the persisted state (`pending` / `done` / `error`), with a 30-minute pending-timeout
recovery. **Goal guidance** follows this pattern:

```
POST /goals/{goal_id}/guidance   # triggerGoalGuidance → 202 {"status":"pending"}; body: {locale?}
GET  /goals/{goal_id}/guidance   # getGoalGuidance → GoalGuidanceResponse
```

`GoalGuidanceResponse` carries `status`, `verdict` (`realistic`/`ambitious`/`unrealistic`),
`guidance` (the streamed prose, with the leading `REALISM:` tag stripped), and `updated_at`.
The daily training-status endpoints (`POST/GET /athlete/training-status`) and the course
pacing plan (`POST/GET /courses/{course_id}/plan`) use the same shape. All are gated by the
LLM subscription check and return `403 llm_subscription_required` when denied on a gated
instance.

Course *analysis* deliberately does **not** use this pattern. Parsing and solving a course is
arithmetic measured in hundreds of milliseconds, so `POST /courses` returns the finished
segment table and a bad file is an immediate `400`/`422` — a job to poll would buy nothing and
cost a round trip. Only the written plan, which calls a model, is asynchronous.

## Example: a paginated collection

```jsonc
// GET /goals
{
  "operationId": "listGoals",
  "summary": "List goals (paginated)",
  "response": "GoalPage"   // allOf: Page + items: Goal[]
}
```

## Security

A single global security scheme applies: HTTP **bearer** auth. The unauthenticated operations
are `/health`, `/version`, `/public/...`, and the credential endpoints under `/auth/...` —
login, refresh, invite `register`, and the self-serve email flows: `POST /auth/signup`,
`POST /auth/verify-email`, `POST /auth/request-password-reset` and
`POST /auth/reset-password`. The signup/verify/reset endpoints are gated at runtime
(`allow_self_signup` and a configured email provider) and rate-limited; the request-reset and
signup endpoints always return a generic response to avoid account enumeration.

### Two credentials, one scheme

The bearer value is **either** a short-lived session **JWT** or a long-lived **personal access
token** (`okp_{id}_{secret}`, opaque and DB-backed). There is still exactly one security scheme
because both travel in the same `Authorization: Bearer …` header — which is also why PATs needed
no new `allow_headers` entry and no CORS change. See
[Auth, roles & onboarding](../architecture/auth.md#personal-access-tokens) for the model.

Operations carry an `x-personal-access-token` extension recording what a token may do with them:

```jsonc
// GET /activities
"x-personal-access-token": { "allowed": true, "scope": "activities:read" }

// GET /messages
"x-personal-access-token": { "allowed": false }
```

The same fact is repeated in each operation's `description`, so it survives into rendered
reference docs. Scope enforcement is **default-deny** at the server: an authenticated operation
carrying no declaration is unreachable by a token rather than open.

### Personal access token endpoints

`/tokens` — create, list, revoke, plus `GET /tokens/scopes` for the vocabulary and the allowed
lifetimes. **Session-authenticated only** (the router itself is closed to tokens, so a token can
never mint another), and there is deliberately **no update operation**: a token's name, scopes
and expiry are immutable, so `PUT`/`PATCH` would have nothing to act on.

`GET`/`DELETE /admin/users/{user_id}/tokens[/{token_id}]` give an administrator list-and-revoke
over one user's tokens — metadata only, never the name, and with no issue-on-behalf counterpart.

### Rate limiting

Limits are keyed by **principal**: `user:{user_id}` for an authenticated request, and the client
address for everything else. The fallback leaves the IP-keyed limits protecting the
unauthenticated endpoints unchanged. The key is the user rather than the token because tokens
can be minted freely — per-token buckets would make every limit multiplicative in a number
nothing caps.
