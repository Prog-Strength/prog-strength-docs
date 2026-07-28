# Adding an Activity Type

**Status**: Living reference · **Last updated**: 2026-07-27

> This is the recipe [`sows/unified-activity-model.md`](sows/unified-activity-model.md)
> promised: after that SOW, adding a training type is supposed to be a
> descriptor plus (optionally) one migration — not a tour through every
> aggregate surface. This doc is that recipe, and its worked example
> (shadowboxing) is a real test in `prog-strength-api`, so it can't drift
> silently out of date the way a plain narrative doc can.

## Overview

Prog Strength stores every training session — a lift, a run, a hike, a
kickboxing class — as one row in the `activities` base table
(`internal/db/migrations/042_unified_activity_model.sql`). The base table
carries only universal columns (type, start time, duration, name, notes,
heart rate, calories, ingest provenance); `activity_type` has **no SQL
`CHECK`** — SQLite can't widen a `CHECK` without rebuilding the table, so the
Go type registry is the single source of truth for which types exist.

A type-specific **detail table** (`activity_<type>_details`) holds whatever
doesn't fit the base row — distance and pace for a run, nothing at all for a
type like kickboxing.

In Go, `internal/activity/registry.go` is the registry: each type registers a
**`Descriptor`** (validation, an optional detail store, a card renderer,
optional custom create/update/routes). `internal/server/server.go` builds the
one production `Registry` at boot. There is no other place a type "plugs in" —
every surface below (list, timeline, dashboard, snapshot) reads the unified
base and the registry, not a per-type switch statement.

Two recipes follow: a **base-only type** (kickboxing-shaped — nothing but
duration and vitals) and a **detail-backed type** (hiking-shaped — needs its
own structured metrics). Do the base-only recipe first even for a
detail-backed type; the detail table and store are additive on top of it.

## Recipe: base-only type (kickboxing-shaped)

A base-only type has no structured metrics of its own — its whole state fits
the `activities` row (start time, duration, name, notes, heart rate, calories).

### 1. Add the `ActivityType` constant

In `internal/activity/activity_type.go`:

```go
const (
	ActivityRunning ActivityType = "running"
	ActivityWalking ActivityType = "walking"
	ActivityCycling ActivityType = "cycling"
	ActivityOther   ActivityType = "other"
	ActivityStrengthTraining ActivityType = "strength_training"
	ActivityKickboxing ActivityType = "kickboxing" // new
)
```

Add it to `ActivityType.Valid()`'s switch too — that method is used by the S3
key builder and other validators that take a type from untrusted input.

### 2. Implement a `Descriptor`

A `Descriptor` (`internal/activity/registry.go`) is a plain struct, not an
interface — build one with a constructor function, following
`NewEnduranceDescriptor` in `internal/activity/endurance_descriptor.go` as the
shape to imitate:

- **`Details: nil`** — no detail store; the unified handler persists nothing
  but the base row (`Repository.CreateManual` / `Repository.UpdateBase`).
- **`ValidateCreate`** — the type's own create/update rules. Return a plain
  `error` for a type-specific rule (it becomes a 400 with your message). Use
  the package sentinels when your rule matches their meaning, since the
  handler maps them to stable error codes API consumers can match on:
  - `activity.ErrInvalidDetails` → 400, code `invalid_details` (structurally
    bad `Details` blob: malformed JSON, unknown field, out-of-domain value).
  - `activity.ErrDistanceRequired` → 400, code `distance_required`
    (distance-first sports created without a positive `distance_meters`).
  - An unregistered type never reaches your descriptor at all — the handler
    catches it earlier and returns **422** listing the valid types
    (`Registry.Lookup` → `ErrUnknownActivityType`).
- **`Summarize` — REQUIRED, not optional in practice.** `Summarize` is `nil`
  by default, and a `nil` `Summarize` means `RenderSummary`
  (`internal/activity/summary.go`) returns `ok=false` for every instance of
  your type. Both the timeline hydrator and the unified `/activities` list
  render cards through `RenderSummary`/`RenderSummaries` — skip `Summarize`
  and your type's sessions **silently have no card anywhere the app renders
  one**. There's no error, no log line; the item just has a blank/absent
  summary. Always implement it.

