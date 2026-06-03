---
status: ready_for_implementation
repos:
  - prog-strength-api
  - prog-strength-mcp
  - prog-strength-agent
  - prog-strength-docs
---

# Nutrition Log Timezone Awareness and Server-Side Aggregation

**Status**: Ready for Implementation · **Last updated**: 2026-06-03

## Introduction

A user opened the **Prog Strength** chat at 7:54 AM Mountain Time and asked "hey what's my nutrition log for today." The agent confidently reported a dinner and a snack that the user had logged *the night before*, the previous calendar day. From the user's perspective, the agent hallucinated entries; from the database's perspective, every row that came back was real, with a valid ID and a `consumed_at` timestamp that genuinely fell within "today" — provided you define "today" as UTC midnight to UTC midnight rather than local midnight to local midnight. The user is in `America/Denver` (UTC-6); their previous evening's logs at `02:47–03:02Z` on June 3rd were `8:47–9:02 PM MT on June 2nd`. Claude asked the tool for `since=2026-06-03T00:00:00Z, until=2026-06-04T00:00:00Z` and got exactly what it asked for. The agent knew the date correctly. It just used UTC midnight when it should have used the user's local midnight.

In the same turn, Claude also miscomputed the totals it presented to the user — adding entry-by-entry macros across five rows, it included a 35 kcal entry in the calorie sum but excluded it from the protein and carbohydrate sums, off by exactly that entry's 2g protein and 7g carbs. The arithmetic infrastructure exists on the Go API (`DailyMacros` already groups and sums in SQL) and is exposed to the agent as `get_daily_macros`, but the agent reached for `list_nutrition_log` + mental arithmetic instead, and the tool descriptions did not push hard enough against that choice.

Both failures are LLM-determinism issues with deterministic-server fixes available. Date-range resolution is a job for Go's `time` package, not a system prompt. Sum of 20 floats is a job for SQL, not for an autoregressive model. After this ships, "what's my nutrition log for today" returns the user's local-day entries, and "what are my macros today" returns server-computed totals — both reproducible to the cent.

## Proposed Solution

Two additive changes on the Go API, plumbed through to the agent via MCP, with no breaking changes to existing query shapes:

1. **`list_nutrition_log` and `get_daily_macros` accept an optional `timezone` (IANA name) and `date` (YYYY-MM-DD) query parameter pair.** When both are supplied, the handler resolves "calendar day X in zone Y" to its UTC bounds via `time.LoadLocation` + `time.Date` and filters by `consumed_at` against those bounds. When neither is supplied, the existing `since`/`until` UTC behavior is unchanged. The `since`/`until` pair also continues to work; when supplied alongside `timezone`, the daily-totals aggregation groups its output by user-local calendar day instead of UTC date.

2. **The agent threads `client_timezone` from each `/chat` request into every nutrition tool call automatically.** The user's timezone is already on the request body (sent by the browser via `Intl.DateTimeFormat().resolvedOptions().timeZone`); today it only feeds the system-prompt date prefix. We intercept tool calls to `list_nutrition_log` and `get_daily_macros` in `model_harness.py` and merge `{"timezone": <client_timezone>}` into the input before forwarding to MCP — Claude doesn't have to know the parameter exists. Combined with a system-prompt nudge that prefers `get_daily_macros` over `list_nutrition_log + sum`, the model's path of least resistance becomes the correct one.

The Go API change is the load-bearing one; the agent and MCP changes are wiring. No data migration is required (timestamps are already stored in UTC; what was wrong was the *query*, not the storage).

## Goals and Non-Goals

### Goals

