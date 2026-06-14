# Mobile Parity Phase 3: Analytics — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement Phase 3 of `sows/mobile-feature-parity-and-testflight.md`: the Progress view migrates to the modernized `movement_pattern` progression API (per-exercise trends + aggregate stats), Personal Records gains a Lifts | Running toggle with expandable history charts, and the calendar gains a day digest and weekly stat chips.

**Architecture:** Pure-JS phase (no native modules — `runtimeVersion` stays `"1"`, every merge ships OTA; the native fingerprint guard must stay green). Web is the parity reference (`../prog-strength-web`): `app/(app)/progress/`, `app/(app)/personal-records/`, `components/calendar/{day-digest,weekly-overview}`. New shared `TimeSeriesChart` (react-native-svg, ticks from `components/charts/ticks.ts`) powers both 1RM-history and best-effort-history charts.

**Verification policy:** No JS test runner (deliberate). Every task ends with `npm run typecheck && npm run lint` + listed simulator checks. PATH for shell work: `export PATH="$HOME/.nvm/versions/node/v24.15.0/bin:$PATH"`. Hooks run on commits (lint-staged + typecheck + commitlint) — commit subjects must be conventional, lower-case.

**Branch:** `feat/mobile-parity-phase-3` off up-to-date `main` in `prog-strength-mobile`.

**Deliberate deviations from web (do not "fix"):**
- No month-level stat-tile row on the calendar (web has 6 tiles above the grid; the SOW scopes mobile to day digest + weekly chips, and phone vertical space goes to the grid + agenda).
- No URL-backed state for pattern/days/PR-view (mobile uses component state, same as the existing segments).
- No tooltip-hover behavior; the progress chart keeps the existing tap-to-inspect SelectedPointCard.

---

### Task 1: Progression API modernization (`lib/api.ts` + call-site shim)

Migrate `listProgression` to the web's modern shape. Web reference: `../prog-strength-web/lib/api.ts` (search `movement_pattern`) — verify every field below against it before writing; web wins on drift.

**Files:**
- Modify: `lib/api.ts` (replace the progression types + fetcher in place, ~lines 288–378)
- Modify: `components/progress/progress-view.tsx` (minimal call-site shim so the branch compiles; full UI migration is Task 4)

- [ ] **Step 1: Replace the progression types.** Keep `MuscleGroupProgressionPoint`, `ExerciseBaseline`, `Trendline` (shapes unchanged — verify); replace `MuscleGroupProgression` and add the new types:

```typescript
/** Echo of the progression request, with the server-resolved muscle groups. */
export type ProgressionFilterInfo = {
  movement_pattern?: string;
  // Legacy single-muscle-group filter; unused by the modern flow.
  muscle_group?: string;
  muscle_groups_included: string[];
};

/** Least-squares trend for one exercise within the query window. */
export type PerExerciseTrend = {
  exercise_id: string;
  session_count: number;
  // %/month; null when below the session threshold or degenerate.
  slope_per_month: number | null;
  trendline: Trendline | null;
};

export type ProgressionAggregate = {
  lifts_tracked: number;
  lifts_progressing: number;
  median_slope_per_month: number | null;
  min_sessions_threshold: number;
};

export type MuscleGroupProgression = {
  filter: ProgressionFilterInfo;
  since: string;
  until: string;
  baseline_model: string; // e.g. "recency_weighted_current"
  exercise_baselines: ExerciseBaseline[];
  points: MuscleGroupProgressionPoint[];
  per_exercise_trends: PerExerciseTrend[];
  aggregate: ProgressionAggregate | null;
};

export type MovementPattern = "push" | "pull" | "legs" | "core" | "all";
```

