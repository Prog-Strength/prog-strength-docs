---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-mcp
  - prog-strength-agent
  - prog-strength-web
  - prog-strength-mobile
  - prog-strength-docs
---

# Nutrition Log Timezone Awareness and Server-Side Aggregation

**Status**: Shipped · **Last updated**: 2026-06-03

## Introduction

A user opened the **Prog Strength** chat at 7:54 AM Mountain Time and asked "hey what's my nutrition log for today." The agent confidently reported a dinner and a snack that the user had logged *the night before*, the previous calendar day. From the user's perspective, the agent hallucinated entries; from the database's perspective, every row that came back was real, with a valid ID and a `consumed_at` timestamp that genuinely fell within "today" — provided you define "today" as UTC midnight to UTC midnight rather than local midnight to local midnight. The user is in `America/Denver` (UTC-6); their previous evening's logs at `02:47–03:02Z` on June 3rd were `8:47–9:02 PM MT on June 2nd`. Claude asked the tool for `since=2026-06-03T00:00:00Z, until=2026-06-04T00:00:00Z` and got exactly what it asked for. The agent knew the date correctly. It just used UTC midnight when it should have used the user's local midnight.

In the same turn, Claude also miscomputed the totals it presented to the user — adding entry-by-entry macros across five rows, it included a 35 kcal entry in the calorie sum but excluded it from the protein and carbohydrate sums, off by exactly that entry's 2g protein and 7g carbs. The arithmetic infrastructure exists on the Go API (`DailyMacros` already groups and sums in SQL) and is exposed to the agent as `get_daily_macros`, but the agent reached for `list_nutrition_log` + mental arithmetic instead, and the tool descriptions did not push hard enough against that choice.

Both failures are LLM-determinism issues with deterministic-server fixes available. Date-range resolution is a job for Go's `time` package, not a system prompt. Sum of 20 floats is a job for SQL, not for an autoregressive model. After this ships, "what's my nutrition log for today" returns the user's local-day entries, and "what are my macros today" returns server-computed totals — both reproducible to the cent.

## Proposed Solution

Pre-launch is the right window to harden the API's date contract, and we use it. The existing `since` / `until` RFC3339-UTC parameter shape on the nutrition endpoints — the one Claude composed wrong — is removed entirely. It is replaced by a single coherent contract: `timezone` is always required, and queries are expressed as either a single `date` or a `start_date` / `end_date` pair (both inclusive), all in `YYYY-MM-DD`. The Go API resolves those dates to UTC instants via the supplied IANA zone using `time.LoadLocation` + `time.ParseInLocation` and filters on `consumed_at` from there. Callers stop doing date arithmetic; the server centralizes it.

The change is breaking. It propagates outward: MCP tool schemas change shape, web and mobile clients move from "compute the local-day UTC bounds yourself" to "send date strings + your timezone." The web and mobile clients already know the user's timezone (`Intl.DateTimeFormat().resolvedOptions().timeZone`); they were just doing more work than they needed to. The agent reads `client_timezone` off every `/chat` request body and the harness now injects it into every nutrition tool call before forwarding to MCP — Claude never has to pass the parameter, and never has to remember it exists. We ship the cross-repo cutover in one coordinated dispatch so there's no window where a deployed client and a deployed API disagree.

While the cross-repo wave is in flight, we also fix the arithmetic issue by getting Claude out of the math business. The existing `get_daily_macros` MCP tool already aggregates server-side in SQL; the tool descriptions are reworked, a short paragraph is added to the agent's system prompt, and the harness optionally rewrites a `list_nutrition_log` call to `get_daily_macros` when the model is clearly asking for totals — but the rewrite is a follow-up; the prompt nudge is the v1 mechanism.

No data migration. `consumed_at` is already UTC and precise; what was wrong was the *query*, not the storage.

## Goals and Non-Goals

### Goals

- Replace `since` / `until` on `GET /nutrition-log` and `GET /nutrition-log/daily-macros` with a strict, required-timezone contract:
  - `timezone` — **required**, IANA name (e.g. `America/Denver`, `UTC`, `Europe/Berlin`). Validated against Go's `time.LoadLocation`. Unknown or malformed name returns 400 with `"invalid timezone <name>: <reason>"`.
  - `date` — `YYYY-MM-DD`, single calendar day in `timezone`. Mutually exclusive with `start_date`/`end_date`.
  - `start_date` + `end_date` — `YYYY-MM-DD`, both inclusive, both must be supplied together. `end_date < start_date` returns 400.
  - Exactly one of the two shapes must be supplied; supplying neither, both, or only one half of a range pair returns 400 with a clear message naming the missing field.
  - 0 occurrences of `since` or `until` in the new handlers' request parsing. The old shape is gone, not deprecated.
