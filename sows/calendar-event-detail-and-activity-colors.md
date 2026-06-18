---
type: sow
status: shipped
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Calendar Event Detail & Activity-Type Colors

**Status**: Shipped · **Last updated**: 2026-06-18

## Introduction

The `/calendar` page (the shipped true-month-grid redesign) is functional but
two things make it feel less cohesive than the rest of the app:

1. **Clicking a planned event doesn't reliably surface the event.** Today a
   month-grid pill click *selects the day and expands a row in the timeline
   below* (`onSelectPlanned`); the read-only detail modal only opens from the
   timeline banner. The user's expectation — click the event, see its details,
   edit if you want — isn't the direct path.
2. **An activity's color isn't consistent between the grid and the timeline.**
   The month grid colors a pill by **activity type** (run = teal, lift = blue,
   via the `--discipline-run-*` / `--discipline-lift-*` CSS tokens), but the
   daily timeline's left bar is colored by **status** (planned = violet accent,
   completed = a hardcoded emerald, skipped = muted gray). So "W7 D4 – Long Run"
   shows a run-tinted pill on the grid but a **violet** bar in the timeline — the
   same event, two different colors. There is no central activity-type → color
   mapping; the color logic is scattered inline across `day-cell.tsx` and the
   four banner components, with `emerald-*` hardcoded for the completed state.

This SOW makes events **click-to-detail** and makes **activity type own the
color** everywhere, with the color logic centralized behind one abstraction so a
future "let the user pick their calendar colors" feature is a change in one
place. The read-only→edit surface **already exists** — `PlannedWorkoutModal` has
a view mode with a pencil → edit transition — so this is mostly a *routing and
theming* refactor, not new modal construction.

This is a **web-only** change (plus docs bookkeeping). **No backend change** —
the planned-workout view/edit API (`createPlannedWorkout`/`updatePlannedWorkout`,
`GET /planned-workouts`) and the logged-session detail pages are untouched. It
conforms to [`../design-system.md`](../design-system.md): dark slate, the violet
accent reserved for app chrome/selection (no longer impersonating a status), and
the existing `--discipline-*` activity tokens.

## Proposed Solution

Two coordinated changes against the existing calendar components.

### 1. The activity-color abstraction (the cohesion fix)

A new `lib/activity-colors.ts` becomes the **single source of truth** mapping an
activity type to its theme color tokens — modeled on the existing
`lib/macro-colors.ts` precedent:

```ts
// activity type → CSS-variable token set. The indirection is deliberate:
// today this is a static map onto the --discipline-* tokens; a future
// user-customization feature swaps the resolver's source without touching
// any call site.
export type ActivityType = "lift" | "run"; // extensible: "mobility" | "core" reserved in globals.css

export const ACTIVITY_COLORS: Record<ActivityType, { dot: string; bg: string; fg: string }> = {
  run:  { dot: "var(--discipline-run-dot)",  bg: "var(--discipline-run-bg)",  fg: "var(--discipline-run-fg)" },
  lift: { dot: "var(--discipline-lift-dot)", bg: "var(--discipline-lift-bg)", fg: "var(--discipline-lift-fg)" },
};

export function activityColors(type: ActivityType) { return ACTIVITY_COLORS[type]; }
```

Every place that currently picks a color inline routes through `activityColors(type)`:

- **Month-grid pills** (`day-cell.tsx`): `doneToneClasses`, `PlannedPill`, and
  `CompletedPlannedPill` resolve their bg/fg/border from `activityColors(type)`
  instead of inline `var(--discipline-*)` strings.
- **Timeline banners** (`workout-banner.tsx`, `run-banner.tsx`,
  `planned-banner.tsx`, `completed-planned-banner.tsx`): the left accent bar and
  focus ring resolve from `activityColors(type).dot`.

The type for an event is derived through the existing `disciplineOf(event)`
helper (`components/calendar/derivations.ts`) — which already maps any
`CalendarEvent` (logged workout, logged run, `PlannedWorkout.activity_kind`,
completed-planned) to `"lift" | "run"`. That stays the one place event → type is
decided; `activity-colors.ts` is the one place type → color is decided.

**Status stops stealing the color slot.** The colored bar/accent is **always the
activity-type color**. Planned / completed / skipped are conveyed by **shape and
badge**, the cues the grid pills already use:

- **Planned** — dashed outline + `PLANNED` badge (+ the sync icon when
  `google_sync_status` is set), as today.
