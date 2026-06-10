# Per-User Daily Usage Cap — Implementation Plan

> **For agentic workers:** Implement task-by-task. Each task is implemented by a
> subagent, then spec-reviewed and code-quality-reviewed before moving on.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bound external API cost per user with a per-user daily allowance,
denominated in dollars on the API backend and surfaced to the user only as a
percentage of their daily quota. The API owns the cost source of truth (price
map + telemetry rows), exposes `GET /me/usage` returning
`{percent_used, capped, resets_at}` (no dollar figures), and a new
`POST /internal/telemetry/speak` to record TTS character spend. The agent
pre-checks usage before every Claude/TTS call via a new `UsageGate`
(30s in-process cache, soft-allow on API failure) and returns 429 when capped.
The web settings page renders a usage bar; the chat page disables send/mic/voice
when capped and converges on 429.

**SOW:** `sows/per-user-daily-usage-cap.md`

**Repos & merge order (rollout):** `prog-strength-api` → `prog-strength-agent`
(ships `USAGE_GATE_ENABLED=false` first, then flip) → `prog-strength-web` →
`prog-strength-docs` (status flip).

**Branch in every repo:** `feat/per-user-daily-usage-cap`.

## Decisions / deviations from the SOW's literal text

- **Telemetry migration number:** the SOW says `016_agent_speak_calls.sql`, but the
  repo's `internal/db/telemetry_migrations/` only goes to `003`. Use the next
  sequential telemetry migration number: **`004_agent_speak_calls.sql`**.
- **No down-migration file:** the telemetry migration system in this repo has no
  `_down.sql` convention (it embeds and applies forward-only). Follow the repo
  convention — the migration is additive (`CREATE TABLE IF NOT EXISTS`), so a
  rollback is a no-op on reads. Do **not** invent a down-file.
- **Prometheus metric names & labels:** use the SOW's exact names and bounded
  labels. API: `api_usage_query_duration_seconds` (Histogram, no labels),
  `api_usage_capped_total` (Counter, no labels). Agent:
  `agent_usage_gate_blocks_total{surface}`,
  `agent_usage_gate_check_duration_seconds` (no labels),
  `agent_usage_gate_check_outcome_total{outcome}`. **Never** label by `user_id`
  (unbounded cardinality).
- **UsageProvider mount point:** mount in `app/(app)/layout.tsx` (the
  authenticated-route group layout) per the SOW — usage requires a token, so it
  belongs in the authed subtree, not the root layout.

---

# Part A — prog-strength-api (ships first; additive, no callers yet)

Module: chi router, domain-oriented `internal/<domain>/` layout, no Makefile
(`go build ./...`, `go test ./...`, `golangci-lint run`). Telemetry DB is a
separate file from app.db; reach it via the telemetry `*sql.DB` handle.

## Task A1 — Migration `004_agent_speak_calls.sql`

Files: `internal/db/telemetry_migrations/004_agent_speak_calls.sql`.

- [ ] Create the migration (the embed glob `telemetry_migrations/*.sql` picks it
  up automatically; the migrate runner tracks applied version). Schema:
  ```sql
  CREATE TABLE IF NOT EXISTS agent_speak_calls (
      id TEXT PRIMARY KEY,
      user_id TEXT NOT NULL,
      session_id TEXT,
      model TEXT NOT NULL,
      chars INTEGER NOT NULL,
      voice TEXT NOT NULL,
      started_at DATETIME NOT NULL,
      ended_at DATETIME NOT NULL,
      error TEXT
  );
  CREATE INDEX IF NOT EXISTS idx_agent_speak_calls_user_started
      ON agent_speak_calls(user_id, started_at);
  ```
  `session_id` is nullable (TTS is sometimes called outside a session). Mirror the
  `(user_id, started_at)` index shape `agent_turns` already has. No FK to users
  (telemetry.db is decoupled from app.db).
