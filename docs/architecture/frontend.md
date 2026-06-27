# Frontend

The web app ([`openkoutsi-web`](https://github.com/openkoutsi/openkoutsi-web)) is a
**Next.js 15** application (App Router) written in TypeScript, styled with Tailwind CSS, with
charts rendered by Recharts. It is a pure client of the backend REST API.

## Structure

```
src/
  app/[locale]/…        # App Router pages (localized via the [locale] segment)
  components/           # UI: activities, charts, plan, profile, workouts, onboarding, ui
  lib/                  # api.ts, auth.tsx, and domain helpers (metrics, calendar, plans, …)
  i18n/                 # next-intl configuration
  types/                # shared TypeScript types
```

Routing is localized through a `[locale]` segment. Pages are organized by feature
(dashboard, activities, plans, workouts, profile, admin, setup).

!!! info "Change from v1 — routing"
    v1 nested the whole app under a team slug: `app/[locale]/t/[slug]/…`. v2 is **token-scoped**:
    the team slug is dropped from every route, so pages live directly under `app/[locale]/…`.
    `middleware.ts` guards protected pages using only the non-secret `session` cookie; the
    backend remains the real authority on access.

## Talking to the backend

`src/lib/api.ts` centralizes API access:

- **`apiFetch`** attaches the in-memory access token as a `Bearer` header and sends requests to
  `NEXT_PUBLIC_API_URL`. The access token is held in memory only (never persisted to storage).
- On a `401`, it transparently calls the refresh endpoint once and retries the request.
- **`setSessionCookie` / `clearSessionCookie`** manage a non-sensitive `session=1` cookie. It
  carries no secret — it only lets the Next.js middleware gate protected pages before the
  client-side auth provider has run.

`src/lib/auth.tsx` provides the auth context (current session, login/logout). In v2 it exposes
the session without a `teamSlug` prop, and the team-scoped refresh path
(`/api/teams/{slug}/auth/refresh`) becomes a plain token-scoped `/api/v2/auth/refresh`.

## Admin & setup

- The **admin** page is the **instance admin** console (users, invitations, and instance-wide
  LLM settings). The v1 superadmin page and team/join-request flows are removed.
- The **setup** page is a first-run wizard that creates the first **instance admin** (no team
  fields).

## Conventions

- **Internationalization** — all UI strings are translated; the app ships English and Finnish
  message catalogs that are kept in sync.
- **Mobile-first** — the UI must be usable on small screens with no element overflow and no
  horizontal page scrolling.