- Add optional `timezone` (IANA name, e.g. `"America/Denver"`) and `date` (YYYY-MM-DD) query parameters to `GET /nutrition-log`. When both are present, resolve the day's UTC bounds in Go and filter `consumed_at` against them. When only one of the two is present, return 400 with a clear message. When neither is present, fall through to the existing `since`/`until` path unchanged.
- Add the same `timezone` parameter to `GET /nutrition-log/daily-macros`. When supplied alongside `since`/`until`, group results by user-local calendar day instead of UTC day. When supplied alongside `date`, return one row for that local day. Without `timezone`, current UTC-day grouping is preserved.
- Add a Go helper `dayBoundsUTC(date, tz string) (time.Time, time.Time, error)` in `internal/nutrition/` that owns the date-arithmetic and is the only place IANA names get parsed. Reuse it from both handlers and the repository.
- Surface the new parameters on the matching MCP tools (`list_nutrition_log`, `get_daily_macros`) as optional, with descriptions that explain when to use `timezone`+`date` vs `since`+`until`. Pass through verbatim — no client-side date math in MCP.
- In the agent, intercept calls to `list_nutrition_log` and `get_daily_macros` in `model_harness.py` and inject `{"timezone": <client_timezone>}` from the chat request into the tool input when the field is not already present. The model never has to remember the parameter.
- Update the agent's system prompt (`prompt.py`) with one short paragraph instructing it to prefer `get_daily_macros` for daily totals over `list_nutrition_log` plus manual summation, with a one-line example showing the preferred call.
- Add Go tests covering: UTC equivalence, an America/Denver day boundary crossing UTC midnight, DST spring-forward (a 23-hour day), DST fall-back (a 25-hour day), unknown IANA name, malformed date string, the `since`/`until` + `timezone` aggregation grouping, and the conflict cases (only one of `date`/`timezone` supplied).
- Add MCP tool tests that assert the new parameters are forwarded to the API client unchanged.
- Add agent tests that assert `client_timezone` is auto-injected into `list_nutrition_log` and `get_daily_macros` calls but not into other tools.

### Non-Goals

- **Persisting a per-user timezone on the user row.** The chat request already supplies `client_timezone` per turn and that signal is more current than a stored preference (the user's machine knows where they are right now; the database doesn't). When a non-chat context eventually needs a server-known timezone (a scheduled daily summary, an email), that justifies the per-user column. Not now.
- **Frontend adoption of the new parameters.** The web and mobile clients already compute local-day bounds on the client and pass `since`/`until` to the API; that works and remains correct. The new parameters are strictly additive for agent-side callers. Touching the clients here would expand the SOW and gain nothing the clients don't already have.
- **Retroactive correction of past agent answers.** The `agent_messages` rows for previously-bad answers stay as-is. The fix applies to future turns only.
- **Timezone awareness on other domain queries (bodyweight, workouts).** Same shape of bug is plausible in both, but each has its own UX surface and rolling them in here turns one focused PR into three or four. File follow-ups as those bugs surface — the `dayBoundsUTC` helper this SOW introduces is reusable when they do.
- **A new aggregation tool exposed to the agent.** `get_daily_macros` already exists and already aggregates server-side via SQL; this SOW makes it timezone-aware and re-points the agent to prefer it. No new MCP surface area.
- **Re-tuning the router classifier.** The router correctly classified the bad turn as `intent=general`; the hallucination happened downstream of routing. The router prompt doesn't need to change here.
- **Schema migrations.** `nutrition_log_entries.consumed_at` is already UTC and precise; nothing on the storage side needs to move. This is purely a query-shape change.

## Implementation Details

### Go API: `dayBoundsUTC` helper

New file `internal/nutrition/day_bounds.go`. Pure function, no I/O, easy to test in isolation:

