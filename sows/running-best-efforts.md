---
status: draft
repos:
  - prog-strength-api
  - prog-strength-mcp
  - prog-strength-web
  - prog-strength-docs
---

# Running Best Efforts (Running PRs)

**Status**: Draft · **Last updated**: 2026-06-09

## Introduction

Prog Strength tracks lift personal records well — each headline lift has a row in `personal_records` that records the heaviest weight ever moved, the rep count, and the workout it happened in. A lifter who hits 305 × 3 on bench sees that surfaced as a PR; the agent can congratulate them; the chat can reason about the gap between PR weight and estimated 1RM. None of that exists for the running side of training. A runner who ships a fast 5K segment inside a 10-mile easy run has nothing to show for it inside Prog Strength — the activity row carries an *average* pace across the whole run, and the average is a poor signal for "did I just set a 5K PR?"

This SOW gives running its analogue. For every running activity, the system computes the runner's best time over each of a small fixed set of standard distances — 1 mile, 2 mile, 5K, 10K, half marathon, full marathon — and surfaces, per runner, the all-time best across activities at each distance. Critically, a "5K best effort" is the fastest 5K *window* found inside any activity, not just activities that happen to total ~5K: a runner whose 10-mile long run happens to contain their fastest-ever 5K should see that fact reflected. The agent gets a new tool to read these so it can reason across lifting and running progression in the same conversation ("your bench press is up 15 lb since March and your 5K best effort dropped 12 s in the same window — you're rebuilding well after the deload").

The work is scoped tight on purpose. The standard distance set is fixed at six entries up front. There is no per-user custom distance, no race-day vs. training distinction, no pace-zone analytics, no week-over-week trend charts. Those are clean follow-ups once we see how the agent actually uses the data and which surfaces runners ask for.

The same SOW also redesigns the existing `/personal-records` web page to host the new running data alongside the existing lift PRs. The redesign is additive — a segmented Lifts / Running view-switcher beside the page title, a parallel running-card layout, and a per-item progression chart on both views that makes the cross-domain story ("pace is improving while squat 1RM climbs") legible in one place. The existing dark aesthetic, card grid, and headline-lift customization flow are preserved; nothing already on the page is removed or restyled.

## Proposed Solution

Best efforts are derived from the raw trackpoint stream at *upload time*, computed once per running activity, and persisted to a new `activity_best_efforts` side table. The compute path lives inside the existing `internal/activity/` summarizer, in the same transaction that writes the activity row and its downsampled trackpoints — there is no separate write path and no recompute trigger, because activities are immutable in place (the only "edit" path is delete-then-reupload).

Per activity, the summarizer runs a distance-anchored two-pointer sweep over the *raw* trackpoint stream (not the ~300-point downsample — those points are 50–150 m apart on a typical run, far too coarse for a 1-mile window). For each of the six target distances, the sweep finds the minimum-time window whose cumulative distance is ≥ the target, with linear interpolation at the right edge to remove the systematic bias from sample boundaries. Activities shorter than a target distance produce no row for that distance. The output is at most six rows per activity, written to `activity_best_efforts(activity_id, distance_key, duration_seconds)`.

The read side is a single grouped query: for the calling user, find `MIN(duration_seconds)` per `distance_key` across `activity_best_efforts` rows whose parent activity is live and `activity_type='running'`. A new `GET /running/best-efforts` endpoint returns the result, including the activity ID and start time of each record-setting activity so the UI (and agent) can link back to the run. The query lives behind the existing user-auth middleware and reuses the activity repository — no separate read path against S3, no on-demand recompute.

A new `internal/activity/best_efforts.go` houses the standard-distance enum and the sweep function. The enum is the single source of truth shared between the summarizer, the repository's read query, the DTO, and the MCP tool description — adding or renaming a distance later is a one-symbol change, not a grep.

S3 archival of the original TCX (already done by the shipped activity pipeline) is the safety net for the distance-set decision. If a future SOW needs a new distance (1500 m, 15K), the in-process backfill described below re-parses each archived TCX and rewrites the table. Today's runners only pay for today's six distances.

MCP grows a new `running.py` module — modeled on the existing `workouts.py` / `nutrition.py` registration pattern — exposing a `get_running_best_efforts` tool that proxies the new endpoint. This is the first running-domain MCP tool; the agent currently has no read access to the running side, so this also closes the long-standing "agent can't talk about runs" gap for the most-requested running fact. List/get tools for individual activities are out of scope here; the brief stays narrow.

On the web, the existing `app/(app)/personal-records/page.tsx` is reshaped into a two-view page. A segmented control beside the existing "Customize" button toggles between Lifts and Running; the selection is URL-backed via `?view=lifts|running` so it survives reload and is linkable, and data fetching moves to TanStack Query with one query per view (`["personal-records", "lifts"]` and `["personal-records", "running"]`). The existing PR card layout becomes the model for the running card — distance name as the title, best time as the headline figure, pace as the secondary stat, "Set on {date}" + "View activity →" — minus the "Time for a max?" status badge, which is a lift-side concept (estimated 1RM gap) with no running analogue. Below each card on either view sits an expandable per-item progression chart: the lift chart plots estimated 1RM over time from the existing `exercise_one_rep_max_history`; the running chart plots best-effort time at that distance over time from `activity_best_efforts`. Both API and runtime unit handling reuse the existing weight-unit (lb/kg) and `DistanceUnitProvider` (mi/km) plumbing — the redesign does not introduce any new unit infrastructure.

## Goals and Non-Goals

### Goals

