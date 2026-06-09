---
status: draft
repos:
  - prog-strength-api
  - prog-strength-agent
  - prog-strength-web
  - prog-strength-docs
---

# Per-User Daily Usage Cap — Bound External API Cost Per User

**Status**: Draft · **Last updated**: 2026-06-09

## Introduction

Prog Strength's chat agent makes calls to two metered external APIs every time a user opens a conversation: Anthropic's Claude API for the LLM turns themselves, and OpenAI's TTS API for voice mode. Both bill by usage — tokens and characters respectively — and neither has any user-scoped ceiling today. The only existing guardrail is an in-process per-user daily character cap on TTS (`TTSGenerator._Quota`), and even that is invisible to the user, lost on agent restart, and unique to one of the two surfaces. The LLM path has no cap at all.

This is fine while the user base is one person testing locally. It is not fine the moment we open registration: a single buggy client (an infinite chat loop, a held-down mic, a malicious script with a stolen JWT) can run up an arbitrary external API bill on accounts the operator pays for. There is no current circuit breaker that prevents one user's runaway behavior from blowing through the month's budget.

The target shape is a per-user daily allowance, denominated in dollars on the backend (because spend is the actual thing being bounded) and surfaced to the user only as a percentage of their daily quota (because the user does not need to see the operator's cost structure). The window is daily — not monthly — so that if a user does hit the cap, they wait hours rather than weeks for fresh allowance. With a global cap of roughly $0.17/day (≈ $5/user/month), a user with the most aggressive plausible chat behavior costs no more than $5/month in external API fees, and the operator can release to the public without worrying about a 4-figure invoice from a single misbehaving account.

What changes for the user: a new "Usage" section on the settings page showing a progress bar with their daily usage percentage and a "resets in Xh Ym" countdown. While they are under the cap, nothing else changes. When they hit the cap, the chat send button and voice-mode toggle disable with a clear message about when their allowance resets, and any attempt to send returns a 429 immediately rather than spending more money.

## Proposed Solution

**One read endpoint on the API, two thin call sites.** The API owns the cost source of truth: it knows the price-per-million-tokens of each Claude model and the price-per-million-characters of each TTS model, reads existing telemetry rows to compute today's spend, compares against a single env-configurable daily cap, and exposes a `GET /me/usage` endpoint that returns only `{percent_used, capped, resets_at}` — no dollar figures cross the wire to the frontend.

The agent calls `GET /me/usage` before every Claude or TTS call as a pre-flight gate. Results are cached in-process for 30 seconds per user — the cap is daily so 30 seconds of staleness is harmless, and 99% of chat turns never hit the API for the check. If the API is unreachable, the gate soft-allows; telemetry still records actual cost after the call, so the next check converges. A two-layer belt: the existing `TTSGenerator._Quota` stays as a process-local backstop in case the API gate is unreachable, so blast radius inside one process is still bounded.

The web settings page hits the same `GET /me/usage` endpoint to render the bar. A `useUsage()` hook mounted at the `(app)/layout.tsx` level shares the snapshot across settings and chat — both refresh on window focus and on a one-minute interval — so the two pages never race to fetch.

Cost is **derived at read time** from the rows that already land in `telemetry.db`. `agent_turns` already records per-turn token counts; the LLM half is a SUM-with-prices query against that table. The missing half is TTS: today `/speak` is fire-and-forget with no telemetry row, so we add one small sibling table (`agent_speak_calls`) that the agent writes to after every `/speak` call. No new ledger; no double bookkeeping. The cost engine reads two tables, applies the price map, returns a number.

The rollover window is anchored on the user's local timezone (the chat already sends `client_timezone`; we extend the same contract to `GET /me/usage` and the new TTS telemetry write). User-local rollover matches the mental model of "I get a fresh allowance tomorrow morning" rather than "my allowance resets at 5 PM today." UTC is the fallback when the client doesn't supply a timezone.

## Goals and Non-Goals

### Goals

- A new `GET /me/usage` endpoint on `prog-strength-api` returns `{percent_used, capped, resets_at}` for the authenticated user, derived from `telemetry.db` and a server-side price map. No dollar figures appear in the response body.
- A new `POST /internal/telemetry/speak` endpoint persists per-TTS-call character usage to a new `agent_speak_calls` table in `telemetry.db`.
- The agent's `/chat` and `/speak` handlers pre-check `GET /me/usage` via a new `UsageGate` module (30-second in-process cache, soft-allow on API failure) and return HTTP 429 with explanatory copy when the user is over cap.
- The agent's `TTSGenerator.generate` fire-and-forgets a `POST /internal/telemetry/speak` after each call (success or failure), mirroring the existing telemetry POST pattern.
- The settings page renders a "Usage" section with a percentage bar, dynamic color (accent / amber at 80% / red at 100%), and a "Resets in Xh Ym" countdown computed client-side from `resets_at`.
- The chat page disables the send button, mic button, and voice-mode toggle whenever the shared `useUsage()` snapshot reports `capped === true`, and surfaces a banner above the composer with the same copy as the settings 100% state.
- A 429 response from `/chat` or `/speak` triggers a toast and forces a `useUsage().refresh()` so the UI converges on the actual cap state immediately.
- The daily window rolls over at the user's local midnight, using the `client_timezone` the chat already sends; UTC is the fallback when missing.
- A single global daily cap is set via env (`DAILY_USAGE_CAP_USD`) on the API, applied to all users uniformly.
- Per-model prices for Claude and OpenAI TTS are env-configured on the API; the agent never carries a price table.
- New Prometheus metrics — `agent_usage_gate_blocks_total{surface}`, `agent_usage_gate_check_duration_seconds`, `agent_usage_gate_check_outcome_total{outcome}` on the agent; `api_usage_query_duration_seconds`, `api_usage_capped_total` on the API — feed a Grafana panel on the agent dashboard.
- Enforcement is gated behind a config flag (`USAGE_GATE_ENABLED`) for one deploy so telemetry can be validated in prod before the cap blocks any user.

### Non-Goals

- **STT cost tracking.** Web and mobile both use on-device speech recognition (`lib/speech.ts`, `expo-speech-recognition`); there is no metered STT surface today. If STT ever moves to Whisper, a follow-up SOW will extend the cost model to a third surface using the same shape.
- **Per-user cap overrides.** All users share the same `DAILY_USAGE_CAP_USD`. A future `users.daily_cap_usd_override` column can be added without touching the gate logic.
- **Account-tier-driven caps (free vs paid).** Subscription plumbing is out of scope.
- **Exposing dollar amounts to the user.** The response DTO has no `usd` field; this is a hard contract, not a UI style choice.
- **Eliminating the existing `TTSGenerator._Quota` in-process character cap.** It stays as a second layer that bounds blast radius inside one process even if the API gate is unreachable. Two cheap belts.
- **Pre-reserve + post-reconcile cost accounting.** The pre-call check + post-call record approach permits one call's worth of overage when a user is right at the boundary. Acceptable at $5/month/user; not worth the transactional complexity to tighten.
- **Mobile UI changes.** The mobile client gets the same 429 protection at the agent layer for free, but the in-product bar and disabled-state UX ship to web only in this SOW. Mobile follow-up tracks the same design.
- **Real-time updates on the settings bar.** Polling every minute + on window focus is the contract. No SSE / WebSocket push.
- **Resetting the cap from the admin side mid-day.** No "reset user X's usage" button. If you absolutely need to today, delete their `agent_turns` and `agent_speak_calls` rows for the current local date.

## Implementation Details

### Data Model

`telemetry.db` migration `016_agent_speak_calls.sql` adds one new table:

| Column | Type | Description |
| --- | --- | --- |
| `id` | text | UUID. Primary key. Agent-generated, mirrors `agent_turns.id`. |
| `user_id` | text | The authenticated user id. Indexed alongside `started_at` for per-user daily-spend queries. |
| `session_id` | text | Chat session id. Nullable — `/speak` is sometimes called outside a session context. Lets us join TTS spend to the conversation that drove it. |
| `model` | text | OpenAI TTS model id (`gpt-4o-mini-tts`, `tts-1`, etc.). Drives the per-call price lookup. |
| `chars` | integer | Character count actually sent to OpenAI — post-Markdown-strip, post-validation. The number we charge against the budget. |
| `voice` | text | Voice id used. Ops/debug only; not priced. |
| `started_at` | datetime | UTC. Indexed with `user_id`. |
| `ended_at` | datetime | UTC. |
| `error` | text | Nullable. Non-null means OpenAI rejected the call. We still record so retries cannot escape the cap by always failing. |

Composite index: `(user_id, started_at)`. Matches the index shape `agent_turns` already has so the daily-spend SUMs scan symmetrically. 90-day TTL applies the same way as on `agent_turns`.

No foreign key to `users` — `telemetry.db` is a physically separate database file from `app.db` (see `internal/db/telemetry_migrate.go`); they are intentionally decoupled.

`agent_turns` requires **no schema change**. The token columns it already carries (`input_tokens`, `output_tokens`, `cache_creation_tokens`, `cache_read_tokens`, `model`, `started_at`, `user_id`) are everything the LLM half of the cost SUM needs.

### Cost Engine

Lives at `internal/usage/` on the API. Three files:

- `price_table.go` — loads model prices from env at startup. Map of `model_id → PriceRates{input_per_mtok, output_per_mtok, cache_write_per_mtok, cache_read_per_mtok}` for Claude; map of `model_id → PriceRates{chars_per_mchar}` for TTS. Unknown model on read → log a structured `price_table_missing_model` warning and contribute 0 to the SUM (cap-bar undercounts rather than 500s).
- `ledger.go` — computes `spend_today_usd(user_id, local_day_window_utc) -> float64`. Two SUM queries against `telemetry.db`, one per table. App-side aggregation across models so prices stay in Go config rather than SQL.
- `window.go` — pure helper. `local_day_window(now, tz) -> (start_utc, end_utc)`. Tz fallback is UTC.

The SQL shape (per-model, app-side aggregate):

```sql
SELECT model,
       SUM(input_tokens),
       SUM(output_tokens),
       SUM(cache_creation_tokens),
       SUM(cache_read_tokens)
FROM agent_turns
WHERE user_id = ? AND started_at >= ? AND started_at < ?
GROUP BY model;
```

```sql
SELECT model, SUM(chars)
FROM agent_speak_calls
WHERE user_id = ? AND started_at >= ? AND started_at < ?
GROUP BY model;
```

Two round trips on different tables, both hitting the `(user_id, started_at)` index. Expected query time well under 10 ms at the scale this app operates at; if it ever isn't, switch to a per-user daily rollup table — same read contract, no caller changes.

### Price Configuration

A single env var carries the whole price map as JSON, keyed by the exact model id the agent records to `agent_turns.model` / `agent_speak_calls.model`. One env var keeps the price table atomic — partial updates that leave a model half-priced are impossible.

```
DAILY_USAGE_CAP_USD = "0.17"

USAGE_PRICE_TABLE_JSON = '{
  "claude": {
    "claude-sonnet-4-6": {
      "input_usd_per_mtok":        3.00,
      "output_usd_per_mtok":       15.00,
      "cache_write_usd_per_mtok":  3.75,
      "cache_read_usd_per_mtok":   0.30
    },
    "claude-haiku-4-5-20251001": {
      "input_usd_per_mtok":        0.80,
      "output_usd_per_mtok":       4.00,
      "cache_write_usd_per_mtok":  1.00,
      "cache_read_usd_per_mtok":   0.08
    }
  },
  "openai_tts": {
    "gpt-4o-mini-tts": { "usd_per_mchar": 15.00 },
    "tts-1":           { "usd_per_mchar": 15.00 }
  }
}'
```

Exact rates are pinned by the operator at deploy time; the values above are illustrative. Repricing is a single env bump and restart — no schema change, no agent change. The agent never sees any of these values.

Model id alignment matters: the keys above must match what the agent writes today. The agent's chat harness records `t.model` from the Anthropic SDK response, and the new `agent_speak_calls.model` will be `config.openai_tts_model` (the same string passed to OpenAI). The cost-engine's "model missing from price table" warning is the canary that catches a drift between these two sources.

### Rollover Window

The user's IANA timezone is the rollover anchor. Sources, in priority order:

1. Query param `tz` on `GET /me/usage` (web sends `Intl.DateTimeFormat().resolvedOptions().timeZone`).
2. Body field `client_timezone` on `POST /internal/telemetry/speak` (agent forwards what it received on the originating `/speak` request — TTS calls today do not carry tz, so this is typically empty; the local-date the call lands in is still computable from the API's UTC view of the call's `started_at` so this is acceptable for the TTS write path).
3. UTC fallback when both are missing.

`window.go::local_day_window(now, tz)` returns the half-open interval `[start_utc, end_utc)` such that the wall-clock range `[00:00, 24:00)` in the user's tz maps cleanly to UTC. DST transitions stretch or shrink one day; queries still resolve correctly because the bounds are explicit UTC timestamps.

`resets_at` in the `GET /me/usage` response is the same `end_utc` value, emitted in RFC3339. The frontend renders the countdown.

A user whose tz changes mid-day (travel, DST) may briefly see their bar jump because the start-of-window changed. Acceptable — the next request lands on the correct window.

### API Surface

**`GET /me/usage`** — JWT-gated, same middleware as `/me`.

Query params:
- `tz` (optional, IANA): User's local timezone. Defaults to UTC.

Response:
```json
{
  "data": {
    "percent_used": 64,
    "capped": false,
    "resets_at": "2026-06-10T06:00:00Z"
  }
}
```

- `percent_used`: `min(100, round(spend_today_usd / cap_usd * 100))`. Clamped at 100 so the visual bar cannot overflow.
- `capped`: `true` iff `spend_today_usd >= cap_usd`. The gate's source of truth.
- `resets_at`: ISO timestamp of the user's next local midnight, in UTC.

The DTO is a typed struct with **no** `usd` field — by construction, no rename or JSON-tag mistake can leak the dollar value.

Status codes:
- `200`: usage computed successfully (including the `capped: true` case).
- `401`: missing or invalid JWT (same shape as `/me`).
- `500`: cost-engine SQL failure (rare; logged).

**`POST /internal/telemetry/speak`** — internal-only, no auth (Caddy refuses to proxy `/internal/*` to the public internet; this matches `/internal/telemetry/turns`).

Body:
```json
{
  "id": "uuid",
  "user_id": "uuid",
  "session_id": "uuid|null",
  "model": "gpt-4o-mini-tts",
  "chars": 184,
  "voice": "alloy",
  "started_at": "2026-06-09T18:22:10Z",
  "ended_at":   "2026-06-09T18:22:11Z",
  "error": null
}
```

Insert into `agent_speak_calls`, return `204 No Content`. The agent fires this as a fire-and-forget task — failures log on the API side and are silently dropped on the agent side, mirroring the existing `record_turn` posture.

### Agent — `UsageGate`

New module `prog_strength_agent/usage_gate.py`. One class, one error type, one method exposed to callers.

```python
class CapExceeded(Exception):
    """User has hit their daily allowance. Maps to HTTP 429 at the
    handler boundary."""

@dataclass
class UsageSnapshot:
    percent_used: int
    capped: bool
    fetched_at_monotonic: float

class UsageGate:
    def __init__(
        self,
        api_base_url: str,
        *,
        cache_ttl_seconds: float = 30.0,
        timeout_seconds: float = 0.5,
    ): ...

    async def check_or_raise(
        self,
        *,
        user_id: str,
        tz: str | None,
    ) -> None:
        """Pre-call gate. Raises CapExceeded if the cached or freshly
        fetched /me/usage says capped=true. Soft-allow on any API
        failure (timeout, 5xx, network) — a flaky API must never
        block chat for a non-overspending user."""

    def invalidate(self, user_id: str) -> None:
        """Drop the cached snapshot for one user. Called immediately
        after a CapExceeded raise so the next call can't false-allow
        from a stale cache entry."""
```

**Cache shape:** `dict[user_id, UsageSnapshot]` guarded by an `asyncio.Lock` per user (lazily created in a parent lock). The per-user lock prevents stampedes — when a chat fans out three concurrent calls right after the 30-second TTL expires, only one of them refreshes; the others wait and read the result.

**Soft-allow on API failure** is deliberate. Telemetry post-writes record actual usage every turn, so spend converges regardless of whether the gate enforces. Cost of an API outage on the gate path: bounded by `(num_active_users × calls_during_outage × per_call_cost)`, which at this scale is dollars at worst. Cost of a hard-fail on outage: every user's chat breaks. Soft-allow wins by an order of magnitude.

**`USAGE_GATE_ENABLED=false`** short-circuits `check_or_raise` to a no-op so the first deploy lands the telemetry writes without changing user-facing behavior. Telemetry writes are not gated.

### Agent — Handler Wiring

`server.py` constructs one `UsageGate` at module load (alongside the existing `tts_generator`, `telemetry_client`, `api_client`) and threads it into `/chat` and `/speak`:

```python
# /chat
auth = authenticate(request, config.jwt_signing_key)
try:
    await usage_gate.check_or_raise(
        user_id=auth.user_id, tz=req.client_timezone,
    )
except CapExceeded as e:
    return Response(
        content=str(e),
        status_code=429,
        media_type="text/plain; charset=utf-8",
    )
# existing routing + harness streaming continues...
```

```python
# /speak
auth = authenticate(request, config.jwt_signing_key)
try:
    await usage_gate.check_or_raise(user_id=auth.user_id, tz=None)
except CapExceeded as e:
    return Response(content=str(e), status_code=429, ...)
# existing TTSGenerator.generate call continues; its _Quota
# QuotaExceeded path is the second belt.
```

The streaming-TTS path inside `voice_streamer` (where every sentence triggers `tts.generate`) inherits the gate at the `/chat` boundary — once a turn is admitted past the gate, its sentences are not re-checked. If the user happens to tip over cap mid-turn, that turn completes; the next turn is rejected. Acceptable per the pre-call/post-record contract.

### Agent — TTS Telemetry Write

`TTSGenerator.generate` gains a fire-and-forget POST to `/internal/telemetry/speak` after the OpenAI call returns (success or failure). The existing `TelemetryClient` grows one new method `record_speak(record: SpeakCallRecord)` that mirrors `record_turn` — wraps `asyncio.create_task` over a `_send_speak` that POSTs the body and swallows exceptions.

`SpeakCallRecord` is a small dataclass with the body fields above. `TTSGenerator.generate` is the natural place to construct it because it already has `user_id`, `text` (→ `chars = len(text)`), `voice`, and the `model` it called. `session_id` is not on the `/speak` request today; passing it through requires a small `SpeakRequest` field — added in this SOW.

The character count written to telemetry is the **same value** charged against the existing `_Quota` — `len(text)` after validation and (for the streaming path) after Markdown stripping. The two counts cannot drift.

### Web — Settings Page

`app/(app)/settings/page.tsx` gets a new section **above** the existing "Units" section:

```
┌─────────────────────────────────────────────────────┐
│  Usage                                              │
│  ┌───────────────────────────────────────────────┐  │
│  │  Daily AI allowance                           │  │
│  │  ▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░  64%                    │  │
│  │  Resets in 4h 12m                             │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

- Bar fill color: `var(--accent)` for 0–79%, `var(--warning)` (new token, mapped to amber) for 80–99%, `var(--danger)` (new token, red) at 100%.
- Below the bar: `Resets in {Xh Ym}`. Computed client-side from `resets_at` minus `Date.now()`; re-rendered every 60 s.
- At 100%, the subtext changes to: **"You've used your daily AI allowance. New allowance available in {Xh Ym}."**
- API failure state: bar renders as a hatched/disabled track with subtext "Usage unavailable right now."

### Web — Shared Usage State

New `lib/api.ts` helper:

```ts
export async function getMyUsage(
  token: string,
  tz: string,
): Promise<{ percent_used: number; capped: boolean; resets_at: string }>;
```

New `lib/usage-context.tsx`:

```ts
export type UsageSnapshot = {
  percentUsed: number;
  capped: boolean;
  resetsAt: Date;
  loading: boolean;
  error: string | null;
};

export function UsageProvider({ children }: { children: React.ReactNode }): JSX.Element;
export function useUsage(): UsageSnapshot & { refresh: () => Promise<void> };
```

`UsageProvider` is mounted in `app/(app)/layout.tsx` so both `settings` and `chat` consume the same snapshot. Internally:
- Fetches on mount, on `window` `focus`, and on a 60-second interval.
- Reads `Intl.DateTimeFormat().resolvedOptions().timeZone` for the `tz` query param.
- Coalesces concurrent `refresh()` calls so a chat 429 + settings poll don't double-fetch.
- On 401, clears the token and routes to `/login` (mirrors the existing `getMe` pattern in settings).

### Web — Chat Page

`app/(app)/chat/page.tsx` reads `useUsage()` and:

- When `capped === true`:
  - Send button: `disabled`, with `aria-disabled` and a tooltip "Daily AI allowance used. Resets in {Xh Ym}."
  - Mic button: `disabled`, same tooltip.
  - Voice-mode toggle: `disabled`. If voice mode was already on, it is forced off and the user gets a small inline note.
  - Banner above the composer: **"You've used your daily AI allowance. New allowance available in {Xh Ym}."**
- On a `/chat` response with status `429` (cache-miss race where the gate flipped to capped between the last `useUsage` refresh and the send):
  - Show a toast with the same copy.
  - Call `useUsage().refresh()` to converge the UI state.
  - Strip the user's optimistic message from the conversation transcript so they can edit and re-send tomorrow.

### Observability

New Prometheus metrics. All cardinalities bounded.

Agent (`prog-strength-agent`):
| Metric | Type | Labels | Notes |
| --- | --- | --- | --- |
| `agent_usage_gate_blocks_total` | Counter | `surface` (`chat`, `speak`) | Bumped each time `check_or_raise` raises `CapExceeded`. |
| `agent_usage_gate_check_duration_seconds` | Histogram | — | Wall-clock for `check_or_raise`. Buckets dense around 0–50 ms (cache hits) and 50–500 ms (API roundtrips). |
| `agent_usage_gate_check_outcome_total` | Counter | `outcome` (`hit`, `miss`, `allowed`, `blocked`, `api_error`) | One bump per call. `hit`/`miss` = cache; `allowed`/`blocked` = decision; `api_error` = soft-allow path. |

API (`prog-strength-api`):
| Metric | Type | Labels | Notes |
| --- | --- | --- | --- |
| `api_usage_query_duration_seconds` | Histogram | — | Wall-clock for the cost-engine SUM queries. |
| `api_usage_capped_total` | Counter | — | Bumped each time `GET /me/usage` returns `capped: true`. |

A new Grafana panel on the agent dashboard renders `rate(agent_usage_gate_blocks_total[5m])` stacked by `surface`. An ops dashboard renders `api_usage_capped_total` over time as a leading indicator of "users who can no longer use the product."

### Rollout

1. **API ships first** with the migration, the cost engine, both new endpoints, and the new env vars set. No callers yet — the migration is additive, the endpoint is unused, behavior is unchanged.
2. **Agent ships second** with `USAGE_GATE_ENABLED=false`. Telemetry writes start landing in `agent_speak_calls`. Verify in prod by querying `telemetry.db` directly: rows have sane char counts, the daily-spend SUM returns plausible numbers, `GET /me/usage` against a test user reports a reasonable percentage.
3. **Flip `USAGE_GATE_ENABLED=true`** on the agent. Same deploy ships the web UI (settings bar + chat disabled states). The web bar will start rendering for all users.
4. **Watch metrics for a week.** Specifically: `agent_usage_gate_blocks_total` (any blocks during normal use means the cap is too low), `api_usage_query_duration_seconds` (any p99 over 50 ms means the index isn't being hit), `agent_usage_gate_check_outcome_total{outcome="api_error"}` (any sustained rate means the soft-allow path is the only thing keeping chat working — a real problem to investigate).

### Error Paths

| Failure | Effect |
| --- | --- |
| API down / 5xx on `GET /me/usage` from agent | `UsageGate` soft-allows. Logs `usage_gate_api_error`. Telemetry still records spend; next successful check converges. |
| API down / 5xx on `GET /me/usage` from web | Settings bar renders disabled with "Usage unavailable right now." Chat does not block — user can send; agent's gate is what actually enforces. |
| Bad `tz` value | API falls back to UTC silently (mirrors the chat prompt's existing fallback). |
| Migration fails on deploy | Roll back via `016_agent_speak_calls_down.sql`. Agent telemetry writes get 4xx; the broad-except in `TelemetryClient._post` swallows them. Chat keeps working. |
| Price-table env missing a model | Cost SUM contributes 0 for that model. Logs `price_table_missing_model` with the model id. Cap-bar undercounts rather than 500ing. |
| Agent restarts mid-day | Cache clears. First check per user re-fetches from the API, restoring the correct snapshot. No state lost. |
| User changes tz mid-day | Bar may visibly jump on the next refresh. Acceptable. |

### Testing

| Layer | Tests |
| --- | --- |
| API — `internal/usage/window_test.go` | Unit: local-day window math across UTC, fixed offsets, DST forward (spring), DST back (fall). Verifies the half-open `[start, end)` always covers exactly 24 wall-clock hours except across DST transitions where it covers 23 or 25. |
| API — `internal/usage/ledger_test.go` | Unit with `telemetry.db` fixtures: zero rows, single-turn day, multi-model day, day spanning midnight UTC but not user-local midnight. |
| API — `internal/usage/handler_test.go` | Handler: happy path returns expected JSON; missing `tz` falls back to UTC; boundary case `spend == cap` returns `percent_used: 100, capped: true`; missing JWT → 401. |
| API — `internal/telemetry/speak_handler_test.go` | Handler: well-formed body inserts the row; malformed body 400; row insert returns 204. |
| Agent — `tests/test_usage_gate.py` | Pytest: cache hit returns no-op; cache miss fetches and stores; API timeout soft-allows and logs; `CapExceeded` raise calls `invalidate`; stampede test runs 10 concurrent `check_or_raise` and asserts one HTTP call. |
| Agent — `tests/test_server_usage.py` | `/chat` returns 429 with expected copy when `check_or_raise` raises; `/speak` returns 429 with expected copy when `check_or_raise` raises; happy path passes through unchanged; `USAGE_GATE_ENABLED=false` short-circuits both. |
| Agent — `tests/test_speak.py` | Existing test extended to assert `TelemetryClient.record_speak` is called once per `generate` invocation with the expected `SpeakCallRecord`. |
| Web — `app/(app)/settings/page.test.tsx` | RTL: renders 0% accent bar, 50% accent bar, 80% amber bar, 100% red bar with switched copy, error state with disabled bar. |
| Web — `app/(app)/chat/page.test.tsx` | RTL: when `useUsage()` reports `capped`, send button is disabled, mic button is disabled, banner appears; when 429 lands from `/chat`, toast appears and `refresh` is called. |
| Web — `lib/usage-context.test.tsx` | RTL: fetches on mount, refetches on `focus`, refetches on interval, coalesces concurrent `refresh()` calls. |

Manual smoke before flipping `USAGE_GATE_ENABLED=true`:
- Burn the cap by repeatedly hitting `/chat` against a test user with a known low cap. Confirm 429 lands at the expected boundary. Confirm the settings bar fills to 100% and copy flips. Confirm "Resets at" matches the test user's local midnight.
- Disconnect the API (stop the container, agent still up): confirm `/chat` and `/speak` still serve, telemetry queues are draining as expected once the API is back.

## Open Questions

1. **Should the agent emit a `capped` SSE event so an in-progress chat that exceeds cap mid-turn can be cancelled by the client?** Options: (a) leave the contract as "admitted turns complete," accepting one turn of overage per user per day at the boundary; (b) add a `capped` SSE event the harness emits if a post-call telemetry write reveals the user just tipped over, which the client uses to immediately close the stream. **Tentative lean: (a).** The cost of one turn at the boundary is bounded by `max_tokens`, and the protocol complexity of (b) is meaningful for the win.
2. **Should we expose `percent_used` to the agent itself for system-prompt awareness?** A user at 90% could in theory be told "you're approaching your daily allowance" by the assistant. Options: (a) no — keep the cap purely a UI/gate concern; (b) yes — pass the snapshot into the system prompt so the model can soften its replies as cap approaches. **Tentative lean: (a).** Bleeding cap-state into the model's voice risks the assistant lecturing the user about cost; the bar in settings is sufficient.
3. **Cache TTL of 30 seconds — too long, too short, or right?** A shorter TTL (5–10 s) reduces the "one call sneaks past at the boundary" window; a longer TTL (60 s) reduces API load. **Tentative lean: 30 s as designed.** Revisit only if `agent_usage_gate_check_duration_seconds` shows the cache-miss path is contributing meaningful latency.
