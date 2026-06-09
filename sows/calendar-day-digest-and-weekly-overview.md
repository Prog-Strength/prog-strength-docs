---
status: shipped
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Calendar Page: Day Digest, Weekly Overview, and Running Stats

**Status**: Shipped · **Last updated**: 2026-06-09

## Introduction

The calendar page has done one job since it shipped: render a Monday-first month grid with workouts and runs as colored pills. That's good for the "did I train this week?" glance, but it's the only thing the page is good at. Three friction points have built up:

First, every time the user wants to know what they actually did on a given day, they have to click into the activity's modal or navigate away. A morning run plus an afternoon lift is three clicks to inspect from the calendar: open the run page, hit back, open the workout modal. The day's shape — "long run, easy lifts, both warm" — is data the page already has but doesn't surface.

Second, running is now a first-class activity, but the page is still workout-biased. Two of the four month-level stat tiles cover running (Run Time, Run Distance); the other two (Lift Time, Activities) are general. What's missing is the *running-quality* signals the dashboards on `/running` already surface — average pace, longest run — which would let the calendar answer "is my running getting better?" without leaving the page.

Third, the calendar has no weekly perspective. A training week is the unit a runner or lifter actually plans around, and the page renders six of them per view but says nothing about any of them. Weekly volume, weekly distance, sessions per week — all of these matter at the planning layer and none of them are visible on the calendar today.

This SOW redesigns the calendar page around three additions: a **day digest** that opens below the grid when a date is selected, showing every activity on that day with an expandable workout breakdown and a high-level run summary; a **weekly overview column** to the right of the grid with per-week aggregated stats; and an **expanded running tile set** that adds Avg Pace and Longest Run to the existing four. The pill grid keeps its existing role as the at-a-glance surface. Everything else gets a meaningful answer on the same page.

## Proposed Solution

The page keeps its current shape — header with month label + Today/prev/next, stat tiles, weekday header, six-week grid — and grows three new regions:

**1. Day digest below the grid.** Clicking any cell selects that date and renders a `DayDigest` panel underneath the grid. Selected state shows as a filled background on the cell (visually distinct from the existing "today" ring). The panel header is the long-form date ("Friday, June 8, 2026") plus the day's at-a-glance count ("2 activities · 1 run · 1 lift"). Below the header, one **banner per activity** ordered by start time.

Each banner has two click targets that behave differently:

- **Banner body** (icon + title + key stats) → navigates to the activity-specific page. Workouts go to `/workouts/[id]`; runs go to `/running/[id]`. Click on the banner is the "I want the full page" affordance.
- **Chevron on the right** → toggles an inline **digest dropdown**. Doesn't navigate. The dropdown is where the in-place data lives.

The digest dropdown content differs by activity type:

- **Workout dropdown** renders the same information the existing `WorkoutDetailsModal` (driven by `components/workout-details.tsx`) shows: exercise list with sets, reps, weights, RPE, notes, and total volume. This is "everything in the activities list item dropdown" — the user gets the full breakdown without opening a modal.
- **Run dropdown** renders high-level stats only — distance, duration, average pace, average heart rate, elevation gain, and a short text route or location label if present. No splits, no charts. A `View full run details →` link inside the dropdown navigates to `/running/[id]` for the splits/HR/elevation graphs (which are heavy and already live on the dedicated page).

When the page first loads, the selected date defaults to **today**. If today has no activities, the panel shows a quiet empty state ("No activities on Friday, June 8. Plan one →" with a link to the workout/run creation flow). When the user navigates to a previous month, the selected date follows the cursor (defaults to the first day of the cursored month).

**2. Weekly overview column to the right of the grid.** The grid widens from `max-w-5xl` to `max-w-6xl` and gains a 7th visual column (technically a sibling block) on the right: one tile per week, vertically aligned with that week's row. Each weekly tile is a compact card showing:

