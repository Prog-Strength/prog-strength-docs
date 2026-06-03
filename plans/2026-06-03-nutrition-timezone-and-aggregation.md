# Nutrition Log Timezone Awareness and Server-Side Aggregation — Implementation Plan

> **For agentic workers:** Implement task-by-task. Each task is implemented by a
> subagent, then spec-reviewed and code-quality-reviewed before moving on.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the nutrition-log read endpoints timezone-aware and server-aggregated.
Replace the `since`/`until` (RFC3339-UTC) query contract on `GET /nutrition-log`
and `GET /nutrition-log/daily` with a strict, required-`timezone` contract
expressed as a single `date` or an inclusive `start_date`/`end_date` pair in
`YYYY-MM-DD`. The Go API resolves those local dates to UTC instants via IANA
zones. Group daily-macros by user-local calendar date. Propagate the new shape
through MCP tools, the agent harness (auto-inject `client_timezone`), the agent
prompt (prefer `get_daily_macros`), and the web + mobile clients.

**SOW:** `sows/nutrition-timezone-and-aggregation.md`

**Breaking, coordinated, pre-launch cutover.** No backwards compatibility, no data
migration, no schema change. Merge order: api → mcp → agent → web + mobile → docs.

## Repos & contract

The daily-macros route in the codebase is `GET /nutrition-log/daily` (the SOW
calls it "daily-macros" loosely). **Keep the existing route path** `/nutrition-log/daily`;
renaming routes is out of scope.

New query contract on both `GET /nutrition-log` and `GET /nutrition-log/daily`:

```
?timezone=<IANA>&date=<YYYY-MM-DD>
?timezone=<IANA>&start_date=<YYYY-MM-DD>&end_date=<YYYY-MM-DD>
```

- `timezone` **required**, validated via `time.LoadLocation`.
- Exactly one of `{date}` or `{start_date, end_date}`.
- Both range bounds inclusive; `end_date >= start_date`.
- Validation failures → 400 with a clear message. **Zero** `since`/`until` in the
  new handlers' request parsing.

---

## Task 1 — Go API: `dayBoundsUTC` + `loadTimezone` helper (+ unit tests)

Files: `internal/nutrition/day_bounds.go`, `internal/nutrition/day_bounds_test.go`.
Module: `github.com/jwallace145/progressive-overload-fitness-tracker`, Go 1.25.3.

- [ ] New `internal/nutrition/day_bounds.go`:
  - `func dayBoundsUTC(date string, loc *time.Location) (time.Time, time.Time, error)` —
    `time.ParseInLocation("2006-01-02", date, loc)`; returns `d.UTC()` and
    `d.AddDate(0,0,1).UTC()` (end exclusive; 23h/25h on DST days). Wrap parse
    errors as `fmt.Errorf("invalid date %q: %w", date, err)`.
  - `func loadTimezone(name string) (*time.Location, error)` — wraps
    `time.LoadLocation`; on error returns `fmt.Errorf("invalid timezone %s: %v", name, err)`
    so handlers emit identical 400 messages.
- [ ] `internal/nutrition/day_bounds_test.go`:
  - UTC equivalence (date in `time.UTC`).
  - `America/Denver` non-DST day boundary (verify the UTC offset).
  - DST spring-forward: 2025-03-09 in `America/Denver` → interval is 23h.
  - DST fall-back: 2025-11-02 in `America/Denver` → interval is 25h.
  - Invalid date string → error.
  - `loadTimezone` invalid IANA name → error with the `invalid timezone` prefix.
- [ ] Verify: `go test ./internal/nutrition/...`, `gofmt -l`.

## Task 2 — Go API: `time/tzdata` import + repository `DailyMacros(loc)` grouping

Files: `cmd/api/main.go`, `internal/nutrition/repository.go`,
`internal/nutrition/sqlite_repository.go`, `internal/nutrition/memory_repository.go`,
and rename in `internal/nutrition/nutrition_test.go`.

- [ ] `cmd/api/main.go`: add `_ "time/tzdata"` to the import block so
  `LoadLocation` works on a minimal base image.
- [ ] Repository interface (`repository.go`): change
  `DailyMacros(ctx, userID string, since, until time.Time)` →
  `DailyMacros(ctx, userID string, since, until time.Time, loc *time.Location)`.
  `ListNutritionLogEntries` signature is **unchanged** (stays `since, until *time.Time`).
- [ ] `sqlite_repository.go` `DailyMacros`: select raw rows in `[since, until)`
  (drop the `strftime('%Y-%m-%d', consumed_at)` SQL grouping), then group in Go
  via `consumed_at.In(loc).Format("2006-01-02")`, summing calories/protein/fat/carbs
  and counting per local-day key, returning `[]DailyMacros` sorted by date ascending.
  Preserve `deleted_at IS NULL` filtering and the `DailyMacros` struct shape.
- [ ] `memory_repository.go` `DailyMacros` (~line 226): same `loc` param; bucket by
  `e.ConsumedAt.In(loc).Format("2006-01-02")` instead of `.UTC()`.
