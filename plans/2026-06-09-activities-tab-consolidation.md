# Activities Tab Consolidation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Collapse the sidebar's separate `Workouts` and `Running` entries into one `Activities` entry with three URL-backed sub-views — Overview (new default), Workouts, Running — sharing a single page-level timeframe selector. The Running view gains a mileage/time analytics card; the Overview view is a new digest surface. Pure `prog-strength-web` UI/UX reorganization: no API, MCP, agent, or mobile changes.

**Architecture:** A new `app/(app)/activities/page.tsx` shell owns (1) the `?view=` URL param (source of truth for the active sub-view, default `overview`), (2) the shared `timeframe` state and its pills, and (3) the toolbar (view switchers + the running-only Upload TCX button). The shell threads a computed `days: number | null` into three view components — `ActivitiesOverviewView`, `WorkoutsView`, `RunningView` — each of which owns its own data fetches. The existing `/workouts` and `/running` parent routes become server-side redirects; detail routes keep their URLs. Four chart components (two existing + two new + one combined) and the Running analytics card consume a small set of newly-lifted shared helpers so the weekly-bucket and formatting logic lives in one place.

**Tech Stack:** Next.js 16.2.6 App Router (client components), React 19, Tailwind v4, recharts 3. **This is NOT the Next.js you know** — consult `node_modules/next/dist/docs/` before writing routing code (see notes below). No test suite exists in `prog-strength-web`; verification is `npm run lint`, `npm run typecheck`, `npm run build`, and the manual smoke plan in the SOW. Each task ends with verification commands and a commit.

**SOW:** `sows/activities-tab-consolidation.md` (in this repo). The SOW is the source of truth for design intent, the four-tile metric definitions, the prop contracts, and the manual verification plan; this plan sequences the work into reviewable tasks and pins the exact extractions.

---

## Working directory

All paths below are relative to `prog-strength-web/`. Run all `npm` commands from that directory. The feature branch is `feat/activities-tab-consolidation`.

## Next.js 16 notes (read before coding)

- **Server-side `redirect()`** (Tasks: routing): import `redirect` from `next/navigation`. A Server Component may call it synchronously; it throws `NEXT_REDIRECT` and needs no `return`. See `node_modules/next/dist/docs/01-app/03-api-reference/04-functions/redirect.md`. The `/workouts` and `/running` parent stubs become one-line server components — **do not** add `"use client"` to them.
- **`useSearchParams`** (Task: shell): a Client Component calling `useSearchParams` must be wrapped in a `<Suspense>` boundary or the production build fails with "Missing Suspense boundary with useSearchParams." The shipped `app/(app)/nutrition/page.tsx` is the validated pattern: an outer default export wrapping an inner component in `<Suspense fallback={null}>`. Mirror it exactly. See `node_modules/next/dist/docs/01-app/03-api-reference/04-functions/use-search-params.md`.
- **Route group `(app)`**: the new `activities/` directory lives under `app/(app)/` alongside `workouts/`, `running/`, `nutrition/`. It inherits the `(app)/layout.tsx` shell (sidebar). No special config.

## Invariant for every task

After each task: `npm run typecheck` passes, `npm run lint` passes, `npm run build` succeeds. The repo starts green and must stay green at every commit. New components that aren't yet rendered anywhere are still type-checked by the build. Do not delete a symbol's last consumer and its definition in different tasks — each task is self-consistent.

---

## Task 1: Lift chart helpers — `lib/chart-format.ts` and `lib/weekly-buckets.ts`

**Why first:** Four chart components (two existing — `WorkoutDurationChart`, `WorkoutVolumeChart`; two new — `RunningMileageChart`, `RunningTimeChart`; plus `ActivitiesCombinedChart`) all need the same Monday-anchored weekly-bucket windowing and the same Y-tick / hours / week-range formatters. Lifting them once now means every later chart task imports one implementation instead of copy-pasting `summarize`.

**Files:**

- Create: `lib/chart-format.ts`
- Create: `lib/weekly-buckets.ts`
- Modify: `components/workout-duration-chart.tsx` (consume both)
- Modify: `components/workout-volume-chart.tsx` (consume both)

- [ ] **Step 1: Create `lib/chart-format.ts`**

Lift the three formatters currently duplicated inside `workout-duration-chart.tsx` (and the volume chart). Keep the exact behavior:

