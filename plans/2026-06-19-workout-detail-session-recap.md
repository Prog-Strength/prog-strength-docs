# Plan: Workout Detail — Session-Recap

**SOW**: [`sows/workout-detail-session-recap.md`](../sows/workout-detail-session-recap.md)
**Repo**: `prog-strength-web` (single PR) + `prog-strength-docs` (status flip)
**Branch**: `feat/workout-detail-session-recap`

## Goal

Recompose `/workouts/[id]` (`app/(app)/workouts/[id]/page.tsx`) into the
`session-recap` editorial reading column selected by `dx/workout-detail`:
note promoted to an editorial lead, PRs as a `--warning` celebratory subhead,
quiet supporting stat line, a single "what it trained" muscle strip (radar +
bars killed), and a detail-page-specific quiet exercise list. Migrate the
surface to design-system v0.4 tokens. No API or data-model changes; everything
derived client-side. The shared `WorkoutDetailsBody` is left byte-for-byte
untouched.

## Constraints / load-bearing boundaries

- **Do not touch** `components/workout-details.tsx` (`WorkoutDetailsBody`) or
  its three other consumers (workouts-view, calendar workout-digest, timeline
  summary). The detail page gets its own list.
- **No new design tokens.** Use existing CSS vars: `--warning` (PR), `--danger`
  (delete), `--accent` (add-exercise link + superset tag only), `--foreground`/
  `--muted`/`--faint`, near-black ramp `--background`/`--surface`/`--border`.
  No steel-blue / `--discipline-lift-*`, no chart color.
- **Reuse production helpers**: `workoutVolume`/`predominantUnit`
  (`lib/workout-volume.ts`, unit-aware) and `setsByCategory`
  (`lib/muscle-set-distribution.ts`) — not the mockup's simplified volume or
  pre-tallied muscle data.
- Honor the dumbbell convention (`weight` is per dumbbell).
- Preserve every edit affordance (Edit details, Delete, + Add exercise,
  per-exercise pencil) wired to existing handlers/modals.
- Preserve `CompletesPlanBanner` plan-linkage (mockup omitted it).

## Orphan-component decision

- `MuscleGroupRadarChart` is still used by `timeline/_components/WorkoutTimelineSummary`
  and `components/workouts-analytics.tsx` → **keep it.**
- `MuscleGroupSetBars` is used **only** by this page → **delete it** (and its
  test if any) as orphan cleanup after removal, per the SOW's tentative lean.

## Phased tasks

### Task 1 — `lib/workout-recap.ts` + unit tests

A small, unit-tested derivation module. Exports:

- `leadHeadline(workout): { kicker, headline }` — kicker = formatted date.
  With PRs: headline = top-PR story (highest weight, ties by reps), singular
  `A new PR — {w} × {r} on {lift}` vs plural `{Word} PRs, topped by {w} × {r}
  on {lift}`. No PRs: headline = workout title via `workoutTitle` fallback
  (auto `Workout - …` name → formatted date; reuse `hasMeaningfulName` from
  `components/workout-details.tsx`). `shortLift` strips
  `Barbell `/`Dumbbell `/`Seated `/`Standing `/`Lying ` prefix.
- `topSetIndex(exercise): number` — heaviest set, ties broken by reps; first of
  equal-weight for a flat scheme.
- `prSubhead(events): string` — `"{n} new personal record(s) — including {top
  1–2 breaks as weight × reps}."`
- `populatedCategories(workout, exercises): { category, value }[]` — wraps
  `setsByCategory([workout], exercises)`, filters `value > 0`, sorts descending.

Tests: `leadHeadline` for 0/1/4 PRs (top-PR selection, singular/plural,
shortLift stripping, auto-name → date); `topSetIndex` ramp/flat/tie-by-reps;
`prSubhead` for 1 vs many; `populatedCategories` descending + empty-safe.

### Task 2 — `app/(app)/workouts/[id]/_components/recap-exercise-list.tsx` + tests

Detail-page-specific quiet list. Props: `workout`, `exerciseMap`,
`onEditGroup(group, groupIndex)`. Per group:
- Consecutive same `superset_group` collapse into one block with a small
  `--accent` `superset` tag; standalone = one line.
- Each line: exercise name (from `exerciseMap`), set count, top set
  (`{n} × · top {weight}`) via `topSetIndex` + a weight formatter that honors
  the per-dumbbell convention (reuse the `isPerDumbbell` logic shape — append
  "per dumbbell" / "/hand" rather than implying combined load).
- Per-group edit pencil revealed on hover/focus → `onEditGroup`.
Reuse the same `groupExercises` adjacency rule as the shared body. Tests:
superset grouping, top-set readout, pencil callback, dumbbell label.

### Task 3 — Recompose `page.tsx`

Replace the dashboard stack with the centered `max-w-2xl` editorial column in
DOM order: header (back-link, demoted subtitle `{title} · {date} · {duration?}`,
Edit details + Delete top-right) → `CompletesPlanBanner` (when planned) → lead
(kicker, `text-4xl` headline, `--warning` PR subhead) → reflection
(`whitespace-pre-wrap` prose / italic `--faint` placeholder) → quiet `<dl>` stat
line (Volume · Sets · Exercises · Duration? · PRs?) → "What it trained" strip →
"The work" (`+ Add exercise` + `RecapExerciseList`). Remove the inline
`PRBanner`, `MuscleGroupRadarChart`, `MuscleGroupSetBars`, `WorkoutSummaryStats`,
`WorkoutDetailsBody` imports/usages from this page. Keep loading/error/redirect
and all handlers/modals. Delete the now-orphaned
`components/muscle-group-set-bars.tsx`.

### Task 4 — Page render tests

Component render tests for both fixtures (canonical 4-PR + degrade
zero-PR/no-note/no-ended_at/flat/superset) covering every state in the SOW.
Verify edit affordances open the right modals via preserved handlers and the
plan banner renders when planned.

### Task 5 — Local gate

`npm run typecheck && npm run lint && npm run format:check && npm run test &&
npm run build`. Fix anything that fails. Confirm shared body + its three
consumers are unchanged.

### Task 6 — Review

Spec review against the SOW + code-quality review (subagents). Address findings.

### Task 7 — PRs

Web feature branch + PR (conventional title). Docs status flip + PR using the
required template.

## Testing strategy

Vitest + Testing Library, co-located. Unit tests for `lib/workout-recap.ts`;
component render tests for the recap list and the page. Gate: lint, format,
typecheck, test, build (CI's matrix).
