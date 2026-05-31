# Bodyweight: Multiple Measurements Per Day + Page Redesign

**Status**: Draft · **Last updated**: 2026-05-31

## Introduction

Lifters weigh themselves more than once a day. The classic protocol is right after waking up (true fasted weight) and again before bed, but plenty of users also weigh post-workout, pre/post a long flight, or whenever else they're curious. The current Prog Strength bodyweight UI silently treats each day as a single data point — the chart line connects one reading per day, and the entries list shows everything but feels like a stream rather than something you can navigate. The data model has always allowed multiple per day (`measured_at` is a timestamp, not a date, with no uniqueness constraint), so this is a frontend redesign on top of an already-permissive backend.

This SOW updates the web + mobile bodyweight pages to be useful for someone weighing in twice a day, every day, for years on end. The chart needs to show the trend through daily averages while still acknowledging the spread of intraday readings; the entries list needs to become a paginated table dense enough that scrolling through three years of measurements doesn't feel infinite; and the page top needs immediate, scannable signal — time-range tabs, summary statistic tiles — so the user gets the answer to "how have I been trending?" before they finish scrolling.

## Proposed Solution

The backend stays untouched. `GET /bodyweight` already returns every measurement for the user; the clients fetch the full list once on page load and do all filtering / aggregation / pagination in memory. For a user weighing in twice a day, three years of data is ~2,200 rows × ~100 bytes JSON ≈ 220 KB — a one-shot fetch is genuinely cheaper than the engineering overhead of server-side range filtering or pagination, and you're correctly questioning whether true pagination ever becomes necessary. (Answer: realistically no. A user would need to weigh in 10× a day for a decade to approach a hundred thousand rows, and even then the JSON payload would be ~10 MB — annoying, but still a single-request problem, not a paginate-the-API problem.)

The page top is rebuilt as a vertically-flowing toolbar:

1. **Time-range tabs** (30 days, 60 days, 90 days, All) drive both the chart and the table — picking 30 days means the chart shows the last 30 days *and* the table shows the last 30 days. Same window, two views.
2. **Horizontal separator** below the tabs — the white line you described — visually marks "everything below is your current selection."
3. **Statistic tiles** (Average, Min, Max, Delta) computed against the currently-selected window, so the numbers stay in sync with whatever the chart is showing.
4. **Chart** with daily averages as the smoothed trend line, plus a lighter scatter of the raw same-day points so the user can see the spread without losing the trend.
5. **Paginated table** below the chart, dense rows, with each row being one measurement (not one day). Same-day measurements appear as adjacent rows.

The mobile client gets the same shape, adapted for narrower widths: tabs on a single row, a 2×2 stat-tile grid instead of 4-wide, chart at the same height proportion, table rendered as cards or a tight list rather than a true HTML table.

## Goals and Non-Goals

### Goals

- The web and mobile bodyweight pages render correctly when a user logs multiple measurements on the same calendar date. No deduping, no "overwrite," no silent drops — every row from `GET /bodyweight` is preserved and surfaced somewhere on the page.
- The chart's smoothed trend line goes through the *daily average* (mean of all measurements on that calendar day in the user's local TZ). Same-day raw readings appear as a lighter-colored scatter so the spread is visible but doesn't pull the eye away from the trend.
- A time-range selector (30 / 60 / 90 / All) drives both chart and table simultaneously. Default selection is 30 days on first page load; selection is component-state only, not persisted across page navigations.
- Four statistic tiles above the chart show: Average weight, Minimum, Maximum, and Delta (absolute lb + percent) over the currently-selected window. All values reflect the visible window, not the all-time history.
- The entries list becomes a compact table (web) or card list (mobile), with pagination at 20 rows/page. Pagination is purely client-side over the full fetched list, filtered by the active time range.
- One API call on page load (`GET /bodyweight` for the user, no params). All filtering, aggregation, and pagination happen client-side.
- Mobile parity: same five components (tabs, separator, stat tiles, chart, table) adapted for narrower widths.

### Non-Goals

