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
(dashboard, activities, plans, workouts, profile, admin, setup) and live directly under
`app/[locale]/…`. Routing is **token-scoped** — there is no team slug in any path.
`middleware.ts` guards protected pages using only the non-secret `session` cookie; the backend
remains the real authority on access.

## Talking to the backend

`src/lib/api.ts` centralizes API access:

- **`apiFetch`** attaches the in-memory access token as a `Bearer` header and sends requests to
  the backend origin resolved at runtime by `getApiUrl()` — from the `API_URL` env var on the
  server, and from `window.__ENV__` (injected by the root layout from the same var) in the
  browser — so a single built image can target any environment. The access token is held in
  memory only (never persisted to storage).
- On a `401`, it transparently calls the refresh endpoint once and retries the request.
- **`setSessionCookie` / `clearSessionCookie`** manage a non-sensitive `session=1` cookie. It
  carries no secret — it only lets the Next.js middleware gate protected pages before the
  client-side auth provider has run.

`src/lib/auth.tsx` provides the auth context (current session, login/logout). It refreshes
against the `/api/auth/refresh` endpoint.

## Admin & setup

- The **admin** page is the **instance admin** console (users, invitations, and instance-wide
  LLM settings).
- The **setup** page is a first-run wizard that creates the first **instance admin**.

## Conventions

- **Internationalization** — all UI strings are translated; the app ships English and Finnish
  message catalogs that are kept in sync.
- **Mobile-first** — the UI must be usable on small screens with no element overflow and no
  horizontal page scrolling.
