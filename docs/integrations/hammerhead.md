# Hammerhead (Karoo)

Hammerhead activities are imported through the **Hammerhead bridge** and the generic
provider-sync pipeline, like Wahoo's. The integration is deliberately **one-way**
(openkoutsi#69): the API could also accept routes, workouts and training metrics, but
nothing is ever written to the athlete's Hammerhead account. That keeps it as simple as
Strava and Wahoo.

## Connecting

OAuth 2.0 authorization code at `https://api.hammerhead.io/v1/auth/oauth/{authorize,token,deauthorize}`,
driven by the generic `/integrations/hammerhead/connect` and `/callback` routes. The token response
carries `user_id`, so — unlike Wahoo — `provider_athlete_id` needs no profile call.

Scope requested: `activity:read` only, which covers activities **and** webhooks. The token
response carries no `scope` field, so a user who declined it is found out on the first 403,
which the client raises as `ScopeNotGranted`.

A refusal on the consent screen redirects to the callback with `?error=access_denied&state=…`
and no `code`; the callback sends the browser back with `?hammerhead=denied`.

## Activity import

`HammerheadClient` (`services/providers/hammerhead.py`) plugs into the generic
[provider sync pipeline](../architecture/backend.md), with one twist.

The list endpoint (`GET /activities?page&perPage&startDate`) returns only `id`, `name`,
`createdAt`, `duration`, `distance` — no start time, no sport, no power/HR/cadence. `createdAt` is
a record-creation timestamp, and using it as the start time would break the ±5-minute duplicate
match against the same ride from Strava or Wahoo. So:

1. `list_activities()` returns **sparse** records (`createdAt` as a provisional start).
2. The pipeline skips anything already imported on `(provider, external_id)` alone — no further
   calls.
3. For an activity it will import or repair, the pipeline calls the new
   `BaseProviderClient.complete_activity()` hook **before the duplicate-window query**.
   Hammerhead's implementation downloads the FIT (`GET /activities/{id}/file`) and the detail
   (`GET /activities/{id}`, for `activityType` and the name), and returns a record dated by the
   FIT. The FIT is cached on the client, so the import that follows does not fetch it again.

Pagination uses `perPage=100` and stops on `currentPage >= totalPages` — no empty-page probe.
Requests go through `provider_client()` (usage attribution) and 429/5xx map through
`providers/throttling.py` like every provider.

- **Sport mapping:** `RIDE` → `Ride`, `EBIKE` → `EBikeRide`, `MOUNTAIN_BIKE` → `MountainBikeRide`,
  `GRAVEL` → `GravelRide`, `EMOUNTAIN_BIKE` → `EMountainBikeRide`, `VELOMOBILE` → `Velomobile`.
- **Source priority:** `hammerhead` with a FIT = **2**, beside Wahoo-with-FIT and above Strava (3)
  — the Karoo FIT is the device recording.

## Webhooks → bridge

`hammerhead_bridge/` follows the bridge pattern with two differences from Wahoo's:

- **HMAC, always on.** `X-Hmac-Signature` is HMAC-SHA256 of the raw body keyed with
  `HAMMERHEAD_WEBHOOK_SECRET`, verified against the bytes as received and compared in constant
  time. With no secret configured every delivery is refused.
- **Hammerhead does not redeliver** — a non-200 is ignored. The bridge only verifies, persists
  and returns 200; the queue (issue #50 claim/ack/nack, `max_queue_events`, 7-day purge) is the
  only retry an event gets.

The payload is only `{activityId, userId}`. `process_hammerhead_webhook` resolves the user via
`provider_connections.provider_athlete_id`, refreshes the token, and imports through
`complete_activity()` — two authenticated calls. It is idempotent on the activity id, which
covers Hammerhead's documented manual retry as well as bridge redelivery.

```mermaid
sequenceDiagram
    participant H as Hammerhead
    participant B as Hammerhead bridge
    participant A as Main app
    H->>B: POST /webhook {activityId, userId} + X-Hmac-Signature
    B->>B: verify HMAC over raw body, queue, 200
    A->>B: POST /events/claim
    B-->>A: events (claim token)
    A->>H: GET /activities/{id}/file + GET /activities/{id}
    A->>A: parse FIT → dedup window → user DB (priority 2)
    A->>B: POST /events/{id}/ack (or /nack on failure)
```

## Configuration

| Variable | Where | Purpose |
|---|---|---|
| `HAMMERHEAD_CLIENT_ID`, `HAMMERHEAD_CLIENT_SECRET` | Main app | OAuth app credentials |
| `HAMMERHEAD_BRIDGE_URL` | Main app | Base URL of the Hammerhead bridge to poll |
| `HAMMERHEAD_BRIDGE_SECRET` | Main app **and** bridge | Shared secret for polling auth (≥ 32 chars) |
| `HAMMERHEAD_WEBHOOK_SECRET` | Bridge | HMAC key for `X-Hmac-Signature` |

## Open questions

Not answered by the published spec, to confirm against the live API: rate limits (each new
activity costs a detail call plus a FIT download), units of the summary's `duration`/`distance`
(unused — the FIT is authoritative), the encoding of `X-Hmac-Signature` (the bridge accepts hex or
base64), whether granted scopes are ever returned, and whether the API licence requires visual
attribution.