- Add a new migration `internal/db/migrations/016_activity_best_efforts.sql` creating `activity_best_efforts(activity_id, distance_key, duration_seconds)` with the composite primary key `(activity_id, distance_key)`, an `ON DELETE CASCADE` foreign key to `activities(id)`, and a `CHECK` constraint on `distance_key` against the standard distance enum. Add a secondary index `(distance_key, duration_seconds)` to serve the per-distance MIN query.
- Define the standard distance set in one place — `internal/activity/best_efforts.go` — as a `[]StandardDistance` slice of `{Key, Meters, DisplayName}`. The keys are `1mi`, `2mi`, `5k`, `10k`, `half_marathon`, `marathon`; meters are `1609.344`, `3218.688`, `5000`, `10000`, `21097.5`, `42195`. All six are referenced by key from every other surface (DB CHECK, summarizer, repository, DTO, MCP tool description).
- Implement `bestEfforts(tps []parsedTrackpoint, targets []StandardDistance) map[string]float64` in `best_efforts.go`. One O(n) two-pointer sweep per target distance, with linear interpolation at the right edge (see § Algorithms). Activities whose total cumulative distance is < the target distance produce no entry for that target.
- Wire `bestEfforts` into `summarize` (in `internal/activity/tcx_summarizer.go`) gated on `actType == ActivityRunning`. The function returns an additional `[]ActivityBestEffort` slice on the `Activity` struct, populated only for running activities and empty for walks/rides.
- Extend `Repository.Create` (interface + memory + sqlite implementations) to write the best-effort rows in the same transaction as the activity and trackpoints. `Repository.SoftDelete` is unchanged — the FK cascade is defensive only since rows are live-only filtered at read time.
- Add `Repository.GetUserRunningBestEfforts(ctx, userID)` returning one row per `distance_key` (the user's current best at that distance), each row carrying `duration_seconds`, `activity_id`, `activity_start_time`. Implements the per-distance MIN with row attribution via `ROW_NUMBER() OVER (PARTITION BY distance_key ORDER BY duration_seconds ASC, start_time ASC)` so the *earliest* activity wins on duration ties (preserves the original date's claim, mirroring the lift-PR tie semantics from `personal-records.md`).
- Add a new `GET /running/best-efforts` HTTP endpoint behind the existing `auth.RequireUser` middleware. Returns one entry per standard distance the user has ever achieved (entries for distances the user has not yet covered are omitted). Response shape under § API Surface.
- Add a new MCP module `prog_strength_mcp/running.py` registering one tool: `get_running_best_efforts()`. The tool reads the inbound Authorization header (same pattern as every other MCP tool), forwards to `GET /running/best-efforts`, and returns the response verbatim. Register the module in `prog_strength_mcp/server.py` alongside the existing `workouts.register(mcp, api)` line.
- Add `list_running_best_efforts` to `APIClient` in `prog_strength_mcp/api_client.py`, mirroring the shape of the existing forwarding methods.
- Backfill: on the next API boot after the migration ships, walk every live `activity_type='running'` row in the existing `activities` table whose `activity_best_efforts` row count is zero, re-fetch its TCX from S3, re-parse, recompute the sweep, and insert the rows. Gated by the same `existing > 0` pattern the rest of the repo's backfills use so reruns are no-ops. See § Backfill.
- Tests per § Tests, including a targeted `intervals.tcx`-style fixture with a fast 5K embedded in a longer run to pin the "fast window inside a slower long run is what gets recorded" behavior.
- Document an `activities_best_efforts_version` constant in `best_efforts.go` (initial value `1`). The version number isn't stored per row today — see Open Question #1 — but the constant is the anchor a future "we changed the algorithm, force recompute" branch hangs on.
- Add `GET /personal-records/{exercise_id}/history` returning a per-exercise estimated-1RM time series derived from `exercise_one_rep_max_history` so the lift card's progression chart has a clean per-card data source independent of the muscle-group dashboard endpoint.
- Add `GET /running/best-efforts/{distance_key}/history` returning every activity that achieved a best effort at that distance, sorted by `activity_start_time` ascending, so the running card's progression chart can read directly from `activity_best_efforts`.
- **Web: redesign `app/(app)/personal-records/page.tsx`** as a two-view page:
  - Add a segmented control next to the existing "Customize" button reading "Lifts" / "Running". URL-backed via `?view=lifts|running` (default `lifts`). The "Customize" button stays visible on Lifts and is hidden on Running (there is nothing customizable on running for v1).
  - Move all data fetching to TanStack Query: `useQuery({ queryKey: ["personal-records", "lifts"], ... })` and `useQuery({ queryKey: ["personal-records", "running"], ... })`. Queries are lazy-mounted per active view so switching tabs doesn't refetch the inactive side and the active side caches across switches.
  - Add a Running view rendering a card grid that mirrors the Lifts grid (same `grid-cols-1 sm:grid-cols-2 lg:grid-cols-3` shell). Each card surfaces: distance name (e.g. "5K"), best time as the headline figure (e.g. "19:44"), pace as the secondary stat (e.g. "6:21 /mi"), "Set on {date}", and a "View activity →" link to `/running/{activity_id}`. Distance-set entries the user has not yet achieved render as empty-state cards mirroring the existing "NO RECORD YET" lift placeholder.
  - Each card on either view exposes an inline expand affordance that reveals a compact progression line chart below the card. Lift charts plot estimated 1RM over time; running charts plot best-effort time over time (lower-is-better axis annotated accordingly). Series is lazy-fetched on first expand via TanStack Query keyed by `["pr-history", view, id]`, then cached for the page lifetime.
  - Reuse the existing `DistanceUnitProvider` formatters (`formatPace`, `formatDistance`) for all running-side numbers. Pace is rendered as `formatPace(duration_seconds / (distance_meters / 1000))` plus a "/mi" or "/km" suffix from `unitLabel`. Lift-side weight formatting is unchanged.
  - Preserve the dark aesthetic, card grid spacing, typography scale, and the existing PRCard component's status badge ("Time for a max?"). The Running card explicitly omits any status badge.
  - Sidebar nav, layout shell, and the route path `/personal-records` are unchanged.