- [ ] **Step 2: Replace the fetcher** (match web's param name exactly):

```typescript
/**
 * GET /workouts/progression?movement_pattern=…&since=…&until=….
 * The server resolves the pattern to muscle groups (echoed in
 * filter.muscle_groups_included). since/until default to 90d server-side.
 */
export async function listProgression(
  token: string,
  movementPattern: MovementPattern,
  since?: string,
  until?: string,
): Promise<MuscleGroupProgression> {
  const params = new URLSearchParams({ movement_pattern: movementPattern });
  if (since) params.set("since", since);
  if (until) params.set("until", until);
  const resp = await fetch(`${config.apiUrl}/workouts/progression?${params.toString()}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  const got = await unwrap<MuscleGroupProgression | null>(resp, null);
  if (!got) throw new Error("progression unavailable");
  return got;
}
```

- [ ] **Step 3: Shim the call site.** In `components/progress/progress-view.tsx`, change the `listProgression(t, muscleGroup, …)` call to `listProgression(t, "all", sinceISO, untilISO)` and adjust any property access that no longer typechecks (`data.muscle_group` → `data.filter`, single `trendline` → `per_exercise_trends`) with the MINIMAL edits needed to compile — if the chart consumed the old single `trendline`, pass `null`/empty equivalents for now and leave a `// TODO(task 4)` marker. Do not redesign the UI here.
- [ ] **Step 4:** typecheck + lint pass. Commit: `feat(api): progression fetcher — movement_pattern, per-exercise trends, aggregate (web twin parity)`

---

### Task 2: PR-history + best-efforts API (`lib/api.ts`)

Web reference: `../prog-strength-web/lib/api.ts` (search `OneRMHistory`, `BestEffort`). Verify and follow web exactly (including URL paths and encodeURIComponent usage).

