# Plan: Activities Overview — Instrument-Panel

**SOW**: [`sows/activities-overview-instrument-panel.md`](../sows/activities-overview-instrument-panel.md)
**Repos**: `prog-strength-web` (implementation), `prog-strength-docs` (status flip)
**Branch**: `feat/activities-overview-instrument-panel`

## Goal

Rebuild the body of the Activities → Overview tab
(`components/activities/activities-overview-view.tsx`, `?view=overview`) from
three near-identical stat-tile rows around one chart into the **instrument-panel**
composition: a compact KPI row over a tight grid of small framed
small-multiple charts. Conform to design-system v0.4.1 (no token/accent/type
change). No backend change — every number is derived client-side from the data
the Overview already fetches (workouts, running sessions, steps, steps goal).

## Constraints (from SOW)

- `scope: in-system` — use existing CSS vars (`--discipline-run-*`,
  `--discipline-lift-*`, `--accent`, `--success`, `--surface-3`, `--muted`,
  `--faint`, `--border`, `--radius-card`) and `@/lib/chart-colors`. **No raw
  hex, no new hue, no token edit.** Note the implemented v0.4.1 web tokens are
  `--discipline-lift-dot: #9aa6d6` and `--discipline-run-dot: #7fae9e`
  (mirrored in `lib/chart-colors.ts` as `CHART_LIFT_LINE` / `CHART_RUN_LINE`);
  these are the canonical web tokens — use them, don't re-tone to the doc's
  table values.
- No backend / API / SDK change. No new endpoint. `prog-strength-api` not touched.
- Keep the data layer of the view **verbatim**: the four fetches, the
  loading/error/empty guards, the 401 redirect, the `days` prop, the
  100-workout truncation note.
- Don't promote the DX mockup code; the `design-explore/*` route stays gated.
- The accent (`--accent`) is used **once** for emphasis (the consistency
  readout) — never as an activity hue.

## Work breakdown

### Task 1 — Derivation module `lib/activities-overview-stats.ts` (+ tests)

A pure, React-free module typed against `lib/api.ts` shapes (`Workout`,
`RunningSession`, `StepsEntry`). One entry function
`deriveOverviewStats({ workouts, sessions, steps, days, now })` returning a
typed `OverviewStats`. Bucketing uses local-date helpers consistent with
`parseLocalDate` / `isoDate` (re-exported from `lib/steps-stats.ts`) and a
real `now: Date` (injected for testability; the view passes `new Date()`).

Constants: `METERS_PER_MILE = 1609.344`, `KM_PER_MILE = 1.609344`.

Fields:

- **headline** — `totalActiveSeconds` (lifting `ended_at − performed_at`,
  completed only + running `duration_seconds`), `workoutCount`, `runCount`,
  `sessionCount`, `avgSessionSeconds` (0 sessions ⇒ 0; render guards `—`).
- **avgRunPaceSecPerMi** — `Σ distance_meters / Σ duration_seconds` over runs
  that carry pace (`avg_pace_sec_per_km != null`), converted to sec/mi;
  `null` when no run in the window carries pace. (Weighted only over runs
  with data — a manual run contributes neither distance nor duration to the
  pace numerator/denominator.) Also expose `bestPaceSecPerMi` (min of
  non-null `best_pace_sec_per_km`, converted) → `null` when none.
- **weekly** — Monday-anchored week buckets over the window (reuse
  `buildWeeklyBuckets` from `lib/weekly-buckets.ts` for the bounded/all span
  logic). Per week: `mileageMeters` (sum), `paceSecPerMi` (`null` when the
  week has no pace-carrying run — gap, not zero). Bars/sparkline read these;
  weeks with no run are gaps.
- **disciplineSplit** — `{ liftSeconds, runSeconds, liftPct, runPct }`.
  Single-modality degrades to `100 / 0` (and `0 / 0` ⇒ both 0, render guards).
- **consistency** — `daysActive` (distinct local calendar days with any
  workout / run / step entry), `totalDays` (`days` for bounded windows;
  earliest-entry→today span for `All`), `currentStreak` (consecutive active
  days ending **at today**, 0 if today inactive), `longestStreak`.
- **effort** — `avgHrBpm` (mean of non-null `avg_heart_rate_bpm`),
  `maxHrBpm` (max of non-null `max_heart_rate_bpm`), `elevationGainMeters`
  (Σ non-null `elevation_gain_meters`); each `null` when no run reports it.