- [ ] Verify the migration applies cleanly: `go test ./internal/db/...` and any
  existing migrate test; confirm `MigrateTelemetry` advances to version 4.

## Task A2 — Cost engine: `internal/usage/window.go` (+ tests)

Files: `internal/usage/window.go`, `internal/usage/window_test.go`.
Add `_ "time/tzdata"` to `cmd/api/main.go` import block if not already present
(so `time.LoadLocation` works on a minimal base image — check first; the nutrition
work may have added it).

- [ ] `func LocalDayWindow(now time.Time, tz string) (startUTC, endUTC time.Time)`:
  load `tz` via `time.LoadLocation`; on empty/invalid tz fall back to `time.UTC`.
  Compute the user's local calendar date for `now`, take local `00:00` as start and
  local `00:00` next day as end, return both `.UTC()`. Half-open `[start, end)`.
- [ ] Tests: UTC equivalence; fixed offset zone; DST spring-forward day (interval
  23h); DST fall-back day (interval 25h); empty/invalid tz → UTC fallback; verify
  `endUTC` is the next local midnight (this becomes `resets_at`).
- [ ] Verify: `go test ./internal/usage/...`, `gofmt -l`.

## Task A3 — Cost engine: `internal/usage/price_table.go` (+ tests)

Files: `internal/usage/price_table.go`, `internal/usage/price_table_test.go`.

- [ ] Define:
  ```go
  type ClaudeRates struct {
      InputPerMTok      float64
      OutputPerMTok     float64
      CacheWritePerMTok float64
      CacheReadPerMTok  float64
  }
  type TTSRates struct{ PerMChar float64 }
  type PriceTable struct {
      Claude    map[string]ClaudeRates
      OpenAITTS map[string]TTSRates
  }
  ```
- [ ] `func LoadPriceTable(jsonStr string) (PriceTable, error)` — parse the
  `USAGE_PRICE_TABLE_JSON` shape from the SOW (keys `"claude"` / `"openai_tts"`,
  json fields `input_usd_per_mtok`, `output_usd_per_mtok`,
  `cache_write_usd_per_mtok`, `cache_read_usd_per_mtok`, `usd_per_mchar`). Empty
  string → empty table (no error).