```ts
/**
 * Formatting helpers shared by the activity charts (workout duration /
 * volume, running mileage / time, combined). Lifted out of
 * workout-duration-chart so all chart families render identical axes,
 * tooltips, and summary values.
 */

/** "0h" / "45m" / "2h" / "2h 30m" from a minute count. */
export function formatHours(minutes: number): string {
  if (minutes <= 0) return "0h";
  const totalMinutes = Math.round(minutes);
  const hours = Math.floor(totalMinutes / 60);
  const mins = totalMinutes % 60;
  if (hours === 0) return `${mins}m`;
  if (mins === 0) return `${hours}h`;
  return `${hours}h ${mins}m`;
}

/** Y-axis tick for a minutes series: "0" / "45m" / "2h" / "1.5h". */
export function formatYTick(minutes: number): string {
  if (minutes <= 0) return "0";
  if (minutes >= 60) {
    const h = minutes / 60;
    return Number.isInteger(h) ? `${h}h` : `${h.toFixed(1)}h`;
  }
  return `${Math.round(minutes)}m`;
}

/** "May 12 – 18" / "May 26 – Jun 1" from a Monday date. */
export function formatWeekRangeFromMonday(monday: Date): string {
  const sunday = new Date(monday);
  sunday.setDate(sunday.getDate() + 6);
  const sameMonth = monday.getMonth() === sunday.getMonth();
  const monStr = monday.toLocaleDateString("en-US", { month: "short", day: "numeric" });
  const sunStr = sameMonth
    ? String(sunday.getDate())
    : sunday.toLocaleDateString("en-US", { month: "short", day: "numeric" });
  return `${monStr} – ${sunStr}`;
}
```

- [ ] **Step 2: Create `lib/weekly-buckets.ts`**

Generalize `workout-duration-chart.tsx`'s `summarize` window-and-bucket logic into a reusable builder parameterized on a timestamp accessor and a per-week accumulator. The builder returns chronologically-ordered weeks with empty weeks zero-filled.

```ts
/**
 * Monday-anchored weekly bucketing shared by every activity chart.
 *
 * For bounded windows (7/30/90d) the span runs from `days` ago to today
 * so even an empty user sees a chart shaped like the window. For "all"
 * (days === null) the span runs from the oldest item to today. Weeks with
 * no items still appear (zero-filled by `factory`) so a multi-week gap
 * shows up as a dip instead of being smoothed away.
 */

export type WeekBucket<T> = T & {
  weekKey: string;
  weekStart: Date;
  // Unix-ms of the Monday so recharts' numeric XAxis places points linearly.
  t: number;
};

export function startOfMonday(d: Date): Date {
  const local = new Date(d.getFullYear(), d.getMonth(), d.getDate());
  // getDay: 0=Sun..6=Sat; Monday-anchored offset: Sun→6, Mon→0, Tue→1…
  const offset = (local.getDay() + 6) % 7;
  local.setDate(local.getDate() - offset);
  return local;
}

export function isoDate(d: Date): string {
  const y = d.getFullYear();
  const m = String(d.getMonth() + 1).padStart(2, "0");
  const day = String(d.getDate()).padStart(2, "0");
  return `${y}-${m}-${day}`;
}

function addDays(d: Date, n: number): Date {
  const out = new Date(d);
  out.setDate(out.getDate() + n);
  return out;
}

/**
 * @param items        the source rows (workouts or running sessions)
 * @param days         window length, or null for "all"
 * @param getTimestamp maps an item to the Date it falls on
 * @param factory      builds a fresh zero-filled accumulator for a week
 * @param accumulate   folds one item into its week bucket (mutates the bucket)
 */
export function buildWeeklyBuckets<T, B>({
  items,
  days,
  getTimestamp,
  factory,
  accumulate,
}: {
  items: T[];
  days: number | null;
  getTimestamp: (item: T) => Date;
  factory: () => B;
  accumulate: (bucket: WeekBucket<B>, item: T) => void;
}): WeekBucket<B>[] {
  const now = new Date();
  const since =
    days !== null
      ? new Date(now.getTime() - days * 24 * 60 * 60 * 1000)
      : items.length > 0
        ? new Date(Math.min(...items.map((it) => getTimestamp(it).getTime())))
        : now;

  const byKey = new Map<string, WeekBucket<B>>();
  const orderedKeys: string[] = [];
  for (
    let cursor = startOfMonday(since);
    cursor.getTime() <= startOfMonday(now).getTime();
    cursor = addDays(cursor, 7)
  ) {
    const key = isoDate(cursor);
    orderedKeys.push(key);
    byKey.set(key, { ...factory(), weekKey: key, weekStart: new Date(cursor), t: cursor.getTime() });
  }

  for (const it of items) {
    const bucket = byKey.get(isoDate(startOfMonday(getTimestamp(it))));
    if (bucket) accumulate(bucket, it);
  }

  return orderedKeys.map((k) => byKey.get(k)!);
}
```

