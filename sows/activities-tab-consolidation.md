---
status: shipped
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Activities Tab: Consolidate Workouts + Running + Overview

**Status**: Shipped · **Last updated**: 2026-06-09

## Introduction

The **Prog Strength** web app's sidebar has accreted entries one feature at a time, and the bill is starting to come due. It currently surfaces Chat, Workouts, Exercises, Calendar, Nutrition, Running, Bodyweight, Progress, Personal Records, and Settings — ten rows of vertical real estate, with Workouts and Running sitting one above the other as siblings answering the same fundamental question ("what physical activity did I do?"). The lifter who opens the app to check on the week doesn't think "I'm going to look at my workouts and then look at my runs"; they think "how active was I this week?" and then drill into whichever modality they actually care about.

Beyond the conceptual split, the two pages have visibly diverged in polish. Workouts already ships a top-of-page analytics card with a 3-up icon switcher (Time Lifting, Volume, Body Parts) and timeframe pills (7d / 30d / 90d / all). Running ships a 4-tile metrics banner and a paginated list — useful, but no charting at all and no timeframe selector to scope the view. A user who wants to see "my mileage trend over the last 30 days" cannot get there from the app today; they have to either eyeball the list or open another tool.

This SOW consolidates the two pages into a single **Activities** entry in the sidebar, with three sub-views (Overview, Workouts, Running) selectable from a toolbar that mirrors the underline-active pattern already shipped on the Nutrition page. The Overview view is the new default and answers the "how active was I?" question at a glance. The Workouts view inherits today's `/workouts` page wholesale. The Running view inherits today's `/running` page and gains a new analytics card with two chart views — Mileage and Time — using the same recharts pattern and Monday-anchored weekly buckets as Workouts. Timeframe pills move up to the page level so the same window scopes all three views, and the Running list switches from cursor pagination to timeframe-filtered fetching to match.

After this work ships, the lifter who opens the app sees one Activities entry where two used to live. Clicking it lands them on a clean, scannable digest of their week; one click swaps in the workouts list with its existing analytics; one more click swaps in the running list with mileage and time-spent charts that respect the same timeframe. Nothing about the data, the API, or the agent surface changes — this is a pure UI / UX reorganization in `prog-strength-web` aimed at simplifying the sidebar and modernizing two pages that have drifted apart.

## Proposed Solution

A new page `app/(app)/activities/page.tsx` becomes the thin shell that owns three concerns: (1) the URL `?view=` parameter as the source of truth for the active sub-view, (2) the timeframe pills, lifted out of the existing Workouts page so the same window scopes Overview / Workouts / Running uniformly, and (3) the toolbar with the three view-switcher buttons and any view-specific action button (Upload TCX, which today lives on `/running`). The body slot under the toolbar swaps between three new view components — `ActivitiesOverviewView`, `WorkoutsView`, `RunningView` — based on the URL parameter.

The sidebar collapses the existing `/workouts` and `/running` NAV entries into one `/activities` entry, slotted where Workouts is today (between Chat and Exercises). A new "activity pulse" line glyph anchors the entry visually. The two unused icons (`DumbbellIcon`, `RunIcon`) get deleted from `components/sidebar.tsx` since no other sidebar entry references them.

The existing parent routes `/workouts` and `/running` become server-side redirects to the new URLs (`/activities?view=workouts` and `/activities?view=running` respectively), so bookmarks and copy-pasted links keep working. Detail routes `/workouts/[id]` and `/running/[id]` keep their URLs intact — Calendar, Progress, and Personal Records all link to them — and only their internal "back" links get retargeted at `/activities?view=…`.

`WorkoutsView` is `/workouts/page.tsx`'s current body lifted into a component with one structural change: the timeframe is now a prop (`days: number | null`) passed in from the shell rather than locally-owned state. The existing `WorkoutsAnalytics` 3-view switcher (Time Lifting / Volume / Body Parts) stays exactly as today.

`RunningView` is `/running/page.tsx`'s current body lifted similarly. The 4-tile metrics banner — driven by the API's `GET /running/metrics` and its fixed "current week / current month / recent 30d / all time" buckets — remains as-is because those tiles answer "where am I right now?" regardless of selected window; the selected window scopes the new charts and the run list. Above the run list, a new `RunningAnalytics` card mirrors `WorkoutsAnalytics`: a shared summary header (Total distance, Total time, Runs), an icon-button row with two views, and two new chart components — `RunningMileageChart` and `RunningTimeChart` — both implemented as recharts `AreaChart`s with Monday-anchored weekly buckets identical to `WorkoutDurationChart`'s structure. The run list itself switches from cursor pagination to timeframe-filtered fetching using the existing `since` / `until` query-param mode the API already supports.

`ActivitiesOverviewView` is the new digest surface and the page's default. It renders four pieces, top to bottom: (1) a 4-up StatTile row — Total time, Total sessions, Workouts, Runs — for the selected window; (2) a new `ActivitiesCombinedChart` — a stacked weekly area chart showing minutes spent lifting (blue) and minutes spent running (green) week over week, so the consolidated trend lives somewhere; (3) a secondary StatTile row — Total volume (lb/kg), Total mileage (mi/km), PRs, Avg session length; (4) nothing else. The view is deliberately a digest, not its own dashboard — single-modality drill-down belongs on the two dedicated tabs.