- **Completed** — solid + a check + `COMPLETED` badge. The **hardcoded
  `emerald-*` is retired** — a completed planned event now reads in its
  activity-type color with a completion check, not green.
- **Skipped** — muted/`var(--muted)` + strikethrough + `SKIPPED` badge.

This is what makes the timeline match the grid: the planned long-run's bar
becomes the **run** color in both places, and the violet accent goes back to
meaning selection/today, not "planned."

### 2. Click-to-detail (the UX fix)

A pill on the grid is **exactly one event**, so a pill click is unambiguous:

- **Planned pill click** → open `PlannedWorkoutModal` in **read-only view** for
  that plan (and select its day). The user reads the details; the in-modal pencil
  enters edit mode (existing behavior; pencil already hidden for
  completed/skipped, which stay view-only).
- **Logged pill click** (workout/run) → navigate to that session's detail page
  (`/workouts/{id}` / `/running/{id}`) — logged sessions keep their richer pages.
- **`+N more` / empty cell background** → select the day and scroll the daily
  **timeline** into view, which lists every event for the day as its own
  clickable row. Each timeline row opens the same target as its pill (planned →
  modal, logged → detail page). This is how multi-event days resolve without any
  "which event did you mean?" disambiguation.

The timeline's `PlannedBanner` already opens the read-only modal on body click —
that stays; this change makes the **grid pill** open it directly too, so both
entry points behave identically.

## Goals and Non-Goals

### Goals

- **Introduce `lib/activity-colors.ts`** as the single activity-type → color
  abstraction (static map onto `--discipline-*` today; structured so a future
  user-preference source swaps in behind `activityColors()` with no call-site
  changes).
- **Route every calendar color through it** — month-grid pills (`day-cell.tsx`)
  and all four timeline banners — removing the scattered inline `var(--discipline-*)`
  references and the hardcoded `emerald-*` completed-state colors.
- **Make activity type own the color on the timeline**, matching the grid: the
  banner left bar/accent uses `activityColors(disciplineOf(event)).dot`, not the
  status color. The planned-event bar is the run/lift color, not violet/emerald.
- **Convey status via shape + badge** (planned = dashed + `PLANNED`, completed =
  solid + check + `COMPLETED`, skipped = muted + strikethrough + `SKIPPED`),
  consistently between grid pills and timeline rows.
- **Planned pills open the read-only detail modal directly** (view mode, pencil →
  edit), selecting the day as a side effect. Reuses `PlannedWorkoutModal`; no new
  modal.
- **Overflow (`+N more`) and empty-cell clicks** select the day and scroll the
  timeline into view (the per-day event list), so multi-event days are fully
  reachable.
- **Preserve logged-session behavior**: workout/run pills and timeline rows
  navigate to `/workouts/{id}` / `/running/{id}` (the detail pages stay the edit
  surface for logged sessions).
- **Keep the violet accent semantic**: selection / today / focus only — it no
  longer doubles as the "planned" status color.
- **Intentional at every state**: a multi-event day (June 18's two planned
  events), planned vs completed vs skipped, run vs lift color parity between grid
  and timeline, and a completed-planned (merged plan + logged) row.
- Keep web CI green (lint/format/typecheck/test/build) with tests for the color
  resolver and the new click routing.

### Non-Goals

- **Any backend change.** `prog-strength-api` is not in `repos:`. The
  planned-workout view/edit API and logged-session endpoints are unchanged.
- **Redesigning the month grid or the `PlannedWorkoutModal` form.** The
  true-month-grid layout, the stat tiles, the week-rollup column, and the modal's
  view/edit form keep their current structure; this SOW changes click routing and
  color sourcing, not layout.
- **Editing logged workouts/runs from the calendar.** Logged sessions keep
  navigating to their detail pages; no read-only modal for them (the
  "planned-only" scope decision).
- **A user-facing color-customization UI.** This SOW builds the *abstraction
  seam* so that feature is later a localized change; it does not add a settings
  surface or persist user color choices.
- **Adding new activity types.** `mobility` / `core` tokens stay reserved in
  `globals.css`; the abstraction leaves room for them but this SOW ships
  `lift` + `run`.
- **Re-tokenizing the whole status system.** Retiring `emerald-*` as the
  completion *color* is in scope; a broader `--status-*` token taxonomy is not
  needed once status is shape+badge.

## Implementation Details

### `lib/activity-colors.ts`

