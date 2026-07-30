---
status: draft
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-mcp
  - prog-strength-docs
---

# Hiking Activity Type

**Status**: Draft · **Last updated**: 2026-07-29

> First new activity type since the unified activity model landed, and
> therefore the first real test of the recipe in
> [`adding-an-activity-type.md`](../adding-an-activity-type.md) — whose worked
> example for a detail-backed type is, verbatim, "hiking-shaped." If this SOW
> needs to touch a switch statement outside a descriptor, the invariant that
> doc promises is broken and the contract test should have caught it.

## Introduction

**Prog Strength** can record a lift, a run, a walk, a ride, and a bucket
called "other." It cannot record a hike. For a user in Denver whose training
is substantially mountain hiking, that gap is not cosmetic: a Saturday on
Quandary Peak is six hours of load-bearing aerobic work that the app either
loses entirely or files as `other`, where it lands in no analytics surface and
renders a card that says "Activity."

The gap exists because hiking is the first sport whose *headline metric isn't
distance or pace*. A 6.7-mile hike at 22:00/mi looks, to every existing
surface, like a bad run. What makes it a hard session is 3,450 feet of
vertical gain from a 10,800-foot trailhead to a 14,265-foot summit — and the
API derives exactly one of those four numbers today (`elevation_gain_meters`),
stores none of the other three, and has no way to accept a hike in the first
place: Garmin's TCX `<Sport>` vocabulary is Running/Biking/Other, so an
uploaded hike normalizes to `ActivityOther` with no user override.

When this ships, a user uploads a hike TCX, tells the upload modal it's a
hike, and gets a Hiking tab alongside Workouts and Running: mileage, total
vertical gain, high point, low point, average pace, and average gain per mile
over the selected timeframe, above a list of hikes that each open a detail
page with a real elevation profile and route map.

## Proposed Solution

Hiking registers as a fifth endurance-shaped type — one migration, one
descriptor registration, no new detail-store code — by widening the *shared*
endurance detail shape rather than inventing a hike-specific one.

Three columns are added to the endurance detail shape:
`elevation_loss_meters`, `elevation_high_meters`, `elevation_low_meters`. They
go on all four existing `activity_*_details` tables plus a new
`activity_hike_details`, so `NewSQLiteEnduranceDetailStore` — already
table-parameterized — serves hiking unchanged, one TCX derivation serves every
endurance type, and Denver *runs* pick up loss/high/low for free.
`sqlite_repository.go` anticipates this explicitly: *"a future fifth detail
table then only needs its alias added here and in `activityJoins`."*

Ingest gets the missing seam: `POST /activities/tcx` accepts an optional
`activity_type` form field that overrides the `<Sport>`-tag normalization.
This is the general fix for a general problem — TCX's three-value sport
vocabulary can't express walking or hiking — not a hiking special case.

On the read side, **no new endpoint**. The Hiking tab already fetches
`/activities?type=hiking&since=&until=` for its list, and the unified list
projection coalesces detail columns onto every row, so all six tiles are
arithmetic over data already in the client's hands. This follows the
precedent set by `lib/activities-overview-stats.ts` and the deliberate
client-side-sum decision on the nutrition macro rings.

## Goals and Non-Goals

### Goals

- A user can upload a hike TCX and have it stored as `activity_type = "hiking"`.
- `POST /activities/tcx` accepts an optional `activity_type` override, validated
  against the registry, rejecting non-endurance types.
- TCX ingest derives elevation gain, loss, high point, and low point from
  trackpoint altitude in one pass; all four are `nil` when no trackpoint
  carries altitude.
- A Hiking tab on `/activities` shows six timeframe-scoped tiles — distance,
  total vertical gain, high point, low point, average pace, average gain per
  mile — above a timeframe-filtered hike list.
- `/hiking/[id]` renders a hike detail page on the activity session-recap
  grammar: stat tiles, elevation profile, route map, heart-rate recap, notes.
- Elevation renders in the user's unit (feet under `mi`, meters under `km`)
  through one shared `formatElevation` helper.