### Critical implementer note

The web repo's `AGENTS.md` and `CLAUDE.md` are explicit: **"This is NOT the Next.js you know"**. The version in use has breaking changes from training-data Next.js — APIs, conventions, and file structure may all differ. Before writing any code in `prog-strength-web`, the implementer must consult the relevant guide in `node_modules/next/dist/docs/` and heed any deprecation notices. This applies in particular to: server-side `redirect()` from a route file (used by the `/workouts` and `/running` parent stubs), `useSearchParams` semantics in client components, and the route group `(app)` conventions. The existing nutrition-tabs work (see `prog-strength-web/docs/superpowers/specs/2026-05-31-nutrition-tabs-design.md`) demonstrates the validated pattern for tab-via-search-param in this codebase and should be the structural model.

## Goals and Non-Goals

### Goals

- Reduce the sidebar by collapsing `Workouts` and `Running` into one new `Activities` entry positioned between Chat and Exercises. Remove the unused `DumbbellIcon` and `RunIcon` definitions from `components/sidebar.tsx` once their NAV entries are gone.
- Choose a new "activity pulse" line glyph for the Activities entry — a stylized waveform / heartbeat line is the conventional shorthand and reads at 16×16 without ambiguity. Inlined SVG in the same idiom as the existing sidebar icons (viewBox `0 0 24 24`, `currentColor` stroke, `strokeWidth="1.75"`). No icon library.
- Build a new page `app/(app)/activities/page.tsx` that:
  - Reads `view` from `useSearchParams`. Allowed values: `overview` (default), `workouts`, `running`. Missing or unrecognized → `overview`.
  - Owns the timeframe selector state (`timeframe: "7d" | "30d" | "90d" | "all"`, default `30d`) and renders the timeframe pills in the page header. The pills are visually and behaviorally identical to today's pills on `/workouts/page.tsx` — same rounded-full styling, same accent fill when active, same set of options.
  - Computes `days: number | null` from the active timeframe and threads it into all three views as a prop.
  - Renders a toolbar row below the header: left group holds view-specific action buttons (Upload TCX only, visible when `view === "running"`); right group holds three `ToolbarButton`s — `Overview`, `Workouts`, `Running` — using the existing `ToolbarButton` component and its `active` prop (the underline pattern shipped by the nutrition tabs work). Active state ties to the URL `view` param.
  - Renders a `border-b border-[var(--border)]` separator under the toolbar.
  - Renders one of `<ActivitiesOverviewView />`, `<WorkoutsView />`, `<RunningView />` based on `view`.
  - Switches tabs via `router.replace("/activities?view=...")` (not push) so rapid switching doesn't pollute back history. Switching to `overview` strips the param entirely (`/activities`).
- Convert the existing `app/(app)/workouts/page.tsx` and `app/(app)/running/page.tsx` parent routes to server-side redirects:
  - `/workouts` → `redirect("/activities?view=workouts")`
  - `/running` → `redirect("/activities?view=running")`
  - Use the framework's server-side `redirect()` helper from `next/navigation` (re-verify the canonical import per the Next.js docs in `node_modules/next/dist/docs/` — the rule from `AGENTS.md` applies). The body of each file becomes a one-line server component that calls `redirect` synchronously.
- Update detail-page back links so the breadcrumb returns the user to the new URL:
  - `app/(app)/running/[id]/page.tsx`: every `href="/running"` and `router.push("/running")` becomes `href="/activities?view=running"` and `router.push("/activities?view=running")` respectively. There are four call sites today (lines 135, 147, 160, 179).
  - `app/(app)/workouts/[id]/page.tsx`: same retarget — every existing back-link to `/workouts` becomes `/activities?view=workouts`. The implementer must `grep` the file for `"/workouts"` and `\`/workouts\`` references and confirm only the back-link sites get rewritten (detail-page links to other workouts must continue to point at `/workouts/{id}` because that route is still owned by the detail page).
- Build `ActivitiesOverviewView` as the new default body:
  - Hero stat row: `grid grid-cols-2 md:grid-cols-4 gap-3` with four `StatTile`s — Total time, Total sessions, Workouts, Runs. Values computed client-side from the fetched data over the selected window.
  - Combined weekly activity chart: a new `components/activities/activities-combined-chart.tsx` — a stacked recharts `AreaChart` with two series (`workout_minutes`, `running_minutes`), Monday-anchored buckets identical to `WorkoutDurationChart`'s window logic, blue (`#3b82f6`) for workouts and green (`#10b981`) for running. Empty weeks render as zero so the line is continuous over gaps. Legend keys on top-left of the chart. Same chrome-less structure as the existing duration chart: outer card border + summary header live on the parent view; the chart component renders the chart block + the truncated note.
  - Secondary stat row: `grid grid-cols-2 md:grid-cols-4 gap-3` with four `StatTile`s — Total volume (formatted with thousands separators + unit), Total mileage (formatted via `useDistanceUnit().formatDistance`), PRs (sum of `personal_records_set.length`), Avg session length (total minutes / total session count).
  - Fetches: on mount and on timeframe change, parallel `Promise.all` for `listWorkouts({ since, limit: 100 })` and `listRunningSessions({ since, until, limit: 200 })`. Uses the existing `getMe(token)` once for `displayUnit` (lifted to the page shell — see State ownership). Catches 401 → `clearToken()` + `router.replace("/login")`. Renders a single inline red error panel on other failures, same idiom as the other pages.