- **Backend changes.** Schema, repository, handler, MCP tools — none of these need to change. The bodyweight surface has always been timestamp-indexed; the only thing assuming one-per-day was the client UI. This SOW does not modify any API code.
- **Server-side pagination.** Realistic per-user volume is small enough that fetch-all-and-filter-client-side is the simpler, faster solution. Revisit if anyone ever ships a user with 50,000+ measurements.
- **Time-range persistence across sessions.** Picking "90 days" doesn't write anything to localStorage or to a user-preferences row. Each page mount defaults back to 30 days. Persistence is a clean follow-up if anyone asks for it.
- **Unit conversion at display time.** A user with mixed `lb` and `kg` rows (rare but possible via the agent's `log_bodyweight` tool) will see whichever unit they actually logged in each row. The stat tiles and chart compute purely in the user's *preferred* unit (from their profile) — measurements in the other unit are converted at compute time but the table preserves the original unit. No data migration.
- **Editing existing measurements.** The current page only supports add + delete; this SOW preserves that and does not add inline edit. Filed separately if it ever comes up.
- **Same-day grouping in the table.** Each measurement is its own row. We considered nesting same-day entries under a date header, but flat-with-timestamp is denser and scrolls better — and same-day rows naturally group when sorted by `measured_at` desc.
- **Custom time ranges.** No date-range picker for "show me May only." The four presets + "All" cover the common cases. A user who needs more specificity can use the agent's `list_bodyweight` MCP tool with `since`/`until` arguments.

## Implementation Details

### Backend (`prog-strength-api`)

Nothing. The bodyweight schema, repository, handler, and DTOs already do exactly what this SOW needs:

```sql
-- internal/db/migrations/006_nutrition_and_bodyweight.sql
CREATE TABLE bodyweight_entries (
    id          TEXT PRIMARY KEY,
    user_id     TEXT NOT NULL,
    weight      REAL NOT NULL,
    unit        TEXT NOT NULL,
    measured_at DATETIME NOT NULL,           -- timestamp, not date
    created_at  DATETIME NOT NULL,
    deleted_at  DATETIME
);
CREATE INDEX idx_bodyweight_user_measured
    ON bodyweight_entries(user_id, measured_at DESC) WHERE deleted_at IS NULL;
```

No uniqueness constraint on `(user_id, date(measured_at))`. `GET /bodyweight` already returns all rows ordered by `measured_at DESC`. `POST /bodyweight` accepts any timestamp. The MCP `log_bodyweight` + `list_bodyweight` tools pass through transparently. Confirming this is the explicit point of the SOW — no backend PR needed.

### Web client (`prog-strength-web`)

The current `app/(app)/bodyweight/page.tsx` is ~500 lines: a `Chart`, a `Legend`, an `EntryForm`, and a list of `EntryRow`s. The redesign keeps the form but rebuilds the rest. Concretely:

- **Time-range tabs** as a horizontal row above the chart, using the same ghost-button style as the nutrition page's toolbar (white text + label, no fill, no border). Active tab gets the "pressed" treatment from the date-tile-strip (accent fill + inset shadow + 1px down-translate). Tabs: `30 days`, `60 days`, `90 days`, `All`. Default: 30 days.
- **Horizontal separator** (`border-b border-[var(--border)]`) directly below the tabs.
- **Statistic tiles** in a 4-column grid. Same `<div className="rounded-lg border bg-surface px-4 py-3">` shape we use elsewhere. Each tile: label (uppercase muted) + value (large + tabular-nums). Tiles:
  - **Average** — mean weight over the window, in the user's preferred unit.
  - **Min** — lowest single measurement over the window, with its date as the sublabel.
  - **Max** — highest single measurement over the window, with its date as the sublabel.
  - **Delta** — `(last_measurement - first_measurement) in window`, shown as `+1.4 lb` or `-2.1 lb` with the percent change in the sublabel (`+0.8%`).
- **Chart** stays recharts-based since we already have the dep. Switch from a single line to a `ComposedChart`:
  - X axis: dates (one tick per calendar day in the window).
  - Daily averages computed client-side: group measurements by `startOfLocalDay(measured_at)`, average within each group. The trend line passes through these averages.
  - Raw measurements rendered as a `Scatter` series in a lighter color so same-day spread is visible without dominating.
  - Y axis: weight, in the user's preferred unit.
- **Table** below the chart, replacing the existing vertical `EntryRow` list. Plain `<table>` with `<thead>` + `<tbody>`. Columns: Date, Time, Weight, Unit, Actions (delete). 20 rows per page. Pagination controls below the table: page number + total pages, `‹`/`›` chevrons, jump-to-first/last. Page state resets to 1 when the active time-range tab changes.

State on the page:
- `entries: BodyweightEntry[]` — the full fetched list, never mutated except for delete.
- `range: "30" | "60" | "90" | "all"` — active tab; default "30".
- `page: number` — 1-indexed current page in the table; resets on range change.
- `entriesInRange` — `useMemo` derived from `entries + range`.
- Stat tiles + chart + table all read off `entriesInRange`; pagination further slices the table.

### Mobile client (`prog-strength-mobile`)

Same five components, narrower-width adaptations:

- **Tabs**: same single row of four ghost buttons. May need to compact text to "30d / 60d / 90d / All" if `30 days / 60 days / ...` overflows on phone widths.
- **Separator**: same horizontal divider via `border-b border-border`.
- **Stat tiles**: 2×2 grid instead of 4-wide. Tiles are bigger and more thumbable on phone screens — a 4-wide row of tiles would shrink each tile below readable threshold.
- **Chart**: same recharts logic ported to `react-native-svg-charts` or whatever the mobile chart library is. (Today `components/nutrition/bodyweight-chart.tsx` exists — confirm what it uses before implementing.) Same daily-average trend + scatter spread.
- **Table → card list**: a true HTML table doesn't read well on phone widths. Render each row as a compact card (date + time on one line, weight on the right). Same 20/page pagination; chevron buttons below.

### Algorithms

**Daily aggregation** for the chart's trend line: group entries by `startOfLocalDay(measured_at)`, average the `weight` within each group, plot the average against the day. Same-day raw points are kept as scatter dots so the user can see the morning-vs-evening spread without it pulling the trend line.

**Time-range filter**: cutoff is `now - rangeDays`. An entry is in-range iff `measured_at >= cutoff`. "All" disables the filter entirely.

**Delta calculation**: pick `first = entriesInRange[earliest]` and `last = entriesInRange[latest]`. Delta = `last.weight - first.weight` in user's preferred unit. Percent = `delta / first.weight * 100`. Both shown with a sign (`+` or `-`).

**Unit handling**: the user's profile holds their preferred unit. Measurements in the *other* unit get converted at compute time (lb ↔ kg via the standard 2.20462 factor) for stat tiles and chart. Table rows preserve the original unit so a user who logs in kg sometimes can still see their actual logged values.

### Backfill or Migration

None. Existing rows are already valid under the new UI — they just stop being assumed-one-per-day. Users with no same-day entries see no behavior change; users with same-day entries finally see all of them.

## Open Questions

1. **Daily aggregation: mean, median, or earliest-of-day?** Mean is the obvious choice and matches your suggestion. Median would be more robust to a single bad reading (e.g., scale read-error). Earliest-of-day is what most fitness apps default to since the "true" weight is generally the fasted morning reading. Tentative lean: mean. It's the most predictable and matches what users would compute on paper. If users object to having their post-meal weights drag the average up, we can revisit.

2. **What does "Delta" actually compare?** First-vs-last *measurement* in the window (potentially same day if both at the start), first-vs-last *day-average* in the window, or first-day-average-vs-most-recent-day-average? Tentative lean: first-day-average-vs-most-recent-day-average. Matches what the chart's trend line is showing, less susceptible to picking a single bad reading at either endpoint. Worth confirming.

3. **Default time-range tab on page load.** 30 days is my lean — covers a recent training block, doesn't dominate the chart with noise, generally what someone wants to see when they open the page. 60 days makes sense for a longer cycle. "All" is too much for the chart on first paint. Lean: 30.

4. **Page size for the table.** 20 rows is the lean (compact + scannable, doesn't require a tiny phone to scroll). 10 feels too few when users log 2/day. 50 feels like a lot of rows to scan. Worth checking — easy to change.

5. **Do the stat tiles + chart adapt to today's date being the most recent measurement, or use `now` as the reference?** If the user hasn't weighed in today, "30 days" — does it mean "30 days back from now" or "30 days back from the last measurement"? Lean: 30 days back from `now` (today). Simpler mental model, and gaps in measurements show up as gaps in the chart rather than the chart sliding earlier in time.

6. **Mobile: card list vs true table.** A 4-column HTML table on a 375px wide phone is bad. Cards are friendlier but less dense. An alternative: a 2-column table (date + weight) with time as a sub-line under the date. Tentative lean: cards for v1, revisit if users want more density.