- [ ] **Step 3: Refactor `components/workout-duration-chart.tsx`**

Replace the inline `summarize` window/bucket loop with `buildWeeklyBuckets`, and import `formatHours` / `formatYTick` / `formatWeekRangeFromMonday` from `lib/chart-format`. Keep `summarize`'s outer shape (it still computes `totalMinutes`, `sessionCount`, `openWorkouts`, and the `weeks` array with a `minutes` field per week) so the JSX is unchanged. The per-week accumulator carries `minutes`; the `getTimestamp` is `(w) => new Date(w.performed_at)`; completed-only / non-positive-span guards stay inside `accumulate`. Delete the now-duplicated `startOfMonday`/`isoDate`/`addDays`/`formatHours`/`formatYTick`/`formatWeekRangeFromMonday` local definitions in this file. The rendered chart must be byte-for-byte identical in behavior.

- [ ] **Step 4: Refactor `components/workout-volume-chart.tsx`**

Same treatment: its `summarize` becomes a `buildWeeklyBuckets` call with a `volume` accumulator (`(b, w) => b.volume += workoutVolume(w, displayUnit)`), `getTimestamp` `(w) => new Date(w.performed_at)`. Keep its own Y-tick / tooltip formatting (volume is unit-suffixed, not hours) — only the windowing + any shared week-range label move to the imports. Remove its duplicated `startOfMonday`/`isoDate`/`addDays` helpers.

- [ ] **Step 5: Verify and commit**

```bash
npm run typecheck && npm run lint && npm run build
```

Open `/workouts` in `npm run dev` and confirm the Time-lifting and Volume charts render identically to before (same buckets, same empty-week dips, same tooltips). Then:

```bash
git add lib/chart-format.ts lib/weekly-buckets.ts components/workout-duration-chart.tsx components/workout-volume-chart.tsx
git commit -m "refactor: lift weekly-bucket + chart-format helpers into lib"
```

---

## Task 2: Lift shared UI components — `stat-tile`, `view-button`, `toolbar-button`

**Why:** The Overview and Running views need `StatTile` (currently running-namespaced), `RunningAnalytics` needs the same icon-toggle `ViewButton` that lives inline in `WorkoutsAnalytics`, and the Activities toolbar needs the `ToolbarButton` that lives inline in the Nutrition page. Lift all three to neutral homes now so later tasks import, not duplicate.

**Files:**

- Create: `components/stat-tile.tsx` (lifted verbatim from `app/(app)/running/_components/StatTile.tsx`)
- Delete: `app/(app)/running/_components/StatTile.tsx`
- Modify: `app/(app)/running/page.tsx` and `app/(app)/running/[id]/page.tsx` — update the `StatTile` import path (grep for `_components/StatTile`).
- Create: `components/view-button.tsx` (lifted from the `ViewButton` inside `components/workouts-analytics.tsx`)
- Modify: `components/workouts-analytics.tsx` — delete the inline `ViewButton`, import from `@/components/view-button`.
- Create: `components/toolbar-button.tsx` (lifted from the `ToolbarButton` inside `app/(app)/nutrition/page.tsx`)
- Modify: `app/(app)/nutrition/page.tsx` — delete the inline `ToolbarButton`, import from `@/components/toolbar-button`.

- [ ] **Step 1: `components/stat-tile.tsx`** — move the file's contents verbatim (the `StatTile` with `value`/`label`/`sub`/`tone` props). Keep the same exported name `StatTile`.

- [ ] **Step 2: Delete `app/(app)/running/_components/StatTile.tsx`** and repoint its importers. `grep -rn "_components/StatTile" app` to find every site (the running page and the running detail page); rewrite each to `import { StatTile } from "@/components/stat-tile";`.

- [ ] **Step 3: `components/view-button.tsx`** — lift the `ViewButton` from `workouts-analytics.tsx` (the `label`/`active`/`onClick`/`children` icon-toggle button with `aria-pressed` and the accent-when-active styling). Export it as `ViewButton`. In `workouts-analytics.tsx`, remove the local definition and import it. The three icon components (`ClockIcon`, `DumbbellIcon`, `RadarIcon`) stay in `workouts-analytics.tsx` — only the button shell moves.