- Build `WorkoutsView` by extracting the body of today's `app/(app)/workouts/page.tsx`:
  - Move the page's render output (analytics card + week-grouped paginated list + edit modal) into `components/activities/workouts-view.tsx`.
  - Replace the locally-owned `timeframe` state with a `days: number | null` prop from the shell. Drop the timeframe-pill render block — those pills now live on the shell.
  - The week-grouping helpers (`groupByWeek`, `startOfMonday`, `isoDate`, `formatWeekLabel`, `formatWeekRange`, `formatTotalDuration`, `formatDate`, `formatDuration`), the row component (`WorkoutRow`), the icons (`ChevronIcon`, `PencilIcon`, `TrophyIcon`, `TrashIcon`), and the `PaginationFooter` / `EmptyState` / `WeekHeader` / `StatTile` (inline) components all move into the view file as-is, since they have no other consumers.
  - The `WorkoutModal` import and modal mount stay inside the view; the edit-modal state (`editing: Workout | null`) is view-local.
  - The view continues to call `getMe(token)` for `displayUnit` only if the shell does not already provide it — wire the shell's already-fetched `displayUnit` through as a prop to avoid double-fetching.
- Build `RunningView` by extracting the body of today's `app/(app)/running/page.tsx`:
  - Move the page's render output (4-tile metrics banner + run list + Upload TCX modal) into `components/activities/running-view.tsx`.
  - The 4-tile metrics banner (driven by `getRunningMetrics`) renders unchanged — those fixed-bucket tiles answer a different question than the timeframe-driven charts and explicitly stay as a fixed-window status row.
  - Replace cursor pagination with timeframe-filtered fetching. The `listRunningSessions` call uses `since` + `until` derived from the selected window (`since = now - days * 86400s`, `until = now`; for `days === null` ("all"), omit both). The "Load more" button is removed — the timeframe selector now drives windowing. If the returned page is the API's max (`limit: 200`), surface the same "Showing the most recent N runs" truncation note the workouts chart uses.
  - Insert a new `<RunningAnalytics />` card above the run list (and below the 4-tile metrics banner).
  - The Upload TCX button moves out of the view's local header and into the shell's toolbar left group. The handler (`handleUploaded`) stays inside the view; the shell forwards a callback that the view passes to `UploadTCXModal`. (Alternative: the shell renders the modal itself with a refetch callback the view exposes via context or a ref. The simpler approach — view owns the modal, shell renders the button — is fine because there's exactly one view-specific action and inverting control here would add ceremony for no payoff.)
- Build `RunningAnalytics` as `components/running-analytics.tsx`:
  - Owns a `view: "mileage" | "time"` state, default `"mileage"`.
  - Shared summary header above the chart: Total distance (in display unit), Total time, Runs count — all computed from the same `sessions` array passed in as a prop.
  - Icon-button row with two buttons styled identically to `WorkoutsAnalytics`'s `ViewButton`: a road / route icon for Mileage, a clock icon for Time. Use the existing inline-SVG idiom; the clock icon already exists in `WorkoutsAnalytics` and should be lifted into a shared `components/icons/` module if the implementer prefers, but inlining is fine for v1.
  - Renders `<RunningMileageChart />` or `<RunningTimeChart />` based on `view`. Both receive `sessions: RunningSession[] | null`, `days: number | null`, `displayUnit: "mi" | "km"`, and a `truncated` / `fetchLimit` pair mirroring `WorkoutDurationChart`'s props.
- Build `RunningMileageChart` as `components/running-mileage-chart.tsx`:
  - Same recharts `AreaChart` structure as `WorkoutDurationChart`.
  - Monday-anchored weekly buckets covering the selected window (or oldest-run-to-today when `days === null`).
  - Y-axis is total weekly distance in the user's preferred unit (`mi` or `km`). The distance conversion uses the existing `useDistanceUnit` context's `formatDistance` for ticks and tooltips; the underlying summation runs in meters and converts at display time.
  - Empty weeks render as zero so gaps are visible.
  - Same chrome-less structure: outer card border and summary header live on `RunningAnalytics`; this component renders the chart block + the truncated note.
- Build `RunningTimeChart` as `components/running-time-chart.tsx`:
  - Identical structure to `RunningMileageChart` but Y-axis is total weekly duration in minutes (or hours when ≥60min, via the existing `formatYTick` pattern). The summation uses `session.duration_seconds`.
- Build `ActivitiesCombinedChart` as `components/activities/activities-combined-chart.tsx`:
  - A stacked `AreaChart` with two series: `workout_minutes` (blue, `#3b82f6`) and `running_minutes` (green, `#10b981`). Use `stackId="activity"` so the recharts areas stack rather than overlap.
  - Monday-anchored weekly buckets identical to `WorkoutDurationChart`'s window logic.
  - Tooltip shows both series labels and totals; legend at the top of the chart names the two series.
  - Empty weeks render as `{workout_minutes: 0, running_minutes: 0}` so the stack is continuous.
- Wire the Activities page into the application's natural flow:
  - Cold load of `/` (the root) continues to redirect to `/chat` (no change here — Chat remains the brand-link home).
  - Cold load of `/activities` with no query string → Overview tab.
  - Cold load of `/activities?view=workouts` / `/activities?view=running` → respective tab.
  - Cold load of `/activities?view=anything-else` → fall back to Overview, same pattern as the nutrition tabs.

### Non-Goals

- **No API, MCP, agent, or mobile changes.** This is a pure `prog-strength-web` reorganization. No new endpoints, no new MCP tools, no agent-side prompt changes, and no mobile-app work. The mobile app's existing navigation is untouched and its own consolidation, if desired, is a separate SOW.
- **No deletion of the `/workouts/[id]` and `/running/[id]` detail routes.** They keep their URLs (calendar, progress, and personal-records all link directly to them; rewriting those callers and the bookmarks users may have is more disruption than the consolidation justifies). Only the parent `/workouts` and `/running` routes redirect.
- **No backward-compatibility hacks beyond the parent-route redirects.** Old in-app links to `/workouts` and `/running` in the codebase (sidebar entry removal, detail-page back links) are rewritten to the new URLs in this work. The redirects are a safety net for external bookmarks, not a permanent shim.
- **No new aggregate metrics beyond what's listed.** No streak counter on the Overview, no "calories burned" estimate, no cardio-vs-strength balance score, no per-week PR rate, no estimated-1RM trend graph. The four hero tiles + four secondary tiles + combined chart is the v1 surface; new metrics land in follow-ups once we know which ones the user reaches for.
- **No per-view timeframe override.** One timeframe at the page level scopes all three views. A "Workouts shows 30d but Running shows 90d" UX is not supported in v1.
- **No URL persistence of inner switcher state.** The `WorkoutsAnalytics` 3-view switcher (Time / Volume / Body Parts) and the new `RunningAnalytics` 2-view switcher (Mileage / Time) reset to their defaults on refresh. Matches today's behavior for `WorkoutsAnalytics`. Persistence via search params is a clean follow-up if any of these stops feeling right.
- **No URL persistence of the timeframe.** Refreshing `/activities` returns to the 30-day default. Matches today's `/workouts` behavior. Same follow-up note.
- **No animation of the tab-switch or chart-switch transitions.** CSS transitions are fine if they happen for free (existing `transition` classes on buttons); no Framer Motion, no react-spring.
- **No mobile/responsive overhaul.** The existing pages' responsive behavior is preserved (e.g., `grid-cols-2 md:grid-cols-4` stays). The mobile sidebar collapsing is unchanged.
- **No icon-library introduction.** All new icons (activity pulse for the sidebar, road/route for the Mileage view, clock for the Time view, anything else) are hand-rolled inline SVGs in the existing idiom. The "no library" precedent is set by `components/sidebar.tsx` and `components/workouts-analytics.tsx` and is preserved.
- **No new tests.** The repo has no existing test suite (`vitest.config.ts` is configured but no tests exist for the pages this work touches). Verification is manual, per the manual verification plan below. Adding a test suite is a separate, larger undertaking.
- **No refactor of the existing `WorkoutsAnalytics` switcher into a generic "AnalyticsCard with N views" abstraction.** The two cards (`WorkoutsAnalytics`, the new `RunningAnalytics`) share a visual idiom but their content shapes differ enough that a shared abstraction would obscure rather than clarify. Three similar files is fine; the abstraction is a future-you problem if a third card lands.
- **No combined activity feed.** The Overview is a digest of aggregates, not a chronological list interleaving workouts and runs. A "what did I do this week, in order?" view exists on the Calendar page already.

## Implementation Details

### Routing & Sidebar

`components/sidebar.tsx`:

- Remove the `{ href: "/workouts", … }` entry and the `{ href: "/running", … }` entry.
- Insert a new `{ href: "/activities", label: "Activities", icon: <ActivityIcon /> }` entry in the same slot Workouts occupies today (between Chat and Exercises).
- Define a new `ActivityIcon` inline-SVG component using a stylized pulse waveform: a horizontal baseline with one peak + one valley + one peak, mirroring the universal "activity" glyph. Use viewBox `0 0 24 24`, `currentColor` stroke, `strokeWidth="1.75"`, `strokeLinecap="round"`, `strokeLinejoin="round"` — identical idiom to the other inline icons.
- Delete the `DumbbellIcon` and `RunIcon` function definitions since they're no longer referenced from this file.

The active-highlight logic stays as-is (prefix-match plus exact match), so `/activities?view=workouts` still lights up the entry.

`app/(app)/workouts/page.tsx`:

- Replace the entire file with a server component that calls `redirect("/activities?view=workouts")` from `next/navigation`. The body collapses to a single line.

`app/(app)/running/page.tsx`:

- Same shape — `redirect("/activities?view=running")`.

`app/(app)/workouts/[id]/page.tsx` and `app/(app)/running/[id]/page.tsx`:

- Retarget every back-link reference from the parent route to the new URL:
  - `href="/workouts"` → `href="/activities?view=workouts"`
  - `router.push("/workouts")` → `router.push("/activities?view=workouts")`
  - `href="/running"` → `href="/activities?view=running"`
  - `router.push("/running")` → `router.push("/activities?view=running")`
- Leave detail-self-references (`/workouts/{id}`, `/running/{id}`) untouched.

### Page shell — `app/(app)/activities/page.tsx`

```tsx
"use client";

import { useEffect, useMemo, useState } from "react";
import { useRouter, useSearchParams } from "next/navigation";
import { clearToken, getToken } from "@/lib/auth";
import { getMe, type User } from "@/lib/api";
// View imports, ToolbarButton, etc.

type View = "overview" | "workouts" | "running";
type Timeframe = "7d" | "30d" | "90d" | "all";

const TIMEFRAMES: { id: Timeframe; label: string; days: number | null }[] = [
  { id: "7d", label: "Last 7 days", days: 7 },
  { id: "30d", label: "Last 30 days", days: 30 },
  { id: "90d", label: "Last 90 days", days: 90 },
  { id: "all", label: "All", days: null },
];

export default function ActivitiesPage() {
  const router = useRouter();
  const search = useSearchParams();
  const rawView = search.get("view");
  const view: View =
    rawView === "workouts" || rawView === "running" ? rawView : "overview";

  const [timeframe, setTimeframe] = useState<Timeframe>("30d");
  const days = useMemo(
    () => TIMEFRAMES.find((t) => t.id === timeframe)?.days ?? null,
    [timeframe],
  );

  const [user, setUser] = useState<User | null>(null);
  // … cold-load getMe + auth handling identical to existing pages

  const setView = (next: View) => {
    if (next === "overview") router.replace("/activities");
    else router.replace(`/activities?view=${next}`);
  };

  // … render header (h1 + timeframe pills), toolbar (left = view-specific
  //    actions, right = three ToolbarButtons with active prop), separator,
  //    body switch on `view`
}
```

State ownership:

| State | Owner | Notes |
| --- | --- | --- |
| `view` | URL (`useSearchParams`) | Source of truth |
| `timeframe` | `ActivitiesPage` | Shared by all three views |
| `user` (for `displayUnit`, `weight_unit`) | `ActivitiesPage` | One-time fetch shared across views |
| Per-view list / chart data | each view component | View-local; refetches on `days` change |
| `WorkoutsAnalytics` inner switcher | `WorkoutsAnalytics` | Unchanged from today |
| `RunningAnalytics` inner switcher | `RunningAnalytics` | New |

### Toolbar — `ToolbarButton`

Reuse the existing `ToolbarButton` component that was introduced for the nutrition tabs (`active?: boolean` prop, applies the 2px underline classes when true). If the implementer finds the component lives in `components/nutrition/` and is logically nutrition-specific in its current home, lift it to `components/toolbar-button.tsx` so the activities page can consume it without an awkward import path. If it's already in a neutral location, import as-is.

Layout:

- Outer: `flex items-center justify-between`.
- Left group: `flex gap-5`. Renders the Upload TCX button when `view === "running"`. Empty when `view === "overview"` or `view === "workouts"`.
- Right group: `flex gap-5`. Three `ToolbarButton`s — `Overview`, `Workouts`, `Running`. Each sets the URL `view` param on click via `setView`.

Active styling matches the nutrition tabs pattern: a 2px underline (`border-b-2 border-[var(--foreground)] -mb-px pb-3`) on whichever button corresponds to the current `view`. Pills used by `/workouts` today for timeframe selection are reused verbatim for the timeframe selector (those move into the header above the toolbar).

### Activities Overview View — `components/activities/activities-overview-view.tsx`

```tsx
"use client";

export function ActivitiesOverviewView({
  days,
  displayUnit,
  distanceUnit,
}: {
  days: number | null;
  displayUnit: "lb" | "kg";
  distanceUnit: "mi" | "km";
}) {
  // … parallel fetch on mount + on days change:
  //   - listWorkouts({ since, limit: 100 })
  //   - listRunningSessions({ since, until, limit: 200 })
  //
  // Compute (memoized on workouts + sessions):
  //   - totalWorkoutMinutes, totalRunningMinutes, totalMinutes
  //   - workoutCount, runCount, totalSessions = workoutCount + runCount
  //   - totalVolume   = sum(workoutVolume(w, displayUnit))
  //   - totalMileage  = sum(session.distance_meters)
  //   - prCount       = sum(workouts[].personal_records_set.length)
  //   - avgSessionMinutes = totalMinutes / totalSessions (guarded)
  //
  // Render:
  //   <Hero4Up> Total time | Sessions | Workouts | Runs </Hero4Up>
  //   <ActivitiesCombinedChart workouts={...} sessions={...} days={days} />
  //   <Secondary4Up> Total volume | Total mileage | PRs | Avg session </Secondary4Up>
}
```

`Total time` formatting reuses the existing `formatHours(minutes)` helper from `components/workout-duration-chart.tsx`; lift that helper into `lib/format.ts` (the existing module from the nutrition work) so both the combined chart and the running-time chart can consume it.

`Total mileage` formatting uses `useDistanceUnit().formatDistance(totalDistanceMeters)` and appends `unitLabel`.

`Avg session length` is `formatHours(totalMinutes / totalSessions)`. When `totalSessions === 0`, render `—`.

Each StatTile uses the existing `app/(app)/running/_components/StatTile.tsx` component (lift it to `components/stat-tile.tsx` so it's not running-namespaced; update the running-view imports accordingly). Same visual idiom — large value, uppercase muted label, optional sub-line — already in use across the app.

### Combined chart — `components/activities/activities-combined-chart.tsx`

```tsx
"use client";

import {
  Area,
  AreaChart,
  CartesianGrid,
  Legend,
  ResponsiveContainer,
  Tooltip,
  XAxis,
  YAxis,
} from "recharts";
import type { Workout, RunningSession } from "@/lib/api";

export function ActivitiesCombinedChart({
  workouts,
  sessions,
  days,
  truncated,
  fetchLimit,
}: {
  workouts: Workout[] | null;
  sessions: RunningSession[] | null;
  days: number | null;
  truncated: boolean;
  fetchLimit: number;
}) {
  // Bucket both inputs into Monday-anchored weeks over the same window
  // (lifted from workout-duration-chart's `summarize` — extract that
  // window-and-bucket helper to lib/weekly-buckets.ts so all four chart
  // components consume one implementation).
  //
  // Each bucket: { t, weekStart, workout_minutes, running_minutes }.
  //
  // <AreaChart data={buckets}>
  //   <Area stackId="activity" dataKey="workout_minutes" stroke="#3b82f6" fill={…} />
  //   <Area stackId="activity" dataKey="running_minutes" stroke="#10b981" fill={…} />
  //   <Legend />
  //   <Tooltip /> formatter returns "Lifting / Running" labels with
  //   formatHours values.
  // </AreaChart>
}
```

The truncation note ("Showing the most recent N sessions") renders when either input is truncated. Y-axis tick formatter reuses `formatYTick` from `workout-duration-chart.tsx` (also lift to `lib/format.ts` or a new `lib/chart-format.ts`).

### Workouts View — `components/activities/workouts-view.tsx`

The content of today's `app/(app)/workouts/page.tsx` body lifted into a component with the following prop contract:

```ts
type WorkoutsViewProps = {
  days: number | null;
  displayUnit: "lb" | "kg";
};
```

Internals (week grouping, row component, modal mount, pagination, expand state) move verbatim. The only structural deletions are: the timeframe-pill render block, the local `timeframe` state and `TIMEFRAMES` constant, the `getMe` effect, and the `user` state — all owned by the shell now.

The view continues to drive the existing `WorkoutsAnalytics` component, which already accepts `days` and `displayUnit` as props. No changes to `WorkoutsAnalytics` itself.

### Running View — `components/activities/running-view.tsx`

Today's `app/(app)/running/page.tsx` body lifted similarly. Prop contract:

```ts
type RunningViewProps = {
  days: number | null;
  uploadModalOpen: boolean;
  onCloseUploadModal: () => void;
};
```

The `uploadModalOpen` state is owned by the shell because the Upload TCX *button* lives in the shell toolbar — the modal itself stays in the view so the upload handler (`handleUploaded`) keeps direct access to the view's `refetch`. Pattern: shell toggles a bool, view renders the modal when bool is true and calls back to close.

Fetching changes:

- The `getRunningMetrics` call stays — the 4-tile metrics banner uses fixed buckets.
- `listRunningSessions` now uses timeframe filtering. Compute:
  ```ts
  const since = days !== null
    ? new Date(Date.now() - days * 86400_000).toISOString()
    : undefined;
  const until = days !== null
    ? new Date().toISOString()
    : undefined;
  listRunningSessions(token, { since, until, limit: 200 });
  ```
  When `days === null` ("all"), omit `since` and `until` entirely.
- The "Load more" button + `nextBefore` state are removed.
- A `truncated` flag (`sessions.length >= 200`) feeds into the `RunningAnalytics` charts so they can show the truncation note when needed.

Render order inside the view:

1. 4-tile fixed metrics banner (unchanged)
2. `<RunningAnalytics sessions={sessions} days={days} truncated={truncated} fetchLimit={200} distanceUnit={unitLabel} />` (new card)
3. Run list (`<ul>` of `<RunListRow>`)
4. `UploadTCXModal` mounted when `uploadModalOpen` is true

### Running Analytics — `components/running-analytics.tsx`

Mirrors `WorkoutsAnalytics`'s structure:

```tsx
type View = "mileage" | "time";

export function RunningAnalytics({
  sessions, days, truncated, fetchLimit, distanceUnit,
}: { … }) {
  const [view, setView] = useState<View>("mileage");

  // Shared summary header values:
  const totalSeconds = sumDurationSeconds(sessions ?? []);
  const totalMeters  = sumDistanceMeters(sessions ?? []);
  const runCount     = sessions?.length ?? 0;

  return (
    <section className="rounded-lg border border-[var(--border)] bg-[var(--surface)] p-4">
      <header className="flex items-baseline gap-8">
        {/* Total distance | Total time | Runs — same big-tile chrome as
            WorkoutsAnalytics's header */}
      </header>
      <div className="mt-4 flex gap-2">
        <ViewButton label="Mileage" active={view === "mileage"} onClick={() => setView("mileage")}>
          <RouteIcon />
        </ViewButton>
        <ViewButton label="Time" active={view === "time"} onClick={() => setView("time")}>
          <ClockIcon />
        </ViewButton>
      </div>
      {view === "mileage" && <RunningMileageChart … />}
      {view === "time"    && <RunningTimeChart    … />}
    </section>
  );
}
```

The `ViewButton` component is structurally identical to the one inside `WorkoutsAnalytics`. The implementer should lift it to `components/view-button.tsx` so both analytics cards consume one component rather than duplicating it — the underline-active variant of this same button is what the toolbar tabs use, so the consolidation is a small but real cleanup.

### Running Mileage Chart — `components/running-mileage-chart.tsx`

Structural copy of `WorkoutDurationChart`. Replace the workout-specific aggregation with:

```ts
function summarize(sessions: RunningSession[], days: number | null) {
  // Window setup is the same — derive `since` from `days` or from oldest
  // session, build Monday-anchored weeks, etc. Lift this window logic
  // out of WorkoutDurationChart's `summarize` into `lib/weekly-buckets.ts`
  // and parametrize on a `getTimestamp(item) -> Date` callback so both
  // chart families consume one implementation.

  for (const s of sessions) {
    const bucket = weeksByKey.get(isoDate(startOfMonday(new Date(s.start_time))));
    if (bucket) bucket.distanceMeters += s.distance_meters;
  }
}
```

Y-axis tick formatter converts meters to display unit and renders integers (e.g., `12 mi`, `18 km`). Tooltip shows `formatDistance(bucketMeters)` + `unitLabel`. Gradient and stroke colors reuse the `#3b82f6` blue from `WorkoutDurationChart` for consistency.

### Running Time Chart — `components/running-time-chart.tsx`

Identical to `RunningMileageChart` but each bucket holds `minutes` (sum of `duration_seconds / 60`) and the Y-axis uses the existing `formatYTick(minutes)` from the lifted chart-format module.

### Shared helpers to lift

To avoid duplication across the four chart components (existing `WorkoutDurationChart`, existing `WorkoutVolumeChart`, new `RunningMileageChart`, new `RunningTimeChart`, new `ActivitiesCombinedChart`):

- `lib/weekly-buckets.ts` — `buildWeeklyBuckets({ days, items, getTimestamp, factory })`. Returns `WeekPoint[]` in chronological order with empty weeks zero-filled.
- `lib/chart-format.ts` — `formatYTick(minutes)`, `formatWeekRangeFromMonday(monday)`, `formatHours(minutes)` (or extend the existing `lib/format.ts`).
- `components/view-button.tsx` — the icon-only `aria-pressed` toggle button used by both analytics cards.
- `components/stat-tile.tsx` — lifted from `app/(app)/running/_components/StatTile.tsx` so the Overview view, the running view, and any future surface share one tile component.

These extractions are part of this SOW because lifting them once is cheaper than copy-pasting four times and reading like an oversight in code review.

### Component file plan

**New files:**

- `app/(app)/activities/page.tsx` — page shell.
- `components/activities/activities-overview-view.tsx`
- `components/activities/workouts-view.tsx`
- `components/activities/running-view.tsx`
- `components/activities/activities-combined-chart.tsx`
- `components/running-analytics.tsx`
- `components/running-mileage-chart.tsx`
- `components/running-time-chart.tsx`
- `components/view-button.tsx`
- `components/stat-tile.tsx`
- `lib/weekly-buckets.ts`
- `lib/chart-format.ts` (or additions to existing `lib/format.ts`)

**Modified:**

- `components/sidebar.tsx` — see Routing & Sidebar.
- `app/(app)/workouts/page.tsx` — collapsed to a server-side redirect.
- `app/(app)/running/page.tsx` — same.
- `app/(app)/workouts/[id]/page.tsx` — back-link retargeting.
- `app/(app)/running/[id]/page.tsx` — back-link retargeting.
- `components/workouts-analytics.tsx` — only if the `ViewButton` extraction lands here too (replace the inline component with the import).
- `components/workout-duration-chart.tsx` — replace inline `summarize` window logic with `buildWeeklyBuckets` import.
- `components/workout-volume-chart.tsx` — same.
- `app/(app)/running/_components/RunListRow.tsx` — no behavior change, but verify the import paths still resolve after `RunningView` moves to `components/activities/`.

**Deleted:**

- `app/(app)/running/_components/StatTile.tsx` — replaced by `components/stat-tile.tsx`. Update the two existing imports (`RunningView`, anywhere else that grep finds).

### Data flow

Each view owns its own fetches. Tab switches do not trigger fetches in other views. Timeframe changes trigger refetches in whichever view is currently rendered.

| Resource | Owner | Refetch trigger |
| --- | --- | --- |
| `user` (for `displayUnit`, `weight_unit`) | `ActivitiesPage` | Mount, 401 |
| `workouts` (overview) | `ActivitiesOverviewView` | Mount, `days` change |
| `sessions` (overview) | `ActivitiesOverviewView` | Mount, `days` change |
| `workouts` (workouts tab) | `WorkoutsView` | Mount, `days` change |
| `exercises` (workouts tab) | `WorkoutsView` | Mount only |
| `metrics` (running tab) | `RunningView` | Mount, after Upload TCX |
| `sessions` (running tab) | `RunningView` | Mount, `days` change, after Upload TCX |

The implementer must verify that `listWorkouts` and `listRunningSessions` both honor `Promise.all` parallel dispatch without sharing in-flight token state — they do today; this is a sanity check, not a change.

### Auth and error handling

Every fetch follows the existing pattern:

- `const token = getToken()`. If null → `router.replace("/login")`.
- On 401 from any endpoint → `clearToken()` + `router.replace("/login")`. Use the existing `handleAuthError` helper pattern from `running/page.tsx` and replicate it in the new views.
- On non-401 error → render inline red panel above the affected view's body. Same idiom as today's pages.
- The shell renders no error itself — all error surfaces live inside the view components since each view owns its own fetches.

### Verification (manual)

There is no existing test suite for the pages this work touches, so verification is manual.

**Golden path:**

1. Cold-load `/activities` → Overview tab renders, timeframe is "Last 30 days", four hero tiles populate, combined chart renders with at least one non-empty week if the test account has activity.
2. Click `Workouts` → URL becomes `/activities?view=workouts`, body swaps to the workouts analytics card + week-grouped list, underline appears on Workouts. The analytics card's 3-view switcher (Time / Volume / Body Parts) defaults to Time.
3. Click each of the three inner switcher icons in turn → charts swap as today.
4. Change timeframe to `Last 7 days` → both the analytics card and the list refetch and re-render with the narrower window.
5. Click `Running` → URL becomes `/activities?view=running`, body swaps to the 4-tile metrics banner + running analytics card + run list, underline appears on Running, Upload TCX button appears in the toolbar left group.
6. Click each of the two new running-analytics icons (Mileage, Time) → charts swap.
7. Change timeframe to `Last 90 days` → the running analytics charts and the run list re-fetch over the wider window; the 4-tile fixed banner is unaffected.
8. Click Upload TCX → existing modal opens; uploading triggers refetch.
9. Click `Overview` → URL strips the `?view` param (back to bare `/activities`), Overview tab renders again.
10. Sidebar: only one `Activities` entry is visible; `/workouts` and `/running` are no longer in the NAV.
11. Direct-load `/workouts` → redirects to `/activities?view=workouts`.
12. Direct-load `/running` → redirects to `/activities?view=running`.
13. Direct-load `/workouts/{valid id}` → detail page renders; the "back" link goes to `/activities?view=workouts`.
14. Direct-load `/running/{valid id}` → detail page renders; the "back" link goes to `/activities?view=running`.
15. Calendar still surfaces both workouts and runs and their detail-page links resolve.

**Edge cases:**

- Direct-load `/activities?view=garbage` → falls back to Overview.
- Empty timeframe (no workouts, no runs in window) → Overview renders four zero tiles, combined chart renders the empty-state copy, secondary tiles render zeros. Workouts view shows its existing EmptyState. Running view shows its existing EmptyState.
- Timeframe = "All" with sparse history → buckets span from the oldest activity to today. No upper truncation note unless the API caps actually bite.
- 401 from any fetch → user lands on `/login`.
- Mobile viewport (narrow) → hero / secondary 4-up tile grids collapse to 2-up (`grid-cols-2`), toolbar tabs wrap if necessary, charts remain readable. (Don't introduce a horizontal scroll on the toolbar; the three tab buttons + Upload TCX must fit on a 375px viewport. If they don't with the chosen labels, shorten to `Mileage` / `Time` and `Upload`.)
- Browser back button after switching tabs: because we use `router.replace`, intra-tab switches don't accumulate history. Back from `/activities?view=running` lands on whatever page preceded `/activities`. Intentional.

### Branch + PR

- One feature branch in `prog-strength-web`, named `activities-tab-consolidation`.
- One pull request against `main`. Description should include the SOW path and a screenshot or two of the new Activities page on Overview, Workouts, and Running.
- One pull request against `prog-strength-docs/main` flipping this SOW's status frontmatter to `shipped` and the body's `**Status**:` to `Shipped` after the web PR is merged.

## Open Questions

1. **Should the Upload TCX button stay on Activities or move into a global header / floating action button?** Options: keep it in the Activities toolbar (only visible on the Running view); promote to a global "+ Add" menu in the top-right that lets the user choose Upload TCX or Quick-add bodyweight / nutrition. Tentative lean: keep it on Activities. The global FAB is a clean follow-up once we have two or three more "ingest from external" flows that all deserve top-level placement; one flow doesn't justify the abstraction.
2. **Should the combined Overview chart be a stacked area or two side-by-side line series?** Options: stacked area (chosen here — emphasizes total volume of activity), grouped bars by week (emphasizes side-by-side comparison), or two distinct line series (de-emphasizes total, easier to read each modality independently). Tentative lean: ship the stacked area for v1; we can revisit if user feedback says the total is misleading or the per-modality trend is hard to read.
3. **Should the timeframe pills persist across reloads via search param or localStorage?** Options: search param (deep-linkable, matches the `view` param), localStorage (persists across deep-links too), neither (current behavior — resets to 30d). Tentative lean: ship "neither" matching today's `/workouts` behavior; persistence is a follow-up if anyone notices.