- Add a single Go helper `dayBoundsUTC(date string, tz *time.Location) (time.Time, time.Time, error)` in `internal/nutrition/`. It is the only place IANA-zone-aware date math happens. Reused by both endpoints and by the repository when grouping the daily-macros output.
- Group `GET /nutrition-log/daily-macros` results by **user-local calendar date in the supplied `timezone`**, not by UTC date. Compute the grouping in Go (`consumed_at.In(loc).Format("2006-01-02")`), not in SQL — SQLite's `date(consumed_at, ?)` modifier takes a fixed UTC offset and is wrong on DST transition days.
- Mirror the contract on the MCP tools `list_nutrition_log` and `get_daily_macros`: both add required `timezone` plus the one-of `date` / `start_date`+`end_date` shape. Remove `since` / `until` from the tool schemas.
- In `prog-strength-agent`, plumb `client_timezone` from `/chat` request body into `ModelHarness.stream_chat` and have the harness inject `{"timezone": <client_timezone>}` into the input of every `list_nutrition_log` and `get_daily_macros` tool call before forwarding to MCP. Inject only when the model didn't already supply it. Other tool calls are untouched.
- Update the agent's system prompt (`prompt.py`) with one short paragraph instructing it to prefer `get_daily_macros` for any daily-totals question and to never sum macros across many `list_nutrition_log` rows by hand. One concrete example showing the right tool call.
- Update `prog-strength-web` to call the new shape:
  - `/nutrition` page reads the user's timezone from `Intl.DateTimeFormat().resolvedOptions().timeZone`, sends `date=<selected-day>&timezone=<tz>` instead of computing `since`/`until` UTC instants locally.
  - Any other call site of the daily-macros or list endpoints (currently `app/(app)/nutrition/page.tsx`, `components/nutrition/*`, `components/quick-add-modal.tsx`, `components/date-tile-strip.tsx`) moves to the new shape.
  - TypeScript types tracking the request shape are updated; no `since`/`until` references remain on the nutrition surface after the PR.
- Update `prog-strength-mobile` to the same new shape. Capture timezone via the mobile equivalent of `Intl.DateTimeFormat().resolvedOptions().timeZone` (React Native exposes it the same way under Hermes); send `date` / `start_date` + `end_date` + `timezone` parameters.
- Add Go API tests covering:
  - `dayBoundsUTC` against UTC equivalence, a non-DST zone day boundary, a DST spring-forward day (23-hour day in `America/Denver` — March 9, 2025), a DST fall-back day (25-hour day — November 2, 2025), an invalid IANA name, an invalid date string.
  - Handler validation: missing `timezone` → 400; both `date` and `start_date` → 400; only `start_date` (no `end_date`) → 400; `end_date < start_date` → 400; unknown IANA name → 400.
  - End-to-end: a fixture with one entry per UTC hour for 48 hours yields the expected per-day grouping under both `UTC` and `America/Denver` timezones, including the DST transition day showing a 23h or 25h cluster on the right date.
- Add MCP tool tests that assert `timezone` + one-of `date` / `start_date`+`end_date` are forwarded to the API client unchanged, and that calls without `timezone` raise at the boundary (before hitting the API).
- Add agent tests covering: with a chat request `client_timezone="America/Denver"`, a model-emitted `list_nutrition_log` tool call without `timezone` has it injected before MCP sees the call; same for `get_daily_macros`; tool calls to unrelated tools (e.g. `list_pantry_items`) are untouched; a chat request with no `client_timezone` does no injection and the model's call passes through (the API will then 400; that's a deliberate fast-fail).

### Non-Goals