```go
func NewKickboxingDescriptor() *activity.Descriptor {
	return &activity.Descriptor{
		Type: activity.ActivityKickboxing,
		ValidateCreate: func(req activity.CreateRequest) error {
			if req.DurationSeconds == nil || *req.DurationSeconds <= 0 {
				return fmt.Errorf("kickboxing needs a positive duration_seconds")
			}
			return nil
		},
		Summarize: func(a activity.Activity, _ any) activity.Summary {
			title := "Kickboxing"
			if a.Name != nil && strings.TrimSpace(*a.Name) != "" {
				title = *a.Name
			}
			dur := activity.FormatDuration(float64(a.DurationSeconds))
			return activity.Summary{Title: title, Subtitle: dur, Metrics: []string{dur}}
		},
	}
}
```

### 3. Register it in `internal/server/server.go`

Add your descriptor to the `activity.NewRegistry(...)` call alongside the
existing endurance and strength descriptors (around the `activityRegistry :=`
block). `NewRegistry` **panics at boot on a duplicate `Type`**
(`internal/activity/registry.go`, `NewRegistry`) — this is deliberate: a
wiring bug (two descriptors for the same type) fails the deploy loudly
instead of one descriptor silently shadowing another at runtime.

### Done

That's the entire per-type surface for a base-only type. See "What comes free"
below for exactly what this buys, verified by a real contract test.

## Recipe: detail-backed type (hiking-shaped)

Everything in the base-only recipe above, **plus**:

### 4. One migration: `activity_<type>_details`

Add a new numbered migration under `internal/db/migrations/`. Use
`activity_run_details` in `042_unified_activity_model.sql` as the literal
template:

```sql
CREATE TABLE activity_hike_details (
    activity_id TEXT PRIMARY KEY REFERENCES activities(id) ON DELETE CASCADE,
    distance_meters REAL NOT NULL,
    elevation_gain_meters REAL,
    ...  -- whatever hiking needs that the base row doesn't carry
);
```

Two rules carried forward from 042, non-negotiable for a new detail table:

- **1:1 on `activity_id`, `ON DELETE CASCADE`.** The base row owns lifecycle;
  the detail row is disposable.
- **No `CHECK` enums on the *base* table for your new type.** The registry is
  the enum. (A `CHECK` *inside* your own detail table, like
  `activity_run_details.environment`, is fine — that's a column-domain
  constraint on data your type alone owns, not a type-taxonomy gate.)

### 5. A `DetailStore`

`DetailStore` (`internal/activity/registry.go`) is three methods:
`Load`/`Save`/`Delete`, all keyed by `(userID, activityID)`, all folding
missing/soft-deleted/wrong-owner into the single `activity.ErrNotFound`
sentinel — implementations must translate their own not-found at that seam.

- **If your shape matches the endurance types** (a single flat detail row,
  no bulk-load fan-out needed beyond what the base list already joins in),
  reuse `NewSQLiteEnduranceDetailStore` (`internal/activity/endurance_detail_store.go`)
  — it's already table-parameterized (`sqliteEnduranceDetailStore{db, table}`),
  so a new endurance-shaped type is just another `table` value, not new code.
- **Otherwise implement `DetailStore` directly** against your own table.
  Optionally implement `BulkDetailLoader` (`LoadMany(ctx, userID, ids)
  (map[string]any, error)`) too — the unified list's `RenderSummaries`
  (`internal/activity/summary.go`) uses it to batch-load details for a whole
  page in one round trip instead of one query per row. Skipping it isn't
  wrong — your cards still render from the base row alone — but it's a perf
  cliff for a type with a meaningful per-row query.

### 6. `DecodeDetails`

`DecodeDetails func(raw json.RawMessage) (any, error)` parses the incoming
`Details` blob (already past your `ValidateCreate`) into the typed value your
`DetailStore.Save` and `Summarize` expect. Return `nil, nil` for an absent or
JSON-`null` blob — see `decodeEnduranceDetails` in
`internal/activity/endurance_descriptor.go` for the pattern, including the
"typed nil inside `any` reads as non-nil" gotcha it comments on. Only the
generic create/update path calls this; a type with custom `Create`/`Update`
(see below) decodes inline instead.

### The `NewEnduranceDescriptor` shortcut