- `design-system.md` gains a **hike** discipline hue, registered in
  `activity-colors.ts`.
- The MCP `log_activity` / `list_activities` docstrings list hiking so the
  agent knows it exists.

### Non-Goals

- **Timeline feed publishing for hikes.** Card *rendering* comes free via
  `Summarize`; publishing needs a `timeline_post.source_type` `CHECK` widen, a
  `SourceType` mapping, and a `sessionHref` entry. See Open Questions.
- **Mobile parity.** Follow-up SOW, per the established per-phase pattern.
- **Best efforts / max-effort estimation for hiking.** Pace PRs over varied
  terrain and altitude are not comparable session to session; the machinery
  stays running-only.
- **Planned-workout reconciliation for hikes.** `matchSession` stays
  running-gated on the TCX path.
- **Overview-tab inclusion.** The Overview's headline instrument is a
  lifting-vs-running time split; making it three-way is its own design
  problem.
- **Manual (non-TCX) hike entry.** No manual endurance-log form exists in web
  today and this SOW doesn't add one. The API's `POST /activities` path
  already accepts a manual hike for the agent and for MCP.
- **Treadmill-style calibration for hikes.** GPS distance on a trail is not
  the systematic-error problem indoor running is.

## Implementation Details

### Data Model

New table `activity_hike_details`, created in migration
`044_activity_hike_elevation.sql`. Columns mirror `activity_run_details`
(`042_unified_activity_model.sql`) plus the elevation triple:

| Column | Type | Description |
| --- | --- | --- |
| `activity_id` | text | Primary key, `REFERENCES activities(id) ON DELETE CASCADE`. |
| `distance_meters` | real | `NOT NULL`. Trail distance. |
| `raw_distance_meters` | real | `NOT NULL DEFAULT 0`. Pre-calibration distance; equals `distance_meters` at ingest. |
| `avg_pace_sec_per_km` | real | Nullable. Retained for shape parity; not featured in the hike summary. |
| `best_pace_sec_per_km` | real | Nullable. Retained for shape parity; not surfaced. |
| `elevation_gain_meters` | real | Nullable. Sum of positive consecutive altitude deltas (total ascent). |
| `elevation_loss_meters` | real | Nullable. Sum of negative consecutive altitude deltas, stored positive (total descent). |
| `elevation_high_meters` | real | Nullable. Maximum trackpoint altitude — the summit. |
| `elevation_low_meters` | real | Nullable. Minimum trackpoint altitude — typically the trailhead. |
| `environment` | text | `NOT NULL DEFAULT 'outdoor' CHECK(environment IN ('outdoor','indoor'))`. |
| `route_geojson` | text | Nullable. Route geometry for the detail-page map. |

The same three elevation columns are added to `activity_run_details`,
`activity_walk_details`, `activity_cycle_details`, and
`activity_other_details` via `ALTER TABLE ... ADD COLUMN` (nullable, no
default — SQLite handles this without a table rebuild).

Two rules carried forward from 042 and honored here: the detail row is 1:1 on
`activity_id` with `ON DELETE CASCADE` (the base row owns lifecycle), and **no
`CHECK` enum is added to the base `activities` table** — the Go registry is
the type enum. The `environment` `CHECK` inside the detail table is fine:
that's a column-domain constraint on data hiking alone owns.

No new index. The detail tables are keyed by `activity_id` and reached only
through the base row's joins.

### Type Registration

1. **`internal/activity/activity_type.go`** — add
   `ActivityHiking ActivityType = "hiking"` and include it in the
   `Valid()` switch (used by the S3 key builder and any validator taking a
   type from untrusted input).

2. **`internal/activity/endurance_descriptor.go`** — `EnduranceDetails` gains
   `ElevationLossMeters`, `ElevationHighMeters`, `ElevationLowMeters`
   (`*float64`, `omitempty`). `enduranceLabel` gains a `"Hike"` case.

