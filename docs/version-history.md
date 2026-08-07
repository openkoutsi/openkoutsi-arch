# Version history

The rest of this documentation describes only the **current** architecture. This page is where
the architectural changes **between major versions** are recorded, so the reference pages don't
have to carry the history inline.

## v1 → v2

v2 collapses the multi-tenant, team-based design into a **single-instance, per-user** model with
a **simplified, token-scoped API**. The defining changes, by area:

### Data & storage

- v1 was multi-tenant: a registry DB held users *and teams*, and each **team** had its own
  `teams/{id}/team.db` containing every member's athlete data. v2 **removes teams entirely** —
  each user's data lives in `users/{id}/user.db`, and team-level configuration became a single
  instance-wide setting (`instance_settings`).
- **Removed registry tables:** `teams`, `team_memberships`, `join_requests`, and `data_consents`
  (the last absorbed into the `users` row).
- FIT files and avatars moved from team-scoped storage to per-user storage, re-keyed to the
  user's encryption key.

#### Migration from v1 (team-based)

The collapse from teams to per-user databases was a one-time data migration. For each athlete the
migration script:

1. Initializes the user's database.
2. Copies every team-DB row into it, preserving IDs.
3. **Re-encrypts** FIT files and avatars from the old team key to the user key, and moves them
   under the user's directory.
4. Collapses roles (a user who was an administrator in any team becomes an instance
   administrator) and maps consent onto the user row.
5. Copies the first team's LLM settings into `instance_settings`.

Users who belonged to **multiple teams** were merged into a single athlete, with activities
de-duplicated by `(provider, external_id)`. After a verified run the team tables and the
`teams/` directory were removed.

### Auth & roles

- v1 JWTs carried a `team_id` and many routes were nested under `/api/teams/{slug}/…`. v2 tokens
  carry only **`sub`** (the user) and **`roles`**; the authenticated user fully determines scope,
  and login no longer needs a `team_slug`.
- The v1 **coach** and **superadmin** roles were **dropped**: coaching across athletes no longer
  exists in the single-instance model, and superadmin duties folded into the single instance
  admin.
- Data-consent moved from a standalone `DataConsent` table onto the `users` row.

### API (v1 → v2)

- **Pagination** — v1 paginated only `/activities`; every other collection returned a bare JSON
  array. v2 returns the shared `Page` envelope for every collection.
- **Token-scoped paths** — the `/teams/{slug}` prefix was removed; scope comes entirely from the
  bearer token.
- **No trailing slashes** — the v1 mix of `/athlete/` vs `/messages` was normalized to
  slash-free collection roots.
- **`/athlete`** — the redundant v1 `PUT` (identical to `PATCH`) was removed, leaving `GET` +
  `PATCH`.
- **Messages** — the v1 `/superadmin/messages*` set that mirrored `/messages*` 1:1 was replaced
  by a single `/messages` resource with a `scope` query parameter.
- **Metrics** — read-only analytics were consolidated under `/metrics`:

    | v2 | Replaced (v1) |
    |---|---|
    | `GET /metrics/bests/{kind}` (`distance` \| `power`) | `/distance/bests`, `/power/bests` |
    | `GET /metrics/ftp` | `/power/ftp-estimate` |
    | `GET /metrics/ftp/history` | `/metrics/ftp-history` |

- **Push actions** — the hardcoded, Wahoo-specific `…/push/wahoo` was generalized to the
  provider-agnostic `…/push/{provider}` with `ProviderPushRequest` / `ProviderPushResponse`.

### Frontend

- **Routing** — v1 nested the whole app under a team slug (`app/[locale]/t/[slug]/…`). v2 is
  token-scoped: the slug is dropped from every route, so pages live directly under
  `app/[locale]/…`.
- **Auth context** — `src/lib/auth.tsx` no longer exposes a `teamSlug` prop, and the team-scoped
  refresh path (`/api/teams/{slug}/auth/refresh`) became the plain `/api/v2/auth/refresh`.
- **Admin & setup** — the v1 superadmin page and the team / join-request flows were removed; the
  admin console became the instance-admin console and setup creates the first instance admin with
  no team fields.

## Notable v2 additions

Feature deltas layered onto v2 after the initial collapse.

### Auth & API — email-based signup + self-serve password reset

- **Email as a login identifier** — `users.email` (unique, nullable) and
  `users.email_verified_at` were added; `users.username` became nullable. Email is a login
  identifier **alongside** username (login accepts either), so invited/legacy accounts are
  unaffected. New `email_verification_tokens` table mirrors `password_reset_tokens`.
- **Admin-gated self-serve signup** — `instance_settings.allow_self_signup` (default off) plus a
  configured email provider let anyone register by email: `POST /auth/signup` creates a pending
  account and emails a verification link; `POST /auth/verify-email` activates it. Invitations
  keep working regardless.
- **Self-serve password reset** — `POST /auth/request-password-reset` emails a reset link to a
  verified account (always a generic response, no enumeration); the existing
  `POST /auth/reset-password` is unchanged.
- **Outbound email dependency** — verification and reset messages are sent through the swappable
  email module (Lettermint default, EuroMail alternative). Optional: with no provider configured
  the email-dependent features stay unavailable rather than erroring.

## Notable v2 removals

Breaking changes to the published v2 API, recorded here because there is no deprecation
window for them.

### API — the LLM chat proxy was removed

`POST /api/llm/chat` is **gone** (issue #45). It was a general-purpose passthrough: it took a
client-supplied `messages` array and forwarded it to the caller's resolved LLM, streaming or
one-shot. Alongside it went the `LlmChatRequest` and `ChatMessage` schemas, and with them the
last `text/event-stream` response in the API.

It was removed rather than deprecated because a client that supplies the message array
**controls the system prompt** — which would make the coach-scope and medical-boundary
guardrails removable by anyone holding an access token. It had been kept as the intended
foundation for conversational features; that work concluded the opposite, and builds its
messages server-side like every other AI feature.

The removal is breaking in principle only: the endpoint was never wired to a UI, never
documented for users, and has no known consumers — the frontend's own
`src/lib/llm.ts` explicitly noted that the browser must not call it. The BYOK configuration
endpoints (`/access`, `/models`, `/servers`, `/test-connection`, `/test-my-connection`) are
unaffected, as are the SSRF defences shared with the remaining outbound paths.