- **stepsSeries** — day- or week-bucketed steps over the window; **unlogged
  days are gaps (`null`)**, never zero bars. Empty ⇒ empty array (instrument
  hidden by the view).

**Tests** (`lib/activities-overview-stats.test.ts`, table-driven over fixtures
mirroring the DX's `Last 90 days` block + single-modality + no-steps + `All`):
avg pace pace-weighted & **null when all runs manual**; discipline split
`100/0`; consistency `daysActive` + current/longest streak across a rest-week
gap; weekly mileage/pace gaps-not-zeros and `n = 1`; effort null when no
HR/elevation; steps trend unlogged = gap.

### Task 2 — Instrument components

Small local components co-located under
`components/activities/overview/` (page-private to the overview):

- `PaceSparkline` — hand-rolled SVG line over `weekly[].paceSecPerMi`,
  null weeks are gaps; `run` hue; "Best m:ss" caption. (`null`-pace window ⇒
  empty-state `— /mi`.)
- `MileageBars` — hand-rolled SVG/div bars over `weekly[].mileageMeters`,
  `run` hue.
- `SplitDonut` — hand-rolled SVG donut over `disciplineSplit`; lift vs run
  hues; `100/0` renders a single ring cleanly; legend with `fmtHm`.
- `StepsBars` — div bars over `stepsSeries`; gap days render as a faint void
  (`--surface-3`), logged days `--accent` (the steps trend, the one place the
  mockup uses accent on a chart — keep consistent but the **headline accent
  emphasis** stays on the consistency readout).
- `EffortReadout` — labeled figures: avg/max HR (bpm), elevation gain (ft when
  `distanceUnit === "mi"`, else m). `—` for each null field; whole instrument
  shows a calm empty line when all three null.
- Reuse the existing `ActivitiesCombinedChart` for **Weekly activity**
  (carried over, not rebuilt).

Each instrument is wrapped by a shared `Instrument` frame
(14px panel, hairline border, small title + sub-label) mirroring the mockup's
`Instrument`. Conversion helpers (`fmtHm`, `fmtPaceMi`) live in the stats
module or reuse `lib/chart-format` / `lib/distance-unit-context`.

### Task 3 — Rebuild `activities-overview-view.tsx` (+ tests)

Keep the data layer verbatim. Replace the render body (the three stat-tile
rows + lone chart `<section>`) with:

1. **KPI row** — Total time · Sessions (`N workouts · M runs`) · Avg run pace
   · Days active + streak. Small uniform type, hierarchy from position; the
   **accent used once** on the days-active/streak readout.
2. **Instrument grid** — responsive `grid-cols-1 sm:grid-cols-2 lg:grid-cols-3`;
   weekly activity spans two columns; pace sparkline, mileage bars, time-split
   donut, steps mini-bars (hidden when `stepsSeries` empty), effort readout.
   Absent instruments (no runs ⇒ no pace/mileage; all-null effort) collapse so
   the grid never leaves a hole.

`Total volume`, `PRs`, `Steps goal %` are removed. `workoutVolume` import drops
if unused.

**Tests** (extend `activities-overview-view.test.tsx`): KPI row omits
volume/PRs/goal-% and shows Avg run pace + Days active; each instrument renders
for the representative fixture; hard states degrade — no pace (`— /mi`,
sparkline gaps), no steps (instrument hidden), single-modality (donut `100/0`,
no hole), rest week (dip reads intentional) — without broken frames. Stub
`ActivitiesCombinedChart` as today.

### Task 4 — Review + gate + ship

- Spec-review and code-quality-review (subagents).
- Gate: `npm run lint && npm run format:check && npm run typecheck &&
  npm run test && npm run build` all green locally.
- Push `feat/activities-overview-instrument-panel`; open PR in
  `prog-strength-web`.
- In `prog-strength-docs`: flip `dx/activities-overview.md` →
  `status: selected` (winning idiom `instrument-panel`) and the SOW →
  `status: shipped` / `Shipped` / `Last updated: 2026-06-22`; open the docs PR
  with the required template.

## Out of scope

Backend/API; other Activities tabs; the period filter & tab-bar semantics;
design-system/token changes; promoting DX mockup code.