- Add `lib/api.ts` types and client methods for the new endpoints: `listRunningBestEfforts(token)` returning `RunningBestEffort[]`, `getExerciseOneRMHistory(token, exerciseId)`, and `getRunningBestEffortHistory(token, distanceKey)`. Match the existing helper style (`unwrap` + bearer header).

### Non-Goals

- **Per-user custom distances** (1500 m, 15K, marathon-relay leg, etc.). The standard set is fixed for v1. Custom distances need a user-prefs surface, a per-user table, and recompute-on-add — a clean follow-up.
- **Lift-PR-style "event log" of new best-effort breaks.** The lift PR system records every PR break in `personal_record_events` so workouts can be badged inline. Running activities have no equivalent badge surface in v1 (no Running tab badge spec), and the agent gets the same signal from the current-bests query plus the activity's `start_time`. Adding an events table is a clean follow-up when there's a UI surface to feed.
- **Mobile.** Web only. The API is stable from day one so the mobile app can pick up `/running/best-efforts` and the redesigned PR page in its own dispatch.
- **Restyling the existing lift PR card.** The redesign is additive — segmented control, running view, and the expandable progression chart — not a refresh of the lift card. The "Time for a max?" badge, weight typography, "View workout →" link, and the customize flow all stay as-is.
- **A unified "Personal Records" data model spanning lifts and running.** The two views are rendered by the same page but read from two distinct endpoints (`/personal-records` for lifts, `/running/best-efforts` for running) and two distinct underlying tables. There is no joined DTO and no "all your PRs" feed — that would force normalization between weight-units and time-units that buys nothing.
- **Compact always-visible inline chart on every card.** Charts are expand-on-click, not eager. See § Web — Page Redesign for the rationale and Open Question #6 for the alternative.
- **Pace-zone or HR-zone analytics on the window.** A best-effort row carries duration only. Average HR or pace breakdown across the winning window is a derived view the agent can ask for later by re-reading the trackpoints; it is not persisted.
- **Moving-time / paused-segment handling.** The sweep operates on elapsed time as-is. A run that includes a 30-second stop at a traffic light has that 30 s baked into any window crossing it. Garmin's TCX `<Trackpoint>` stream does not consistently carry a "paused" signal, and the runtime to separate moving from paused time on the fly is non-trivial. Accepted limitation; document inline.
- **Walking, cycling, or other-sport best efforts.** The sweep runs only for `activity_type='running'`. Walks could have a 5K best effort in principle, but the comparison surface is running-specific (pace, race-distance norms), and the existing summarizer already gates pace fields on running for this reason. Mixing sport-classes in one PR query is the kind of mistake that surfaces as nonsense numbers.
- **In-place activity editing.** Activities remain immutable; the only edit path is delete-then-reupload. The lack of a recompute trigger depends on this; if in-place editing is ever introduced, the recompute call has to be wired into the same Update path that already exists for the lift-PR system. Documented as an inline assumption.
- **A per-row `best_efforts_version` column.** The version constant exists in Go so we can grep for stale rows later, but until we actually change the algorithm, paying for a column write per row buys nothing. Open Question #1.
- **Re-parse-on-read.** Reads always hit `activity_best_efforts`. The S3 re-parse path is reserved for the one-time backfill at migration time; the read path stays a single grouped query.
- **Lap-anchored splits** (best mile-split per activity, best 5K-by-laps). Garmin's `<Lap>` boundaries are anchor-arbitrary; the sweep ignores them and treats the activity as one continuous trace. Lap-anchored splits are a follow-up if runners ask for them by activity.
- **Trend charts.** No "your 5K best over time" plot. The data is there to compute it; the surface is deferred.

## Implementation Details

### Data Model

One new migration: `internal/db/migrations/016_activity_best_efforts.sql`.

**Table `activity_best_efforts`:**

| Column | Type | Description |
| --- | --- | --- |
| `activity_id` | TEXT | FK to `activities(id)` with `ON DELETE CASCADE`. |
| `distance_key` | TEXT | One of `1mi`, `2mi`, `5k`, `10k`, `half_marathon`, `marathon`. Enforced by `CHECK(distance_key IN (...))`. |
| `duration_seconds` | REAL | Elapsed time across the fastest window of that distance inside the activity. Real (not integer) because interpolation produces sub-second precision. |

`PRIMARY KEY (activity_id, distance_key)`. Composite key matches the access pattern — every read either targets a single activity (rare; not exposed in v1) or scans across activities by `distance_key` (the bests query).

**Indexes:**

- `CREATE INDEX idx_activity_best_efforts_distance ON activity_best_efforts(distance_key, duration_seconds);` — serves the per-distance MIN. The composite PK alone would force a full scan ordered by `activity_id`, which doesn't help here.

**Foreign keys:** `activity_id → activities(id) ON DELETE CASCADE`. Activities soft-delete via `deleted_at` rather than hard-delete (see migration 015), so cascade fires only on a true row removal — defensive. Live-row filtering is done at read time by joining against `activities.deleted_at IS NULL`.

No `user_id` column on the table. The user is denormalized away — every read joins against `activities` for the live filter and activity-type filter anyway, so `user_id` would only ever be redundant with the FK target. Saves one write per row and one constraint to maintain.

### Standard Distance Set

`internal/activity/best_efforts.go`:

