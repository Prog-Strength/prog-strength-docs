# Calendar Day Digest & Weekly Overview Implementation Plan

> **For agentic workers:** Implement this plan task-by-task. Every task is implemented by a subagent, then spec-reviewed and code-quality-reviewed before moving on. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign the calendar page (`prog-strength-web`) around three additions — a **day digest** panel below the grid, a **weekly overview column** to the right of the grid, and an **expanded running stat-tile set** (Avg Pace, Longest Run) — while keeping the existing Monday-first month grid as the at-a-glance surface. No backend changes; everything reads from the already-fetched 6-week window.

**Architecture:** All work lives in `prog-strength-web`. The calendar page (`app/(app)/calendar/page.tsx`) gains a `selected` date state (defaults to today, re-anchors to the 1st on cursor change), two new `useMemo`-derived aggregates (`weeklyStats`, expanded running stats), and renders new components from a fresh `components/calendar/` directory. The shared workout body is factored out of `WorkoutDetailsModal` so the inline workout digest renders from one source of truth.

**Tech stack:** Next.js 16 (App Router, client components), React 19, TypeScript, Tailwind v4, Vitest + React Testing Library. Read `node_modules/next/dist/docs/` before using any unfamiliar Next API — this is a non-standard Next build.

**Repo path:** `/workspace/prog-strength-web`

**Source SOW:** `prog-strength-docs/sows/calendar-day-digest-and-weekly-overview.md`

**Conventions discovered in the codebase (reuse, do not reinvent):**
- `useDistanceUnit()` (`lib/distance-unit-context.tsx`) exposes `formatDistance(meters)` (bare numeric, caller adds `unitLabel`), `formatPace(secPerKm)` (`m:ss`), `unit`/`unitLabel`.
- `formatDuration(seconds)` from `lib/format.ts` → `m:ss` / `h:mm:ss`.
- `RunningSession` carries `start_time`, `distance_meters`, `duration_seconds`, `avg_pace_sec_per_km`, `avg_heart_rate_bpm`, `elevation_gain_meters`, `name`. **No location/route field exists** → graceful omit.
- `Workout` carries `performed_at`, `ended_at`, `name`, `exercises[]`. `WorkoutSet` is `{reps, weight, unit}` — **there is no RPE field** → graceful omit RPE.
- `hasMeaningfulName(name)` from `components/workout-details.tsx` is the shared rule for falling back from auto-generated `"Workout - <date>"` names.
- `localDateKey`, `buildMonthGrid`, `gridFetchBounds`, `formatTotalDuration` already exist inside `page.tsx`. Export/reuse the helpers the new components need rather than duplicating.
- Pill color language: workout = `var(--accent)`, run = `teal-500/300`.
- Existing test style: Vitest globals, `@testing-library/react`, jsdom. `next/navigation` and context providers must be mocked/wrapped in component tests.

---

## Phase 1 — Shared building blocks

### Task 1: Factor out `WorkoutDetailsBody` + optional total-volume footer

**Files:**
- Modify: `components/workout-details.tsx`
- Modify: `app/(app)/workouts/[id]/page.tsx`
- Modify: `components/activities/workouts-view.tsx`

- [ ] **Step 1:** Rename the existing `WorkoutDetails` component to `WorkoutDetailsBody`. Keep `WorkoutDetailsModal` and `hasMeaningfulName` exactly as-is (the modal now renders `<WorkoutDetailsBody />`).
- [ ] **Step 2:** Add an optional `showTotalVolume?: boolean` prop (default `false`) to `WorkoutDetailsBody`. When true, render a total-volume footer line at the bottom: total volume = Σ(`reps × weight`) across every set of every exercise, formatted as `"{number} {unit}"` using the workout's predominant set unit (lb/kg). If all sets are bodyweight (weight 0) the volume is 0 — still render `"0 lbs"` (or the unit) so the digest test has a deterministic target. Use a small muted label like `Total volume`.
- [ ] **Step 3:** Update the two external consumers (`workouts/[id]/page.tsx`, `activities/workouts-view.tsx`) to import/render `WorkoutDetailsBody` instead of `WorkoutDetails`. They pass no `showTotalVolume` so their output is unchanged.
- [ ] **Step 4:** `npm run typecheck` and `npx vitest run` stay green.

**Acceptance:** Modal and existing surfaces render identically; `WorkoutDetailsBody` is the single exported body component; total-volume footer is opt-in.