- [ ] Rename `TestDailyMacros_AggregatesPerUTCDate` →
  `TestDailyMacros_AggregatesPerLocalDate_UTC` and pass `loc=time.UTC` (equivalence
  preserved). Update `TestDailyMacros_RangeExcludesSoftDeleted` call site to pass a `loc`.
  Add `TestDailyMacros_AggregatesPerLocalDate_NonDST` (`America/Denver`),
  `…_DSTSpringForward` (entries straddling missing 2 AM, 2025-03-09),
  `…_DSTFallBack` (entries in repeated 1 AM, 2025-11-02). Assert exact macro sums
  and `entry_count`, and that entries land on the correct local-day key.
- [ ] Verify: `go build ./...`, `go test ./internal/nutrition/...`, `gofmt -l`.

## Task 3 — Go API: rewrite both handlers + handler validation tests

Files: `internal/nutrition/handler.go`, `internal/nutrition/nutrition_test.go`
(or a new `handler_test.go`).

- [ ] Add a shared parser, e.g.
  `func parseDateRangeQuery(r *http.Request) (start, end time.Time, loc *time.Location, err error)`,
  implementing the validation order from the SOW:
  1. `timezone` present + `loadTimezone` ok → else 400 `timezone is required` /
     `invalid timezone <name>: <reason>`.
  2. Exactly one of `{date}` / `{start_date,end_date}` → else 400 naming the issue
     (`supply either date or start_date+end_date, not both`,
     `end_date is required when start_date is supplied`,
     `start_date is required when end_date is supplied`, `date or start_date is required`).
  3. Each date parses via `ParseInLocation` → 400 on failure.
  4. Range: `end_date >= start_date` → else 400 `end_date must be on or after start_date`.
  5. Single `date`: `start, end = dayBoundsUTC(date, loc)`.
     Range: `start, _ = dayBoundsUTC(start_date, loc)`; `_, end = dayBoundsUTC(end_date, loc)`.
- [ ] Rewrite `listLogEntries`: drop `parseSinceUntil`; use the new parser; call
  `repo.ListNutritionLogEntries(ctx, userID, &start, &end)`.
- [ ] Rewrite `dailyMacros`: use the new parser; call
  `repo.DailyMacros(ctx, userID, start, end, loc)`.
- [ ] Remove the now-unused `parseSinceUntil` helper. **Zero** `since`/`until` in
  request parsing of these two handlers.
- [ ] Handler tests: missing `timezone` → 400; both `date` and `start_date` → 400;
  `start_date` without `end_date` → 400; `end_date < start_date` → 400; unknown IANA → 400;
  happy `date`+`timezone` returns entries whose `consumed_at ∈ [localStartUTC, localEndUTC)`;
  happy range spans the inclusive window; a `date`+`UTC` query matches the equivalent
  explicit-UTC-instant window (regression bridge).
- [ ] Verify: `go build ./...`, `go test ./...`, `gofmt -l`.

## Task 4 — MCP: tool schemas + client + tests

Files: `src/prog_strength_mcp/nutrition.py`, `src/prog_strength_mcp/api_client.py`,
new `tests/test_nutrition_tools.py`. Python 3.12, `uv`, FastMCP, pytest+respx.