```go
package activity

// StandardDistance is one of the fixed-set targets the best-effort sweep
// produces a row for on every running activity. The Key is the value stored
// in activity_best_efforts.distance_key — keep it stable across releases
// since the DB CHECK constraint references it by exact string.
type StandardDistance struct {
    Key         string
    Meters      float64
    DisplayName string
}

// StandardDistances is the v1 set. Order is display order (shortest first);
// downstream code that iterates assumes this order, but no algorithm
// depends on it.
var StandardDistances = []StandardDistance{
    {Key: "1mi",           Meters: 1609.344,  DisplayName: "1 Mile"},
    {Key: "2mi",           Meters: 3218.688,  DisplayName: "2 Mile"},
    {Key: "5k",            Meters: 5000,      DisplayName: "5K"},
    {Key: "10k",           Meters: 10000,     DisplayName: "10K"},
    {Key: "half_marathon", Meters: 21097.5,   DisplayName: "Half Marathon"},
    {Key: "marathon",      Meters: 42195,     DisplayName: "Marathon"},
}

// bestEffortsVersion is bumped when the sweep algorithm changes in a way
// that requires existing rows to be recomputed. It's not stored per-row
// today (see Open Question #1) — it's the constant a future backfill
// branch tests against.
const bestEffortsVersion = 1
```

The migration's `CHECK` constraint hard-codes the same six keys; a startup-time test (`TestStandardDistances_MatchMigrationCheck`) parses the migration file and asserts the two sets are identical, so a typo in either place fails CI rather than silently dropping rows at runtime.

### Algorithms

**Two-pointer sweep with right-edge interpolation.** Given the raw trackpoint stream (ordered by elapsed time, monotonically non-decreasing cumulative distance) and a target distance T:

```
walk left, right indices over trackpoints:
  for right in 1..N-1:
    # Tighten left while [left+1, right] still covers T meters
    while left+1 < right and tps[right].dist - tps[left+1].dist >= T:
      left++

    span = tps[right].dist - tps[left].dist
    if span < T:
      continue  # right hasn't extended far enough yet

    # Interpolate the exact crossing time at distance tps[left].dist + T.
    # That distance falls inside the segment [prev, right], where prev is
    # max(left, right-1) — the loop guarantees [left, right-1] spans < T,
    # so the crossing is in the right-most segment.
    prev := max(left, right-1)
    seg_d := tps[right].dist - tps[prev].dist
    if seg_d <= 0:
      continue  # degenerate; zero-distance segment
    target_end := tps[left].dist + T
    ratio := (target_end - tps[prev].dist) / seg_d
    end_t := tps[prev].time + ratio * (tps[right].time - tps[prev].time)
    window_s := end_t - tps[left].time
    if window_s < best:
      best = window_s
```

Why two-pointer: the sweep is O(n) per target. With six targets and ~3,000–15,000 trackpoints per typical activity, the whole sweep across all distances completes in well under 10 ms on a modern CPU — small enough that wiring it into the upload-time transaction has zero practical cost.

Why interpolate the right edge: trackpoint cumulative distance is sampled, not continuous. A target of exactly 5,000 m almost never lands on a sample boundary — typical samples are 5–15 m apart. Taking "the first window that meets or exceeds 5,000 m" introduces a systematic bias toward windows slightly longer than 5,000 m, which inflates the reported time. Linear interpolation between the bracketing samples removes that bias to within sub-second precision (the trackpoints are clean, ~1 Hz, with locally-linear distance).

Why only the right edge (and not also the left): for the optimal window position the minimum lies at a piecewise-linear breakpoint of the window-time function, which is either when the left edge crosses a sample or when the right edge crosses one. Sweeping all sample-boundary lefts with right-edge interpolation reaches every "left-aligned" critical point. The "right-aligned" critical points are at most one sample period away in time — a fraction of a second at typical sample rates — and dropping them keeps the algorithm one-pass and obvious. The residual asymmetric bias is acceptable for v1; if a runner ever notices it (they will not), symmetrize in a follow-up.

**Operates on the raw trackpoints, not the ~300-point downsample.** The downsample is sized for chart rendering: on a marathon, 300 points stride ~140 m apart, which makes any 1-mile-window math useless. The summarizer hands `bestEfforts` the un-downsampled `parsedTrackpoint` slice it already has in memory at that point in the pipeline.

**Activities shorter than the target distance.** If `tps[N-1].dist - tps[0].dist < T`, no row is emitted for that target. A 2-mile easy run emits rows for `1mi` and `2mi` only — the `5k`, `10k`, `half_marathon`, and `marathon` keys never appear for that activity.

**Paused / stopped time.** Out of scope. The sweep treats elapsed time as continuous; a runner who stops at a traffic light has the stop time inside any window that straddles it. Documented in non-goals.

### Write Path

The upload pipeline (`IngestTCX` → `summarize` → `repo.Create`) gains a small addition. In `summarize`:

```go
if actType == ActivityRunning {
    // existing avg/best pace code
    a.BestEfforts = bestEfforts(parsed.Trackpoints, StandardDistances)
}
```

`bestEfforts` returns `[]ActivityBestEffort{Key, DurationSeconds}` — at most six entries, omitting distances the activity is too short for. The slice is hung off `Activity`:

```go
type Activity struct {
    // ... existing fields ...
    BestEfforts []ActivityBestEffort
}

type ActivityBestEffort struct {
    DistanceKey     string
    DurationSeconds float64
}
```

`Repository.Create` writes `activity_best_efforts` rows in the same transaction that writes the activity row and the trackpoints. On a duplicate-source `ErrDuplicate` rollback (the existing dedup path), the best-effort rows roll back with everything else — no orphan possible. The memory repository mirrors the behavior so `handler_test.go` against the in-memory repo stays accurate.