---

### Task 2: `RunDigest` inline dropdown content

**Files:**
- Create: `components/calendar/run-digest.tsx`
- Create: `components/calendar/run-digest.test.tsx`

- [ ] **Step 1:** `RunDigest({ run }: { run: RunningSession })`. Renders a compact stat list:
  - Distance — `${formatDistance(run.distance_meters)} ${unitLabel}`
  - Duration — `formatDuration(run.duration_seconds)`
  - Avg pace — `${formatPace(run.avg_pace_sec_per_km)} /${unitLabel}` (omit row if pace is null)
  - Avg HR — `${run.avg_heart_rate_bpm} bpm` (omit if null)
  - Elevation gain — `${Math.round(run.elevation_gain_meters)} m` for km-unit users / `ft` conversion for mi-unit users (omit if null). Use `unit === "mi"` → feet (`meters * 3.28084`, rounded), else meters.
  - Location/route label — **no field exists**; omit entirely (leave a code comment noting the graceful-omit).
  - A `View full run details →` link (`next/link`) to `/running/${run.id}`.
- [ ] **Step 2:** Use `useDistanceUnit()` for all unit math. No charts, no splits.
- [ ] **Step 3:** Test (`run-digest.test.tsx`): renders required fields (distance, duration); gracefully omits avg HR / elevation when null; includes the `View full run details` link pointing at `/running/<id>`. Wrap render in a `DistanceUnitProvider` or mock `useDistanceUnit`.

---

### Task 3: `WorkoutDigest` inline dropdown content

**Files:**
- Create: `components/calendar/workout-digest.tsx`
- Create: `components/calendar/workout-digest.test.tsx`

- [ ] **Step 1:** `WorkoutDigest({ workout, exerciseMap })` renders `<WorkoutDetailsBody workout={workout} exerciseMap={exerciseMap} showTotalVolume />`. Thin wrapper — the source of truth is the shared body (Task 1).
- [ ] **Step 2:** Test: renders an exercise list with a set/rep/weight line; the total-volume footer shows the computed volume. Provide a small `Workout` + `exerciseMap` fixture.

---

### Task 4: `WorkoutBanner` and `RunBanner`

**Files:**
- Create: `components/calendar/workout-banner.tsx`
- Create: `components/calendar/run-banner.tsx`
- Create: `components/calendar/workout-banner.test.tsx`
- Create: `components/calendar/run-banner.test.tsx`

- [ ] **Step 1:** Each banner is a `<div role="group">` containing two buttons:
  - **Banner body** button (icon + title + key stats), `onClick` → calls an `onNavigate()` callback (the page passes `() => router.push("/workouts/<id>")` or `/running/<id>`). Icon follows pill color language (workout = accent, run = teal).
  - **Chevron** button on the right with `aria-expanded={open}` and `aria-controls={dropdownId}`; `onClick` toggles local `open` state. Does NOT navigate.
  - When `open`, render the dropdown (`WorkoutDigest` / `RunDigest`) below, lazily (closed banner does not mount the digest). Use a stable `useId()` for `aria-controls`/the dropdown's `id`.
- [ ] **Step 2:** Banner title: workout uses `hasMeaningfulName(name) ? name : <time>`; run uses `name?.trim() || runFallbackName(start_time)` style. Key stats: workout → time + activity count or duration; run → distance + pace. Keep focus rings (`focus-visible`) on both buttons.
- [ ] **Step 3:** Support a controlled-open affordance so the page can auto-expand a banner when its pill was clicked: accept optional `defaultOpen?: boolean` (or expose open state via props). Simplest: `defaultOpen` initializes local state. The page keys the banner so a pill click can force it open (see Task 7).
- [ ] **Step 4:** Tests for both: clicking the body fires the navigate callback (and does NOT toggle the dropdown); clicking the chevron toggles `aria-expanded` and shows/hides the dropdown content. Mock `next/navigation`'s `useRouter` if needed; prefer testing via the `onNavigate` callback prop.

---

### Task 5: `DayDigest` panel

**Files:**
- Create: `components/calendar/day-digest.tsx`
- Create: `components/calendar/day-digest.test.tsx`