If your new type is genuinely endurance-shaped (distance + pace + elevation +
route, same fields as run/walk/cycle/other), skip steps 5 and part of 2/6
entirely: call `activity.NewEnduranceDescriptor(yourType, activity.NewSQLiteEnduranceDetailStore(db, yourType))`
in `server.go` directly, same as the four existing endurance registrations.
You still need the migration (step 4) creating your `activity_<type>_details`
table with the exact endurance column set, but you write zero new descriptor
code — hiking that's just "a run with a different label" is one migration and
one registration line.

## What comes free

Everything below is asserted by a real, runnable test —
`internal/activity/contract_test.go` — that registers a fake base-only type
(`shadowboxing`) known to **nothing** in `internal/activity`,
`internal/dashboard`, or `internal/server` outside that one test file, then
drives it through the production handlers. If a future change makes any of
these require touching a switch statement outside your descriptor, that test
fails — it's the alarm, not just documentation.

- **The unified surface**: `POST /activities`, `GET /activities`,
  `GET /activities/{id}`, `PUT /activities/{id}`, `DELETE /activities/{id}`,
  and `GET /activities?type=<yours>` filtering — all through the exact routes
  `internal/server/server.go` mounts, with zero handler code aware your type
  exists. (`TestContract_BaseOnlyType_UnifiedSurface`)
- **Rendered summaries** on every one of those responses via your
  `Summarize`. (Same test.)
- **Timeline card rendering** — `RenderSummary`/`RenderSummaries`
  (`internal/activity/summary.go`) are exactly what the timeline hydrator
  calls to build post cards, and what the unified list calls to build list
  rows. One card renderer, two callers.
  (`TestContract_BaseOnlyType_RendersTimelineCard`)
- **Training snapshot active-days**: `internal/snapshot/aggregate.go`'s
  `sessionDates` counts a day with *any* logged session as active, no
  per-type switch. (`TestContract_BaseOnlyType_CountsAsSnapshotActiveDay`)
- **Dashboard streak days**: the dashboard's `streakDates` lights a local day
  for any non-strength session the same way, no per-type switch.
  (`TestContract_BaseOnlyType_LightsDashboardStreak`)
- **MCP `log_activity` (forthcoming)**: once stage 2 of the unified-activity-model
  rollout lands generic `log_activity`/`list_activities` MCP tools, those
  tools drive the same registry-backed `/activities` surface — a new type
  picked up there needs no MCP-side change either. Not shipped yet as of this
  writing; noted here so this doc doesn't need a rewrite when it lands.

## What does NOT come free

Be honest with yourself about this list before promising a type is "done."

- **Timeline post *publishing* — as opposed to card *rendering*.**
  `timeline_post.source_type` carries a SQL `CHECK` in
  `internal/db/migrations/020_timeline.sql`:
  `CHECK(source_type IN ('workout', 'run', 'pr', 'best_effort'))`. Your
  descriptor's `Summarize` makes a card render *if* a post exists, but
  nothing about registering a descriptor creates a `timeline_post` row for
  your type. Publishing feed posts for a new type additionally needs:
  1. A migration widening that `CHECK` to include your `source_type`.
  2. A mapping from your `ActivityType` to a `timeline.SourceType`
     (`internal/timeline/source_type.go`) wherever posts get created for your
     type (a publish hook analogous to the workout/run ones).
  3. A `sessionHref` entry in `internal/server/timeline_hydrator.go` so a
     published post knows what web URL to link to.

  Card rendering and post publishing are different concerns with different
  costs — don't conflate "the contract test's card renders" with "this type
  posts to the feed."

- **Type-specific analytics.** Tiles, dedicated metrics endpoints, PR-like
  surfaces (best-effort tracking, 1RM history, headline exercises) are
  bespoke per type today — running's `/activities/running-metrics` and
  best-effort machinery, strength's PR detection, are hand-built, not
  registry-derived. A new type gets none of that by registering a descriptor.