- **Backwards-compatible co-existence of the old `since` / `until` shape.** Pre-launch; not worth the dual-write/dual-read complexity. One coordinated cutover lands the strict shape everywhere at once.
- **Persisting a per-user timezone on the user row.** The chat request already supplies `client_timezone` per turn and that signal is more current than a stored preference (the user's machine knows where they are right now; the database doesn't). When a non-chat context eventually needs a server-known timezone (a scheduled daily summary, an email), that justifies the per-user column. Not now.
- **Retroactive correction of past agent answers.** The `agent_messages` rows for previously-bad answers stay as-is. The fix applies to future turns only.
- **Timezone-awareness on other domain queries (bodyweight, workouts).** Same shape of bug is plausible in both, but each has its own UX surface and rolling them in here turns one focused dispatch into three or four. File follow-ups as those bugs surface — the `dayBoundsUTC` helper this SOW introduces is reusable when they do.
- **A new aggregation tool exposed to the agent.** `get_daily_macros` already exists and already aggregates server-side via SQL; this SOW makes it timezone-aware and re-points the agent to prefer it. No new MCP surface area.
- **Harness-side rewrite of `list_nutrition_log` → `get_daily_macros` when the model is clearly asking for totals.** Tempting (most reliable way to guarantee server-side aggregation), but it puts intent-classification logic in the harness that overlaps the model's job. The prompt nudge plus better tool descriptions is v1; if Claude still reaches for the wrong tool after this ships, a harness-side rewrite becomes a sensible follow-up.
- **Re-tuning the router classifier.** The router correctly classified the bad turn as `intent=general`; the hallucination happened downstream of routing. The router prompt doesn't need to change here.
- **Schema migrations.** `nutrition_log_entries.consumed_at` is already UTC and precise; nothing on the storage side needs to move. This is purely a query-shape change.

## Implementation Details

### Go API: `dayBoundsUTC` helper

New file `internal/nutrition/day_bounds.go`. Pure function, no I/O, easy to test in isolation:

```go
// dayBoundsUTC returns the UTC instants that bracket the given calendar
// day in loc. The end bound is exclusive, matching SQL BETWEEN-like
// half-open range semantics used downstream. On DST transition days the
// returned interval is 23h or 25h respectively; callers should not
// assume 24 hours.
func dayBoundsUTC(date string, loc *time.Location) (time.Time, time.Time, error) {
    d, err := time.ParseInLocation("2006-01-02", date, loc)
    if err != nil {
        return time.Time{}, time.Time{}, fmt.Errorf("invalid date %q: %w", date, err)
    }
    return d.UTC(), d.AddDate(0, 0, 1).UTC(), nil
}
```

A sibling `loadTimezone(name string) (*time.Location, error)` wraps `time.LoadLocation` with a consistent error shape so handlers produce identical 400 messages.

Go's `time.LoadLocation` requires either the system tzdata or the `time/tzdata` package imported. Import `_ "time/tzdata"` in `cmd/api/main.go` so the binary is self-contained and reproducible across the AL2023 base image and a developer's macOS laptop.

### Go API: `GET /nutrition-log` handler

Replace the existing `since`/`until` parsing entirely. The new shape:

```text
?timezone=<IANA>&date=<YYYY-MM-DD>
?timezone=<IANA>&start_date=<YYYY-MM-DD>&end_date=<YYYY-MM-DD>
```

Validation order (fail at the first error):

1. `timezone` present and `loadTimezone` succeeds → otherwise 400 `"invalid timezone <name>: <reason>"` (or `"timezone is required"` when missing entirely).
2. Exactly one of `{date}` or `{start_date, end_date}` is supplied → otherwise 400 with a message naming the offending combination (`"supply either date or start_date+end_date, not both"`, `"end_date is required when start_date is supplied"`, etc.).
3. Each supplied date parses with `time.ParseInLocation("2006-01-02", ..., loc)` → 400 on parse failure.
4. For range: `end_date >= start_date` → 400 otherwise (`"end_date must be on or after start_date"`).
5. Convert to UTC bounds:
   - Single `date`: `start, end := dayBoundsUTC(date, loc)`
   - Range: `start, _ := dayBoundsUTC(start_date, loc); _, end := dayBoundsUTC(end_date, loc)` — `end` is the UTC instant of the *next* local day after `end_date`, making `end_date` inclusive.
6. Call the repository with `(start, end)`. No further filtering needed; the repo signature stays UTC-only.

### Go API: `GET /nutrition-log/daily-macros` handler

Same validation as above. Repository signature changes:

```go
// Before:
func (r *SQLiteRepository) DailyMacros(ctx context.Context, userID string, since, until time.Time) ([]DailyMacros, error)

// After:
func (r *SQLiteRepository) DailyMacros(ctx context.Context, userID string, since, until time.Time, loc *time.Location) ([]DailyMacros, error)
```

Implementation: select raw rows in `[since, until)`, group in Go via `consumed_at.In(loc).Format("2006-01-02")`, sum per group, return one `DailyMacros` per local-day key sorted ascending. The in-memory repository (`memory_repository.go:226`) gets the same change so handler tests stay green. The existing `TestDailyMacros_AggregatesPerUTCDate` is renamed and re-pointed at `loc = time.UTC` to keep covering the equivalence; a new `TestDailyMacros_AggregatesPerLocalDate_DSTSpringForward` and `…_DSTFallBack` pin the transition behavior.