- [ ] **Step 1:** `DayDigest({ date, events, exerciseMap, autoExpandId })`. `date` is a `Date` (or the local-date key + a Date); `events` is the `CalendarEvent[]` for that day (already start-time sorted). Export the `CalendarEvent` type from `page.tsx` (or a shared module) so the digest and page agree on shape.
- [ ] **Step 2:** Header: long-form date (`"Friday, June 8, 2026"`) + at-a-glance count line (`"2 activities · 1 run · 1 lift"`, pluralized correctly; omit zero parts).
- [ ] **Step 3:** Body: one banner per event in start-time order — `WorkoutBanner` for `kind: "workout"`, `RunBanner` for `kind: "run"`. Pass `defaultOpen` when `event` matches `autoExpandId` (the pill-clicked activity).
- [ ] **Step 4:** Empty state when `events.length === 0`: quiet message `"No activities on {short date}. Plan one →"` with a link to an existing activity surface (`/activities` — there is no dedicated create route; link there). Leave a comment referencing the SOW non-goal.
- [ ] **Step 5:** Tests: renders the date header; renders one banner per event; renders banners in start-time order; renders the empty state when there are no events.

---

### Task 6: `WeeklyTile` and `WeeklyOverviewColumn`

**Files:**
- Create: `components/calendar/weekly-tile.tsx`
- Create: `components/calendar/weekly-overview.tsx`
- Create: `components/calendar/weekly-overview.test.tsx`

- [ ] **Step 1:** Define the weekly-stat shape `{ weekStart: Date; activities: number; liftMinutes: number; runMeters: number; runMinutes: number }` (export from a shared module or `weekly-overview.tsx`).
- [ ] **Step 2:** `WeeklyTile({ week, isCurrent })` — compact card showing, each omitted when its value is 0:
  - Sessions count (`"3 activities"` / `"1 activity"`)
  - Lift time (`formatTotalDuration(liftMinutes)`) — only if `liftMinutes > 0`
  - Run distance (`${formatDistance(runMeters)} ${unitLabel}`) — only if `runMeters > 0`
  - Run time (`formatTotalDuration(runMinutes)`) — only if `runMinutes > 0`
  - `isCurrent` → subtle accent border.
- [ ] **Step 3:** `WeeklyOverviewColumn({ weeks, todayKey })` renders the right-side stack on `md+` (`hidden md:flex`, one tile per week vertically). The current week is the one whose 7-day key set contains `todayKey`.
- [ ] **Step 4:** Tests: produces one tile per week; omits zero-value rows from a tile; accents the current week's tile.

> Note: the `<md` inline-chip variant is wired in Task 8 (it must interleave with grid rows). Build `WeeklyOverviewColumn` to also export/render a `WeeklyChip` (compact single-line variant) for that use, or expose the tile in a chip mode.

---

## Phase 2 — Page integration

### Task 7: Selected-date state, expanded stat tiles, click model, day-digest wiring

**Files:**
- Modify: `app/(app)/calendar/page.tsx`

- [ ] **Step 1:** Export `CalendarEvent`, `localDateKey`, and `formatTotalDuration` from `page.tsx` (or move shared helpers to `components/calendar/shared.ts`) so the new components consume one definition. Keep `page.tsx` under 600 lines — move `DayCell`/inline components out only if needed.
- [ ] **Step 2:** Add `const [selected, setSelected] = useState<string>(() => localDateKey(new Date()))` and a `useEffect` keyed on `cursor` that re-anchors `setSelected(localDateKey(new Date(cursor.year, cursor.month, 1)))`. Add `autoExpand` state `{ id: string } | null` cleared on selection change.
- [ ] **Step 3:** Expanded stat tiles: add a `monthRunningStats` memo computing Avg Pace (total run seconds / total run meters → format via `formatPace`-style mm:ss per unit; only when `runMeters > 0`) and Longest Run (`Math.max` of cursor-month run distances; format via `formatDistance`). Re-flow the tile grid from `grid-cols-2 md:grid-cols-4` to **`grid-cols-2 md:grid-cols-3 lg:grid-cols-6`** with six tiles in this order: Lift Time, Run Time, Run Distance, Avg Pace, Longest Run, Activities. The two running-quality tiles render but get a `hidden` class (no layout-shift) when the month has no runs.
- [ ] **Step 4:** Update `DayCell`: clicking the cell whitespace selects the day (`setSelected(key)`) and scrolls the digest into view (`digestRef.current?.scrollIntoView({ behavior: "smooth", block: "nearest" })`). The `+N more` overflow becomes a button that also selects + scrolls. A workout/run pill click selects the day, scrolls, AND sets `autoExpand` to that activity's id (auto-expand its digest banner). Add hover states (pill: background lift; cell: border highlight) and selected/today visual states (selected = filled accent bg; today = ring; both can stack). Pass `selected`/`onSelect` down to `DayCell`.
- [ ] **Step 5:** Render `<DayDigest>` below the grid (with the `digestRef`), passing the selected date, `eventsByDate.get(selected) ?? []`, `exerciseMap`, and `autoExpand?.id`. Remove the old `WorkoutDetailsModal` pill-opens-modal behavior (pills now drive the digest) — but keep the modal import only if still used; otherwise delete the `viewing` state and modal. (Navigation to full pages now happens from the banner body.)
- [ ] **Step 6:** `aria-expanded`/focus rings on new interactive elements; preserve existing keyboard accessibility. `npm run typecheck` + `npx vitest run` green.