- Sessions count ("3 activities")
- Lift time ("2h 15m") — present only if the week had any lifts
- Run distance ("12.5 mi" / "20.1 km" — respects the user's distance unit)
- Run time ("1h 40m") — present only if the week had any runs

Stats with zero values for the week are hidden, not shown as "—", to keep the column visually light. The current week (the one containing today) gets a subtle accent border so a glance at "this week so far" is immediate.

On screens narrower than `md` (768px) the weekly column collapses: each weekly tile becomes a compact summary chip that sits **above** that week's row of cells rather than beside it. Same data, vertical stacking, no horizontal scroll.

**3. Expanded running stat tiles.** The current four-tile row grows to six:

- Lift Time
- Run Time
- Run Distance
- Avg Pace (mm:ss per mi/km, only shown if any runs exist this month)
- Longest Run (distance, only shown if any runs exist this month)
- Activities

Layout: 2-up on mobile, 3-up on tablet, 6-up from `lg`. The conditional running tiles render as empty placeholders that hide via CSS (`hidden` class) when there's no data; this avoids layout shift when switching between a month with runs and a month without.

**Visual polish.** Several small refinements ride along:

- Pills get hover state (subtle background lift) since they're now clickable inside a wider interaction model.
- Day cells get hover state too (border highlight) so the "selectable cell" affordance is discoverable.
- The selected date gets a filled accent background; today gets a ring; the two states stack visually if today is selected.
- The "+N more" overflow on a busy day cell becomes clickable: clicking it both selects the day AND scrolls the day-digest panel into view.
- Banner icons follow the existing pill color language (workout = lift accent, run = run accent) so a glance at the digest reads "what kind of day was it" from color alone.

**Click model summary** for the calendar grid after this work:

| Click target | Action |
| --- | --- |
| Day cell (whitespace) | Select day; scroll digest into view |
| Workout pill | Select day; scroll digest into view; expand that workout's digest |
| Run pill | Select day; scroll digest into view; expand that run's digest |
| "+N more" overflow | Select day; scroll digest into view |
| Digest banner body | Navigate to activity page |
| Digest banner chevron | Toggle that banner's dropdown |
| Digest dropdown's "View full" link (runs only) | Navigate to run page |

The shift is intentional: pills become *day-selectors with activity context*, not the only path into activity detail. The activity-specific pages are still one click away from the digest, but the most common need ("what did I do that day?") is answered in place.

## Goals and Non-Goals

### Goals

- Refactor `app/(app)/calendar/page.tsx` to support a selected-date state, defaulting to today, that persists across month cursor changes by snapping to the first day of the new cursored month.
- Add a `DayDigest` component rendered below the grid when a date is selected. The component takes the selected date, the day's events, and the exercise map; it produces a header + one banner per activity in start-time order.
- Add `ActivityBanner` (or two variants — `WorkoutBanner` and `RunBanner`) with two interaction targets: a navigation-on-click body and a digest-toggle chevron on the right.
- Add `WorkoutDigest` dropdown rendering the same content as `components/workout-details.tsx` (the modal's body). Factor the shared layout out of `WorkoutDetailsModal` into a `WorkoutDetailsBody` so both the modal and the inline digest render from one source of truth.
- Add `RunDigest` dropdown rendering distance, duration, average pace, average heart rate, elevation gain, and a route/location label if available. Include a `View full run details →` link to `/running/[id]`. No charts or splits.
- Add a `WeeklyOverviewColumn` block sibling to the grid on `md+` screens, showing one `WeeklyTile` per week vertically aligned with that week's row. Each tile shows sessions count, lift time, run distance, and run time, omitting zero-value rows.
- On `<md` screens, render each `WeeklyTile` as a compact chip stacked above its week's row instead of beside it.
- Add Avg Pace and Longest Run tiles to the existing stat row, conditional on the month having any runs. Re-flow the tile grid to 2 / 3 / 6 columns at the existing tailwind breakpoints.
- Update the day-cell click handler so the cell whitespace AND the "+N more" overflow both select the day and scroll the digest into view via `scrollIntoView({ behavior: "smooth", block: "nearest" })`.
- Add hover states to pills and cells (background lift on pill, border highlight on cell).
- Visually distinguish selected (filled background), today (ring), and selected-AND-today (filled background plus ring) on day cells.
- All existing keyboard accessibility (focus rings, tab order on pills) carries forward; the new banners and chevron toggles get matching `aria-expanded` and focus rings.
- Maintain the existing data-fetching shape: `listWorkouts` + `listRunningSessions` over the visible 6-week window. The digest pulls from already-fetched data; no per-day fetches.
- Cover the page's new behaviors with component tests: day selection updates the digest, banner click navigates, chevron toggle expands without navigating, weekly column reflects the visible window's events, the new tiles hide when their data is absent.

### Non-Goals

- **Activity creation flows.** The empty-state CTA ("Plan one →") links to the existing workout/run create routes; this SOW does not introduce new creation surfaces.
- **Run charts in the digest.** Splits, HR over time, elevation profile, and pace curve stay on `/running/[id]` only — they're too heavy to render inline for every selected run and would require their own data fetch.
- **Multi-day selection or range selection.** One date at a time. Range-based stats are already covered by the new weekly overview column.
- **Per-day fetches.** The page already over-fetches a 6-week window; the digest reads from that cache. No new endpoints.
- **New backend endpoints.** Everything the digest needs is in the existing `Workout` and `RunningSession` payloads. If a field is missing (e.g., a run's "location label"), the digest gracefully omits it rather than triggering a new API.
- **Quick-edit from the digest.** Edits still happen on the activity-specific pages. The digest is read-only.
- **Print/export view.** Out of scope.
- **Drag-and-drop reordering or scheduling.** The calendar remains a view of past activity, not a planning surface.
- **Cross-month digest persistence.** When the user moves to a new month, the digest re-anchors to the first day of that month rather than trying to preserve the prior selection's absolute date. Simpler model, no edge cases at month boundaries.
- **Custom date picker.** Date selection happens by clicking a cell. No popup picker; this is a calendar page.
- **Mobile-specific digest layout differences beyond the weekly chip stacking.** The digest itself is responsive but doesn't restructure on small screens — the banner + dropdown pattern fits a narrow viewport already.

## Implementation Details

### Component structure

The calendar page currently has its `DayCell`, `NavButton`, and `StatTile` as inline components inside `page.tsx`. Add the new pieces as siblings, and split out the heavy ones into their own files in `components/calendar/` to keep `page.tsx` under 600 lines.

```
app/(app)/calendar/
  page.tsx                  Shrinks; coordinates fetches, cursor, selected date.
components/calendar/
  day-digest.tsx            Selected-date panel header + activity banners.
  workout-banner.tsx        Banner body + chevron, navigates on body click.
  run-banner.tsx            Same shape, run-specific content.
  workout-digest.tsx        Inline dropdown content for a workout.
  run-digest.tsx            Inline dropdown content for a run.
  weekly-overview.tsx       The right-side column and the small-screen chip variant.
  weekly-tile.tsx           One week's compact card.
components/workout-details.tsx     Refactor: split body out of modal.
```

The split factors the existing modal's body into `WorkoutDetailsBody` so both `WorkoutDetailsModal` and `WorkoutDigest` consume it. The modal stays for any other surface that opens it (today, only the calendar — but the workout page on `/workouts/[id]` may want it later, so the split is worth the cost).

### Selected-date state

```ts
// inside CalendarPage
const [selected, setSelected] = useState<string>(() => localDateKey(new Date()));
// On cursor change, re-anchor to the first of the new month.
useEffect(() => {
  setSelected(localDateKey(new Date(cursor.year, cursor.month, 1)));
}, [cursor]);
```

The selected state is the `YYYY-M-D` local-date key the rest of the page already uses. The digest pulls from `eventsByDate.get(selected) ?? []`.

### Weekly aggregation

Compute weekly buckets from the already-computed `days` array. A week is one row of seven cells in the existing grid. For each week, sum the matching subset of `workouts` and `runs`:

```ts
const weeklyStats = useMemo(() => {
  const weeks: Array<{
    weekStart: Date;
    activities: number;
    liftMinutes: number;
    runMeters: number;
    runMinutes: number;
  }> = [];
  for (let i = 0; i < days.length; i += 7) {
    const weekDays = days.slice(i, i + 7);
    const keys = new Set(weekDays.map(localDateKey));
    let activities = 0, liftMinutes = 0, runMeters = 0, runMinutes = 0;
    for (const w of workouts ?? []) {
      const key = localDateKey(new Date(w.performed_at));
      if (!keys.has(key)) continue;
      activities += 1;
      if (w.ended_at) {
        const ms = new Date(w.ended_at).getTime() - new Date(w.performed_at).getTime();
        if (ms > 0) liftMinutes += Math.round(ms / 60000);
      }
    }
    for (const r of runs ?? []) {
      const key = localDateKey(new Date(r.start_time));
      if (!keys.has(key)) continue;
      activities += 1;
      runMinutes += Math.round(r.duration_seconds / 60);
      runMeters += r.distance_meters;
    }
    weeks.push({ weekStart: weekDays[0], activities, liftMinutes, runMeters, runMinutes });
  }
  return weeks;
}, [days, workouts, runs]);
```

`WeeklyOverviewColumn` consumes this array and renders six tiles. The current-week tile gets the accent border via comparison against `localDateKey(new Date())` falling inside its week's key set.

### Layout shift for the weekly column

Wrap the existing grid in a flex container so the weekly column sits beside it on `md+`:

```tsx
<div className="flex flex-col gap-4 md:flex-row md:items-start">
  <div className="flex-1">
    {/* weekday header + 6×7 day grid as today */}
  </div>
  <WeeklyOverviewColumn weeks={weeklyStats} todayKey={todayKey} />
</div>
```

`WeeklyOverviewColumn` renders the right-side stack on `md+` and the inline chips on `<md`. The chips variant requires interleaving with the rows, which means the inline chips actually need to live INSIDE the grid wrapper, one per row. Implementation: render both, hide one per breakpoint via `hidden md:flex` / `flex md:hidden`. This is layout duplication, but it's cleaner than DOM-restructuring at runtime — and matches the pattern used in the existing `<md` adaptations on `/progress`.

Bump the outer `max-w-5xl` to `max-w-6xl` so the weekly column doesn't squeeze the grid.

### Pace and longest-run computation

Avg Pace: total run time in seconds / total run distance in meters → format as `mm:ss / mi` or `mm:ss / km` based on the user's distance unit. Computed only when `runMeters > 0`.

Longest Run: `Math.max(...runs.map(r => r.distance_meters))` filtered to the cursor month. Formatted via the existing `formatDistance` helper.

Both feed into `monthStats` (existing) or a parallel `monthRunningStats` memo if `monthStats` gets crowded.

### Run digest content

The run digest dropdown shows:

- Distance (formatted via `formatDistance`)
- Duration (mm:ss or h:mm:ss)
- Average pace (mm:ss / unit)
- Average heart rate, if `r.avg_heart_rate` is present
- Elevation gain, if `r.elevation_gain_meters` is present (formatted: `120 m` / `400 ft` based on unit)
- Location/route label, if present (currently no field for this in the `RunningSession` type — graceful omit)
- A `View full run details →` link to `/running/[id]`

Treat every optional field as a graceful omit. The digest is read-only; if a field isn't in the API response, skip it rather than rendering "—".

### Workout digest content

The workout digest reuses `WorkoutDetailsBody` (refactored out of `WorkoutDetailsModal`). Same layout: exercise list with sets, reps, weights, RPE, notes; total volume at the bottom. The modal remains for any external surface; the calendar uses the body component directly inside the banner dropdown.

### Banner click semantics

Each banner is a `<div role="group">` containing:

- A `<button class="banner-body">` covering everything to the left of the chevron. Its `onClick` calls `router.push(...)`.
- A `<button class="banner-chevron" aria-expanded={isOpen} aria-controls={dropdownId}>` on the right. Its `onClick` toggles local state.

Keyboard: Tab moves between body and chevron within the banner; Space/Enter on either fires its handler. The dropdown content's tab order continues into it when expanded.

### Tests

Component tests (Vitest + React Testing Library, matching existing patterns):

- `day-digest.test.tsx`: renders the date header; renders one banner per event; renders an empty state when there are no events; renders banners in start-time order.
- `workout-banner.test.tsx` and `run-banner.test.tsx`: clicking the body calls a navigation callback; clicking the chevron toggles an `aria-expanded` attribute and reveals/hides the dropdown; the body click does NOT toggle the dropdown.
- `workout-digest.test.tsx`: renders an exercise list with set/rep/weight; computes total volume.
- `run-digest.test.tsx`: renders required fields; gracefully omits optional ones; includes the View full run link.
- `weekly-overview.test.tsx`: produces one tile per week; omits zero-value rows from each tile; accents the current week's tile.
- `calendar/page.test.tsx`: selecting a day updates the digest; cursor change re-anchors to the first of the new month; clicking a pill selects the day AND expands the matching digest banner.

### Performance considerations

- `weeklyStats` and `monthStats` both depend on `workouts` / `runs` / `cursor` (and `days`); existing `useMemo` shape extends naturally.
- The digest re-renders only when `selected` or `eventsByDate` changes; no new fetches.
- Banner dropdowns mount lazily — closed banners don't render `WorkoutDetailsBody`. This avoids paying the cost of building the exercise list for every closed banner on a busy day.

## Open Questions

- **Should clicking a pill auto-expand its digest dropdown, or just scroll into view + select?** Auto-expand is more direct ("show me this lift") but adds state coupling between the grid and the digest. The proposed solution above does auto-expand; flag this for a review pass on the implementation PR if it feels noisy in practice.
- **Run location/route label** is not currently in the `RunningSession` type. If we want this in the digest, the API needs a small extension. Out of scope for v1; graceful omit until the field exists.
- **Avg HR on the per-month tile row?** Currently not proposed (would make 7 tiles). Keep at 6 for now; revisit if the running stat surface grows further.