### MCP: tool schema updates

In `prog-strength-mcp/src/prog_strength_mcp/nutrition.py`:

`list_nutrition_log`:

```python
@mcp.tool
async def list_nutrition_log(
    timezone: Annotated[
        str,
        Field(description="IANA timezone (e.g. America/Denver) the date params are interpreted in."),
    ],
    date: Annotated[
        str | None,
        Field(default=None, description="Single calendar day, YYYY-MM-DD. Mutually exclusive with start_date/end_date."),
    ] = None,
    start_date: Annotated[
        str | None,
        Field(default=None, description="Inclusive start of a multi-day range, YYYY-MM-DD. Required if end_date is supplied."),
    ] = None,
    end_date: Annotated[
        str | None,
        Field(default=None, description="Inclusive end of a multi-day range, YYYY-MM-DD. Required if start_date is supplied."),
    ] = None,
) -> list[dict[str, Any]]:
    ...
```

Same parameter set on `get_daily_macros`. Implementation forwards verbatim; validation that exactly one shape is supplied is the API's job (single source of truth), but raise locally if `timezone` is missing — that's a clear MCP-client bug to surface early.

Remove the existing `since` / `until` parameters from both tool signatures. Update the docstrings; the new docstring for `list_nutrition_log` ends with:

> "For totals over a day or a range, use `get_daily_macros` — it returns sums computed by the API. Do not list entries and sum macros yourself; arithmetic across many items is unreliable."

### Agent: thread `client_timezone` through the harness

In `prog-strength-agent/src/prog_strength_agent/server.py`, the `/chat` handler already reads `client_timezone` off the request body and passes it to `build_chat_system_prompt`. Thread it as a new kwarg into `_route_and_stream(messages, user_token, telemetry, system_prompt, *, client_timezone: str | None = None)` and forward it from there into the harness.

`ModelHarness.stream_chat` gains a `client_timezone: str | None = None` kwarg. The tool-call site (currently around `model_harness.py:226-228`) becomes:

```python
_NUTRITION_TZ_TOOLS = {"list_nutrition_log", "get_daily_macros"}

# Inside the tool-use loop, when block.type == "tool_use":
tool_input = dict(block.input or {})
if (
    block.name in _NUTRITION_TZ_TOOLS
    and "timezone" not in tool_input
    and client_timezone
):
    tool_input["timezone"] = client_timezone
result = await session.call_tool(block.name, tool_input)
```

When `client_timezone` is not on the chat request (older clients, scripted callers), the harness does not inject. The downstream API will then 400 — that is the deliberate fast-fail so the gap is loud, not silent.

### Agent: nudge toward `get_daily_macros`

In `prog-strength-agent/src/prog_strength_agent/prompt.py`, add a short paragraph near the nutrition logging guidance:

```text
When the user asks for daily nutrition totals or a per-day summary
("how many calories today," "what's my protein for the day,"
"how did I do today on macros"), call get_daily_macros with date and
timezone — it returns totals computed by the API. Do NOT call
list_nutrition_log and add up the macros yourself; arithmetic across
many items is unreliable, and the API computes it exactly.
```

Place it adjacent to the existing nutrition-logging instructions so the model sees both rules together.

### Web client

`prog-strength-web` currently computes the user's local-day UTC bounds in the browser and passes `since`/`until` to the nutrition endpoints. The cutover:

- Inspect each call site in `app/(app)/nutrition/page.tsx`, `components/nutrition/nutrition-log-view.tsx`, `components/quick-add-modal.tsx`, `components/date-tile-strip.tsx`, and `app/(app)/bodyweight/page.tsx` (the last one only if it touches nutrition endpoints; if it's purely bodyweight, leave it alone — bodyweight isn't in this SOW).
- Replace the UTC-bound computation with: read the timezone once via `Intl.DateTimeFormat().resolvedOptions().timeZone`; pass `date=<selectedDay>&timezone=<tz>` for single-day reads or `start_date=<a>&end_date=<b>&timezone=<tz>` for ranges.
- Update the TypeScript types describing the request to remove `since`/`until` and add the new shape.
- Where a hook caches by query key, include `timezone` in the key so a user changing zones (devtools-change-timezone test path) refetches.

### Mobile client