```go
// dayBoundsUTC returns the UTC instants that bracket the given calendar
// day in the given IANA timezone. The end bound is exclusive, matching
// the existing since/until convention used by the list endpoint. On
// DST transition days the returned range is 23h or 25h respectively;
// callers should not assume a 24-hour interval.
func dayBoundsUTC(date string, tz string) (time.Time, time.Time, error) {
    loc, err := time.LoadLocation(tz)
    if err != nil {
        return time.Time{}, time.Time{}, fmt.Errorf("invalid timezone %q: %w", tz, err)
    }
    d, err := time.ParseInLocation("2006-01-02", date, loc)
    if err != nil {
        return time.Time{}, time.Time{}, fmt.Errorf("invalid date %q: %w", date, err)
    }
    return d.UTC(), d.AddDate(0, 0, 1).UTC(), nil
}
```

`time.LoadLocation` reads from the embedded tzdata via Go's standard library — no extra dependency. On AL2023 the binary will need either the system tzdata package installed or the `time/tzdata` import; we prefer the import so the container image stays minimal and reproducible.

### Go API: `GET /nutrition-log` handler

Extend `handleList` in `internal/nutrition/handler.go`:

| Inputs | Behavior |
| --- | --- |
| neither `date` nor `timezone` | unchanged — falls through to existing `since`/`until` parsing |
| both `date` and `timezone` | call `dayBoundsUTC(date, timezone)`; use as `since`, `until` for the repo call |
| only `date`, no `timezone` | 400 with `"timezone is required when date is supplied"` |
| only `timezone`, no `date` | 400 with `"date is required when timezone is supplied"` |
| `date` + `timezone` + `since`/`until` | reject with 400 — ambiguous; pick one shape |

Error messages mirror the existing `httpresp.Error(w, http.StatusBadRequest, ...)` convention. No change to the response envelope.

### Go API: `GET /nutrition-log/daily-macros` handler

Extend the existing `daily-macros` handler to accept `timezone` (optional) alongside the existing `since` and `until` (and the new `date`). Behavior:

| Inputs | Behavior |
| --- | --- |
| `since` + `until`, no `timezone` | unchanged — groups by UTC date |
| `since` + `until` + `timezone` | groups by user-local calendar day in that zone |
| `date` + `timezone` | one row, totals for that local day |
| `date` without `timezone`, or `timezone` without any range | 400 |

For the "group by user-local date" path, the cleanest implementation is in the repository: convert `consumed_at` to the target timezone before truncating to date. Two viable approaches; pick after writing the test:

1. **In SQL**, using SQLite's `date(consumed_at, ?)` with a fixed-offset modifier — works for non-DST zones but is wrong on DST transition days because the offset is fixed.
2. **In Go**, by selecting raw rows in the UTC range and grouping in Go via `consumed_at.In(loc).Format("2006-01-02")`. Slightly more memory than SQL grouping but correct under DST and simple to read.

Recommendation: Go-side grouping. Day-range queries are small (typical bound is one day to one month), and the correctness gain on DST days is worth the few hundred bytes per row.

### Repository: extend `DailyMacros` signature

Current signature in `sqlite_repository.go:337`:

```go
func (r *SQLiteRepository) DailyMacros(ctx context.Context, userID string, since, until time.Time) ([]DailyMacros, error)
```

Extend to accept the zone:

```go
func (r *SQLiteRepository) DailyMacros(ctx context.Context, userID string, since, until time.Time, loc *time.Location) ([]DailyMacros, error)
```

`loc == nil` means "group by UTC date" (preserves the existing behavior). `loc != nil` means "group by calendar date in loc". Match the same change in `memory_repository.go` so the in-memory repo (used by handler tests) stays in lockstep.

Update the existing `TestDailyMacros_AggregatesPerUTCDate` to keep covering the UTC path, and add a sibling `TestDailyMacros_AggregatesPerLocalDate` for the new path with at least one DST transition fixture.

### MCP: tool schema updates

In `prog-strength-mcp/src/prog_strength_mcp/nutrition.py`, extend `list_nutrition_log` and `get_daily_macros` with two optional parameters:

- `timezone: str | None = None` — "IANA timezone name (e.g. America/Denver). Required when supplying date; optional when supplying since/until."
- `date: str | None = None` — "YYYY-MM-DD calendar day. Resolves to UTC bounds using the timezone parameter. Mutually exclusive with since/until."

