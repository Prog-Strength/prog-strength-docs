---
type: sow
status: draft
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Activities Overview — Instrument-Panel

**Status**: Draft · **Last updated**: 2026-06-21

> Frontend SOW. It implements a chosen DX variant and therefore inherits that DX's
> `scope` — here **`in-system`**. The visual foundation (design-system **v0.4.1**,
> oura-calm) is already decided, so this SOW **conforms** to it; it does **not**
> re-tone the system or touch shared tokens. The work is in `prog-strength-web`;
> `prog-strength-docs` only flips the DX to `selected` and marks this SOW shipped.

## Introduction

The **Overview tab** of `/activities`
(`components/activities/activities-overview-view.tsx`, rendered under
`?view=overview`) is the cross-domain landing surface — the answer to "how's my
training going?" *before* a user drills into the Workouts, Running, or Steps tabs.
It shipped from the activities-page redesign
(`sows/activities-page-redesign.md`, which set design-system v0.4 across all four
tabs at once) with the page's generic furniture: **three near-identical rows of
stat tiles bracketing a single weekly line chart**. It reads like a totals
receipt, not a picture of a training block.

The receipt underdelivers in three specific ways the Design Exploration
`dx/activities-overview.md` (PR Prog-Strength/prog-strength-web#101) called out:

- **The wrong metrics lead.** `Total volume` and `PRs` are lifting-detail figures
  that already head the Workouts tab and mean nothing next to a run; `Steps goal %`
  is a flat percentage that hides the actual trend.
- **Running — half the user's sessions — is a second-class citizen.** It gets a
  single `Runs` count and a `Total mileage` tile, with **no pace, no effort, no
  trend**.
- **There is no sense of consistency.** Nothing answers "did I show up this
  month?" — the emotional core of an activity overview.

The DX explored the tab across **five `in-system` variants** (all sharing v0.4.1:
soft near-black ramp, single periwinkle accent, `run`/`lift` discipline hues,
Manrope, 14px hairline panels), diverging only on **composition, density, and
which metrics lead** — not on palette, type, or accent. The owner selected
**`instrument-panel`** (reference: **Strava** "your stats"): the Overview as a
**panel of trends**. A minimal **KPI row** sits over a **grid of small-multiple
charts** — the weekly activity dual line, a **pace sparkline**, a **weekly mileage
bar trend**, a **discipline time-split donut**, and a **steps mini-bar series** —
each a small framed instrument. It is the most **visual, trend-forward** of the
five: the page becomes **graphs first, numbers second**, running finally reads at a
glance, and consistency earns real estate.

This SOW reimplements that variant **production-quality** against the real tab and
data. Because `scope: in-system`, there is **no palette, accent, type, or
design-system change** — it conforms to v0.4.1. There is **no backend change**:
every number and series the instruments show is derived **client-side** from the
data the Overview already fetches (workouts + running sessions + steps + steps
goal); **no new endpoints are assumed**.

## Proposed Solution

Rebuild the body of the Overview view into the instrument-panel composition,
keeping the tab's orchestration intact: the `days` period prop (from the parent
Activities page's `7 / 30 / 90 / All` filter, default `30`), the existing
client-side fetches (workouts, running sessions, steps, steps goal), the loading /
error / empty guards, and the truncation note. The mockup on the DX branch
(`app/design-explore/activities-overview/_variants/instrument-panel.tsx` +
`_data.ts`) is the **visual spec, not code to promote**.

The composition has two layers:

1. **A minimal KPI row** — the headline numbers demoted to a compact strip so the
   charts can lead: **Total time**, **Sessions** (`N workouts · M runs`), **Avg
   run pace**, **Days active** + **streak**. Small labels, position-and-density
   hierarchy, the periwinkle accent used **once** for emphasis (the streak/days
   readout), not sprayed across the row.

2. **A grid of small-multiple instruments** — each a framed 14px-panel chart with
   a small label, the charts as the hero:
   - **Weekly activity** — the existing dual **Lifting / Running** minutes line,
     carried over (discipline hues, intentional dip on the rest week).
   - **Pace sparkline** — `avg pace` per run/week over the window (`run` hue),
     showing trend; manual pace-less runs are gaps, not zeros.
   - **Mileage bars** — weekly **mileage trend** (`run` hue), giving running visual
     presence beyond one count.
   - **Discipline time-split donut** — share of *time* across **lifting vs
     running** (the two discipline hues), answering "where did my effort go."
   - **Steps mini-bars** — a steps trend (replacing the dropped `Steps goal %`),
     hidden entirely when the window has no step entries.
   - **Effort readout** — **avg/max HR** and **elevation gain** over runs that
     report them, as a small framed instrument (sparkline or labeled figures),
     adding the intensity dimension the page lacks today.

The real work beyond restyling is **a pure, tested derivation layer**: the
rollups the receipt never computed (avg pace, weekly mileage/pace buckets,
discipline time-split, days-active + streak, avg/max HR, elevation) over the
fetched window, each handling the hard states honestly. The mockup's `_data.ts`
derivations are the production reference, ported to read the real API types and a
real "today."

## Goals and Non-Goals

### Goals

- **Add a pure, tested overview-derivation module** (e.g.
  `lib/activities-overview-stats.ts`), porting the mockup's `_data.ts`
  derivations to read the real `lib/api.ts` types over the active window:
  - **Headline rollups** — `totalActiveSeconds`, `sessionCount` (split
    `workouts` / `runs`), `avgSessionSeconds`.
  - **`avgRunPaceSecPerMi`** — Σ`distance_meters` / Σ`duration_seconds` across the
    window's runs (or the mean of non-null `avg_pace_sec_per_km`), converted to
    min·per·mi; **null when no run in the window carries pace** (all manual) →
    rendered `— /mi`, never `NaN`. The average is weighted **only** over runs that
    have the data.
  - **`weeklyMileage`** / **`weeklyPace`** — week-bucketed series over the window
    for the mileage bars and pace sparkline; weeks with no run are gaps, not zeros.
  - **`disciplineSplit`** — Σ lifting duration (`ended_at − performed_at`,
    completed sessions only) vs Σ running duration (`duration_seconds`), as
    `{ liftSeconds, runSeconds, liftPct, runPct }`; a single-modality window
    degrades to `100 / 0` cleanly, not a broken donut.
  - **`consistency`** — `daysActive` (distinct calendar days with **any** workout,
    run, or step entry) over `M` window days, plus **current** and **longest**
    streak (consecutive active days). Local-date bucketing consistent with the
    existing `parseLocalDate` / `isoDate` helpers; a real `new Date()` for "today"
    (the mockup pins `TODAY`).
  - **`effort`** — mean `avg_heart_rate_bpm`, max `max_heart_rate_bpm`, and
    Σ`elevation_gain_meters` over runs that report them; **null/`—` when none do**.
  - **Steps trend** — week- or day-bucketed steps series; **unlogged days are gaps
    (`null`), never zero-step bars** (a day with no entry is missing data, not a
    zero-step day).
- **Rebuild the Overview view body into instrument-panel**, conforming to v0.4.1:
  - **A compact KPI row** — Total time · Sessions (`N workouts · M runs`) · Avg run
    pace · Days active + streak — small uniform type, hierarchy from position, the
    **accent used once** (the consistency readout), absorbing today's three
    stat-tile rows.
  - **A tight, even grid of small framed instruments** — weekly activity dual line,
    pace sparkline, mileage bars, discipline time-split donut, steps mini-bars, and
    the HR/elevation effort readout — the charts as the hero, **discipline hues per
    chart** (`run` green-teal, `lift` steel-blue), **one accent for emphasis**.
  - **`Total volume`, `PRs`, and `Steps goal %` are gone**; `Avg run pace`, the
    mileage/pace trends, the discipline time-split, the days-active consistency
    readout, and HR/elevation are present (the DX's decided content edits).
- **Use the in-system color tokens** — the shared `@/lib/chart-colors` and the CSS
  vars (`--discipline-run-*`, `--discipline-lift-*`, `--accent`, `--success`,
  `--surface-3`), **never raw hex or a new hue**. Lifting reads steel-blue, running
  green-teal, the accent stays **app chrome** (it must never read as an activity
  hue — "activity ≠ selection").
- **Preserve the tab's orchestration** — the `days` prop and the parent's period
  filter; the existing client-side fetches (workouts + running + steps + steps
  goal) and their loading / error / empty guards; the auth/401 redirect; and the
  **workouts truncation note** (the fetch caps at 100 — when hit, the note must
  still surface so older data's absence is explained).
- **Handle every state the DX enumerated**:
  - **A zero/rest week** — the trend dips to zero and the heatmap/streak shows a
    gap that reads as *intentional rest*, not broken/missing data.
  - **A single-modality window** — lifting-only or running-only: the discipline
    split renders `100 / 0`, a missing pace/mileage instrument (no runs) or absent
    lifting trend **degrades gracefully**, leaving no hole.
  - **Null pace / null HR** — manual runs report neither: `Avg run pace`, the pace
    sparkline, and the HR/effort readout show `—` cleanly and weight their averages
    only over runs that carry the data.
  - **No steps history** — the steps instrument is **hidden entirely** (today's
    view already gates on this), not rendered empty.
  - **Sparse vs dense window** — the same composition survives a one-week `7d` and
    a multi-month `All`; trends must not look broken at `n = 1`.
  - **Truncation cap** — the existing 100-workout truncation note still surfaces.
- **Tests** — the derivation module is the priority: `avgRunPace` (pace-weighted,
  **null when all runs manual**), `disciplineSplit` (`100/0` single-modality),
  `consistency` (`daysActive`, current/longest streak across the rest-week gap),
  weekly mileage/pace bucketing (**gaps not zeros**, `n = 1`), `effort`
  (null when no HR/elevation), and the steps trend (unlogged = gap). Plus component
  tests: the KPI row drops volume/PRs/goal-% and shows avg pace + days active; each
  instrument renders for the representative fixture and degrades (no pace, no
  steps, single-modality, rest week) without holes. **CI green**
  (lint/format/typecheck/test/build).

### Non-Goals

- **Any backend / API / SDK change.** Everything is derived client-side from the
  data the Overview already fetches (workouts, running sessions, steps, steps
  goal). **No new endpoint** (`GET /activities/overview`-style aggregation is
  explicitly out of scope — the DX assumes none); `prog-strength-api` is **not** in
  `repos:`.
- **Any design-system or shared-token change.** `scope: in-system` against v0.4.1 —
  conform only; no token/accent/type edit, no `design-system.md` change, no new
  discipline hue.
- **Redesigning the other Activities tabs** (Workouts / Running / Steps) — they are
  out of scope and unchanged. The Overview must keep reading as a sibling of the
  shipped tabs.
- **Restyling the page chrome beyond keeping it functional.** The persistent
  **period filter** (`7 / 30 / 90 / All`) and the **tab bar** are owned by the
  parent Activities page; a variant may relocate/restyle but must keep them
  functional, and this SOW does not change their semantics.
- **Promoting the DX mockup code.** `instrument-panel.tsx` / `_data.ts` are the
  visual spec, not code to copy; the `design-explore/activities-overview` route
  stays gated (`NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE`) and never ships. The other four
  variants are not built; the DX PR is **closed, never merged**.

## Implementation Details

### Derivation module (`prog-strength-web/lib`)

A pure, React-free module (e.g. `activities-overview-stats.ts`) porting the
mockup's `_data.ts` derivations, typed against the real `lib/api.ts` shapes
(`Workout`, `RunningSession`, `StepsEntry`, `StepsGoal`) rather than the mockup's
fixtures. It exposes the rollups the receipt never computed:

- **Durations are derived, not stored.** Lifting duration = `ended_at −
  performed_at` over **completed** sessions only (`ended_at` null ⇒ in-progress ⇒
  excluded). Running duration = `duration_seconds`.
- **Unit conversions** stay where the existing tab does them — stored SI
  (`distance_meters`, `*_sec_per_km`, `*_bpm`, `elevation_gain_meters`) → mi /
  min·per·mi / bpm / ft for display.
- **Nullability is load-bearing.** `avg_pace_sec_per_km`, `best_pace_sec_per_km`,
  and the HR fields are **null on manually-logged runs** (no TCX). Every
  pace/HR/effort aggregate weights only the non-null members and returns `null`
  (→ `—`) when the window has none — never `NaN` or `0`.
- **Bucketing** uses local-date parsing consistent with `parseLocalDate` /
  `isoDate` and a real `new Date()` for "today"; week buckets and the day axis
  treat **missing days as gaps (`null`)**, not zeros, across both bounded windows
  (`7 / 30 / 90`) and `All` (earliest-entry → today).

Table-driven tests over the DX's representative `Last 90 days` fixture (46h 7m
total · 47 sessions · 121.5 mi · 9:14 /mi avg · 151 bpm avg · 4,120 ft · 41 of 90
days active · 3-day current / 9-day longest streak · 3 manual pace-less runs · a
rest-week trough near `May 25`) plus single-modality, no-steps, and `All`-period
fixtures.

### Overview tab rebuild (`components/activities/activities-overview-view.tsx`)

Keep the component's data layer verbatim (the fetches, the loading/error/empty
guards, the truncation note, the auth redirect). Replace the **render body** — the
three stat-tile rows and the lone weekly chart — with the **KPI row** and the
**instrument grid**. Drive every instrument from the derivation module over the
fetched window. Build the small-multiple charts (pace sparkline, mileage bars,
time-split donut, steps mini-bars, effort readout) as small local components
mirroring the mockup but wired to derived data and the shared chart-color tokens;
**carry over the existing weekly Lifting/Running line** rather than rebuilding it.
The instruments lay out as a tight, even responsive grid that holds at both
breakpoints and collapses sensibly when an instrument is absent (no runs → no
pace/mileage; no steps → no steps bars) so the grid never leaves a hole.

### Tests (`prog-strength-web`)

- **`activities-overview-stats.ts`** — the derivations per the fixtures above:
  pace-weighted avg (and **null when all runs manual**), discipline split
  (`100/0`), consistency (`daysActive` + current/longest streak across the
  rest-week gap), weekly mileage/pace (gaps not zeros, `n = 1`), effort (null with
  no HR/elevation), steps trend (unlogged = gap).
- **Component** (extend `activities-overview-view.test.tsx` if present, else add) —
  the KPI row omits `Total volume` / `PRs` / `Steps goal %` and shows `Avg run
  pace` + `Days active`; each instrument renders for the representative fixture;
  and the hard states degrade — **no pace** (manual runs → `— /mi`, sparkline
  gaps), **no steps** (steps instrument hidden), **single-modality** (donut
  `100/0`, absent instrument leaves no hole), **rest week** (dip reads as
  intentional) — without broken frames.

## Rollout

1. **`prog-strength-docs`** — flip `dx/activities-overview.md` to `status:
   selected` (winning idiom `instrument-panel`); mark this SOW `shipped` on merge.
2. **`prog-strength-web`** — the derivation module + Overview-tab rebuild + tests,
   in one PR. Vercel preview to verify across `7 / 30 / 90 / All`, a rest-week
   window, a single-modality window, a no-steps window, manual (pace-less) runs,
   and both breakpoints.

### Verification after rollout

- `/activities` → Overview reads as a **panel of trends** — a compact KPI row over
  a grid of small framed charts, **graphs first, numbers second** — not three rows
  of look-alike tiles around one chart.
- **Running is first-class**: avg pace, the pace sparkline, and the weekly mileage
  bars read at a glance, with **HR / elevation** carrying effort; the page no
  longer hides running behind a single count.
- **Consistency lands**: the **days-active** readout ("41 of 90 days") and the
  current/longest **streak** make "did I show up?" legible, and the rest week reads
  as **intentional rest**, not missing data.
- **The decided content edits hold**: `Total volume`, `PRs`, and `Steps goal %`
  are gone; the discipline **time-split** answers "where did my effort go."
- **Degrades honestly**: manual runs → `— /mi` and sparkline gaps (no `NaN`); no
  steps → the steps instrument is hidden; a single-modality window → a `100/0`
  donut and no holes; sparse `7d` and long `All` both look intentional; the
  100-workout truncation note still surfaces; mobile and desktop both hold.
- It **sits beside Workouts, Running, and Steps** as the same v0.4.1 surface; the
  `design-explore/activities-overview` route stays gated / 404 in production; **no
  DX mockup code shipped**; the DX PR is closed (never merged); `design-system.md`
  unchanged.

## Open Questions

1. **Charts: recharts vs hand-rolled small-multiples.** The mockup hand-rolls the
   sparkline / mini-bars / donut as lightweight SVG/divs; the existing weekly chart
   and the other tabs use recharts (tooltips, responsive axis, shared `ChartCard`
   conventions). **Lean:** keep **recharts** for the instruments that benefit from
   tooltips/axis (weekly line, mileage bars, steps bars) for consistency with the
   sibling tabs, and allow hand-rolled SVG only for the **donut** and the **pace
   sparkline** where recharts is heavier than the glance they need. Confirm at
   review.
2. **Pace sparkline granularity: per-run vs per-week.** Per-run points show every
   session but get noisy over `90d`/`All` and jagged with manual gaps; per-week
   means are smoother and align with the mileage bars. **Lean:** **per-week**
   buckets for the sparkline (matching the mileage trend's x-axis), so the two run
   instruments share a timeline; revisit if per-run reads more informative on short
   windows. Confirm.
3. **Streak denominator and "current" definition.** `daysActive / M` counts
   calendar days in the window; "current streak" can either require today to be
   active or count the most recent consecutive run ending at the last active day.
   **Lean:** report **`daysActive` of `M` window days** for the consistency
   readout, and define **current streak** as the consecutive active run ending at
   **today** (0 if today is a rest day) — honest about "am I on a streak right
   now." Confirm the today-anchored definition reads right.
4. **`All`-period bucketing cost.** Week-bucketing every calendar week from the
   earliest entry to today is wide for a multi-year `All` history. **Lean:**
   week-bucket directly for `7 / 30 / 90` and accept it for `All` at current data
   sizes; if `All` gets heavy, bucket **monthly** for the chart instruments only —
   the KPI rollups are unaffected. Flag for a perf check on the longest realistic
   range.