### Task 8: Weekly overview layout integration

**Files:**
- Modify: `app/(app)/calendar/page.tsx`

- [ ] **Step 1:** Add the `weeklyStats` memo exactly per the SOW (iterate `days` in 7-cell slices, bucket `workouts`/`runs` by `localDateKey` membership, summing `activities`, `liftMinutes` from `ended_at - performed_at`, `runMinutes`, `runMeters`).
- [ ] **Step 2:** Bump the outer container `max-w-5xl` → `max-w-6xl`. Wrap the weekday-header + grid in a flex container: `<div className="flex flex-col gap-4 md:flex-row md:items-start">` with the grid in a `flex-1` block and `<WeeklyOverviewColumn weeks={weeklyStats} todayKey={todayKey} />` beside it (`hidden md:flex`).
- [ ] **Step 3:** `<md` chips: render a `WeeklyChip` (compact summary) above each week's row inside the grid block, hidden at `md+` (`flex md:hidden`). Interleave by rendering rows week-by-week (chunk `days` into 7s) with the chip before each 7-cell row, OR render chips as full-width grid items spanning the row. Keep the existing 6×7 visual grid intact at `md+`.
- [ ] **Step 4:** `npm run typecheck` + `npx vitest run` green; no horizontal scroll on narrow viewports.

### Task 9: Page-level integration tests

**Files:**
- Create: `app/(app)/calendar/page.test.tsx`

- [ ] **Step 1:** Mock `@/lib/api` (`listWorkouts`, `listRunningSessions`, `listExercises`), `@/lib/auth` (`getToken` → a token), and `next/navigation` (`useRouter`). Wrap render in `DistanceUnitProvider` (or mock `useDistanceUnit`). Seed deterministic workouts/runs for the current month.
- [ ] **Step 2:** Tests:
  - Selecting a day updates the digest (header + banners reflect that day's events).
  - Cursor change (click Previous/Next) re-anchors the selection to the 1st of the new month (digest header shows the 1st).
  - Clicking a pill selects the day AND auto-expands the matching digest banner (`aria-expanded="true"` on that banner).
  - The Avg Pace / Longest Run tiles are hidden when the month has no runs.
  - The weekly column reflects the visible window's events (a week with N activities shows `"N activities"`).
- [ ] **Step 3:** Full `npx vitest run`, `npm run typecheck`, `npm run lint`, `npm run format:check` all green.

---

## Rollout

Web-only, no API/migration. Single PR against `prog-strength-web`. No coordination window — merge whenever.

**Hand-test after deploy:**
- Open `/calendar`; today is selected by default; the digest shows today's activities (or the empty state).
- Click a day with a morning run + afternoon lift; banners appear in start-time order; chevron expands the inline digest without navigating; banner body navigates to the activity page.
- Click a workout pill; the day selects, the page scrolls to the digest, and that workout's banner is auto-expanded.
- Verify the weekly column on `md+` shows per-week sessions/lift-time/run-distance/run-time with zero rows omitted and the current week accented; on a narrow viewport the chips stack above each row with no horizontal scroll.
- Switch a month with runs ↔ a month without; Avg Pace and Longest Run tiles appear/hide without layout shift; tile row is 2-up mobile / 3-up tablet / 6-up desktop.
- Toggle the distance unit; digest + weekly + tiles all reflect mi/km.

## Out of scope (per SOW non-goals)

New endpoints, per-day fetches, run charts/splits in the digest, multi-day/range selection, quick-edit, print/export, drag-and-drop, custom date picker, cross-month digest persistence, run location label (no field yet).
