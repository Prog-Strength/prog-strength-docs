# Implementation Plan: Daily Steps Logging

**SOW:** `sows/daily-steps-logging.md` · **Created:** 2026-06-15 · **Branch:** `feat/daily-steps-logging` (all repos)

Steps follow the path **bodyweight** paved: a small Go API domain (per-entry
table + single-row-per-user goal table), an MCP module + agent prompt update,
and web/mobile UI that reuses chart/stat-tile/goal-setting patterns. The one
structural difference: steps are **date-keyed and upserted** (one total per
day) rather than a stream of timestamped readings.

This plan exists primarily to **lock the canonical API contract** so the five
repos can be built in parallel without drift.

---

## Canonical API contract (the coordination artifact)

All repos MUST conform to this exactly. Responses use the standard
`{service, version, message, data}` envelope; clients read `data`.

### Entry DTO (returned by `PUT /steps/{date}` and inside list)
```json
{ "id": "hex", "date": "2026-06-14", "steps": 8400,
  "created_at": "2026-06-14T12:00:00Z", "updated_at": "2026-06-14T12:00:00Z" }
```
`user_id` is NOT exposed (mirrors bodyweight `entryDTO`).

### Routes (mounted behind auth middleware; user id from `auth.UserIDFrom`)

- **`GET /steps?since=&until=&limit=&before=`** — list a user's days, **newest
  first**. Two modes:
  - **Range mode** (`since`/`until`, both `YYYY-MM-DD`, no `limit`): returns all
    days with `since <= date <= until` (both **inclusive**). Powers the chart.
  - **Keyset mode** (`limit` set, optional `before=YYYY-MM-DD`): returns up to
    `limit` rows with `date < before` (when `before` given), newest first.
    Powers the log table.
  - Response `data`: `{ "steps": [Entry...], "next_before": "YYYY-MM-DD" | null }`.
    `next_before` is the date of the last row returned when a full `limit` page
    came back (more may exist); otherwise `null`. In range mode `next_before` is
    `null`.
  - Reject mixing a `limit` with `since`/`until` only if it complicates impl;
    simplest acceptable behavior: if `limit` present, keyset mode wins.
- **`PUT /steps/{date}`** — upsert the total for `{date}`. Body `{"steps": int}`.
  Validates: `{date}` is a real `YYYY-MM-DD` calendar date and **not more than 1
  day in the future** relative to server UTC date (tolerates a tz midnight
  crossing; rejects far-future); `0 <= steps <= 200000`. Returns stored Entry
  (`200`). Idempotent — covers "log today" and "correct a past day".
- **`DELETE /steps/{date}`** — remove a day's entry. `204` on success, `404`
  when the day has no entry.
- **`GET /me/steps-goal`** — current goal. `data`:
  `{ "goal": int, "created_at": str|null, "updated_at": str|null }`. Unset =
  `{ "goal": 0, "created_at": null, "updated_at": null }` (no 404), mirroring
  `getMyBodyweightGoal`.
- **`PUT /me/steps-goal`** — upsert goal. Body `{"goal": int}`,
  `0 < goal <= 200000`. Returns the goal (`200`).

Validation → `400` mapping follows the bodyweight handlers exactly.

---

## Task 1 — API (`prog-strength-api`)