- **Snapshot-vs-dashboard completion nuance — pick one deliberately if your
  type has a draft/in-progress state.** The dashboard's streak gates
  `strength_training` on completion (`workoutCompleted`) — an in-progress
  lift doesn't light the streak day. The training snapshot's active-days
  count does not carry that nuance for any type: from
  `internal/snapshot/aggregate.go`'s `sessionDates`, "any logged session
  counts, whether finished or not — a type with an in-progress state should
  decide whether it wants the dashboard's completion gate or the snapshot's
  any-row count, and implement that choice explicitly rather than inherit
  today's asymmetry by accident." If your type can exist half-finished
  (e.g. a live-logged session saved before it's marked done), decide which
  contract you want and wire it — don't assume registering a descriptor
  answers this for you.

- **Web/mobile card variants.** A generic card renders from `Summary` (title,
  subtitle, metric chips) with no client change — that's what the contract
  test proves. A bespoke visual treatment (a distinct icon, a type-specific
  detail page, a custom color per the design system's "activity tonal hues")
  is per-type frontend work, same as it's always been.

## Worked example: shadowboxing

`internal/activity/contract_test.go` registers a type that doesn't exist in
production — `shadowboxing` — purely to prove the "new types come free"
claim. The test doubles as the pattern's regression guard: if this test ever
needs a code change outside the descriptor below to keep passing, the
invariant broke.

```go
// shadowboxingType is the fake base-only type. Not an ActivityType constant
// anywhere in production code — that's the point.
const shadowboxingType = activity.ActivityType("shadowboxing")

// newShadowboxingDescriptor is the ENTIRE per-type surface a base-only type
// implements: no DetailStore, no migration, no handler or switch edits.
// ValidateCreate requires a positive duration; Summarize renders a
// name-or-default title with a duration chip — the same card the timeline
// hydrator and the unified list share via RenderSummaries.
func newShadowboxingDescriptor() *activity.Descriptor {
	return &activity.Descriptor{
		Type: shadowboxingType,
		ValidateCreate: func(req activity.CreateRequest) error {
			if req.DurationSeconds == nil || *req.DurationSeconds <= 0 {
				return fmt.Errorf("shadowboxing needs a positive duration_seconds")
			}
			return nil
		},
		Summarize: func(a activity.Activity, _ any) activity.Summary {
			title := "Shadowboxing"
			if a.Name != nil && strings.TrimSpace(*a.Name) != "" {
				title = *a.Name
			}
			dur := activity.FormatDuration(float64(a.DurationSeconds))
			return activity.Summary{Title: title, Subtitle: dur, Metrics: []string{dur}}
		},
	}
}
```

The test registers this descriptor into a `Registry` that otherwise mirrors
production (`newContractRegistry`: the four real endurance descriptors, real
detail stores, plus `newShadowboxingDescriptor()`), wires the real activity
and dashboard handlers over one SQLite database behind a fake auth
middleware, and drives HTTP requests at it — `POST /activities`
(duration-less → 400 citing `duration_seconds`; a genuinely unregistered
`"kickboxing"` in the same test → 422 listing `shadowboxing` among the valid
types), list, `?type=shadowboxing` filter, `GET /activities/{id}` (asserting
the response has **no** `details` key for a base-only type), `PUT` rename,
and soft `DELETE`. Separate tests in the same file drive the timeline-card
and snapshot/dashboard paths against the same registry. Read the full file
for the exact assertions — this doc quotes only the descriptor.

## Reference: the shim period

`prog-strength-api` is mid-migration off a legacy dual-domain model. Two
things worth knowing before you touch this code:

- **`/workouts/*` is a deprecated compatibility shim**, kept only so web,
  mobile, and MCP can migrate to `/activities` without a mid-flight break.
  `internal/server/server.go` marks the mount point
  `// Deprecated: stage-5 cleanup removes these /workouts shims once MCP,
  web, and mobile are on /activities`. Don't build new functionality against
  `/workouts`; it goes away in the unified-activity-model rollout's stage 5.
- **`completed_session_kind` (on `planned_workout`) is being collapsed in
  stage 5.** Until then, lookups by completed session are deliberately
  **kind-agnostic** — `planned_workout.Repository.GetByCompletedSession`
  matches on `completed_session_id` alone, ignoring which of `'workout'` /
  `'activity'` the row says, because both wrappers ultimately point at the
  same unified `activities` row. If you're touching planned-workout
  reconciliation for a new type, match that kind-agnostic pattern rather than
  adding a new `completed_session_kind` value.
