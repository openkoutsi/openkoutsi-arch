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
`workouts`, `integrations`, `messages`, `admin`, and `public`.

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
| `GET /metrics/fitness`, `/metrics/fitness/current` | CTL/ATL/TSB series and current values |

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
The daily training-status endpoints (`POST/GET /athlete/training-status`) use the same shape.
Both are gated by the LLM subscription check and return `403 llm_subscription_required` when
denied on a gated instance.

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

A single global security scheme applies: HTTP **bearer** auth carrying a **JWT**. Public
endpoints (`/health`, `/version`, `/public/...`) are the only unauthenticated operations.