New `internal/steps` domain mirroring `internal/bodyweight` anatomy
(`model.go`, `goal.go`, `errors.go`, `metrics.go`, `repository.go`,
`sqlite_repository.go`, `memory_repository.go`, `handler.go`, `handler_goal.go`,
+ `*_test.go`). Registered in `internal/server/server.go`:
`steps.NewHandler(stepsRepo).Mount(r)` (no `userRepo` — steps are unitless, no
user dependency like bodyweight's unit).

### Migration `internal/db/migrations/019_steps.sql`
(019 is the next free index — 018 is the last; the Timeline SOW also eyes 019
but is not yet implemented here, so 019 is ours.)

```sql
CREATE TABLE IF NOT EXISTS user_steps (
    id         TEXT     PRIMARY KEY,
    user_id    TEXT     NOT NULL,
    date       TEXT     NOT NULL,                 -- YYYY-MM-DD, client local day
    steps      INTEGER  NOT NULL CHECK (steps >= 0 AND steps <= 200000),
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE (user_id, date)
);
CREATE INDEX IF NOT EXISTS idx_user_steps_user_date
    ON user_steps(user_id, date DESC);

CREATE TABLE IF NOT EXISTS user_steps_goal (
    user_id    TEXT     PRIMARY KEY,
    goal       INTEGER  NOT NULL CHECK (goal > 0 AND goal <= 200000),
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL
);
```
No `deleted_at` — DELETE is a hard delete (the table spec lists only
created/updated). No FK on `user_id` (matches sibling tables).

### Model / repository
- `Entry{ ID, UserID, Date string, Steps int, CreatedAt, UpdatedAt time.Time }`,
  `Validate()` → `ErrStepsOutOfRange`, `ErrInvalidDate`, `ErrDateInFuture`.
- `Goal{ UserID string, Goal int, CreatedAt, UpdatedAt *time.Time }` (nil
  timestamps = never set), `Validate()` → `ErrGoalOutOfRange`.
- `Repository`: `UpsertEntry(ctx, e) (Entry, error)` (insert-or-replace by
  `(user_id, date)`, preserve `created_at`, bump `updated_at`),
  `List(ctx, userID, since, until *string, limit int, before *string)
  ([]Entry, nextBefore string, err error)`, `Delete(ctx, userID, date) error`
  (ErrNotFound when absent), `GetGoal`, `UpsertGoal`. SQLite via
  `ON CONFLICT(user_id, date)` / `ON CONFLICT(user_id)`; memory via maps keyed
  `userID+"|"+date` and `userID`.
- Date validity: parse with `time.Parse("2006-01-02", date)`; future check vs
  `time.Now().UTC()` truncated to date, allowing `+1` day.

### Tests
Repository (sqlite+memory): upsert replaces rather than duplicates, range read,
keyset read + `next_before`, goal upsert, validation bounds. Handler: `PUT`
upsert + validation 400s (bad date, future date, out-of-range steps), `DELETE`
204/404, list pagination shape, goal get/put, and an **authz test** (user B
cannot read/modify user A's steps — cross-user list returns empty, cross-user
delete 404).

Run `go test ./...` green; `gofmt`/`go vet` clean.

---

## Task 2 — MCP (`prog-strength-mcp`)

New `src/prog_strength_mcp/steps.py` mirroring `bodyweight.py` (`register(mcp,
api)` + `_auth_header_or_raise()`), registered in `server.py`. Add methods to
`api_client.py`.

Tools:
- `log_steps(date: str, steps: int)` — `PUT /steps/{date}`. `date` is
  `YYYY-MM-DD` (agent resolves "today"/"yesterday" before calling). Returns the
  entry.
- `get_steps_goal()` / `set_steps_goal(goal: int)` — `GET`/`PUT /me/steps-goal`.
- `get_steps(since: str|None, until: str|None)` — `GET /steps` range mode for
  coaching context (include it; the agent's `analyze`/logging benefits and the
  prefetch uses it).

`api_client.py` methods: `log_steps(auth, *, date, steps)` (PUT to
`/steps/{date}`), `get_steps_goal(auth)`, `set_steps_goal(auth, *, goal)`,
`list_steps(auth, *, since, until)` (returns the `{steps, next_before}` object;
for the tool, return `data`). Each unwraps `data` and maps non-2xx → `APIError`.

Tests (`tests/test_steps_tools.py`, respx): API-client forwarding + auth header
+ APIError mapping; tool auth-guard (`RuntimeError` on missing header) and
APIError→RuntimeError mapping; `log_steps` forwards `{steps}` to
`/steps/{date}`. `pytest` green, `ruff` clean.

---

## Task 3 — Agent (`prog-strength-agent`)

Add a `log_daily_steps` intent mirroring `log_bodyweight`:
- `intents.py`: add `"log_daily_steps"` to `KNOWN_INTENTS`; register an
  `IntentSpec` with `_log_daily_steps_prefetch` (calls `get_steps` for the last
  ~14 days), `_log_daily_steps_format` (renders "RECENT STEPS …"), and
  `_LOG_DAILY_STEPS_RULES` (resolve relative dates and **confirm the resolved
  date**; mention the goal when set). Keep additions minimal and consistent.
- `model_router.py`: add a `log_daily_steps` line to `ROUTER_SYSTEM_PROMPT`'s
  intent list.
- Update `tests/test_intents.py` (`test_known_intents_enum` expected set) and add
  a prefetch test mirroring the bodyweight one. `pytest` + `ruff` green.

> Note: the MCP tool names (`log_steps`, `get_steps`, `get_steps_goal`,
> `set_steps_goal`) must match what the agent's prefetch calls. If the agent
> prefetch calls a tool by name, it must be `get_steps` with `{since}`.

---

## Task 4 — Web (`prog-strength-web`)

> Fork note: `useSearchParams` must stay under `<Suspense>` (already handled in
> `activities/page.tsx`). recharts for charts; inline SVG icons; vitest + RTL.

### `lib/api.ts` — add (date-keyed; NOT POST/recorded_at):
```ts
export type StepsEntry = { id: string; date: string; steps: number;
  created_at: string; updated_at: string };
export type StepsGoal = { goal: number; created_at: string | null;
  updated_at: string | null };
listSteps(token, { since?, until?, limit?, before? }): Promise<{ steps: StepsEntry[]; next_before: string | null }>
upsertStepsForDate(token, date, steps): Promise<StepsEntry>   // PUT /steps/{date}
deleteStepsForDate(token, date): Promise<void>                // DELETE /steps/{date}
getStepsGoal(token): Promise<StepsGoal>                       // unwrap default {goal:0,...}
putStepsGoal(token, { goal }): Promise<StepsGoal>
```

### Activities shell `app/(app)/activities/page.tsx`
- `View` union → add `"steps"`; accept `?view=steps`.
- Add a **Steps** `ToolbarButton` with an inline `FootprintsIcon`.
- Render `<StepsView days={days} />`.

### `components/activities/steps-view.tsx` (new) — top to bottom:
1. **Stat tiles** (over the timeframe): Avg daily steps, Total steps, Best day,
   Goal (target + "avg vs goal" attainment). Uses `StatTile`.
2. **Daily bar chart** — recharts `BarChart`, one bar per day across the
   window, with a `ReferenceLine` at the goal when set. Replicate the
   bodyweight goal-in-y-axis-domain fix so the goal is always in-domain and the
   line never clips.
3. **Paginated log table** — one row per day (date, steps, edit/delete),
   paginating via the API `before` cursor (keyset mode).
- Goal-setting control (modal mirroring `BodyweightGoalModal`, numeric goal, no
  unit). Fetch chart data via `listSteps({since,until})` (range), table via
  `listSteps({limit, before})` (keyset). Empty-safe CTA before any entry.

### `components/activities/activities-overview-view.tsx`
- Add `listSteps` to the parallel fetch; render a small set of steps tiles (Avg
  daily steps + goal attainment) **only when step history exists**. Combined
  chart unchanged.

### Tests (vitest)
`StepsView`: timeframe sync, goal line presence/absence, log pagination;
Overview steps tiles conditional render. `npm test` green; typecheck/lint clean.

---

## Task 5 — Mobile (`prog-strength-mobile`)

Mirror web under the Activities tab. react-native-svg hand-rolled chart;
`useFocusEffect` load pattern; SecureStore token.

- `lib/api.ts`: same five functions/types as web (date-keyed), using the mobile
  `unwrap` helper.
- `app/(tabs)/activities/index.tsx`: add `"steps"` to `ActivityView`, a Steps
  segment in `SegmentedControl`, render `<StepsView timeframe={timeframe} />`
  (show timeframe pills for steps).
- `components/activities/steps-view.tsx` (new): FlatList with stat-tile header
  (Avg/Total/Best/Goal), `steps-chart` daily bars + goal reference line, and a
  paginated day list with edit/delete; goal-setting sheet mirroring
  `bodyweight-view.tsx`. Empty-safe.
- `components/activities/steps-chart.tsx` (new): SVG daily bars + dashed-green
  in-domain goal line, modeled on `bodyweight-chart.tsx`.
- Overview gains steps tiles when history exists.
- Tests: mobile has no jest harness today; per repo conventions add render
  tests only if a harness is introduced cheaply, otherwise ensure
  `npm run typecheck` and `npm run lint` are green (the enforced gates) and add
  a lightweight pure-logic test for the stats/chart-domain helper if extracted.

---

## Task 6 — Docs (`prog-strength-docs`)

Flip `sows/daily-steps-logging.md` frontmatter `status: shipped`, body
`**Status**: Shipped`, `**Last updated**: 2026-06-15`. Commit on
`feat/daily-steps-logging`. Open the docs PR using the operator template,
referencing all implementation PRs.

---

## Rollout / merge order (for the docs PR)

1. **`prog-strength-api`** — migration `019` + `internal/steps` + endpoints.
   Deploy first: until it's live, every other surface's calls 404.
2. **`prog-strength-mcp`** and **`prog-strength-agent`** — tool module + prompt.
   MCP forwards to the API; the agent references the tools. Merge after the API
   deploys.
3. **`prog-strength-web`** and **`prog-strength-mobile`** — the Steps sub-views.
   Independent of each other; both only need the API live.

Entirely additive, no flag; every surface is empty-safe.

## Verification after rollout
- Web: Activities → Steps → log today's steps via the form → bar appears; set a
  goal → reference line renders and goal tile updates; reload table paginates.
- Agent: "log 8,400 steps yesterday" → agent confirms the resolved date; "set my
  daily step goal to 10,000" → goal persists; "how have my steps been?" reads back.
- Mobile: Activities → Steps mirrors the same; Overview shows steps tiles once
  history exists.
- API: `PUT /steps/{date}` twice for the same day replaces (no duplicate row).