3. **`enduranceSummarize`** gains a hiking branch. Every other endurance type
   renders `distance · duration`; hiking renders **`distance · gain ·
   duration`** (`"6.7 mi · 3,450 ft ↑ · 5:12:40"`), because vertical gain is
   what makes a hike hard and a hike's pace chip reads as a slow run
   everywhere it appears. Gain is omitted from the chip set when
   `ElevationGainMeters` is nil, degrading to the standard two-chip card. This
   is the one place hiking deviates from the shared endurance summarizer.

4. **`internal/activity/sqlite_repository.go`** — add the `hd` alias to
   `activityJoins`, extend `detailCoalesce`'s expression to include it, add
   three `detailCoalesce` projections to `activityColumns` (fallback `""`,
   they're nullable), extend the scan in the same order, and add
   `ActivityHiking → "activity_hike_details"` to `detailTable()`.

5. **`internal/server/server.go`** — register
   `activity.NewEnduranceDescriptor(activity.ActivityHiking, activity.NewSQLiteEnduranceDetailStore(db, "hike"))`
   alongside the four existing endurance registrations. `NewRegistry` panics
   at boot on a duplicate `Type`, so a wiring mistake fails the deploy loudly.

`MountRoutes` stays nil for hiking — no hike-specific routes exist. No
`DetailStore`, `DecodeDetails`, or `ValidateCreate` is written: hiking inherits
all three from `NewEnduranceDescriptor`, including the "distance-first sports
require a positive `distance_meters`" rule, which is correct for a hike.

### Write Path

- **TCX upload with an explicit type** — `POST /activities/tcx` reads an
  optional `activity_type` multipart form field. When present and non-empty it
  is looked up in the registry and overrides `normalizeActivityType`'s
  `<Sport>`-derived value. Unknown type → **422** listing valid types
  (consistent with `POST /activities`). A registered but non-endurance type
  (today: `strength_training`) → **400**, code `invalid_activity_type`:
  strength ingest has its own path (`ingest_strength.go`) and its own
  summarizer, and routing a strength TCX through the endurance summarizer
  would write a garbage detail row. When the field is absent, behavior is
  byte-for-byte what it is today.
- **Elevation derivation** — the summarizer computes the four elevation
  numbers and writes them to the detail row in the same transaction as the
  base row and trackpoints. No separate write.
- **Duplicate upload** — unchanged. `ErrDuplicate` returns the existing live
  row and publishes nothing.
- **Delete** — unchanged. `ON DELETE CASCADE` on `activity_id` disposes of the
  hike detail row with the base row.
- **No timeline publish, no plan match** — the `uploadTCX` success branch
  keeps both gated on `a.ActivityType == ActivityRunning`. A hike is stored
  and listed but does not publish a feed post (see Non-Goals).

### Algorithms

**Elevation derivation (server, at ingest).** `elevationGain` in
`tcx_summarizer.go` becomes `elevationProfile`, returning all four numbers in
one pass over trackpoints so a single traversal answers gain, loss, high, and
low:

```
gain = Σ max(0, alt[i] - alt[i-1])   over consecutive points with altitude
loss = Σ max(0, alt[i-1] - alt[i])   over the same pairs, stored positive
high = max(alt[i])
low  = min(alt[i])
```

Points with no altitude are skipped rather than treated as zero, and the
previous-altitude cursor only advances on a point that has one — so a gap in
the altitude stream doesn't manufacture a cliff. All four return `nil` when
*no* trackpoint carried altitude, which is distinct from a genuinely flat
route returning `0`.

**Tile aggregation (client, over the windowed list).** All six tiles are
computed in a new pure `lib/hiking-stats.ts`, mirroring
`lib/activities-overview-stats.ts`:

```
totalDistance = Σ distance_meters
totalGain     = Σ elevation_gain_meters        (nulls skipped)
highPoint     = max(elevation_high_meters)     (nulls skipped)
lowPoint      = min(elevation_low_meters)      (nulls skipped)
avgPace       = Σ duration_seconds / (Σ distance_meters / 1000)
gainPerMile   = totalGain / (totalDistance / METERS_PER_MILE)
```

Two deliberate choices:

- **Average pace is total time over total distance, not the mean of per-hike
  paces.** A weighted aggregate reconciles with the distance and duration
  tiles beside it; a mean of means does not. This is the same policy
  `computeMetrics` uses for running's `RecentAvgPaceSecPerKm`, and the reason
  is the lesson of
  [`running-detail-metric-alignment.md`](running-detail-metric-alignment.md):
  surfaces that compute the same number under two policies disagree, and users
  do the arithmetic.
- **Null elevation is skipped, never coerced to zero.** Coercing nulls to 0
  makes `min()` return a 0-meter low point, which for a Denver hiker is
  visibly absurd but for a sea-level user is *plausible* — silent wrong data.
  When the window contains no hike with altitude, the high/low/gain tiles
  render an em-dash. Pre-migration rows have nulls in all three columns, so
  this is the normal case until hikes accumulate.

### API Surface

No new endpoints. Two changed contracts:

- **`POST /activities/tcx`** — accepts an optional `activity_type` multipart
  form field. Absent → today's behavior. Unknown → 422 with the valid set.
  Registered but non-endurance → 400 `invalid_activity_type`.
- **Every activity read response** (`GET /activities`, `GET /activities/{id}`,
  and the DTOs in `handler.go`) — the endurance detail block gains
  `elevation_loss_meters`, `elevation_high_meters`, `elevation_low_meters`.
  Additive and nullable; existing clients ignore them. Per the DTO convention
  in `handler.go`, the keys stay present and render `null` when absent rather
  than being omitted.

`GET /activities?type=hiking&since=&until=` needs no server change — the
registry-backed type filter already accepts any registered type.

### Web Surface

**Shell (`app/(app)/activities/page.tsx`).** The `View` union gains
`"hiking"`; the `?view=` parse, `setView`, and the toolbar gain a matching
entry with a mountain icon (a simple two-peak glyph in the existing
16px/1.75-stroke inline-SVG house style). The Upload TCX toolbar button's
condition widens from `view === "running"` to `view === "running" || view ===
"hiking"`, and the shell passes the active view down so the modal can
preselect the right sport.

**`components/activities/hiking-view.tsx`.** Follows `RunningView`'s
structure — `refetch` deriving `[since, until)` from `days`, the same 401
handling, the same error surface — with two simplifications: one fetch instead
of two (no metrics endpoint), and no analytics card. Renders a
`grid-cols-2 md:grid-cols-3` of six `StatTile`s over a `HikeHistoryList`.
Tile labels: DISTANCE, VERTICAL GAIN, HIGH POINT, LOW POINT, AVG PACE, GAIN /
MI (the last two labels swap to `/KM` and `GAIN / KM` under the `km` unit).

**Elevation *loss* is captured but gets no aggregate tile.** Hiking is
overwhelmingly out-and-back or loop, so window-level loss tracks window-level
gain closely enough that a seventh tile would carry almost no information while
diluting the six that do. Loss is stored, returned by the API, and shown on the
detail page, where a single hike's gain-vs-loss asymmetry is genuinely
interesting (a point-to-point descent, a shuttle hike). If a loss tile turns
out to be wanted at the window level, it's a one-line addition to
`hiking-stats.ts` and one more `StatTile`.

**`UploadTCXModal`.** Gains a sport selector — a small pill row (Run / Hike /
Walk / Ride) rather than a `<select>`, matching the timeframe pills already in
the header — defaulting to the tab it was opened from. The chosen value is
sent as the `activity_type` form field. When the user picks nothing and the
default applies, the field is still sent explicitly, so the resulting type is
never a surprise derived from a `<Sport>` tag the user can't see.

**`/hiking/[id]`.** New detail page on the activity session-recap grammar
(`design-system.md` § Activity session-recap), reusing `ElevationChart`,
`RunRouteMap`, `HeartRateRecap`, `SectionKicker`, and `NotesEditor`. Header
stat tiles lead with distance, vertical gain, and duration. Deliberately
absent: `SplitsSpine`, `PaceRecap`, `CalibrateDistanceModal`. The elevation
profile is the page's centerpiece rather than a supporting chart, since it's
the shape of the hike.

Reused components move from `app/(app)/running/[id]/_components/` and
`app/(app)/running/_components/` to a shared location as they're picked up —
`components/activity-detail/` — rather than being imported across route
groups. `RunningView` already reaches sideways into
`../../app/(app)/running/_components/`; a second consumer is the point at
which that should stop rather than double.

**`lib/api.ts`.** The `ActivityType` union gains `"hiking"`. New
`listHikingSessions` wrapping `listActivities(token, {...opts, type: "hiking"})`,
mirroring `listRunningSessions`. The activity types gain the three elevation
fields.

The TCX upload helper is `importRunningTcx(token, file)` returning a
`RunningSession` — a name from before the unified model that is now wrong at
both ends, since the endpoint ingests any endurance type and returns a unified
activity. It gains a third optional `activityType` argument and is renamed
`importActivityTcx`, with call sites updated (`UploadTCXModal` is the only
one). This is a rename of a function this SOW is already changing the
signature of, not a broader `RunningSession`-to-`Activity` type migration —
that alias stays as-is.

**`lib/distance-unit-context.tsx`.** New `formatElevationValue(meters, unit)`
pure helper plus a `formatElevation` binding on the context: whole feet with a
thousands separator under `mi`, whole meters under `km`, em-dash for null.

**Targeted cleanup.** Elevation formatting is currently open-coded in three
places with three behaviors: `components/calendar/run-digest.tsx:38` and
`components/activities/overview/instruments.tsx:342` each reimplement the
`FEET_PER_METER` conversion inline, while `app/(app)/running/[id]/page.tsx:349`
hardcodes `"m"` regardless of the user's unit setting — so the same run reports
its elevation in meters on its detail page and in feet on the calendar. This
work needs the shared helper regardless; those three sites adopt it. Nothing
else is refactored.

### Design System

`design-system.md` bumps to **v0.4.4**, adding a **hike** row to the Activity
tonal hues table and registering it in `lib/activity-colors.ts`. The
`Discipline` union in `components/calendar/derivations.ts` gains `"hike"`, and
`MappedDiscipline` in `activity-colors.ts` widens to `"run" | "lift" | "hike"`
with matching `ACTIVITY_COLORS` and `ACTIVITY_RING` entries.

Hiking cannot reuse run's green-teal or lift's steel-blue, and per the
standing "activity ≠ selection" rule it must stay clearly distinct from the
periwinkle accent. The proposal is a **desaturated clay** — earthy, which
reads right for terrain, and separated from the pinker `--danger` (`#c79292`)
by being warmer and browner:

| Discipline | `--discipline-hike-bg` | `--discipline-hike-fg` | `--discipline-hike-dot` |
| --- | --- | --- | --- |
| **hike** (clay) | `#2a201c` | `#c9a690` | `#b08e77` |

These are pitched to the **actual** values in `app/globals.css`
(run `#16241f`/`#9cc7b8`/`#7fae9e`), which are softer and more desaturated than
the table currently printed in `design-system.md`. That drift is real and is
picked up as an Open Question below.

### Backfill or Migration

1. **Mechanism.** Schema-only. `044_activity_hike_elevation.sql` creates one
   table and adds three nullable columns to four others. There is no backfill:
   `elevation_loss/high/low` are derivable only from trackpoint altitude, and
   while trackpoints *are* retained, recomputing history is not worth a
   migration script for a solo pre-launch dataset. Existing rows keep `NULL`,
   which the null-skipping aggregation already handles correctly, and any
   re-uploaded activity picks up the full profile.
2. **Recoverability.** The migration is additive and idempotent-by-numbering;
   nothing is dropped or rewritten, so a failure leaves prior state intact.
   Should a backfill ever be wanted, it's a truncate-and-recompute over
   `activity_trackpoints` per activity — safe to rerun because the derivation
   is pure.
3. **Scale boundary.** `ALTER TABLE ADD COLUMN` on nullable columns with no
   default is O(1) in SQLite regardless of row count. A later backfill would
   be O(trackpoints) and would need batching somewhere north of a few million
   points — orders of magnitude beyond current data.

### Verification

- **The contract test is the gate.** `internal/activity/contract_test.go`
  registers `shadowboxing`, a type unknown to every package outside that file,
  and drives it through the production handlers. If registering hiking
  requires editing a switch outside the descriptor and the repository's
  declared join/coalesce/detail-table seams, that test tells us the "new types
  come free" invariant broke. It must keep passing untouched.
- **New Go tests.** `elevationProfile` over: a normal ascent/descent, a route
  with no altitude at all (all four nil), a route with an altitude gap (no
  phantom cliff), and a flat route (zeros, not nils). The `activity_type`
  override on `POST /activities/tcx`: absent, valid endurance, unknown (422
  listing valid types), and `strength_training` (400 `invalid_activity_type`).
  A hiking round-trip through create/list/get/delete asserting the elevation
  triple survives, and an `enduranceSummarize` hiking case with and without
  gain.
- **New web tests.** `lib/hiking-stats.ts` unit tests: the weighted-pace
  formula, null-skipping for each of the three elevation aggregates, an
  all-null window rendering em-dashes, and an empty window. A `HikingView`
  render test mirroring `running-view.test.tsx`.
- **Manual check.** Upload a real Denver hike TCX, confirm the summit altitude
  on the high-point tile matches the watch, and confirm the same hike reports
  the same elevation on the tab, the detail page, and the calendar digest.

## Open Questions

1. **Should hikes publish timeline posts?** Options: (a) leave hiking
   feed-silent, as this SOW does; (b) a follow-up SOW widening
   `timeline_post.source_type`'s `CHECK` (migration `020_timeline.sql`), adding
   a `SourceType` mapping, and adding a `sessionHref` entry for `/hiking/[id]`.
   **Lean: (b), as a follow-up.** A 14er summit is the most feed-worthy event
   in the app, but publishing is a genuinely separate concern from rendering —
   the three-step cost is exactly what `adding-an-activity-type.md` warns not
   to conflate with "the card renders."

2. **Does `/hiking/[id]` carry heart-rate zones?** Options: (a) HR recap only,
   as scoped; (b) add `HeartRateZones` too. **Lean: (b).** The `hrzones`
   engine is already type-agnostic and consumes only trackpoint HR, so this is
   a component import rather than new machinery, and time-in-zone is arguably
   *more* informative on a long steady climb than on a run. Scoped out only to
   keep the first pass narrow — cheap to fold in if you want it now.

3. **The clay hex triple.** The values above are a proposal, not a decided
   token set, and a discipline hue is a design-system amendment. **Lean: pick
   the final triple by eye against a real hike card on the dark field before
   the `design-system.md` edit is committed**, since the surrounding neutrals
   are near-black and warm hues shift a lot against them.

4. **`design-system.md`'s hue table has drifted from `globals.css`.** The doc
   prints run as `#16302a`/`#82d3b8`/`#46b893`; the stylesheet says
   `#16241f`/`#9cc7b8`/`#7fae9e`. The doc also asserts activity hues stay
   distinct from the accent, yet `--discipline-lift-dot` is `#9aa6d6` — the
   accent value exactly. Options: (a) add the hike row and leave the drift;
   (b) correct the run/lift rows to match the stylesheet in the same v0.4.4
   edit and either fix or document the lift/accent collision. **Lean: (b) for
   the run/lift hex correction** — a decided-conventions doc that disagrees
   with the code is worse than no doc, and this SOW is already editing that
   table. The lift-dot/accent collision is a real question about whether the
   v0.3 rule still holds and deserves its own answer rather than a quiet
   correction here.
