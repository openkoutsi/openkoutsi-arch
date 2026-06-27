# Authentication, roles & onboarding

## Authentication

The backend authenticates requests with a **JWT bearer token**. The web app holds the access
token in memory and refreshes it transparently on a `401` (see [Frontend](frontend.md)).

- `POST /auth/login` — exchange credentials for an access token (and a refresh cookie).
- `POST /auth/refresh` — mint a new access token from the refresh cookie.
- `POST /auth/logout` — end the session.

All non-public API operations require the bearer token; the OpenAPI document declares a single
`bearerAuth` (HTTP bearer, JWT) security scheme applied globally.

!!! info "Change from v1 — token-scoped, no team_id"
    In v1 the JWT carried a `team_id` and many routes were nested under `/api/teams/{slug}/…`.
    In v2 the token carries only **`sub`** (the user) and **`roles`**. There is no team in the
    token and no `{slug}` in any path — the authenticated user fully determines scope. At login,
    a `team_slug` field is no longer needed.

## Roles

The role model is reduced to two levels:

| Role | Capability |
|---|---|
| **User** | Owns and manages their own athlete profile and training data. |
| **Instance admin** | Everything a user can do, plus instance administration: managing users, issuing invitations, and editing instance-wide settings (including LLM configuration). |

The v1 **coach** and **superadmin** roles are **dropped**. Coaching across athletes no longer
exists in the single-instance model, and superadmin duties fold into the single instance admin.

## Onboarding

Registration is **invite-only**:

1. **First run** — the setup wizard creates the first **instance admin**. No team is created.
2. An admin **issues an invitation** (instance-wide, no team association).
3. A new user **registers** with the invite token; registration is rejected without a valid
   instance-wide invitation.

```mermaid
flowchart LR
    Setup["First-run setup<br/>→ first instance admin"] --> Invite["Admin issues<br/>instance-wide invite"]
    Invite --> Register["User registers<br/>with invite token"]
    Register --> Active["Active user<br/>(own per-user DB)"]
```

## Consent

Data-consent is recorded **on the user row** in the registry DB (the v1 standalone
`DataConsent` table is absorbed into `users`). Users can export their data and delete their
account at any time.