- [ ] Cost helpers on `PriceTable`:
  - `ClaudeCostUSD(model string, in, out, cacheCreate, cacheRead int64) float64`
  - `TTSCostUSD(model string, chars int64) float64`
  Unknown model → log a structured `price_table_missing_model` warning (use the
  repo's logging convention) and contribute `0` (do not error). Cost =
  `tokens / 1_000_000 * rate` summed per rate kind.
- [ ] Tests: parse the SOW example JSON; cost math for a known Claude model
  (all four token kinds) and a known TTS model; unknown model returns 0 and does
  not panic; empty JSON yields empty table.
- [ ] Verify: `go test ./internal/usage/...`.

## Task A4 — Cost engine: `internal/usage/ledger.go` (+ tests)

Files: `internal/usage/ledger.go`, `internal/usage/ledger_test.go`.

- [ ] `type Ledger struct { db *sql.DB; prices PriceTable }` with constructor
  `NewLedger(telemetryDB *sql.DB, prices PriceTable) *Ledger`.
- [ ] `func (l *Ledger) SpendTodayUSD(ctx, userID string, startUTC, endUTC time.Time) (float64, error)`:
  two SUM queries against telemetry.db, grouped by model, app-side aggregation:
  ```sql
  SELECT model, SUM(input_tokens), SUM(output_tokens),
         SUM(cache_creation_tokens), SUM(cache_read_tokens)
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
  Sum `ClaudeCostUSD` + `TTSCostUSD` across the grouped rows. Wrap the two queries
  with the `api_usage_query_duration_seconds` histogram observation (see A7).
- [ ] Tests with a telemetry.db fixture (reuse the migrate-into-`t.TempDir()`
  helper pattern from `internal/telemetry/sqlite_repository_test.go`): zero rows →
  0; single-turn day; multi-model day; a row outside the window is excluded; a
  speak row contributes its TTS cost; combined turns+speak sum.
- [ ] Verify: `go test ./internal/usage/...`.

## Task A5 — `GET /me/usage` handler + DTO + metrics (+ tests)

Files: `internal/usage/handler.go`, `internal/usage/handler_test.go`; wire into
the JWT-gated group in `internal/server/server.go`; metrics in
`internal/server/metrics.go` (or a usage metrics file).

- [ ] Handler depends on a `*Ledger` and the configured `capUSD float64`. Extract
  user id via `authctx.UserIDFrom(ctx)` (same as `/me`). Read `tz` query param
  (optional, defaults UTC).
- [ ] Compute window via `LocalDayWindow(time.Now(), tz)`, spend via
  `SpendTodayUSD`, then:
  - `percent_used = min(100, round(spend/cap*100))` (guard cap<=0 → treat as
    uncapped: percent 0, capped false; log once).
  - `capped = spend >= cap` (and cap > 0).
  - `resets_at = endUTC` in RFC3339.
- [ ] Response DTO is a typed struct with **no** `usd` field:
  ```go
  type usageResponse struct {
      PercentUsed int    `json:"percent_used"`
      Capped      bool   `json:"capped"`
      ResetsAt    string `json:"resets_at"`
  }
  ```
  Return via the repo's `httpresp.OK(w, "...", usageResponse{...})` (the `{"data":...}`
  envelope). On SQL failure → `httpresp.ServerError` (500). Missing JWT → 401 is
  handled by middleware.
- [ ] Metrics: define `api_usage_query_duration_seconds` (Histogram, no labels)
  and `api_usage_capped_total` (Counter, no labels), registered in `init()` like
  existing metrics. Observe query duration around the ledger call; bump
  `api_usage_capped_total` when the response has `capped: true`.
- [ ] Mount: `r.Get("/me/usage", h.getMyUsage)` inside the JWT-gated group (same
  middleware as `/me`). Construct the handler in `server.New` using the telemetry
  DB handle, the parsed price table, and `cfg.DailyUsageCapUSD`.
- [ ] Tests (handler, httptest + authctx like `user/handler_test.go`): happy path
  returns expected JSON with no `usd` key; missing `tz` → UTC; boundary
  `spend == cap` → `percent_used: 100, capped: true`; missing JWT path (test the
  handler returns/relies on middleware — assert 401 via the mounted route if
  feasible, else document middleware covers it).
- [ ] Verify: `go test ./internal/usage/... ./internal/server/...`.

## Task A6 — `POST /internal/telemetry/speak` + repo insert (+ tests)

Files: extend `internal/telemetry/` — `telemetry.go` (add `AgentSpeakCall`
type), `repository.go` (add `InsertSpeakCall`), `sqlite_repository.go` (impl),
`handler.go` (add route + `speak` handler + request struct),
`handler_test.go` / `sqlite_repository_test.go` (tests).

- [ ] `AgentSpeakCall` domain struct mirroring the table columns (SessionID and
  Error as `*string` / nullable; Chars int64; timestamps time.Time).
- [ ] `InsertSpeakCall(ctx, AgentSpeakCall) error` on the repository interface +
  SQLite impl (`INSERT INTO agent_speak_calls (...) VALUES (...)`). Handle NULLs
  for session_id/error.
- [ ] Handler `speak`: decode `speakRequest` (json tags matching the SOW body:
  `id, user_id, session_id, model, chars, voice, started_at, ended_at, error`),
  validate required fields (id, user_id, model, chars, voice, started_at,
  ended_at) → 400 on malformed; insert; return `204 No Content`. Register under
  the existing `/internal/telemetry` route group:
  `r.Post("/speak", h.speak)`.
- [ ] Tests: well-formed body inserts the row + 204; malformed body → 400; null
  session_id/error accepted; row readback matches.
- [ ] Verify: `go test ./internal/telemetry/...`.

## Task A7 — Config wiring + server construction

Files: `internal/config/config.go`, `internal/server/server.go`,
possibly `internal/server/metrics.go` (covered in A5).

- [ ] Add to `Config`: `DailyUsageCapUSD float64` (from `DAILY_USAGE_CAP_USD`,
  parse via `strconv.ParseFloat`, default 0) and `UsagePriceTableJSON string`
  (from `USAGE_PRICE_TABLE_JSON`, default ""). No new required vars (additive,
  safe default).
- [ ] In `server.New`: parse the price table once via
  `usage.LoadPriceTable(cfg.UsagePriceTableJSON)` (log + continue on parse error so
  a bad env doesn't crash boot — empty table just undercounts), build
  `usage.NewLedger(telemetryDB, prices)`, and mount the usage handler in the
  JWT-gated group. Keep the existing telemetry handler wiring; the speak route is
  added inside the existing telemetry handler's `Mount` (A6).
- [ ] Verify: `go build ./...`, `go test ./...`, `golangci-lint run` (if
  available), `gofmt -l`.

---

# Part B — prog-strength-agent (ships second; gate disabled on first deploy)

Python 3.12, FastAPI, `uv`. Tests: `uv run pytest tests/`. Lint:
`uv run ruff check src/ tests/`, format `uv run ruff format`. Singletons built at
module load in `server.py`. `httpx.AsyncClient` is the API call pattern;
fire-and-forget = `asyncio.create_task` + broad except.

## Task B1 — Config: `usage_gate_enabled`

Files: `src/prog_strength_agent/config.py`, `tests/conftest.py` if needed.

- [ ] Add `usage_gate_enabled: bool` to the frozen `Config` dataclass; in
  `from_env` read `USAGE_GATE_ENABLED` (default `"false"`, parse truthy
  `{"1","true","yes","on"}` case-insensitive). `api_url` already exists — reuse it
  for the gate's base URL.
- [ ] Verify: `uv run pytest tests/` still green.

## Task B2 — `usage_gate.py` (UsageGate, CapExceeded, UsageSnapshot) + metrics + tests

Files: `src/prog_strength_agent/usage_gate.py`, `tests/test_usage_gate.py`;
metrics alongside the existing ones (telemetry.py defines them, or a metrics
module — match the repo).

- [ ] Implement per the SOW interface:
  ```python
  class CapExceeded(Exception): ...
  @dataclass
  class UsageSnapshot:
      percent_used: int
      capped: bool
      fetched_at_monotonic: float
  class UsageGate:
      def __init__(self, api_base_url, *, enabled=True,
                   cache_ttl_seconds=30.0, timeout_seconds=0.5): ...
      async def check_or_raise(self, *, user_id, tz): ...
      def invalidate(self, user_id): ...
  ```
- [ ] Behavior:
  - `enabled=False` → `check_or_raise` is a no-op (short-circuit before any cache
    or HTTP work). Telemetry writes are NOT gated (they live in TTS/handlers).
  - Cache: `dict[user_id, UsageSnapshot]` with a per-user `asyncio.Lock` (lazily
    created under a parent lock) to prevent stampedes — concurrent calls after TTL
    expiry → one HTTP fetch, others read the result.
  - On cache hit within TTL (`time.monotonic() - fetched_at < ttl`): use cached
    snapshot; emit `outcome="hit"`.
  - On miss: GET `{api_base_url}/me/usage?tz=...` via `httpx.AsyncClient`
    (timeout 0.5s); parse `data.{percent_used,capped}`; store; emit
    `outcome="miss"`.
  - If `capped` → raise `CapExceeded(<copy>)` and call `invalidate(user_id)` so a
    stale entry can't false-allow next call; emit `outcome="blocked"` and bump
    `agent_usage_gate_blocks_total{surface=...}`. (Surface label: pass `surface`
    into `check_or_raise` or expose via the caller — simplest: add a `surface`
    kwarg to `check_or_raise`. The SOW handler snippets don't pass it, so default
    `surface` from the call site — implement with a `surface: str` kwarg defaulting
    to `"chat"`, and `/speak` passes `surface="speak"`.)
  - Not capped → emit `outcome="allowed"`; return None.
  - Any API failure (timeout, 5xx, network, bad JSON) → **soft-allow** (return
    None), log `usage_gate_api_error`, emit `outcome="api_error"`.
  - Wrap the whole `check_or_raise` body to observe
    `agent_usage_gate_check_duration_seconds`.
- [ ] Metrics (prometheus_client, bounded labels):
  `agent_usage_gate_blocks_total` Counter `["surface"]`;
  `agent_usage_gate_check_duration_seconds` Histogram (no labels);
  `agent_usage_gate_check_outcome_total` Counter `["outcome"]`.
- [ ] Tests (`pytest`, `respx` for httpx, `@pytest.mark.asyncio`): cache hit →
  no HTTP call; cache miss → fetch + store; API timeout → soft-allow + logs +
  `api_error` outcome; `capped=true` → raises `CapExceeded` and calls
  `invalidate`; stampede: 10 concurrent `check_or_raise` after expiry → exactly
  one HTTP call; `enabled=False` → no-op (no HTTP).
- [ ] Verify: `uv run pytest tests/test_usage_gate.py`.

## Task B3 — `TelemetryClient.record_speak` + `SpeakCallRecord` + tests

Files: `src/prog_strength_agent/telemetry.py`, `tests/test_telemetry.py`.

- [ ] Add `SpeakCallRecord` dataclass with the POST body fields (`id, user_id,
  session_id, model, chars, voice, started_at, ended_at, error`). Timestamps as
  ISO strings (mirror how `record_turn` serializes).
- [ ] `record_speak(self, record: SpeakCallRecord) -> None` — fire-and-forget via
  `asyncio.create_task(self._send_speak(record))`; `_send_speak` POSTs to
  `/internal/telemetry/speak` and swallows exceptions (reuse `_post` broad-except
  pattern). No-op safe if client disabled (mirror `record_turn` posture).
- [ ] Tests: `respx` mock asserts one POST to `/internal/telemetry/speak` with the
  expected JSON body; failure is swallowed.
- [ ] Verify: `uv run pytest tests/test_telemetry.py`.

## Task B4 — TTS write: `TTSGenerator.generate` records speak; thread `session_id`

Files: `src/prog_strength_agent/speak.py`, `src/prog_strength_agent/server.py`
(SpeakRequest gains `session_id`), `src/prog_strength_agent/voice_stream.py`
(pass session_id through), `tests/test_speak.py`.

- [ ] `TTSGenerator.generate` gains optional `session_id: str | None = None` and a
  telemetry hook. Cleanest: inject the `TelemetryClient` (or a `record_speak`
  callback) into `TTSGenerator.__init__` so `generate` can fire-and-forget a
  `SpeakCallRecord` after the OpenAI call returns — on **both** success and failure
  (error path sets `error=str(e)`, chars still = `len(text)`). Capture
  `started_at`/`ended_at` around the call. `chars = len(text)` (same value charged
  against `_Quota` — must not drift). `model = self._model`, `voice = resolved
  voice`.
  - If wiring the TelemetryClient into TTSGenerator is awkward vs. the existing
    construction order, instead accept an optional `on_speak: Callable[[SpeakCallRecord], None] | None`
    and have `server.py` pass `telemetry_client.record_speak`. Choose whichever
    matches the repo's DI style; document the choice.
- [ ] Add `session_id: str | None = None` to `SpeakRequest`; `/speak` handler
  passes it into `generate`. In `voice_stream.voice_streamer`, thread the chat
  `session_id` into the per-sentence `tts.generate(...)` calls so streamed TTS
  rows carry the session.
- [ ] Extend `tests/test_speak.py`: assert `record_speak` (or `on_speak`) is
  called once per `generate` with the expected `SpeakCallRecord` (chars, model,
  voice, session_id); assert it still fires on the failure path.
- [ ] Verify: `uv run pytest tests/test_speak.py`.

## Task B5 — Handler wiring: `/chat` + `/speak` gate → 429

Files: `src/prog_strength_agent/server.py`, `tests/test_server_usage.py`.

- [ ] Construct one `UsageGate` at module load alongside `tts_generator` /
  `telemetry_client` / `api_client`, using `config.api_url`,
  `enabled=config.usage_gate_enabled`. (If `api_url` is empty, construct disabled.)
- [ ] In `/chat`, after `authenticate(...)`, before routing/streaming:
  ```python
  try:
      await usage_gate.check_or_raise(user_id=auth.user_id,
                                      tz=req.client_timezone, surface="chat")
  except CapExceeded as e:
      return Response(content=str(e), status_code=429,
                      media_type="text/plain; charset=utf-8")
  ```
- [ ] In `/speak`, same pattern with `tz=None, surface="speak"`. The existing
  `TTSGenerator._Quota` `QuotaExceeded` (429) stays as the second belt.
- [ ] Cap copy (used as the 429 body and shared with web): a clear sentence about
  the daily allowance being used and resetting — keep it short and consistent.
  e.g. `"You've used your daily AI allowance. New allowance available soon."`
  (web computes the precise countdown from `resets_at`).
- [ ] Tests (`tests/test_server_usage.py`, FastAPI TestClient + monkeypatch the
  gate): `/chat` → 429 with expected copy when gate raises; `/speak` → 429 when
  gate raises; happy path passes through unchanged; `USAGE_GATE_ENABLED=false`
  short-circuits both (no 429 even if usage would be capped).
- [ ] Verify: `uv run pytest tests/`.

---

# Part C — prog-strength-web (ships third, with the gate flip)

Next.js 16 / React 19 / TS, Tailwind + CSS vars, vitest + RTL. npm.
`npm run test`, `npm run lint`, `npm run typecheck`, `npm run build`.

## Task C1 — `--warning`/`--danger` tokens + `getMyUsage` helper

Files: `app/globals.css`, `lib/api.ts`.

- [ ] `globals.css`: add `--warning: #f59e0b;` (amber) to `:root` (and
  `--color-warning: var(--warning);` in the `@theme inline` block). `--danger`
  already exists (red); reuse it.
