# Strava

Strava activities are imported through the **Strava bridge** and the generic provider-sync
pipeline. Strava exposes activity **streams** (not FIT files), so its data is imported via the
stream-based path.

## Connecting

A user connects Strava through the OAuth flow at `/integrations/strava/connect`. The resulting
connection (access token, refresh token, expiry, scopes) is stored encrypted in the registry
DB's `provider_connections` table.

## Webhooks → bridge

The Strava bridge (`strava_bridge/`) is a standalone public FastAPI service.

- **`GET /webhook`** — Strava subscription verification. Strava calls it with
  `hub.mode=subscribe`, `hub.verify_token`, and `hub.challenge`; the bridge echoes back
  `hub.challenge` when the verify token matches its `bridge_secret`.
- **`POST /webhook`** — receives an event. **Unauthenticated.** Strava documents the
  `hub.challenge` handshake and nothing else: no signing secret, no header, no statement of
  which bytes would be signed. The bridge does hold an `X-Hub-Signature-256` HMAC check against
  `strava_client_secret`, but it ships off (`strava_verify_webhook_signature = False`) — with it
  on, the fail-closed path answered every real delivery `401`, because Strava sends no such
  header. Only events with `object_type == "activity"` are queued; everything else is
  acknowledged and dropped.

  What carries the trust instead: the activity-only filter above, the unknown-owner drop during
  polling below, the re-fetch from Strava's own API (a forged payload asks for a sync, it cannot
  supply data), and `max_queue_events` bounding a flood. Turn the check back on once Strava
  documents the header, the secret, and the validation sequence — and confirm the documented
  scheme matches the implementation first.

Queued events record the aspect type (create/update/delete), the Strava owner id, and the raw
payload.

## Polling → import

The main app's `strava_bridge_poller` runs every 60 seconds (a no-op if `BRIDGE_URL` /
`BRIDGE_SECRET` aren't configured). It fetches pending events, hands each to
`strava_sync.process_webhook_event`, and claims it regardless of outcome (to avoid infinite retry
loops).

Import uses the **stream-based** path of the [provider sync pipeline](../architecture/backend.md):
the activity's power/HR/cadence/speed/altitude streams are fetched from the Strava API and used
to compute weighted power, training load, intensity, category, streams, and bests.

- **Source priority:** `strava` = **3** (a Wahoo FIT for the same ride, priority 2, would win
  and repopulate the activity).
- **Token refresh:** Strava tokens last ~6 hours; the pipeline refreshes when **≤30 minutes**
  remain (Strava's own recommendation).

```mermaid
sequenceDiagram
    participant S as Strava
    participant B as Strava bridge
    participant A as Main app
    S->>B: POST /webhook (activity event)
    B->>B: queue activity events (signature check off by default)
    A->>B: GET /events/pending
    B-->>A: events
    A->>S: fetch activity streams (refresh token if needed)
    A->>A: compute metrics → user DB (priority 3)
    A->>B: POST /events/{id}/claim
```

## Configuration

| Variable | Where | Purpose |
|---|---|---|
| `STRAVA_CLIENT_ID`, `STRAVA_CLIENT_SECRET` | Main app | OAuth app credentials |
| `BRIDGE_URL` | Main app | Base URL of the deployed Strava bridge to poll |
| `BRIDGE_SECRET` | Main app **and** bridge | Shared secret for polling auth and hub verification |
| `STRAVA_VERIFY_WEBHOOK_SIGNATURE` | Bridge | Require `X-Hub-Signature-256` — **off**; Strava does not document webhook signing |
| `STRAVA_CLIENT_SECRET` | Bridge | Validates `X-Hub-Signature-256`; only read when the flag above is on |

Deploy `strava_bridge/` to a public HTTPS URL and register that URL as the Strava webhook
subscription callback.