- [ ] `nutrition.py` `list_nutrition_log` and `get_daily_macros`: replace `since`/`until`
  with `timezone` (required, `Annotated[str, Field(description=...)]`) plus optional
  `date`, `start_date`, `end_date` (`str | None`, default None, descriptions per SOW).
  Raise `RuntimeError` locally if `timezone` is falsy (clear MCP-client bug). Forward
  `date`/`start_date`/`end_date` verbatim (one-of validation is the API's job). End
  `list_nutrition_log`'s docstring with the "use get_daily_macros for totals; don't
  sum yourself" sentence from the SOW.
- [ ] `api_client.py` `list_nutrition_log` and `get_daily_macros`: replace `since`/`until`
  kwargs with `timezone` (required) + `date`/`start_date`/`end_date` (optional); build
  the query dict, always include `timezone`, include the date params that are set.
  Both hit the existing paths (`/nutrition-log`, `/nutrition-log/daily`).
- [ ] `tests/test_nutrition_tools.py` (respx-mocked API): a call without `timezone`
  raises at the tool boundary before any HTTP; valid `timezone`+`date` forwarded as-is;
  valid `timezone`+`start_date`/`end_date` forwarded as-is; mixed `date`+`start_date`
  does **not** raise at the MCP layer (server validates). Assert the outgoing query
  params contain `timezone` and the date fields, and no `since`/`until`.
- [ ] Verify: `uv run pytest`, `uv run ruff check`.

## Task 5 — Agent: harness timezone injection + prompt nudge + tests

Files: `src/prog_strength_agent/server.py`, `src/prog_strength_agent/model_harness.py`,
`src/prog_strength_agent/prompt.py`, `tests/test_model_harness.py`, `tests/test_prompt.py`.

- [ ] `server.py`: thread `client_timezone` from the `/chat` handler into
  `_route_and_stream(..., *, client_timezone: str | None = None)` and forward into
  `harness.stream_chat(..., client_timezone=client_timezone)`.
- [ ] `model_harness.py`: `stream_chat` gains `client_timezone: str | None = None`.
  At the tool-use site (~226-228) define
  `_NUTRITION_TZ_TOOLS = {"list_nutrition_log", "get_daily_macros"}` (module level) and:
  ```python
  tool_input = dict(block.input or {})
  if block.name in _NUTRITION_TZ_TOOLS and "timezone" not in tool_input and client_timezone:
      tool_input["timezone"] = client_timezone
  result = await session.call_tool(block.name, tool_input)
  ```
  Inject only when the model didn't supply `timezone` and `client_timezone` is set.
  Unrelated tools untouched. No injection when `client_timezone` is None (deliberate
  downstream 400 fast-fail).
- [ ] `prompt.py`: add the "prefer `get_daily_macros` with `date` + `timezone`; don't
  sum `list_nutrition_log` rows yourself" paragraph adjacent to the existing nutrition
  logging guidance (around lines 64-68).
- [ ] `tests/test_model_harness.py`: with `client_timezone="America/Denver"`, a model
  `list_nutrition_log` tool call without `timezone` gets `timezone` merged before
  `session.call_tool`; same for `get_daily_macros`; a `list_pantry_items` call passes
  through untouched; no `client_timezone` → no injection. (Drive the harness's tool
  loop with a fake Anthropic stream + fake MCP session in the style of existing tests.)
- [ ] `tests/test_prompt.py`: assert the system prompt contains the "prefer
  get_daily_macros" guidance and mentions `date` + `timezone`.
- [ ] Verify: `uv run pytest` (or `pytest`), `ruff check`.

## Task 6 — Web client cutover

Files: `lib/api.ts`, `app/(app)/nutrition/page.tsx`. (Other `components/nutrition/*`,
`quick-add-modal.tsx`, `date-tile-strip.tsx` don't call the read endpoints directly;
`bodyweight/page.tsx` does not touch nutrition — leave it.) Next.js 16, no test suite;
verify via `npm run lint`, `npm run typecheck`, `npm run build`.

- [ ] `lib/api.ts`: change `listNutritionLog` options to
  `{ timezone: string; date?: string; startDate?: string; endDate?: string }` and
  `getDailyMacros` to `(token, { timezone, date?, startDate?, endDate? })`; build query
  params `timezone` + `date`/`start_date`+`end_date`. Add an exported request-shape type;
  remove `since`/`until` from the nutrition surface (leave bodyweight/workouts/progression
  alone). The daily endpoint stays `/nutrition-log/daily`.
- [ ] `app/(app)/nutrition/page.tsx`: drop the local `since`/`until` UTC-bound
  computation (`d.toISOString()` / `endOfLocalDay`); read tz once via
  `Intl.DateTimeFormat().resolvedOptions().timeZone`; for the selected day pass
  `{ date: <YYYY-MM-DD of selectedDay>, timezone }`. Use a local-date formatter (not
  `toISOString`, which is UTC) to derive `YYYY-MM-DD`. Remove now-dead `endOfLocalDay`
  if unused elsewhere.
- [ ] Grep confirms no `since`/`until` remain on the nutrition surface.
- [ ] Verify: `npm run lint`, `npm run typecheck`, `npm run build`.

## Task 7 — Mobile client cutover

Files: `lib/api.ts`, `components/nutrition/today-view.tsx`. RN 0.83.6 / Expo 55,
Hermes Intl available; no test suite; verify via `npm run lint` (`expo lint`) and
`npm run typecheck` (`tsc --noEmit`).

- [ ] `lib/api.ts`: mirror web — `listNutritionLog` and `getDailyMacros` take
  `timezone` + `date`/`startDate`/`endDate`; build `timezone` + `date`/`start_date`+`end_date`
  query params; remove `since`/`until` from the nutrition surface (leave workouts/progression).
- [ ] `components/nutrition/today-view.tsx`: drop `d.toISOString()`/`endOfLocalDay`
  bounds (~lines 71-74); read tz via `Intl.DateTimeFormat().resolvedOptions().timeZone`
  and pass `{ date: <local YYYY-MM-DD>, timezone }`.
- [ ] Grep confirms no `since`/`until` remain on the nutrition surface.
- [ ] Verify: `npm run typecheck`, `npm run lint`.

## Task 8 — Docs: mark SOW shipped

File: `sows/nutrition-timezone-and-aggregation.md` on branch
`feat/nutrition-timezone-and-aggregation`.

- [ ] Frontmatter `status: shipped`; body `**Status**: Shipped`;
  `**Last updated**: 2026-06-03`. Commit
  `docs: mark nutrition-timezone-and-aggregation as shipped`.

## PRs

Push `feat/nutrition-timezone-and-aggregation` and open a PR against `main` in each
modified repo plus `prog-strength-docs`. Titles follow `feat(<scope>): …`
conventional-commit style; bodies follow the Summary + Test plan format used in
recent merged PRs. The docs PR body notes that merging marks the SOW shipped.