- [ ] **Step 4: `components/toolbar-button.tsx`** — lift the `ToolbarButton` from `nutrition/page.tsx` (the `onClick`/`icon`/`label`/`active?` ghost button with the `border-b-2 … pb-3` active underline and the `hidden sm:inline` label). Export `ToolbarButton`. In `nutrition/page.tsx`, remove the local definition and import it. Leave the page's icon components (`PlusIcon`, `LogIcon`, etc.) where they are.

- [ ] **Step 5: Verify and commit**

```bash
npm run typecheck && npm run lint && npm run build
```

Smoke `/running` (stat tiles unchanged), `/workouts` (analytics switcher unchanged), `/nutrition` (toolbar tabs unchanged). Then:

```bash
git add components/stat-tile.tsx components/view-button.tsx components/toolbar-button.tsx \
        components/workouts-analytics.tsx app/\(app\)/nutrition/page.tsx \
        app/\(app\)/running/page.tsx app/\(app\)/running/\[id\]/page.tsx
git rm app/\(app\)/running/_components/StatTile.tsx
git commit -m "refactor: lift StatTile, ViewButton, ToolbarButton to neutral homes"
```

---

## Task 3: Running charts + `RunningAnalytics` card

**Why:** The Running view's new analytics card needs two chrome-less charts and a wrapper. Build them before the view that mounts them.

**Files:**

- Create: `components/running-mileage-chart.tsx`
- Create: `components/running-time-chart.tsx`
- Create: `components/running-analytics.tsx`

Shared contract for both charts (mirrors `WorkoutDurationChart`): props `{ sessions: RunningSession[] | null; days: number | null; truncated: boolean; fetchLimit: number; distanceUnit: "mi" | "km" }`. `null` sessions → "Loading…"; empty/zero window → the chrome-less empty-state block; otherwise a recharts `AreaChart` of Monday-anchored weekly buckets built with `buildWeeklyBuckets`. Reuse the `#3b82f6` blue, gradient defs, `CartesianGrid`/`XAxis`/`YAxis`/`Tooltip`/`Area` structure and the `truncated` note copy from `WorkoutDurationChart`. `CHART_HEIGHT = 200`.

- [ ] **Step 1: `components/running-mileage-chart.tsx`**
  - Per-week accumulator: `{ meters: number }`, `accumulate: (b, s) => b.meters += s.distance_meters`, `getTimestamp: (s) => new Date(s.start_time)`.
  - Y value per point: convert meters → display unit at render time. Distance conversion goes through `useDistanceUnit().formatDistance` for tick labels and tooltip; the numeric Y datum can be the converted number (mi or km) so the axis scales. Compute the converted value with the same divisor the context uses (`mi` → /1609.344, `km` → /1000) or expose the converted number directly and format with `formatDistance` on the meters — pick one and be consistent. Y tick formatter renders an integer unit value (e.g. `12 mi`). Tooltip shows `formatDistance(weekMeters) + " " + distanceUnit`.
  - Empty weeks render as zero so gaps are visible. Truncated note identical to the duration chart's, with "sessions".

- [ ] **Step 2: `components/running-time-chart.tsx`**
  - Identical structure; per-week accumulator `{ minutes: number }`, `accumulate: (b, s) => b.minutes += s.duration_seconds / 60`.
  - Y axis uses `formatYTick` (minutes→`Xm`/`Yh`) from `lib/chart-format`; tooltip uses `formatHours`.