`prog-strength-mobile` mirrors web — same call sites in spirit, same cutover. Hermes exposes `Intl.DateTimeFormat().resolvedOptions().timeZone` (Hermes >= 0.74 has full Intl support); if the bundled Hermes version doesn't, add `react-native-localize` and use `getTimeZone()` instead.

### Tests

Go API:
- `day_bounds_test.go`: UTC equivalence, an `America/Denver` non-DST day boundary, DST spring-forward fixture, DST fall-back fixture, invalid date string. (Invalid IANA name is tested at the handler level since `dayBoundsUTC` takes a `*time.Location`.)
- `handler_test.go` (list endpoint): missing `timezone` → 400; `date` and `start_date` both supplied → 400; `start_date` without `end_date` → 400; `end_date` < `start_date` → 400; unknown IANA → 400; happy path with `date` + `timezone` returns entries whose `consumed_at` falls in `[localStartUTC, localEndUTC)`; happy path with range returns entries across the inclusive span; equivalent `date` + `UTC` query and an equivalent UTC-instant `since`/`until` query against the *old* code yield identical entries (regression bridge).
- `daily_macros_test.go`: rename + repurpose `TestDailyMacros_AggregatesPerUTCDate` to pin `loc=time.UTC` behavior; new `TestDailyMacros_AggregatesPerLocalDate_NonDST` against `America/Denver`; new `TestDailyMacros_AggregatesPerLocalDate_DSTSpringForward` with entries straddling the missing 2 AM hour; new `TestDailyMacros_AggregatesPerLocalDate_DSTFallBack` with entries in the repeated 1 AM hour. Assert exact macro sums and `entry_count`.

MCP:
- Extend existing `list_nutrition_log` and `get_daily_macros` tool tests: a call without `timezone` raises at the tool boundary; valid `timezone` + `date` is forwarded to the API client method as-is; valid `timezone` + `start_date`/`end_date` is forwarded as-is; mixed `date` and `start_date` does not error at the MCP layer (server validates).

Agent:
- `test_model_harness.py`: a chat with `client_timezone="America/Denver"`, the model emits a `list_nutrition_log` tool call without `timezone` → the harness merges `timezone="America/Denver"` into the input before MCP receives it. Same for `get_daily_macros`. A `list_pantry_items` tool call goes through untouched. A chat with no `client_timezone` → harness does not inject; the call passes through unchanged.
- `test_prompt.py`: the system prompt contains the "prefer get_daily_macros" guidance and explicitly mentions calling it with `date` + `timezone`.

Web / Mobile:
- Existing component tests for the nutrition view get extended cases for the new request shape; mock the API client to assert the outgoing request body has `date` + `timezone` and no `since` / `until`. Snapshot tests on the page output may need re-baselining if any rendered date string changed because of the query-key change; the rendered content should not change for an in-zone user.

### Rollout

This is a coordinated cross-repo dispatch — the SOW lands one dispatch, the developer worker opens PRs in `prog-strength-api`, `prog-strength-mcp`, `prog-strength-agent`, `prog-strength-web`, and `prog-strength-mobile` from one driver. Merge order matters for keeping a clean staging environment alive during the rollout window:

1. Merge `prog-strength-api` first. Until the matching client PRs merge, the deployed web/mobile/agent are broken (they send the old shape; API returns 400). Acceptable pre-launch — there are no external users — but reviewers should be ready to merge the dependents promptly.
2. Merge `prog-strength-mcp` next; the agent's tool calls would already be 400ing against the new API, so the MCP catching up restores correctness for the in-flight agent code.
3. Merge `prog-strength-agent` (timezone interceptor + prompt + tests). Chat is now end-to-end correct.
4. Merge `prog-strength-web` and `prog-strength-mobile` in parallel. Once both deploy, all consumers are on the new shape.
5. Hand-test in prod: open chat at a time of day where local-today and UTC-today differ; ask "what's my nutrition log for today" and confirm via `sqlite3 /data/telemetry.db` that the `agent_tool_calls.arguments_json` contains the user's timezone and the returned entries are the user's local-day entries.
6. Hand-test: ask "what are my macros for today" and confirm `agent_tool_calls.tool_name == 'get_daily_macros'`, no `list_nutrition_log` call in the same turn, and the displayed numbers match a manual sum of the user's local-day entries — off by zero.
7. Hand-test the web `/nutrition` page in DevTools with the simulated timezone set to America/Denver, confirm the page's request payload contains `date` + `timezone=America/Denver`.

No data migration. No schema change. Each merge step is independently revertable (the cross-repo coordination is in merge order; each PR is internally consistent).