**Files:**
- Modify: `lib/api.ts` (append; Phase 2 may have added `listRunningBestEfforts` — grep first, add only what's missing)

- [ ] **Step 1: Append types + fetchers** (exact shapes from web):

```typescript
export type OneRMHistoryPoint = {
  workout_id: string;
  performed_at: string; // RFC3339
  estimated_1rm: number;
};

export type ExerciseOneRMHistory = {
  exercise_id: string;
  exercise_name: string;
  unit: "lb" | "kg";
  points: OneRMHistoryPoint[];
};

/** GET /personal-records/{exercise_id}/history. */
export async function getExerciseOneRMHistory(
  token: string,
  exerciseId: string,
): Promise<ExerciseOneRMHistory> { /* fetch + unwrap, throw on null — copy web */ }

export type RunningBestEffort = {
  distance_key: string;   // "1mi" | "2mi" | "5k" | "10k" | "half_marathon" | "marathon"
  distance_label: string; // "5K"
  distance_meters: number;
  duration_seconds: number;
  pace_sec_per_km: number;
  activity_id: string;
  activity_start_time: string;
};

/** GET /running/best-efforts — only distances the user has covered. */
export async function listRunningBestEfforts(token: string): Promise<RunningBestEffort[]> { /* unwrap [] */ }

export type BestEffortPoint = {
  activity_id: string;
  activity_start_time: string;
  duration_seconds: number;
};

export type RunningBestEffortHistory = {
  distance_key: string;
  distance_label: string;
  distance_meters: number;
  points: BestEffortPoint[];
};

/** GET /running/best-efforts/{distance_key}/history. */
export async function getRunningBestEffortHistory(
  token: string,
  distanceKey: string,
): Promise<RunningBestEffortHistory> { /* fetch + unwrap, throw on null — copy web */ }
```

(The bodies marked "copy web" must be transcribed from the web twin, matching its exact paths, error messages, and null handling — this file is a deliberate twin.)

- [ ] **Step 2:** typecheck + lint. Commit: `feat(api): 1rm history + running best-efforts fetchers (web twin parity)`

---

### Task 3: Shared time-series chart + tick consolidation

**Files:**
- Create: `components/charts/time-series-chart.tsx`
- Modify: `components/charts/ticks.ts` (add the percent-snapping y-tick variant), `components/progress/progression-chart.tsx` (delete its local `niceYTicks` copy at ~lines 259–272 and import the shared variant)

- [ ] **Step 1: ticks.ts** — add (move from progression-chart verbatim, exported as a named variant):

```typescript
/** Percent-domain y ticks: snaps steps to 5/10/15/20/25/50-style increments. */
export function niceYTicksPercent(min: number, max: number, count: number): number[] { /* moved verbatim */ }
```

Refactor `progression-chart.tsx` to import it; behavior identical (typecheck after this step).

- [ ] **Step 2: TimeSeriesChart.** A date-x-axis line chart following `run-metric-chart.tsx`'s structure (onLayout width, PADDING constants, dark COLOR_GRID/#27272a + COLOR_AXIS/#a1a1aa, Polyline series, niceYTicks grid):

```tsx
export function TimeSeriesChart({
  points,          // { t: number /* epoch ms */; y: number }[] sorted asc by t
  height = 140,    // web parity: 140pt history charts
  color = "#60a5fa",
  yFormat,         // (y: number) => string
  caption,         // optional muted caption under the title, e.g. "lower is faster"
}: { … })
```

- X ticks: 2–3 via `niceXTicks(tMin, tMax, …)` over epoch ms, labels `new Date(t).toLocaleDateString(undefined, { month: "short", day: "numeric" })` ("Apr 18").
- Y ticks: `niceYTicks(yMin, yMax, 4)`, labels via `yFormat`.
- Edge cases like run-metric-chart: width 0 placeholder; `< 2` points → render the single value + "Not enough data yet" muted text instead of an SVG (web behavior).
- Dots r=3 at each point + 2px Polyline.

- [ ] **Step 3:** typecheck + lint + simulator (Progress chart renders identically post-tick-refactor). Commit: `feat(charts): shared time-series chart; consolidate percent tick helper`

---

### Task 4: Progress view modernization

Web reference: `../prog-strength-web/app/(app)/progress/page.tsx` + its `_components` (read for exact copy/format). Mobile file: `components/progress/progress-view.tsx` (699 lines — read fully first).

**Files:**
- Modify: `components/progress/progress-view.tsx`, `components/progress/progression-chart.tsx`

- [ ] **Step 1: Filter migration.** Replace the 11 muscle-group pills with 5 movement-pattern pills — `All | Push | Pull | Legs | Core` (labels title-case, values `all/push/pull/legs/core`), same pill styling as the current MuscleGroupPills (active = `border-accent bg-accent/15`). Keep the 30/60/90d timeframe pills. State: `pattern: MovementPattern` default `"all"`. Remove the Task-1 shim; call `listProgression(t, pattern, sinceISO, untilISO)` and refetch on pattern/timeframe change.
- [ ] **Step 2: Aggregate stat tiles.** Replace the current three tiles with web's three (same tile component style):
  - "Lifts progressing": `${aggregate.lifts_progressing} / ${aggregate.lifts_tracked}`, tone positive when ratio ≥ 0.5, danger when ≤ 0.25, else neutral; "—" when aggregate null.
  - "Median per-exercise slope": `formatSlope(median_slope_per_month)` → `+1.4%/mo` / `−0.8%/mo` / `—`; tone positive > 0.5, danger < −0.5.
  - "Best session": highest `normalized_max` point → `${Math.round(p.normalized_max * 100)}% of baseline` with the date in the label; "—" when no points.
- [ ] **Step 3: Per-exercise trendlines on the chart.** `progression-chart.tsx`: accept `trends: PerExerciseTrend[]` instead of the single `trendline`; render one dashed trendline per exercise (color = that exercise's `exerciseColorMap` color, only when `trendline` non-null). Keep scatter points, 1.0 baseline dashed reference line (label it "baseline"), selection behavior, and y-domain math (include all trendline endpoints + 1.0).
- [ ] **Step 4: Legend slopes.** Extend the legend rows: `[dot] Exercise · baseline N unit  ↑ +1.4%/mo` — arrow ↑/→/↓ per web thresholds (±0.5), slope text green `text-emerald-300` / `text-danger` / `text-muted`; "not enough data" when `slope_per_month` null. Add the "not enough data" banner when EVERY trend has null slope: copy web's text (read it; ~"Keep logging — direction shows up around session 3").
- [ ] **Step 5: Tables intact.** The estimates/sets tables keep working off `points` (no changes beyond compile fixes).
- [ ] **Step 6:** typecheck + lint + simulator: pattern pills refetch (push shows only push-pattern exercises), tiles populate, per-exercise dashed trendlines render, legend shows slopes. Commit: `feat(progress): movement-pattern filter, aggregate tiles, per-exercise trendlines`

---

### Task 5: Personal Records — Lifts | Running + expandable history charts

Web reference: `../prog-strength-web/app/(app)/personal-records/` (read `page.tsx` + `_components/`). Mobile: `components/progress/prs-view.tsx` (194 lines).

**Files:**
- Modify: `components/progress/prs-view.tsx` (grows substantially — extract `components/progress/running-prs-view.tsx` for the running cards to keep files focused)
- Create: `components/progress/running-prs-view.tsx`

- [ ] **Step 1: Sub-toggle.** Inside the PRs segment add a nested `SegmentedControl` — `Lifts | Running` (component state, default lifts; the Nutrition tab's nested-control pattern). "Customize" button renders only in Lifts.
- [ ] **Step 2: Lifts cards — expandable 1RM history.** Each PRCard with a PR gains a chevron footer ("History" + chevron-down/up). First expand → `getExerciseOneRMHistory(token, exercise_id)`, cache in a `useRef<Map<string, ExerciseOneRMHistory>>` for the view's lifetime (re-expand = no refetch); loading spinner inside the card while fetching; error → inline danger text. Render `TimeSeriesChart` with `points: history.points.map(p => ({ t: Date.parse(p.performed_at), y: p.estimated_1rm }))`, `yFormat` = weight in the HISTORY's unit converted to preferred via `convertWeight` (convert y values before charting, label via preferred unit).
- [ ] **Step 3: formatWeight consolidation.** Delete prs-view's local `formatWeight`; use a small null-guard around `lib/units`' `formatWeight` so card weights/1RMs now render in the user's preferred unit (this is a deliberate behavior improvement — note in PR):

```tsx
const fmtWeight = (v: number | null, unit: "lb" | "kg" | null) =>
  v === null || unit === null ? "—" : formatWeight(v, unit, preferred ?? unit);
```

- [ ] **Step 4: Running PRs view** (`running-prs-view.tsx`). Fetch `listRunningBestEfforts`; render ALL six standard distances in order (1 Mile, 2 Mile, 5K, 10K, Half Marathon, Marathon — constant list; merge achieved efforts by `distance_key`): achieved → time (`formatRunDuration`) + pace (`formatPace` + `/unit`) + "Set on {date}" + "View activity →" (`router.push(\`/activities/run/${activity_id}\`)`) + expandable history; not achieved → "—" + "No record yet", no chevron. History expand → `getRunningBestEffortHistory(distance_key)` (same caching pattern) → `TimeSeriesChart` with `y: duration_seconds`, `yFormat: formatRunDuration`, `caption: "lower is faster"`.
- [ ] **Step 5:** typecheck + lint + simulator: toggle switches views without refetch churn; expand a lift card → chart; running cards show your real best efforts; em-dash cards for unachieved distances. Commit: `feat(prs): lifts/running toggle, expandable 1rm + best-effort history charts`

---

### Task 6: Calendar — day digest

Web reference: `../prog-strength-web/components/calendar/day-digest.tsx` (+ run-digest). Mobile: `app/(tabs)/calendar.tsx` (~540 lines; agenda at ~362–424).

**Files:**
- Modify: `app/(tabs)/calendar.tsx`

- [ ] **Step 1: Count line.** Under the selected-day header add web's summary line, zero parts omitted, pluralization correct: `2 activities · 1 run · 1 lift` (runs+workouts counted; "1 activity" when single).
- [ ] **Step 2: Expandable rows.** Agenda WorkoutRow/RunRow gain an inline-expand affordance (chevron on the right; row press still navigates — chevron press toggles):
  - Workout expanded: exercise list (name via exerciseByID + set count + top set `formatWeight(weight, unit, preferred)` per exercise), notes if present.
  - Run expanded: stat rows — duration (`formatRunDuration`), avg pace, avg/max HR ("—" when null), calories, elevation gain.
  - Keep 44pt targets for both press zones (chevron hitSlop).
- [ ] **Step 3:** typecheck + lint + simulator: count line correct on multi-activity days; expand/collapse works; navigation unchanged. Commit: `feat(calendar): day digest — count line + expandable workout/run details`

---

### Task 7: Calendar — weekly stat chips

Web reference: `../prog-strength-web/components/calendar/weekly-overview.tsx` (read for exact stat math + current-week accent). Mobile grid is already built as 6 explicit week rows (calendar.tsx ~293–296).

**Files:**
- Modify: `app/(tabs)/calendar.tsx`

- [ ] **Step 1: Per-week rollup.** Compute `WeeklyStat` per grid row using web's exact math (quoted in the web report; local-date-key membership over the 7 days): `activities`, `liftMinutes` (ended workouts only, ms→min rounded), `runMeters`, `runMinutes`.
- [ ] **Step 2: Chips.** Under each week row with `activities > 0`, render one thin single-line chip, parts joined by " · ", zero parts omitted: `2 activities · 45m · 5.2 mi · 40m` (lift duration via a `formatTotalDuration(minutes)` helper — "45m"/"2h"/"1h 30m", port from web; distance via `formatDistance` + unit). Muted text ~10px; current week (contains today) gets `text-accent`. No chip for empty weeks (keeps the grid compact).
- [ ] **Step 3:** typecheck + lint + simulator: chips align under their weeks, current week accented, math sanity-checked against the Activities overview for the same window. Commit: `feat(calendar): weekly stat chips aligned to grid weeks`

---

### Task 8: Final review + PR

- [ ] **Step 1:** Full `npm run typecheck && npm run lint && npm run format:check && npm run fingerprint:check` (the last MUST pass — this phase is JS-only; a fingerprint change means something native snuck in).
- [ ] **Step 2:** Final whole-branch review (controller dispatches a reviewer over `main...HEAD`).
- [ ] **Step 3:** Push `feat/mobile-parity-phase-3`, open PR noting: pure-JS phase → ships via OTA on merge (~30s to the phone, no TestFlight build); the PR-card weights now convert to the preferred unit (deliberate improvement over the old as-logged rendering); month stat tiles deliberately omitted (SOW scope).

---

## Self-review notes (applied)

- **SOW Phase 3 coverage:** movement_pattern migration ✓ (T1+T4), per-exercise trends + aggregate stats ✓ (T4), PR running view ✓ (T5), per-card progression charts both views ✓ (T3+T5), calendar digest ✓ (T6), weekly chips ✓ (T7).
- **Type consistency:** `MovementPattern`/`PerExerciseTrend`/`ProgressionAggregate` defined T1, consumed T4; history types defined T2, consumed T5; `TimeSeriesChart` defined T3, consumed T5; `niceYTicksPercent` defined T3 step 1, consumed by progression-chart.
- **Judgment calls:** lazy per-card history fetch with a ref-cache instead of TanStack Query (SOW open question #2 leaned "adopt only for history charts" — a ref-cache achieves the same UX without a new dependency; revisit if invalidation needs appear); running PRs render all six distances (web omits unachieved ones from the API result but renders the fixed set — verify in web and match its rendering, not its API).