- [ ] **Step 3: `components/running-analytics.tsx`** (mirrors `WorkoutsAnalytics`)
  - Props: `{ sessions: RunningSession[] | null; days: number | null; truncated: boolean; fetchLimit: number; distanceUnit: "mi" | "km" }`.
  - Owns `view: "mileage" | "time"` state (default `"mileage"`).
  - Card chrome: `section.rounded-lg.border.border-[var(--border)].bg-[var(--surface)].p-4`. Summary header (same big-tile chrome as `WorkoutsAnalytics`'s header — `text-[10px] uppercase` label + `text-2xl` value): **Total distance** (`formatDistance(totalMeters) + " " + distanceUnit`), **Total time** (`formatHours(totalSeconds/60)`), **Runs** (`sessions?.length ?? 0`). Use `useDistanceUnit()` for `formatDistance` — the component is a client component under the app provider.
  - Icon-button row using the lifted `ViewButton` (Task 2): a route/road icon for Mileage, the clock icon for Time. Add a hand-rolled inline `RouteIcon` (viewBox `0 0 24 24`, `currentColor`, `strokeWidth="1.75"`); reuse a local `ClockIcon` copy in the same idiom (or import if you also lifted icons — inlining is fine for v1 per the SOW).
  - Renders `<RunningMileageChart …/>` or `<RunningTimeChart …/>` based on `view`, forwarding all five props.

- [ ] **Step 4: Verify and commit**

```bash
npm run typecheck && npm run lint && npm run build
```

(Not rendered yet; build type-checks them.)

```bash
git add components/running-mileage-chart.tsx components/running-time-chart.tsx components/running-analytics.tsx
git commit -m "feat: running mileage/time charts + RunningAnalytics card"
```

---

## Task 4: `ActivitiesCombinedChart`

**Files:**

- Create: `components/activities/activities-combined-chart.tsx`

- [ ] **Step 1: Build the stacked combined chart**
  - Props: `{ workouts: Workout[] | null; sessions: RunningSession[] | null; days: number | null; truncated: boolean; fetchLimit: number }`.
  - Build one bucket set spanning the same window over the **union** of both inputs' timestamps. Two clean approaches: call `buildWeeklyBuckets` once over a merged item list tagged by kind, or build the week skeleton from `days` and fold both arrays in. Simplest: accumulator `{ workout_minutes: number; running_minutes: number }`; run `buildWeeklyBuckets` over `workouts` for `workout_minutes` (completed-only, `performed_at`→`ended_at` span / 60000) and over `sessions` for `running_minutes` (`duration_seconds/60`, `start_time`), **using the same `days`** so both produce the same week keys, then zip by `weekKey`. Empty weeks are `{workout_minutes:0, running_minutes:0}`.
  - `<AreaChart>` with two `<Area stackId="activity" …>`: `workout_minutes` stroke/`#3b82f6` (blue), `running_minutes` stroke/`#10b981` (green). `isAnimationActive={false}`. Numeric `XAxis dataKey="t"` like the duration chart; `YAxis tickFormatter={formatYTick}`.
  - `<Legend />` at the top naming the two series ("Lifting", "Running"). `<Tooltip>` `labelFormatter` → `formatWeekRangeFromMonday`; `formatter` returns `[formatHours(value), name]` mapping the two dataKeys to "Lifting" / "Running".
  - Empty-state: when both inputs are empty/zero over the window, render the chrome-less empty copy ("No activity in this window."). `null` for either while loading → "Loading…".
  - Truncation note renders when `truncated` is true ("Showing the most recent N sessions…").
  - Chrome-less: the outer card border + the Overview's summary tiles live on the parent view (Task 5), this component renders just the chart block + note.

- [ ] **Step 2: Verify and commit**

```bash
npm run typecheck && npm run lint && npm run build
git add components/activities/activities-combined-chart.tsx
git commit -m "feat: ActivitiesCombinedChart (stacked lifting vs running minutes)"
```

---

## Task 5: View components — `WorkoutsView`, `RunningView`, `ActivitiesOverviewView`

**Why:** These three are the body slots the shell swaps between. Build them before the shell so the shell's imports resolve. Keep `/workouts` and `/running` pages working as-is during this task — they are converted to redirects in Task 6, so until then the extracted code is duplicated (view component + still-live page). That is fine and keeps each task green.

**Files:**

- Create: `components/activities/workouts-view.tsx`
- Create: `components/activities/running-view.tsx`
- Create: `components/activities/activities-overview-view.tsx`

- [ ] **Step 1: `components/activities/workouts-view.tsx`**

Lift the body of today's `app/(app)/workouts/page.tsx` into a component. Prop contract:

```ts
type WorkoutsViewProps = { days: number | null; displayUnit: "lb" | "kg" };
```

Move verbatim into the view file: the week-grouping helpers (`groupByWeek`, `startOfMonday`, `isoDate`, `formatWeekLabel`, `formatWeekRange`, `formatTotalDuration`, `formatDate`, `formatDuration`), the row component (`WorkoutRow`), the icons (`ChevronIcon`, `PencilIcon`, `TrophyIcon`, `TrashIcon`), and the inline `PaginationFooter` / `EmptyState` / `WeekHeader` / `StatTile`. Changes vs the page:
  - Delete the local `timeframe` state, the `TIMEFRAMES` constant, and the timeframe-pill render block (those now live in the shell). Replace every `TIMEFRAMES.find(...).days` use with the `days` prop.
  - Delete the `getMe` effect and the `user` state; take `displayUnit` from the prop instead.
  - Keep: the `listWorkouts({ since, limit: FETCH_LIMIT })` fetch (now keyed on the `days` prop, derive `since` from it), `listExercises()` (mount only), client-side pagination, expand state, the edit-modal (`editing`) state + `WorkoutModal` mount, delete handler, `WorkoutsAnalytics` mount (still passes `days`, `displayUnit`, `truncated`, `fetchLimit`).
  - Inline error panel idiom unchanged. 401 handling unchanged (`clearToken()` + `router.replace("/login")`).
  - Note: this view's inline `StatTile` (week-header tile, `label`/`value` shape) is **different** from the lifted `components/stat-tile.tsx` (`value`/`label`/`sub`/`tone`). Keep the workouts week-header tile inline/local to avoid a prop-shape clash — do not force it onto the shared component.

- [ ] **Step 2: `components/activities/running-view.tsx`**

Lift the body of today's `app/(app)/running/page.tsx`. Prop contract:

```ts
type RunningViewProps = {
  days: number | null;
  uploadModalOpen: boolean;
  onCloseUploadModal: () => void;
};
```

Changes vs the page:
  - Keep the `getRunningMetrics(token, tz)` call and the 4-tile fixed-bucket banner **unchanged** (it answers "where am I now?" regardless of window).
  - Replace cursor pagination with timeframe-filtered fetching: `since = days !== null ? new Date(Date.now() - days*86400_000).toISOString() : undefined`, `until = days !== null ? new Date().toISOString() : undefined`; call `listRunningSessions(token, { since, until, limit: 200 })`. When `days === null`, omit both. Refetch on `days` change and after upload.
  - Remove the "Load more" button and `nextBefore` / `loadingMore` state. Compute `truncated = (sessions?.length ?? 0) >= 200`.
  - Insert `<RunningAnalytics sessions={sessions} days={days} truncated={truncated} fetchLimit={200} distanceUnit={unitLabel} />` **above** the run list and **below** the 4-tile banner.
  - The Upload TCX button moves to the shell toolbar (Task 6). The modal stays here: render `<UploadTCXModal …/>` when `uploadModalOpen` is true; on close call `onCloseUploadModal()`; `handleUploaded` keeps direct access to `refetch` (metrics + sessions) and the optimistic insert + toast.
  - Render order inside the view: (1) 4-tile banner, (2) `<RunningAnalytics/>`, (3) run list (`<RunListRow/>` items — verify the import still resolves; `RunListRow` stays in `app/(app)/running/_components/`), (4) `UploadTCXModal` when open. Keep the existing `EmptyState`.
  - 401 handling via the existing `handleAuthError` pattern.

- [ ] **Step 3: `components/activities/activities-overview-view.tsx`**

New digest. Prop contract:

```ts
type ActivitiesOverviewViewProps = {
  days: number | null;
  displayUnit: "lb" | "kg";
  distanceUnit: "mi" | "km";
};
```

  - Fetch on mount + on `days` change: `Promise.all([ listWorkouts(token, { since, limit: 100 }), listRunningSessions(token, { since, until, limit: 200 }) ])`. `since`/`until` derived from `days` exactly as the running view; for `days === null`, omit. 401 → `clearToken()` + `router.replace("/login")`; other errors → inline red panel (shell renders none).
  - Compute (memoized on `workouts` + `sessions`): `totalWorkoutMinutes` (completed-only, ended−performed), `totalRunningMinutes` (Σ `duration_seconds/60`), `totalMinutes`, `workoutCount`, `runCount`, `totalSessions = workoutCount + runCount`, `totalVolume = Σ workoutVolume(w, displayUnit)`, `totalMileageMeters = Σ session.distance_meters`, `prCount = Σ w.personal_records_set.length`, `avgSessionMinutes = totalSessions ? totalMinutes/totalSessions : 0`.
  - Render top→bottom using the lifted `components/stat-tile.tsx`:
    1. Hero row — `grid grid-cols-2 md:grid-cols-4 gap-3`, four `StatTile`s: **Total time** (`formatHours(totalMinutes)`), **Total sessions** (`String(totalSessions)`), **Workouts** (`String(workoutCount)`), **Runs** (`String(runCount)`).
    2. `<ActivitiesCombinedChart workouts={workouts} sessions={sessions} days={days} truncated={…} fetchLimit={…} />` wrapped in the card chrome (outer border + a small summary header is optional; SOW says the combined chart is chrome-less and the view owns the card border — wrap it in `section.rounded-lg.border…p-4`).
    3. Secondary row — `grid grid-cols-2 md:grid-cols-4 gap-3`, four `StatTile`s: **Total volume** (`totalVolume.toLocaleString(undefined,{maximumFractionDigits:0}) + " " + displayUnit`), **Total mileage** (`formatDistance(totalMileageMeters) + " " + distanceUnit`), **PRs** (`String(prCount)`), **Avg session** (`totalSessions ? formatHours(avgSessionMinutes) : "—"`).
  - `formatDistance` via `useDistanceUnit()`. `formatHours` from `lib/chart-format`. `workoutVolume` from `lib/workout-volume`.
  - For `truncated`/`fetchLimit` on the combined chart: pass `truncated = workouts.length >= 100 || sessions.length >= 200` and a representative `fetchLimit` (e.g. 200) — the note copy is generic.

- [ ] **Step 4: Verify and commit**

```bash
npm run typecheck && npm run lint && npm run build
git add components/activities/workouts-view.tsx components/activities/running-view.tsx components/activities/activities-overview-view.tsx
git commit -m "feat: WorkoutsView, RunningView, ActivitiesOverviewView components"
```

---

## Task 6: Activities page shell + routing + sidebar

**Why:** Wires the three views into one URL-backed shell, collapses the sidebar, and redirects the old parent routes. This is the task that makes the feature visible.

**Files:**

- Create: `app/(app)/activities/page.tsx`
- Modify: `app/(app)/workouts/page.tsx` → redirect stub
- Modify: `app/(app)/running/page.tsx` → redirect stub
- Modify: `app/(app)/running/[id]/page.tsx` → back-link retarget
- Modify: `app/(app)/workouts/[id]/page.tsx` → verify (see note)
- Modify: `components/sidebar.tsx` → ActivityIcon, nav consolidation, delete unused icons

- [ ] **Step 1: `app/(app)/activities/page.tsx` shell**

Mirror the nutrition `Suspense` wrapper pattern exactly:

```tsx
"use client";
import { Suspense, useEffect, useMemo, useState } from "react";
import { useRouter, useSearchParams } from "next/navigation";
// auth, getMe, view imports, ToolbarButton, StatTile not needed here
```

  - Outer default export wraps `<ActivitiesPageInner/>` in `<Suspense fallback={null}>`.
  - Inner: read `view` from `useSearchParams().get("view")`; `view = (raw === "workouts" || raw === "running") ? raw : "overview"`.
  - `timeframe` state (`"7d"|"30d"|"90d"|"all"`, default `"30d"`) with the `TIMEFRAMES` constant (id/label/days) from the SOW; `days = useMemo(...)`.
  - One-time `getMe(token)` for `user` → `displayUnit = user?.weight_unit ?? "lb"`. `distanceUnit` comes from `useDistanceUnit().unitLabel`. 401 → `clearToken()` + `/login`; missing token → `/login`.
  - `setView(next)`: `next === "overview" ? router.replace("/activities") : router.replace(\`/activities?view=${next}\`)`.
  - `uploadModalOpen` state owned here (for the running Upload TCX button in the toolbar).
  - Render: header (`<h1>Activities</h1>` + timeframe pills — the exact rounded-full pill markup from today's `/workouts` header, driven by `timeframe`/`setTimeframe`); a toolbar row (`flex items-center justify-between`): left group (`flex gap-5`) renders the Upload TCX `ToolbarButton` only when `view === "running"` (onClick → `setUploadModalOpen(true)`), right group (`flex gap-5`) three `ToolbarButton`s — Overview / Workouts / Running — each `active={view === …}`, onClick → `setView(…)`; a `border-b border-[var(--border)]` separator under the toolbar; then the body switch.
    - `ToolbarButton` requires an `icon` — give each tab a small inline glyph in the existing idiom (e.g. a pulse/Activity glyph for Overview, the dumbbell for Workouts, a run glyph for Running, an upload arrow for Upload TCX). Labels: Overview / Workouts / Running / Upload TCX. On a 375px viewport the labels collapse to icon-only via the `hidden sm:inline` already baked into `ToolbarButton`, so all controls fit — no horizontal scroll.
  - Body: `view === "overview"` → `<ActivitiesOverviewView days={days} displayUnit={displayUnit} distanceUnit={distanceUnit} />`; `workouts` → `<WorkoutsView days={days} displayUnit={displayUnit} />`; `running` → `<RunningView days={days} uploadModalOpen={uploadModalOpen} onCloseUploadModal={() => setUploadModalOpen(false)} />`.
  - Layout chrome (`<main className="flex flex-1 flex-col overflow-hidden">`, header border, `flex-1 overflow-y-auto px-6 py-6`, `mx-auto max-w-3xl`) mirrors the existing pages so the three views sit in the same column width they do today. (Overview/the four-up grids may use a wider `max-w-4xl`; pick the width that keeps the `md:grid-cols-4` tiles readable — match what `/running` used.)

- [ ] **Step 2: Convert parent routes to redirects**

`app/(app)/workouts/page.tsx` becomes, in full:

```tsx
import { redirect } from "next/navigation";

export default function WorkoutsPage() {
  redirect("/activities?view=workouts");
}
```

`app/(app)/running/page.tsx` likewise → `redirect("/activities?view=running")`. **No `"use client"`** — these are server components. Remove all now-unused imports/helpers that lived in the old page bodies (they now live in the view components from Task 5).

- [ ] **Step 3: Retarget detail back-links**

`app/(app)/running/[id]/page.tsx`: the four `/running` back-link sites (today at lines ~135 `router.push("/running")`, ~147, ~160, ~179 `href="/running"`) become `/activities?view=running`. Leave `/running/{id}` self-references untouched.

`app/(app)/workouts/[id]/page.tsx`: **grep first** — `grep -n '/workouts' app/(app)/workouts/[id]/page.tsx`. Today this file's only nav link is `← Back to Personal Records` (`href="/personal-records"`); there are **no** `/workouts` back-links to retarget. Confirm the grep is empty for `"/workouts"` back-links and make no change beyond that confirmation. (If a `/workouts` parent back-link does exist, retarget it to `/activities?view=workouts`; never rewrite `/workouts/{id}` self-links.)

- [ ] **Step 4: `components/sidebar.tsx`**
  - Remove the `{ href: "/workouts", … }` and `{ href: "/running", … }` NAV entries.
  - Insert `{ href: "/activities", label: "Activities", icon: <ActivityIcon /> }` in the slot Workouts occupied (between Chat and Exercises).
  - Add `ActivityIcon` — inline SVG, viewBox `0 0 24 24`, `currentColor` stroke, `strokeWidth="1.75"`, `strokeLinecap="round"`, `strokeLinejoin="round"`, a pulse/heartbeat polyline (baseline → peak → valley → peak), e.g. `<polyline points="3 12 7 12 10 5 14 19 17 12 21 12" />`.
  - Delete the now-unreferenced `DumbbellIcon` and `RunIcon` function definitions from this file.
  - The active-highlight logic (prefix-match + exact) is unchanged, so `/activities?view=workouts` still lights the entry (pathname is `/activities`).

- [ ] **Step 5: Verify and commit**

```bash
npm run typecheck && npm run lint && npm run build
```

Run the SOW's full manual golden-path + edge-case checklist in `npm run dev`: cold-load `/activities` (Overview, 30d, four hero tiles, combined chart), tab to Workouts/Running (URL updates, underline moves, Upload TCX appears only on Running), timeframe changes refetch the active view, `/workouts` and `/running` redirect, detail back-links land on `/activities?view=…`, `/activities?view=garbage` falls back to Overview, sidebar shows one Activities entry, 375px viewport fits the toolbar. Then:

```bash
git add app/\(app\)/activities/page.tsx app/\(app\)/workouts/page.tsx app/\(app\)/running/page.tsx \
        app/\(app\)/running/\[id\]/page.tsx components/sidebar.tsx
git commit -m "feat: Activities page shell, route redirects, sidebar consolidation"
```

---

## Wrap-up

- Final full verification: `npm run typecheck && npm run lint && npm run build`.
- The web PR description includes the SOW path and screenshots of Overview / Workouts / Running.
- The `prog-strength-docs` PR flips this SOW's `status` to `shipped` (separate task, see the workflow).

## Notes / decisions deferred to the implementer

- **Distance Y-axis datum**: the mileage chart can either store converted display-unit numbers per bucket (simplest for axis scaling) or store meters and convert in formatters. Either is acceptable; keep tick + tooltip consistent with `formatDistance`.
- **Combined-chart truncation**: the note copy is generic ("most recent N sessions"); a single `fetchLimit` (200) is fine even though workouts cap at 100.
- **Overview card width vs the 3-up column**: match whatever max-width keeps the `md:grid-cols-4` tiles from cramping (the running page used `max-w-3xl`; the overview's four-up may read better at `max-w-4xl`). Implementer's call; keep it consistent across the three views' container.