The map above plus a thin resolver. Call sites pass an `ActivityType` (from
`disciplineOf`), never raw tokens. Document in-file that the static map is the
present resolver and the seam for future per-user colors. An unknown/未mapped
type falls back to a neutral token set (defensive; today only `lift`/`run` reach
it).

### `day-cell.tsx`

- `doneToneClasses(discipline)` → derive classes from `activityColors(discipline)`.
- `PlannedPill` → border/text from `activityColors(type)`; **status** drives only
  dashed-vs-solid, the badge, and (skipped) strikethrough — not color. Remove the
  `border-emerald-500/70 text-emerald-300` completed branch in favor of the type
  color + check.
- `CompletedPlannedPill` → already uses the logged discipline tone; route through
  the resolver.
- **Click handlers**: a planned pill calls a new `onOpenPlanned(plan)` (opens the
  modal + selects day) instead of `onSelectPlanned(id)`; logged pills navigate
  (or keep their select+expand — see Open Questions); `+N more` and the cell
  background call the existing day-select + scroll-to-timeline.

### `page.tsx`

- Wire `onOpenPlanned` to the existing `openViewPlan(plan)` (sets
  `editingPlan = plan`, `planningOpen = true`) so a grid pill opens the modal in
  view mode — the same handler the timeline banner already uses.
- Keep `openNewPlan()` ("Plan a workout"), `onSaved={loadWindow}`, and
  `onStartWorkout` as-is.

### Banners (`workout-`, `run-`, `planned-`, `completed-planned-banner.tsx`)

- Left bar + focus ring → `activityColors(disciplineOf(event)).dot`.
- `planned-banner.tsx`: replace the status-driven rail (`--accent` / `emerald-500`
  / `--muted`) with the type color; keep the dashed treatment + status badge +
  the resync affordance; skipped adds muted + strikethrough.
- `completed-planned-banner.tsx`: replace the always-emerald bar with the logged
  session's type color; the `COMPLETED` framing comes from the badge/check and
  the existing disclosures.

### Tokens

No new color tokens required — the change *removes* hardcoded `emerald-*` and
centralizes the existing `--discipline-*` usage. If `design-system.md` doesn't
yet enumerate the `--discipline-*` activity palette, add it there as the
documented activity-color set (small docs reconcile).

## Testing

- **`lib/activity-colors.ts` unit test**: `run`/`lift` resolve to the expected
  token sets; the resolver is the only color source (a grep-style assertion in
  review, plus the unit test).
- **Color-parity component tests**: for a planned run, the grid pill and the
  timeline banner both resolve to the **run** `dot` token (not violet/emerald);
  same for a planned lift; completed-planned uses the logged type color, not
  emerald.
- **Status-rendering tests**: planned → dashed + `PLANNED`; completed → solid +
  check + `COMPLETED` (no emerald); skipped → muted + strikethrough + `SKIPPED`.
- **Click-routing tests**: a planned pill click opens `PlannedWorkoutModal` in
  **view** mode for that plan and selects the day; the in-modal pencil enters edit
  mode; a logged pill navigates to the session detail; `+N more` / cell-background
  selects the day and scrolls the timeline into view; on a **multi-event day**,
  each pill opens its own event and each timeline row opens its own target.
- Web CI green: lint/format/typecheck/test/build.

## Rollout

One **web PR**: add `lib/activity-colors.ts`, route `day-cell.tsx` + the four
banners through it, retire `emerald-*`, switch the timeline bars to type color,
and wire planned-pill → read-only modal. Plus **docs bookkeeping** (this SOW, and
the `design-system.md` activity-palette note if needed). Single shippable change,
no backend dependency.

## Open Questions

1. **Logged grid-pill click: navigate or keep expand?** The clean "a pill opens
   its event" model implies a logged workout/run pill should **navigate** to its
   detail page directly. Today it selects the day and expands the banner in the
   timeline (navigation happens from the banner). Recommend **navigate directly**
   for consistency with planned pills; the fallback is to leave logged pills on
   today's select+expand and only change planned pills. (Planned-pill behavior is
   settled either way.)
2. **Completed-planned color source.** A merged plan+logged event resolves its
   type from the *logged* session (a planned lift fulfilled by a logged run shows
   run color). Recommend keying off the logged session (what actually happened);
   confirm that's the intuition vs. keying off the plan's `activity_kind`.
3. **Skipped legibility without color.** Muted + strikethrough + `SKIPPED` badge
   should read clearly, but skipped is the rarest state — confirm it doesn't need
   a stronger cue than the type color at reduced emphasis.