- [ ] `lib/api.ts`: add
  ```ts
  export type UsageData = { percent_used: number; capped: boolean; resets_at: string };
  export async function getMyUsage(token: string, tz: string): Promise<UsageData> { ... }
  ```
  GET `${config.apiUrl}/me/usage?tz=<encoded>` with Bearer token; unwrap the
  `{data}` envelope via the existing `unwrap` helper. Use query param name `tz`
  (matches the API contract in A5).
- [ ] Verify: `npm run typecheck`.

## Task C2 — `lib/usage-context.tsx` (UsageProvider + useUsage) + tests

Files: `lib/usage-context.tsx`, `lib/usage-context.test.tsx`.

- [ ] Implement per the SOW:
  ```ts
  export type UsageSnapshot = {
    percentUsed: number; capped: boolean; resetsAt: Date;
    loading: boolean; error: string | null;
  };
  export function UsageProvider({ children }): JSX.Element;
  export function useUsage(): UsageSnapshot & { refresh: () => Promise<void> };
  ```
- [ ] Provider behavior (model on `lib/distance-unit-context.tsx`):
  - Fetch on mount, on `window` `focus`, and on a 60s `setInterval`.
  - `tz` from `Intl.DateTimeFormat().resolvedOptions().timeZone`.
  - Coalesce concurrent `refresh()` calls (store the in-flight promise; return it
    if present).
  - Map `getMyUsage` result → snapshot; `resetsAt = new Date(resets_at)`.
  - On 401, `clearToken()` + route to `/login` (mirror settings' getMe pattern).
  - On other errors, set `error` (bar renders unavailable) but don't crash.
  - If no token, stay idle (don't fetch).
- [ ] Tests (RTL + mocked `getMyUsage`): fetches on mount; refetches on `focus`;
  refetches on interval (fake timers); coalesces concurrent `refresh()` (one
  underlying fetch).
- [ ] Verify: `npm run test -- usage-context`.

## Task C3 — Mount `UsageProvider` in `app/(app)/layout.tsx`

Files: `app/(app)/layout.tsx`.

- [ ] Wrap the authed subtree (`<Sidebar/>` + children) with `<UsageProvider>` so
  both settings and chat share one snapshot. Keep the existing auth gate; only
  render the provider once `ready` (token present).
- [ ] Verify: `npm run typecheck`, `npm run build`.

## Task C4 — Settings "Usage" section + tests

Files: `app/(app)/settings/page.tsx`, `app/(app)/settings/page.test.tsx`,
possibly a small `components/usage-bar.tsx`.

- [ ] Add a **Usage** section *above* the existing "Units" section. Render via
  `useUsage()`:
  - Progress bar fill `width: {percentUsed}%`; color: `var(--accent)` 0–79,
    `var(--warning)` 80–99, `var(--danger)` at 100.
  - `{percentUsed}%` label.
  - Subtext "Resets in {Xh Ym}" computed client-side from `resetsAt - Date.now()`,
    re-rendered every 60s.
  - At 100% (`capped`/100): subtext flips to **"You've used your daily AI
    allowance. New allowance available in {Xh Ym}."**
  - `error` state: hatched/disabled track + "Usage unavailable right now."
  - `loading` state: neutral placeholder.
- [ ] Tests (RTL, provide `useUsage` via a mock or a test wrapper): 0% accent bar;
  50% accent; 80% amber; 100% red with flipped copy; error → disabled bar +
  unavailable copy.
- [ ] Verify: `npm run test -- settings`.

## Task C5 — Chat page capped states + 429 convergence + tests

Files: `app/(app)/chat/page.tsx`, `app/(app)/chat/page.test.tsx`.

- [ ] Read `useUsage()`. When `capped === true`:
  - Disable send button, mic button, voice-mode toggle (`disabled` +
    `aria-disabled`; tooltip "Daily AI allowance used. Resets in {Xh Ym}.").
  - If voice mode was on, force it off with a small inline note.
  - Banner above the composer: **"You've used your daily AI allowance. New
    allowance available in {Xh Ym}."**
- [ ] On a `/chat` fetch returning status `429`:
  - Show a toast with the same copy (`useToast().error`/`info`).
  - Call `useUsage().refresh()` to converge.
  - Strip the optimistic user message + blank assistant placeholder from the
    transcript.
- [ ] Tests (RTL): when `useUsage` reports `capped`, send/mic disabled + banner
  shows; when `/chat` returns 429, toast appears and `refresh` is called.
- [ ] Verify: `npm run test -- chat`, `npm run lint`, `npm run typecheck`,
  `npm run build`.

---

# Part D — prog-strength-docs (status flip)

- [ ] On `feat/per-user-daily-usage-cap`, edit `sows/per-user-daily-usage-cap.md`:
  frontmatter `status: shipped`; body `**Status**: Shipped`; body
  `**Last updated**: 2026-06-09`. Commit
  `docs: mark per-user-daily-usage-cap as shipped`.

---

## Per-repo verification gates (run before opening each PR)

- API: `go build ./...` && `go test ./...` && `gofmt -l` (clean) && lint.
- Agent: `uv run pytest tests/` && `uv run ruff check src/ tests/`.
- Web: `npm run test` && `npm run lint` && `npm run typecheck` && `npm run build`.