Soft-delete (`Repository.SoftDelete`) does not touch `activity_best_efforts`. Live-row filtering at read time is the load-bearing mechanism. Hard-delete (which the codebase doesn't expose today) would fire the cascade — defensive.

### API Surface

One new endpoint, behind `auth.RequireUser`.

**`GET /running/best-efforts`**

Returns the calling user's current best across each standard distance. Distances the user has never covered are omitted (the entry simply does not appear); distances the user has covered at least once appear with the activity that produced the current best.

200 response:

```json
{
  "best_efforts": [
    {
      "distance_key": "1mi",
      "distance_label": "1 Mile",
      "distance_meters": 1609.344,
      "duration_seconds": 332.4,
      "pace_sec_per_km": 206.5,
      "activity_id": "act_8f3...",
      "activity_start_time": "2026-05-22T07:12:11Z"
    },
    {
      "distance_key": "5k",
      "distance_label": "5K",
      "distance_meters": 5000,
      "duration_seconds": 1184.7,
      "pace_sec_per_km": 236.9,
      "activity_id": "act_2c1...",
      "activity_start_time": "2026-04-18T06:45:00Z"
    }
  ]
}
```

The result is sorted by `StandardDistances` order (shortest first) so the agent and any future UI render consistently without re-sorting. `pace_sec_per_km` is computed at request time from `duration_seconds / (distance_meters / 1000)` — derivable, not stored.

`current_pace_sec_per_km` is intentionally not split out by user unit preference at the API level; the unit conversion happens at render time (web/mobile) the same way pace conversion already does for the existing running endpoints. The agent does its own unit reasoning in natural language.

Error responses follow the existing `{error, code}` shape; this endpoint produces only `401` (no auth) and `500` (DB failure) — no input validation, no 4xx variants.

**`GET /running/best-efforts/{distance_key}/history`**

Returns every activity that achieved a best effort at the specified distance, ordered by `activity_start_time` ascending. Feeds the per-card progression chart on the web Running view.

200 response:

```json
{
  "distance_key": "5k",
  "distance_label": "5K",
  "distance_meters": 5000,
  "points": [
    { "activity_id": "act_8f3...", "activity_start_time": "2026-01-12T07:02:00Z", "duration_seconds": 1340.2 },
    { "activity_id": "act_2c1...", "activity_start_time": "2026-02-18T07:08:00Z", "duration_seconds": 1312.7 },
    { "activity_id": "act_f04...", "activity_start_time": "2026-04-18T06:45:00Z", "duration_seconds": 1184.7 }
  ]
}
```

`points` is the full series (every activity that achieved a best effort at this distance, not just the record-breakers) so the chart shows real density, not a step function. Order is ascending so a `<LineChart>` consumes it without resorting. `404` with `code: "unknown_distance_key"` if the path parameter is not in the standard set. No pagination — even prolific runners produce a few hundred points per distance at most.

**`GET /personal-records/{exercise_id}/history`**

Returns the per-workout estimated 1RM time series for a single exercise, derived from `exercise_one_rep_max_history`. Feeds the per-card progression chart on the web Lifts view.

200 response:

```json
{
  "exercise_id": "barbell-bench-press",
  "exercise_name": "Barbell Bench Press",
  "unit": "lb",
  "points": [
    { "workout_id": "wk_2a...", "performed_at": "2026-01-04T17:30:00Z", "estimated_1rm": 305.4 },
    { "workout_id": "wk_3b...", "performed_at": "2026-01-11T17:25:00Z", "estimated_1rm": 308.1 }
  ]
}
```

`404` with `code: "unknown_exercise_id"` if the slug doesn't exist in the catalog. The series is one point per workout in which the exercise was performed; the value is the max estimated 1RM across that workout's sets on the exercise (Epley applied per-set, max taken). This is the same value the `MuscleGroupProgression` endpoint already exposes — this endpoint is the per-exercise slice, separate from the per-muscle-group rollup, so the personal-records page doesn't have to wrangle a muscle-group-keyed response.

### MCP Surface

One new module: `prog_strength_mcp/running.py`, modeled on `workouts.py`. One tool:

```python
@mcp.tool
async def get_running_best_efforts() -> dict[str, Any]:
    """Return the calling user's current best time across each standard
    running distance (1 mile, 2 mile, 5K, 10K, half marathon, marathon).

    A "best effort" at a given distance is the fastest window of that
    length found inside any of the user's running activities — including
    a fast segment embedded inside a longer run, not just runs that
    happen to total that distance. Distances the user has never covered
    are absent from the response.

    Use this for any "what's my fastest 5K?" / "PR" / "personal best"
    question on the running side. For lifting PRs, use the dedicated
    lifting endpoints (those live on the workout / personal_records
    surfaces, not here).

    Returns:
        A dict with key `best_efforts`, a list of entries each carrying
        the distance, duration, derived pace, and the activity that set
        the record. The list is sorted shortest distance first.
    """
    auth = _auth_header_or_raise()
    try:
        return await api.list_running_best_efforts(auth)
    except APIError as e:
        raise RuntimeError(f"API error ({e.status_code}): {e.message}") from e
```

`APIClient.list_running_best_efforts(auth)` (new) is a thin GET wrapper mirroring the existing `list_workouts` / `list_nutrition` methods.

`server.py` gains a single new line in the registration block, alphabetized:

```python
running.register(mcp, api)
```

The module is the first running-domain MCP module; future running tools (e.g. `list_running_activities`, `get_running_activity`) will land in the same file rather than creating a new module per tool, matching the per-domain grouping the existing modules use.

### Web — Page Redesign

Touches one route: `app/(app)/personal-records/page.tsx`. The redesign is additive — segmented control + running view + expandable per-card chart — with no changes to the sidebar, the layout shell, the `HeadlineExercisesModal`, or the existing `PRCard` body.

**File structure.** The flat `page.tsx` grows into a small directory:

```
app/(app)/personal-records/
  page.tsx                          # shell: header, segmented control, query orchestration
  _components/
    LiftsView.tsx                   # grid of LiftPRCard, lifted out of today's PRCard
    RunningView.tsx                 # grid of RunningPRCard
    LiftPRCard.tsx                  # current PRCard body, unchanged except for the expand chrome
    RunningPRCard.tsx               # new card
    ProgressionChart.tsx            # expandable chart shell shared by both card types
    ViewSwitcher.tsx                # segmented Lifts / Running control
```

Reasoning: today's `page.tsx` is one file mixing fetch, layout, and card rendering. The redesign adds three new render branches (running cards, expandable charts, view switching) — extracting one card per file and a single `ProgressionChart` shell keeps each file scannable. Matches the `_components/` pattern the existing `/running` route already uses.

**Page shell (`page.tsx`).**

- Reads `?view` from `useSearchParams()`; valid values `lifts` and `running`, default `lifts`. Wrong/missing → `lifts`.
- Renders the header band (title + segmented control + Customize button) and routes to `<LiftsView />` or `<RunningView />` based on the active view.
- Customize button: visible on `lifts`, hidden via conditional render on `running`. There is nothing customizable on running for v1 (the distance set is fixed in backend Go code; see § Standard Distance Set).
- Wraps the page in a `QueryClientProvider` if one isn't already provided higher in the tree (check `app/layout.tsx`). If a global provider exists, this page just calls `useQuery`.

**`<ViewSwitcher />`.** A two-segment pill control. Inactive segment: muted text, transparent background. Active segment: surface background, foreground text, accent underline-or-ring. Updates the URL via `router.replace("/personal-records?view=running")` (use `replace`, not `push`, so the back button doesn't fill with view-toggle history). Sits in the existing header band on the same row as the title — beside the "Customize" button, not below it, to honor the brief's "avoids adding a second horizontal band that pushes cards down."

**Data fetching.** Each view owns one query.

- Lifts: `useQuery({ queryKey: ["personal-records", "lifts"], queryFn: () => listPersonalRecords(token), enabled: view === "lifts" })`. The `enabled` flag means switching away pauses revalidation but the cached snapshot survives so a quick back-and-forth doesn't refetch.
- Running: `useQuery({ queryKey: ["personal-records", "running"], queryFn: () => listRunningBestEfforts(token), enabled: view === "running" })`.
- Per-card history (both views): `useQuery({ queryKey: ["pr-history", view, id], ... enabled: expanded })`. Disabled until the card is expanded so we don't fan out N+M GETs on page load. The cache is per `view + id` so swapping back to a previously expanded card paints instantly.

**`<LiftPRCard />`.** Identical to today's `PRCard` body — title, weight × reps, "Set on", current estimated 1RM with the gap, "Time for a max?" badge, "View workout →" — wrapped in a new card shell that adds:

- A small chevron button at the bottom-right corner that toggles a local `expanded` state.
- When expanded, renders `<ProgressionChart kind="lifts" exerciseId={record.exercise_id} />` below the card body, inside the same card border. The card grows in height; the grid handles the reflow.
- When the user has no PR on that exercise yet (`!hasPR`), the expand chevron is suppressed — nothing to chart.

**`<RunningPRCard />`.** Mirrors the lift card layout for visual consistency:

- Header row: distance display name ("5K", "Half Marathon"). No status badge (decided).
- Body: best time as the headline figure formatted as `m:ss` for sub-hour and `h:mm:ss` for ≥ 1 h via a small helper `formatDuration(seconds)`. Pace as the secondary stat formatted via `formatPace(secPerKm)` from `DistanceUnitContext`, suffixed with `/${unitLabel}`. "Set on {date}" via the same `formatDate` helper the lift card uses.
- "View activity →" link to `/running/{activity_id}`.
- Empty state for distances not yet achieved: render an em-dash and "NO RECORD YET" in the same muted style the lift card uses for never-trained exercises. Expand chevron suppressed.
- Expand chevron + `<ProgressionChart kind="running" distanceKey={entry.distance_key} />` mirror the lift card.

**`<ProgressionChart />`.** One component for both kinds, picks its data fetcher and axis labels off the `kind` prop. Built on Recharts (already in deps via the `/running` charts):

- Compact: ~140 px tall, full card width, no legend, minimal axis labels. The Y-axis label reads "Est. 1RM (lb|kg)" for lifts and "Time" for running; X-axis is "Date" for both.
- Lift Y-axis: estimated 1RM in the row's `unit` (lb or kg). Higher-is-better; default Y domain.
- Running Y-axis: duration in seconds, tick-formatted as `m:ss` via a helper. Lower-is-better; the chart annotation reads "lower is faster" once below the title to avoid the "why does the line slope down when I'm improving?" confusion. (Inverted Y axis is the alternative — see Open Question #4.)
- Single line, no dots, `connectNulls={false}`. Dark theme: `stroke="#60a5fa"`, grid `#27272a`, ticks `#a1a1aa` — same palette as the existing `PaceChart`.
- Loading state: skeleton rectangle of chart height. Error state: muted "Couldn't load history" inline message. Empty (1 point only): render the single point as a dot + a "Not enough data yet" muted label.

**Unit handling.**

- Lifts: continue to use `record.unit` straight off the API row. No change.
- Running: every running display number flows through `DistanceUnitProvider` — `formatPace` for paces, `formatDistance` for distances. The page does not introduce its own conversion helpers.
- Pace from a best-effort row is computed as `duration_seconds / (distance_meters / 1000)` to feed `formatPace`. The API already returns `pace_sec_per_km` on the bests payload; prefer that field when present so the math lives in one place.

**Accessibility.**

- The segmented control is a `<div role="tablist">` with two `<button role="tab" aria-selected>` segments. Keyboard arrow keys move focus across segments; Enter / Space activates. Matches the WAI-ARIA tabs pattern even though the panel content lives below in a separate region.
- The expand chevron is a `<button aria-expanded>` whose label is "Show progression" / "Hide progression" for screen readers.

### Backfill

Runs in-process at API startup the first boot after migration 016 ships, gated by an `existing > 0` check on `activity_best_efforts`. Same shape as `BackfillOneRepMaxHistory`.

For each live `activities` row where `activity_type='running'`:

1. Fetch `tcx_s3_key` from the row.
2. Download the object from the `prog-strength-tcx-uploads` bucket via the existing `s3_archiver`.
3. Run `parseTCX` + `validate` over the bytes. (The file passed validation at upload time, so this is expected to succeed; log and skip on the rare case where it doesn't, with the activity ID for follow-up.)
4. Run `bestEfforts(parsed.Trackpoints, StandardDistances)`.
5. Insert the resulting rows in one transaction per activity.

Why re-parse from S3 instead of from the in-DB downsampled trackpoints: the downsampled trackpoints are too coarse (the chart sample, not the raw stream) to produce honest 1-mile-window math. The S3 object is the canonical source and is exactly what `summarize` would see on a fresh upload.

**Scale boundary.** At current beta volumes (low hundreds of running activities) the backfill completes well under a minute inline with the boot sequence. The S3 GETs dominate; doing them serially is fine at this scale. If the activity table grows past ~10,000 running rows the backfill should split into a separate CLI command, but until then inline-at-boot is the simpler story.

**Recoverability.** Truncate `activity_best_efforts` and restart the API. The boot path finds it empty and rebuilds.

### Tests

**Go API (`internal/activity/`):**

- `best_efforts_test.go` — pure-function tests against synthetic trackpoint traces:
  - `constant pace, exact-multiple distance` — sweep over a synthetic 5,000 m / 1,200 s constant-pace trace; expect exact 1,200 s for the 5K entry, exact 240 s for the 1K-equivalent.
  - `constant pace, non-exact distance` — same trace but sampled every 7 m; expect the 5K entry within ±0.1 s of the analytical answer (proves interpolation removes the boundary bias).
  - `fast window inside a longer slow run` — 10-mile trace at 9:00/mi with a 5K segment at 7:00/mi embedded in the middle; expect the 5K entry to reflect the 7:00/mi pace, not the 9:00/mi average. This is the load-bearing behavior for the feature.
  - `activity too short for some distances` — 1.5 mi trace; expect a `1mi` entry but no `2mi`, `5k`, etc.
  - `monotonic non-strict` — a trace where two consecutive trackpoints share the same cumulative distance (rare GPS glitch); sweep should not divide by zero and should produce a sensible result.
  - `walk / cycling activity` — `summarize` returns no best-effort rows when `actType != ActivityRunning`.
- `tcx_summarizer_test.go` — extend the existing fixtures' assertions to cover the new field. Add an `intervals.tcx`-style fixture with a known embedded fast 5K and pin its expected `BestEfforts` map.
- `sqlite_repository_test.go` — insert an activity with best-effort rows, assert the rows appear, assert `GetUserRunningBestEfforts` returns the per-distance minimum with the correct activity ID, assert two activities with tied durations resolve by earliest `start_time`, assert a walk activity's bests do not appear in the running query.
- `handler_test.go` — happy path `GET /running/best-efforts` for a user with multiple activities, including verifying the `pace_sec_per_km` derivation. Empty case (user with no running activities) returns `{"best_efforts": []}`.
- `TestStandardDistances_MatchMigrationCheck` — parses the SQL migration file, extracts the `CHECK(distance_key IN (...))` token set, asserts it equals `{d.Key for d in StandardDistances}`. Fails CI on drift.

**MCP (`prog-strength-mcp/tests/`):**

- `test_running_tools.py` — patch `APIClient.list_running_best_efforts`, invoke the tool with a fake Authorization header, assert the response is forwarded verbatim and that a missing header raises the expected `RuntimeError`.

**Web (`prog-strength-web`):**

- `lib/api.test.ts` — unit-test the new `listRunningBestEfforts`, `getExerciseOneRMHistory`, and `getRunningBestEffortHistory` client methods against a `msw` mock returning the documented response shapes; assert thrown errors carry the API's `error`/`code` fields.
- `app/(app)/personal-records/_components/RunningPRCard.test.tsx` — render the card with a populated entry under both `mi` and `km` `DistanceUnitProvider` values; assert the pace string switches units and the "View activity →" link points at `/running/{activity_id}`. Render the empty-state variant; assert the expand chevron is suppressed.
- `app/(app)/personal-records/_components/ViewSwitcher.test.tsx` — render with `?view=lifts`, simulate a click on the Running segment, assert `router.replace("/personal-records?view=running")` is called and not `router.push`. Assert keyboard arrow-key focus traversal across segments.
- `app/(app)/personal-records/page.test.tsx` — integration-style test via `msw`:
  - `?view=lifts` (default) renders LiftPRCards from the `listPersonalRecords` mock; the running endpoint is not called.
  - `?view=running` renders RunningPRCards; the lifts endpoint is not called.
  - Clicking the expand chevron on a card fires exactly one `["pr-history", view, id]` query; clicking it again to collapse and re-expand re-uses the cached result (no second network call).
  - The Customize button is hidden on `?view=running`.
- PR description checklist (no E2E framework introduced; mirrors the existing running SOW pattern):
  - Toggle between Lifts and Running; URL updates; reload preserves the view.
  - Expand a lift card; estimated-1RM line chart renders; collapse and re-expand instantly.
  - Expand a running card with multiple activities; line chart renders with date-axis points.
  - Switch distance unit in settings; running cards and the running chart's tooltip both re-render in the new unit.
  - Empty state with zero running activities; the Running view shows six "NO RECORD YET" cards.
  - A user with no PRs at all on a configured headline lift; the existing "NO RECORD YET" lift placeholder still renders.

**Migration test:** existing `db.Migrate()` harness applies cleanly on an empty DB and on a DB pre-populated with activity rows + trackpoints from migration 015 (proves the new table layers on top without disturbing existing data).

### Rollout

Coordinated cross-repo dispatch. Merge order:

1. `prog-strength-api` — migration 016, `best_efforts.go`, summarizer wiring, repository extensions, the three new endpoints (`GET /running/best-efforts`, `GET /running/best-efforts/{distance_key}/history`, `GET /personal-records/{exercise_id}/history`), backfill. Deploys via the existing release workflow. The backfill runs once on the first boot post-deploy; subsequent boots are no-ops via the row-count gate.
2. `prog-strength-mcp` — `running.py` module, `APIClient.list_running_best_efforts`, `server.py` registration. Deploys via the existing release workflow. The MCP tool returns the new endpoint's payload as soon as the API is live.
3. `prog-strength-web` — `personal-records/page.tsx` reshape, `_components/` extraction, `lib/api.ts` client methods, TanStack Query wiring. Deploys via the existing release workflow. The page picks up the new endpoints transparently once the API release lands.
4. `prog-strength-docs` — this SOW, flipped from `draft` to `shipped`.

Each PR is independently revertable. The API change is additive — no existing read or write path changes shape — and the MCP change adds a tool without touching any existing one. Reverting the web release leaves the existing single-view personal-records page in place; the new API endpoints simply go unused. Reverting the API release after the web is live degrades the running view to a single "couldn't load" inline message (TanStack Query surfaces the 404), while the lifts view continues to function from the unchanged `/personal-records` endpoint.

## Open Questions

1. **Per-row `bestEffortsVersion` column.** The constant exists in Go but isn't stored per row. Storing it would let a future algorithm change ship a one-line backfill (`WHERE best_efforts_version < N`) instead of a `TRUNCATE + reboot`. Options: (a) add the column now at one INTEGER per row of cost; (b) defer until we actually change the algorithm and pay the truncate-and-recompute cost once. Tentative lean: (b) — the recompute cost at current scale is seconds, and YAGNI on the column until there's a real algorithm-change event to justify it.

2. **Symmetrize the interpolation at the left edge.** As described in § Algorithms, sweeping sample-boundary lefts with right-edge interpolation leaves a fractional-sample-period asymmetric bias at the left edge. Options: (a) accept the bias for v1; (b) add a symmetric left-edge interpolation pass (still one-pass, still O(n) per target, ~10 extra lines). Tentative lean: (a). The bias is well under a second at typical sample rates and no runner will notice it.

3. **Surface best-effort rows on the existing activity DTO** (`GET /activities/{id}`). Today this SOW only adds them to `/running/best-efforts`. Including them on the per-activity detail response is a small additional join that would let a future activity-detail UI show "this activity holds your current 5K PR" inline without a second request. Options: (a) ship the standalone endpoint only; (b) also embed `best_efforts` on the activity detail DTO. Tentative lean: (a) — defer the embed until there's a UI surface that actually consumes it; it is a strictly additive change to layer on later.

4. **Backfill error handling for missing-from-S3 TCX files.** If an activity row exists but its TCX is missing from S3 (manual deletion, bucket recreation), the backfill currently logs + skips that row. Options: (a) keep the skip behavior; (b) hard-fail the boot so the operator notices; (c) mark the activity with a `best_efforts_unavailable` flag so the read endpoint can render a "no data" entry. Tentative lean: (a) — the S3 bucket has versioning + has been stable since the running feature shipped, and a hard boot failure on operator-induced state divergence is a footgun.

5. **Running progression chart Y-axis orientation.** Running time is "lower is better," which inverts the intuitive direction of a line chart. Options: (a) plot duration directly (axis ascends downward — fastest at top); (b) plot duration with `reverseYAxis` (axis ascends upward, line slopes up as the runner improves); (c) leave the axis natural and annotate "lower is faster" under the chart title. Tentative lean: (c). Inverting the axis fixes the slope-direction tension but every tick label then reads upside-down relative to a stopwatch ("16:00 is higher up the chart than 17:00"), which is its own confusion. Annotation costs nothing and trusts the reader.

6. **Always-visible inline mini-chart vs. expand-on-click.** The brief offered either. The current SOW commits to expand-on-click; an always-visible sparkline would be a different layout decision (would change card aspect ratio, eager-load N+M history queries on view mount, and complicate the empty-state card). Options: (a) ship expand-on-click as specified; (b) replace with an always-visible 60 px sparkline + a "View full chart" link; (c) defer the chart entirely to a follow-up SOW and ship Part 1 + the view switcher first. Tentative lean: (a) — the brief explicitly named it an option, the data contract is symmetric either way, and the deeper chart on expand is more useful than a sparkline that's too small to read.

7. **Lift per-exercise history endpoint vs. extending the muscle-group endpoint.** This SOW adds `GET /personal-records/{exercise_id}/history` as a new endpoint. An alternative is to extend `GET /workouts/progression` to accept `?exercise_id=` (the existing endpoint's code comment in `lib/api.ts` foreshadows future per-exercise filters). Options: (a) new dedicated endpoint as specified; (b) overload `/workouts/progression` with a per-exercise response variant; (c) ship both and deprecate one later. Tentative lean: (a). The personal-records page is the only caller for the per-exercise series in v1, and a separate endpoint keeps the muscle-group dashboard's response shape stable. If a second caller emerges that wants both shapes from one endpoint, overload then.
