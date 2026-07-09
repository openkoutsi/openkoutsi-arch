# LLM & AI features

openkoutsi's coaching intelligence is delivered by a language model, but the platform
deliberately **owns no model and hard-codes no provider**. Every AI feature is built on a small,
uniform abstraction: an **OpenAI-compatible chat-completions API** reached through a
server-side proxy. Anything that speaks that dialect — a local
[Ollama](https://ollama.com/) instance, a hosted OpenAI-compatible endpoint, or a gateway in
front of several providers — can be plugged in without code changes.

The features are **optional**: if no endpoint is configured, or the user never triggers an AI
action, nothing is ever sent to a model and every other feature keeps working.

## The features

Four services under `backend/app/services/` use the LLM, plus one pass-through proxy:

| Feature | Service | Shape |
|---|---|---|
| **Activity analysis** | `llm_activity_analyzer` | Streaming prose |
| **Daily training status** | `llm_training_status_analyzer` | Streaming prose |
| **AI plan generation** | `llm_plan_generator` | One-shot JSON |
| **AI workout generation** | `llm_workout_generator` | One-shot JSON |
| **Chat proxy** | `POST /api/llm/chat` (`backend/app/api/llm.py`) | Streaming or one-shot, general-purpose |

The two **analysers** stream Server-Sent Events (SSE) straight through to the browser so the
user sees text as it is generated. The two **generators** make a blocking call and parse the
model's reply as JSON (`extract_json` strips markdown fences before `json.loads`), retrying once
with a correction nudge if the first response doesn't parse. The **chat proxy** is a
general-purpose passthrough to the configured model — with the API key never reaching the
browser — that a client can drive directly. It isn't wired to a UI today; it exists as the
foundation for future conversational features.

## OpenAI compatibility

Every call targets the OpenAI **`POST {base_url}/chat/completions`** contract and nothing else:

- **Request** — `{"model", "messages": [{role, content}], "stream", …}`. Roles are the usual
  `system` / `user` / `assistant`.
- **Response** — non-streaming replies are read from `choices[0].message.content`; streaming
  replies are consumed as `data:`-prefixed SSE chunks, extracting
  `choices[0].delta.content`, terminated by `data: [DONE]`.
- **Auth** — a bearer token in the `Authorization` header, when a key is configured.

Because that surface is tiny and universally implemented, "provider support" is a matter of
**configuration, not code**. Two deliberate compatibility choices keep a wide range of models
working out of the box:

!!! note "Temperature is omitted by default"
    `temperature_param()` leaves the `temperature` field **out** of the request unless a caller
    passes an explicit value, so each model applies its own default. This keeps thinking-enabled
    models — which reject any temperature other than `1` — working without special-casing.

!!! note "Upstream errors are surfaced, not swallowed"
    `raise_for_llm_status()` reads and includes the provider's response body in the raised error
    and the log line (httpx's built-in `raise_for_status` discards it). That body is where an
    OpenAI-compatible provider explains a 400/422 — e.g. an unsupported parameter for a thinking
    model — so failures are diagnosable.

## Presets: provider- and model-agnostic configuration

A **preset** is a self-contained (or partial) connection. An admin can offer several — e.g.
a fast local model next to one or more hosted providers — and users pick one. Each preset carries:

| Field | Meaning |
|---|---|
| `name` | Stable identifier (the stored selection value) |
| `label` | Human-friendly display name shown in the picker |
| `base_url` | The provider endpoint |
| `model` | The upstream model id sent in the request |
| `api_key` | Per-preset credential (encrypted at rest; see below) |
| `headers` | Extra request headers merged into every call for this preset |
| `body` | Extra chat-completion body params merged into every request |

A preset is **self-contained** — there are no instance-level single-config or env-var fallbacks,
so each preset carries everything it needs (a preset that omits a field simply doesn't have it).
The `headers` and `body` fields are what make the abstraction provider-agnostic in practice:

- **`headers`** carry provider-specific needs — a zero-data-retention header, an API version
  header, a gateway routing header — without the code knowing about any of them.
- **`body`** carries per-model tuning — `max_tokens`, a `reasoning_effort` level, a nested
  thinking config — attached to the one model that needs it. Core fields
  (`model` / `messages` / `stream`) always win, so extras can add but never break the request
  (`apply_body_extras`).

Presets are the instance's **entire** LLM configuration: they live in the
`instance_settings.llm_models` JSON column, and **the first preset in the list is the instance
default**. Users may also bring their own via the athlete's `app_settings` (BYOK — a single
`llm_base_url` / `llm_model` / `llm_api_key`, or personal presets). See the
[data & storage model](data-model.md).

## Resolving one request

All call sites funnel through **`resolve_llm()`** in `backend/app/services/llm_client.py`, which
produces a single `ResolvedLlm` (base URL, model, key, headers, body, **`source`**,
**`key_source`**). A preset is selected by name — a per-request override → the athlete's saved
`llm_model` → the **first preset** in the list (athlete presets take precedence over instance
presets).

!!! warning "No-mixing rule (BYOK)"
    As soon as the athlete configures **their own base URL** (the single `llm_base_url`, or a
    selected athlete preset with a `base_url`), resolution uses **only** athlete-level values —
    the instance's presets, key and headers are ignored entirely. This is the whole point: the
    instance's API key can **never** be sent to a user-chosen server. It also fixes an earlier
    leak where a user who set only their own base URL would inherit and transmit the instance key.

`source` records where the base URL came from (`user` / `instance`); **`source == "user"` is the
canonical "BYOK active" signal** consumed by entitlement gating (issue #9). `key_source` records
where the key came from (`user` / `instance` / `none`).

```mermaid
flowchart TD
    Req["Call site<br/>(chat / generator / analyser / test)"] --> BYOK{"Athlete has own base_url?"}
    BYOK -->|"yes — source = user"| U["base_url / model / key / headers:<br/>athlete only (instance ignored)"]
    BYOK -->|"no — source = instance"| Sel{"Select instance preset by name<br/>(request → athlete.llm_model → first preset)"}
    Sel --> P["Preset's base_url / model / key / headers / body"]
    U --> RL["ResolvedLlm (+ source, key_source)"]
    P --> RL
```

Two thin wrappers adapt this for their callers:

- **`resolve_llm_config(athlete, instance, user_id, *, requested_model, allow_instance_fallback)`**
  — the athlete-aware path used by the chat proxy and the generators. It enforces the use-time
  policy and raises **`LlmConfigError`** with a machine-readable `code` (`no_base_url` → HTTP 400,
  `server_not_allowed` → 403, `instance_fallback_disabled` → 403) that the API layer maps to a
  status. The **allow-list is checked here at use time** (BYOK URLs only), so the generators get
  it too. `allow_instance_fallback=False` is the hook for #9: when the user has no own config it
  raises instead of falling back to the instance.
- **`resolve_instance_llm(instance)`** — the instance-only path used by the automated analysers
  and the admin connection test, which aren't tied to a particular user's overrides.

`GET /api/llm/models` returns the presets a user may select (`{name, label}`) plus their current
effective selection (the first preset when nothing is saved), so the UI picker mirrors exactly
what `resolve_llm()` would choose.

## Security model

The browser never talks to the LLM directly — every call is **proxied server-to-server**:

- **Keys stay on the server.** API keys are encrypted at rest — instance keys with a Fernet key
  (`encrypt_instance_secret`), per-user keys with an HKDF-derived per-user key
  (`encrypt_secret(key, user_id)`) — decrypted in memory only for the outbound request, and
  **never returned to the browser** (`api_key_set` booleans only). The [no-mixing rule](#resolving-one-request)
  guarantees an instance key is never sent to a user-chosen server.
- **SSRF defences.** Because a user-supplied base URL could point at internal services,
  `check_url_safe()` accepts only `http`/`https`, resolves the hostname and **rejects
  link-local addresses** (the cloud metadata range), disables redirects, and connects to the
  pre-resolved IP to blunt DNS rebinding.
- **BYOK allow-list.** An optional admin allow-list (`LLM_ALLOWED_SERVERS`) restricts which base
  URLs a user may **bring**. It is enforced both at **save time** (`PATCH /api/athlete`, which
  also strips/validates the scheme and runs the SSRF check) and at **use time** in
  `resolve_llm_config` — and it only ever restricts user (BYOK) URLs, never admin-configured
  instance presets.
- **Bounded responses.** The proxy caps an upstream response at 32 MB to avoid unbounded memory
  use.

## Testing a connection

Two endpoints share a `_probe_llm_endpoint()` helper that sends a minimal "hello world" chat
completion and confirms a non-empty `choices[0].message.content` comes back. Because they
exercise the **real** request path — not a `GET /models` listing — they also validate a ZDR header
or a thinking config, and return both the prompt sent and the model's reply for display.

- **`POST /api/llm/test-connection`** (admin-only) tests the instance's selected preset.
- **`POST /api/llm/test-my-connection`** (any authenticated user) tests the caller's own **BYOK**
  config. Body values override the saved athlete config so the *Test* button works before saving;
  an omitted `api_key` falls back to the saved encrypted key. It enforces the allow-list + SSRF on
  the tested URL and is rate-limited, since it triggers an outbound request on demand.

## Why this shape

- **No lock-in.** Swapping providers, running fully local, or fronting several models behind a
  gateway are all configuration, never code.
- **One code path.** Every feature resolves config and makes requests the same way, so a change
  to headers, body handling, or SSRF policy applies everywhere at once.
- **Privacy is a deployment choice.** Whether data leaves the server is decided entirely by the
  configured endpoint — a property the [user documentation](https://openkoutsi.github.io/openkoutsi-docs/)
  makes explicit to end users.