Forward to the API client unchanged. Update the existing tool docstring for `list_nutrition_log` so the "prefer get_daily_macros for daily rollups" guidance Claude is currently ignoring is split out into a more prominent location (the system prompt, see below).

### Agent: auto-inject `client_timezone` on nutrition tool calls

`prog-strength-agent/src/prog_strength_agent/model_harness.py`. The tool-call site is `stream_chat`'s inner loop where each `block.input` becomes the MCP call payload (currently around line 226–228). Add the timezone into the input dict for the two tools that take it, before the call:

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

`client_timezone` needs to reach the harness. It already flows into `build_chat_system_prompt` via `server.py`; thread it as a new kwarg on `stream_chat` (default `None`) and pass it from `_route_and_stream`. Backwards-compatible because callers without the kwarg get the existing behavior.

### Agent: nudge toward `get_daily_macros`

In `prog-strength-agent/src/prog_strength_agent/prompt.py`, add a short paragraph near the nutrition logging guidance:

```text
When the user asks for daily nutrition totals or a per-day summary
("how many calories today," "what's my protein for the day,"
"how did I do today on macros"), call get_daily_macros — it returns
totals computed by the API. Do NOT call list_nutrition_log and add
up the macros yourself; arithmetic across many items is unreliable
and the API computes it exactly.
```

Place it adjacent to the existing nutrition-logging instructions so the model sees them together. Cache hit rate stays unaffected — the addition is small and append-only relative to the cached prefix.

### Tests

Go API:
- `day_bounds_test.go`: UTC equivalence, America/Denver day boundaries crossing UTC midnight, DST spring-forward fixture (a 23-hour day in `America/Denver`), DST fall-back fixture (a 25-hour day), invalid IANA name returns error, invalid date string returns error.
- `handler_test.go`: `?date=2026-06-03&timezone=America/Denver` filters to the expected entries; `?date=…` without `?timezone=…` returns 400; unknown timezone returns 400; both pair shapes return identical entries when given equivalent UTC ranges; supplying all four params returns 400.
- `daily_macros_test.go`: extend the existing `TestDailyMacros_AggregatesPerUTCDate` and add `TestDailyMacros_AggregatesPerLocalDate` with a DST fixture; assert grouped totals are exact (calories, protein_g, fat_g, carbs_g) and entry_count matches.

MCP:
- Extend the existing `list_nutrition_log` and `get_daily_macros` tool tests to assert the new parameters are forwarded to the API client method unchanged.

Agent:
- `test_model_harness.py`: with a chat request whose `client_timezone == "America/Denver"`, a model-emitted `list_nutrition_log` tool call without `timezone` in its input has `timezone="America/Denver"` injected before reaching MCP. Same for `get_daily_macros`. Other tool calls are untouched.
- `test_prompt.py`: the system prompt contains the "prefer get_daily_macros" guidance.

### Rollout

1. Land Go API changes. Old MCP and old agent keep working — the new params are optional and additive.
2. Land MCP tool-schema updates. Old agent keeps working — `timezone` is optional on the tool, and the model never had to know about it before.
3. Land agent changes (timezone interceptor + prompt nudge + tests). End-to-end wired.
4. Hand-test in prod: open chat at a time of day where local-today and UTC-today differ; ask "what's my nutrition log for today" and confirm via `sqlite3 /data/telemetry.db` that the resulting `agent_tool_calls.arguments_json` contains the user's timezone and that the returned entries are the user's local-day entries.
5. Hand-test: ask "what are my macros for today" and confirm `agent_tool_calls.tool_name == 'get_daily_macros'`, no `list_nutrition_log` call in the same turn, and the displayed numbers match a manual sum of the user's local-day entries — off by zero.

Each step is independently reverting-safe. No data migration. No schema change.
