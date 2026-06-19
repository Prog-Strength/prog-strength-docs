---
type: sow
status: shipped
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Workout Detail — Session-Recap

**Status**: Shipped · **Last updated**: 2026-06-19

## Introduction

The Workout Detail page (`/workouts/[id]`,
`prog-strength-web/app/(app)/workouts/[id]/page.tsx`) is the one-stop record of a
single logged session — reached from the Workouts list, the Personal Records
`View workout →` links, the calendar, and the agent. The functionality is solid;
the **composition** is not. The page that should make a lifter relive a great
session reads like a generic admin dashboard, and the canonical example is a
*great* session it under-celebrates: four PRs and a 305 lb bench double rendered
as a flat amber box, with the sets that earned them ~1000px down in a list of
identical bullets, two redundant muscle charts (one a degenerate single-workout
radar), and the lifter's own reflection buried as plain body text.

[`dx/workout-detail`](../dx/workout-detail.md) (explored in
[prog-strength-web#88](https://github.com/Prog-Strength/prog-strength-web/pull/88))
re-explored the whole page and produced five variants. The **`session-recap`**
direction was selected. This SOW builds it for real, production-quality,
conforming to the design system.

`session-recap` tells the session as **a short story**: the lifter's **note is
promoted to the editorial lead** (a headline that says what happened + the
reflection as readable prose), the **PRs ride as a celebratory subhead**, and the
numbers, the muscle summary, and the exercises support quietly beneath. It is the
direct answer to the two complaints the DX named most sharply — *the human voice is
buried* and *the PR moment is muted* — and, because this surface **predates the
v0.4 oura-calm re-tone**, conforming its stale blue/zinc/amber chrome to the
current tokens is part of the brief.

## Proposed Solution

Recompose the Workout Detail page into the `session-recap` reading layout: a
centered editorial column (`max-w-2xl`, generous vertical rhythm) replacing the
current dashboard stack. Top to bottom:

1. **Header** — back-link, the workout name + `date · duration` as a small
   subtitle, and the **Edit details / Delete** actions (preserved, top-right).
2. **Editorial lead** — a date kicker, a **large headline** derived from the
   session (the top PR story when there are PRs; the workout title/date when not),
   and a **`--warning` PR subhead** (`4 new personal records — including 305 lb × 2
   and 205 lb × 3.`) shown only when PRs exist.
3. **The reflection** — the workout `notes` as large, relaxed prose (the body of
   the story); a quiet italic placeholder when there's no note.
4. **A quiet stat line** — Volume · Sets · Exercises · (Duration) · (PRs) as a
   small `<dl>`, numbers supporting rather than leading; honest tile-dropping
   (no Duration without `ended_at`, no PRs tile at zero) preserved.
5. **"What it trained"** — the two redundant muscle analytics collapse to **one
   quiet inline strip** of the populated categories (`Back 13 · Chest 12 ·
   Shoulders 4`), zeros dropped — killing the degenerate radar entirely.
6. **The work** — a `+ Add exercise` action and a **detail-page-specific quiet
   exercise list**: one line per exercise (or per superset block), showing the
   exercise name, a `superset` tag where grouped, the set count, and the **top
   set** (`4 × · top 305 lb`), each with its **edit pencil**.

The page keeps owning data (`getWorkout`, `listExercises`,
`getPlannedWorkoutBySession`) and all mutation; the recompose is presentational
plus one small derivation helper. **The shared `WorkoutDetailsBody` is left
untouched** — the detail page gets its own quiet list, so the three other surfaces
that render the shared body are unaffected (see Non-Goals / blast radius).

## Goals and Non-Goals

### Goals

- Recompose `/workouts/[id]` into the `session-recap` editorial reading column,
  faithful to the selected variant.
- **Promote the note to the lead** — an editorial headline + the reflection as
  prose — and degrade cleanly to a session verdict (title/date) when there's no
  note, so a note-less session still reads complete.
- **Promote the PR moment** to a celebratory `--warning` subhead under the
  headline, naming the top breaks; **vanish gracefully** to a plain session record
  when there are zero PRs (the common case).
- **Collapse the redundant muscle analytics** (the radar + the bars) to a single
  quiet "what it trained" strip of populated categories — the degenerate
  single-workout radar is removed, not rendered near-empty.
- Express each exercise's **top set** and **superset grouping** in the quiet list,
  reconstructing the top set from the set sequence (no warm-up flag in the data).
- **Preserve every edit affordance** — Edit details, Delete, + Add exercise, and
  the per-exercise pencil — wired to the existing modals and handlers.
- **Preserve the plan-linkage banner** (`CompletesPlanBanner`) when the session
  fulfilled a planned workout — the mockup omitted it; production must keep it.
- **Migrate this surface to design-system v0.4** — replace the Tailwind `amber-*`
  PR banner and the radar's raw `#3b82f6 / #27272a / #a1a1aa` with `--warning`,
  `--foreground`/`--muted`/`--faint`, and the near-black ramp; Manrope; 14px radius.
- Handle every required state (see Implementation Details → States), verified
  against the canonical 4-PR fixture **and** a zero-PR / no-note / no-`ended_at`
  degrade fixture.

### Non-Goals

- **No change to the shared `WorkoutDetailsBody`** (`components/workout-details.tsx`)
  or its three other consumers — the Workouts list inline-expand
  (`components/activities/workouts-view.tsx`), the Calendar digest
  (`components/calendar/workout-digest.tsx`), and the Timeline card
  (`app/(app)/timeline/_components/WorkoutTimelineSummary.tsx`). The detail page
  introduces its **own** quiet exercise list; the shared body keeps serving the
  other surfaces exactly as today. This is the load-bearing scope boundary.
- **No inline PR-to-set marking and no warm-up/working-set ledger.** Marking the
  🏆 set inline and distinguishing warm-ups from working sets were the
  *`exercise-ledger`* idiom, not `session-recap`. `session-recap` shows only the
  **top set** per exercise. Out of scope.
- **No API or data-model changes.** `GET /workouts/{id}`, `GET /exercises`, and
  `GET /planned-workouts/by-session` are unchanged; everything derived is
  client-side. `repos: prog-strength-api` is intentionally absent.
- **No new design tokens.** This is `scope: in-system`; it does not re-decide
  palette/accent/type. `session-recap` is deliberately **near-monochrome** — it
  does **not** use the lift steel-blue discipline tone or any chart color.
- **The DX exploration code is not merged or salvaged.** PR #88 is a disposable
  throwaway; this is a clean reimplementation. Do not import from
  `app/design-explore/`.

## Implementation Details

### Design-system conformance (read first)

Conform to [`design-system.md`](../design-system.md) **v0.4** (oura-calm) and
reference its tokens — never hard-code hex that duplicates a token. This surface
predates the re-tone, so conforming it is part of the work:

- **Color**: near-black ramp (`--background` / `--surface` / `--border`), text
  `--foreground` / `--muted` / `--faint`. **`--warning`** carries the PR
  celebration (replacing the Tailwind `amber-*` banner). **`--danger`** for the
  Delete action. **`--accent`** (periwinkle) only as app-chrome — the small
  `Add exercise` link and the `superset` tag — never as a card tint. The radar's
  raw `#3b82f6 / #27272a / #a1a1aa` is **deleted** with the radar.
- **No steel-blue, no chart color.** `session-recap` is intentionally monochrome;
  the `--discipline-lift-*` tones are *not* used here (they belong to the other,
  unselected idioms).
- **Typography**: Manrope throughout; the lead **headline** is the one large
  figure (`text-4xl`, tight tracking), everything else small caption/prose — the
  widest type contrast of the five idioms. Numerals carry `-0.03em` / tabular.
- **Form**: 14px panel radius, full-pill buttons, hairline borders.

### Surface being recomposed

`app/(app)/workouts/[id]/page.tsx` (the `WorkoutDetailPage` client component) and
the components it composes today:

- `WorkoutSummaryStats` (`components/workout-summary-stats.tsx`) — its honest
  tile-dropping logic (drop Duration without `ended_at`, drop PRs at zero) moves
  into the recap's quiet stat line; **reuse its `workoutVolume` / `predominantUnit`
  helpers** from `lib/workout-volume.ts` (unit-aware) rather than the mockup's
  simplified volume.
- `MuscleGroupRadarChart` (`components/muscle-group-radar-chart.tsx`) and
  `MuscleGroupSetBars` (`components/muscle-group-set-bars.tsx`) — **removed from
  this page**, replaced by the inline strip. Grep for other usages; if neither is
  used elsewhere, delete the orphaned components (and the raw-hex radar) as
  cleanup — otherwise leave them for their other consumer.
- The inline `PRBanner` (in `page.tsx`) — **replaced** by the editorial lead +
  `--warning` subhead.
- The exercise list — today the shared `WorkoutDetailsBody` with `onEditGroup` —
  is replaced **on this page only** by the new quiet list (below).

### Data shape & derivation (client-side)

From `lib/api.ts` (unchanged): `getWorkout(token, id) → Workout`,
`listExercises() → Exercise[]` (catalog → `exerciseMap: Map<id, Exercise>` for
names), `getPlannedWorkoutBySession(token, id, "workout") → PlannedWorkout | null`.
Types `Workout`, `WorkoutExercise`, `WorkoutSet`, `PersonalRecordEvent`, `Exercise`
are as today.

Add a small, unit-tested helper module (e.g. `lib/workout-recap.ts`) for the
derivations the recap needs:

- **`leadHeadline(workout): { kicker, headline }`** — the editorial lead. With
  PRs: kicker = formatted date, headline = the **top PR story** (highest-weight
  break), e.g. *"Four PRs, topped by 305 lb × 2 on Bench Press"* (singular *"A new
  PR — …"* for one). With **no PRs**: kicker = date, headline = the workout title
  (the auto-name `Workout - <date>` falls back to the formatted date — reuse the
  existing `hasMeaningfulName` from `components/workout-details.tsx`). Port the
  mockup's `lead()` + `shortLift()` (strips the `Barbell `/`Dumbbell `/… equipment
  prefix from the lift name for the headline).
- **`topSetIndex(exercise): number`** — the heaviest set (ties broken by reps),
  used for the `top 305 lb` readout. Port from the mockup; this is how the top set
  is found **without** a warm-up flag the data doesn't have, and it degrades for a
  flat scheme (returns the first of equal-weight sets).
- **The PR subhead string** — `"{n} new personal record(s) — including {top 1–2
  breaks as weight × reps}."`, from `personal_records_set`.

For **"what it trained,"** reuse the production `setsByCategory(workouts,
exercises)` from `lib/muscle-set-distribution.ts` (it already returns
`{ data, hasData }` over the six fixed categories), then filter to `value > 0` and
sort descending — the inline strip is exactly the populated tail. No new muscle
math; the redundant radar/bars are dropped, not re-derived.

### Layout & composition

A single centered reading column. In DOM order:

1. **Header** (hairline-bordered): `← Workouts` back-link (keep the existing
   `/activities?view=workouts` target), then a row with the small subtitle
   (`{title} · {date} · {duration}`, duration omitted without `ended_at`) on the
   left and **Edit details** + **Delete** on the right. The big workout name is
   intentionally *demoted* here — the hero is the editorial headline below.
2. **Plan-linkage banner** — render `CompletesPlanBanner` here (when
   `getPlannedWorkoutBySession` is non-null), integrated into the column above the
   lead so it doesn't fight the PR subhead. Absent for the common unplanned case.
3. **Lead** — kicker (uppercase, `--faint`), the `text-4xl` headline, and the
   `--warning` PR subhead (PRs only).
4. **Reflection** — `notes` as `whitespace-pre-wrap` large prose; the italic
   `--faint` "No reflection on this session." placeholder when empty.
5. **Stat line** — the quiet `<dl>` (Volume · Sets · Exercises · Duration? · PRs?)
   bounded by hairline rules.
6. **What it trained** — the populated-category strip.
7. **The work** — section label + `+ Add exercise`, then the quiet exercise list.

### Quiet exercise list (detail-page-specific)

A new component local to the detail page (e.g.
`app/(app)/workouts/[id]/_components/recap-exercise-list.tsx`) — **not** a change
to the shared `WorkoutDetailsBody`. It renders, per group:

- Consecutive exercises sharing a `superset_group` collapse into **one block**
  with a small `--accent` `superset` tag; standalone exercises are one line each.
- Each line: the exercise name (from `exerciseMap`), the set count, and the **top
  set** (`{sets.length} × · top {fmtWeight(topSet)}`). Honor the **dumbbell
  convention** (`weight` is per dumbbell) — don't imply a combined load.
- A per-group **edit pencil** revealed on hover/focus, wired to the page's
  existing `setGroupEdit({ group, mode: "edit" })` → `ExerciseEditModal`, exactly
  as `WorkoutDetailsBody`'s `onEditGroup` does today.
- `+ Add exercise` keeps calling `setGroupEdit({ group: newExerciseGroup(...),
  mode: "add" })`.

### Edit / delete / create (preserved)

No handler logic changes — only where the affordances render:

- **Edit details** → `setEditingDetails(true)` → `WorkoutDetailsEditModal`.
- **Delete** → `handleDelete()` (confirm → `deleteWorkout` → redirect to
  `/activities?view=workouts`).
- **Per-exercise pencil** and **+ Add exercise** → `setGroupEdit(...)` →
  `ExerciseEditModal`, via the new quiet list.

### States to handle (render/verify both fixtures)

- **Four-PR canonical** — editorial headline names the top PR; `--warning` subhead
  lists the top breaks; stat line shows the PRs tile; top sets read (`bench 4 × ·
  top 305 lb`); the finishing superset reads as one block.
- **Zero-PR** — no subhead; headline falls back to title/date; the PRs tile drops;
  the page still reads as a complete session record, not a stripped trophy page.
- **No note** — the italic `--faint` placeholder; the lead still stands on the
  headline.
- **No `ended_at`** — no Duration in the subtitle or stat line; no time framing
  breaks.
- **Degenerate / sparse muscles** — the strip shows only populated categories
  (`Legs 12` alone in the degrade fixture); never a near-empty six-axis chart.
- **Supersets** — `superset_group` blocks read as one unit (both fixtures carry
  one).
- **Auto-named workout** — `Workout - May 14` falls back to the date as the title.
- **Plan-linkage present vs absent** — `CompletesPlanBanner` integrates above the
  lead when planned; absent otherwise.
- **Long / dense session** — up to ~12 exercises; the quiet list stays scannable.

## Testing

- **Unit (Vitest)** for `lib/workout-recap.ts`: `leadHeadline` for 0 / 1 / 4 PRs
  (top-PR selection, singular vs plural, `shortLift` prefix stripping, auto-name →
  date fallback); `topSetIndex` for a ramp, a flat scheme, and a weight tie broken
  by reps; the PR-subhead string for 1 vs many breaks.
- **"What it trained"** derivation: `setsByCategory` filtered/sorted yields the
  populated tail in descending order and is empty-safe.
- **Component render** of both fixtures (mirror the DX's canonical 4-PR Chest &
  Back day and the zero-PR / no-note / no-`ended_at` degrade) covering every state
  above.
- **Regression**: Edit details, Delete, + Add exercise, and the per-exercise
  pencil still open the correct modals via the preserved handlers; the
  plan-linkage banner renders when planned; **the shared `WorkoutDetailsBody` and
  its three other consumers are byte-for-byte unchanged**.
- `lint`, `typecheck`, and the project build pass.

## Rollout

Single PR to `prog-strength-web` (the recomposed `page.tsx`, the new
`recap-exercise-list` component, `lib/workout-recap.ts` + tests, removal of the
radar/bars + inline `PRBanner` from this page, and any orphan-component cleanup),
plus this SOW's status flip in `prog-strength-docs`. No flags, migrations, or API
coordination — a client-side recompose of one existing page. Verify against a real
account: a genuine multi-PR session **and** a plain no-PR/no-note session, plus a
planned-workout session to confirm the plan banner, before merge. Standard
rebase-only merge.

## Open Questions

1. **Status**: leave `draft` for review, or flip to `ready_for_implementation` so
   the autonomous developer can dispatch it?
2. **Orphaned muscle components**: if `MuscleGroupRadarChart` / `MuscleGroupSetBars`
   have no other consumer once removed from this page, delete them (and the raw-hex
   radar) as cleanup, or leave them in place? Tentative lean: delete if orphaned —
   they're the stale pre-v0.4 chrome the re-tone is retiring.
3. **Headline voice**: the lead headline is generated (*"Four PRs, topped by 305 lb
   × 2 on Bench Press"*). Confirm the auto-generated phrasing is acceptable, or
   should the no-PR case lean on the workout name only (simpler, less "authored").
   Tentative lean: keep the generated headline; it's the whole point of the idiom.
