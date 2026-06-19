# Bodyweight Readings Table — Timeline-Rail Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the `/bodyweight` log region (desktop `<table>`, mobile card list, and shared pagination) with a single new presentational client component that renders the log as a vertical **timeline rail** — calendar-day nodes, per-reading beads with intra-day spread, and an accent/muted day-over-day delta connector.

**Architecture:** A pure, unit-tested helper (`lib/bodyweight-grouping.ts`) turns the flat `BodyweightEntry[]` into ordered day-groups (local-calendar-date buckets carrying average, spread, and day-over-day delta) and packs those groups into pages without ever splitting a day across a page boundary. A new client component (`components/bodyweight/bodyweight-readings-timeline.tsx`) renders those day-groups as the rail at both breakpoints via CSS container queries. `app/(app)/bodyweight/page.tsx` keeps owning fetch / mutation / goal state and swaps the old table+cards+pagination for the new component, wiring its existing `editingEntry` / `deletingEntry` / `actionTarget` handlers in as callbacks.

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript, Tailwind v4 (container queries), Vitest + Testing Library (jsdom).

---

## Design-system & SOW constraints (read before any task)

- **Tokens only** — conform to `design-system.md` v0.4. Use CSS variables already defined in `app/globals.css`: `--accent` (`#9aa6d6` periwinkle), `--accent-soft`, `--accent-line`, `--muted` (`#7d818c` slate), `--faint`, `--foreground`, `--surface`, `--surface-2`, `--border`, `--border-strong`, `--danger`. **Never hard-code a hex that duplicates a token.** No new palette/accent/type.
- **Accent earns meaning, not chrome.** `--accent` is used for a **down** day-over-day move (toward the cut goal) and active/hover affordances only. An **up** move is `--muted` slate. The rail itself is structural (`--border-strong`). Do not introduce a second hue.
- **Typography** — Manrope throughout (already the app default); numerals carry tight tracking. Use the existing `tabular-nums` utility for figures. No display face.
- **No API / data-model / chart / stat-tile changes.** Everything (day grouping, averages, deltas, spreads) is derived client-side, exactly as the chart's smoothing layer already does.
- **Do not import from `app/design-explore/`** (it does not exist in this repo; PR #80 is a disposable throwaway — clean reimplementation only).

## Decisions resolved from the SOW's Open Questions

- **Delta basis**: day-average vs previous **day-average** (SOW-specified; Open Q2 default). The connector sign is driven by `deltaVsPrevDay`.
- **Per-node goal context**: **toolbar only** (Open Q3 default — "simplest, fewest accents"). The `Log` + goal affordance toolbar stays in `page.tsx` exactly as today, directly above the new component. The timeline component does **not** render per-node goal context and therefore does **not** take a `goal` prop — passing an unused prop would trip `no-unused-vars` lint, and per-node goal context is an explicit non-goal. (The page continues to own goal state and the toolbar; this honors "the page keeps owning goal state" while keeping the component's prop surface honest.)

## File Structure

- **Create** `lib/bodyweight-grouping.ts` — pure helpers: the `DayReading` / `DayGroup` types, `groupByLocalDay(entries, displayUnit)`, and `packDayGroupsIntoPages(groups, readingCap)`. No React, no DOM. Reuses `convertWeight` from `lib/bodyweight-trend.ts` (DRY).
- **Create** `lib/bodyweight-grouping.test.ts` — Vitest unit tests for both helpers.
- **Create** `components/bodyweight/bodyweight-readings-timeline.tsx` — the client component rendering the rail, beads, spread, delta connector, empty state, and internal whole-day-packed pagination.
- **Create** `components/bodyweight/bodyweight-readings-timeline.test.tsx` — component render tests against static fixtures for each required state, at both breakpoints (DOM assertions; container queries don't lay out in jsdom).
- **Modify** `app/(app)/bodyweight/page.tsx` — remove the `BodyweightTable`, `Pagination`, `PaginationBtn` components, the `PAGE_SIZE` constant, the `page` state + `pageEntries`/`totalPages` slice, and render `<BodyweightReadingsTimeline />` in place of `<BodyweightTable />`. Keep the toolbar, all modals, the action sheet, and all handlers. Remove now-unused row-date/time helpers only if they have no remaining caller.

---

## Task 1: `groupByLocalDay` — flat readings → ordered day-groups

**Files:**
- Create: `lib/bodyweight-grouping.ts`
- Test: `lib/bodyweight-grouping.test.ts`

The helper buckets `BodyweightEntry[]` by **local calendar date** over `measured_at`, converts weights to the display unit (reusing `convertWeight`), and returns day-groups **newest day first**. Each group carries the day average, the intra-day spread (`null` for a single reading), and `deltaVsPrevDay` (this day's average minus the chronologically previous day's average; `null` for the oldest day). Readings inside a day are sorted **chronologically ascending** (morning → evening) so a bead cluster reads top-to-bottom in time order.

- [ ] **Step 1: Write the failing test**

Create `lib/bodyweight-grouping.test.ts`:

```ts
/// <reference types="vitest/globals" />

import { groupByLocalDay } from "./bodyweight-grouping";
import type { BodyweightEntry } from "./api";

let nextId = 0;
function entry(weight: number, measured_at: string, unit: "lb" | "kg" = "lb"): BodyweightEntry {
  return { id: `e${nextId++}`, weight, unit, measured_at, created_at: measured_at };
}

describe("groupByLocalDay", () => {
  it("returns an empty array for no entries", () => {
    expect(groupByLocalDay([], "lb")).toEqual([]);
  });

  it("buckets readings by local calendar date, newest day first", () => {
    const groups = groupByLocalDay(
      [
        entry(180, "2026-06-12T08:00:00"),
        entry(181, "2026-06-14T08:00:00"),
        entry(179, "2026-06-13T08:00:00"),
      ],
      "lb",
    );
    expect(groups.map((g) => g.dateKey)).toEqual(["2026-06-14", "2026-06-13", "2026-06-12"]);
  });

  it("clusters multiple readings on one day into a single group with spread", () => {
    const groups = groupByLocalDay(
      [entry(180, "2026-06-15T07:30:00"), entry(183, "2026-06-15T22:15:00")],
      "lb",
    );
    expect(groups).toHaveLength(1);
    expect(groups[0].readings).toHaveLength(2);
    // chronological ascending within the day
    expect(groups[0].readings.map((r) => r.id)).toEqual(["e3", "e4"]);
    expect(groups[0].average).toBeCloseTo(181.5, 5);
    expect(groups[0].spread).toBeCloseTo(3, 5);
  });

  it("gives a single-reading day a null spread", () => {
    const groups = groupByLocalDay([entry(180, "2026-06-15T07:30:00")], "lb");
    expect(groups[0].spread).toBeNull();
  });

  it("keeps a late-PM and an early-AM reading ~80 min apart on different nodes", () => {
    // Jun 15 11:24 PM and Jun 16 12:44 AM are ~80 min apart but different local dates.
    const groups = groupByLocalDay(
      [entry(180, "2026-06-15T23:24:00"), entry(181, "2026-06-16T00:44:00")],
      "lb",
    );
    expect(groups.map((g) => g.dateKey)).toEqual(["2026-06-16", "2026-06-15"]);
  });

  it("computes deltaVsPrevDay as average minus the chronologically previous day's average", () => {
    // Oldest day 180 -> next day 178 (down -2) -> newest 179 (up +1).
    const groups = groupByLocalDay(
      [
        entry(180, "2026-06-12T08:00:00"),
        entry(178, "2026-06-13T08:00:00"),
        entry(179, "2026-06-14T08:00:00"),
      ],
      "lb",
    );
    // newest first: Jun14 (+1 vs Jun13), Jun13 (-2 vs Jun12), Jun12 (null)
    expect(groups[0].deltaVsPrevDay).toBeCloseTo(1, 5);
    expect(groups[1].deltaVsPrevDay).toBeCloseTo(-2, 5);
    expect(groups[2].deltaVsPrevDay).toBeNull();
  });

  it("converts mixed units to the display unit before averaging", () => {
    // 81.65 kg ~= 180.01 lb; averaged with 180 lb stays ~180 lb in display unit.
    const groups = groupByLocalDay(
      [entry(180, "2026-06-15T07:00:00", "lb"), entry(81.65, "2026-06-15T19:00:00", "kg")],
      "lb",
    );
    expect(groups[0].average).toBeCloseTo(180.005, 1);
    // each reading's weight is carried in the display unit
    expect(groups[0].readings[1].unit).toBe("lb");
    expect(groups[0].readings[1].weight).toBeCloseTo(180.01, 1);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npm run test -- lib/bodyweight-grouping.test.ts`
Expected: FAIL — `groupByLocalDay` is not exported / module not found.

- [ ] **Step 3: Write the implementation**

Create `lib/bodyweight-grouping.ts`:

```ts
import type { BodyweightEntry } from "./api";
import { convertWeight } from "./bodyweight-trend";

/** One reading inside a day-group. Weight is in the display unit; the
 * original RFC3339 `measured_at` is retained for the bead's time label. */
export type DayReading = {
  id: string;
  weight: number;
  unit: "lb" | "kg";
  measured_at: string;
};

/** A single calendar day's worth of readings — one node on the rail. */
export type DayGroup = {
  /** Local calendar date "YYYY-MM-DD"; the node identity / grouping key. */
  dateKey: string;
  /** That day's readings, chronologically ascending (morning → evening). */
  readings: DayReading[];
  /** Mean weight for the day, in the display unit (the node headline). */
  average: number;
  /** max − min weight when the day has >1 reading; null for a single reading. */
  spread: number | null;
  /** average − previous (older) day's average; null for the oldest day.
   * Sign drives the connector color: down = accent, up = muted. */
  deltaVsPrevDay: number | null;
};

/** Local-calendar-date key ("YYYY-MM-DD") for an RFC3339 timestamp, using the
 * browser's local time — the same local-date bucketing the chart uses. */
function localDateKey(iso: string): string {
  const d = new Date(iso);
  const pad = (n: number) => String(n).padStart(2, "0");
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`;
}

/**
 * Group flat readings into ordered day-groups, newest day first. Weights are
 * converted to `displayUnit`; the average, spread, and day-over-day delta are
 * all computed in that unit. Bucketing is by **local** calendar date over
 * `measured_at`, so a late-PM and an early-AM reading across midnight land on
 * different nodes.
 */
export function groupByLocalDay(
  entries: BodyweightEntry[],
  displayUnit: "lb" | "kg",
): DayGroup[] {
  const byDay = new Map<string, DayReading[]>();
  for (const e of entries) {
    const key = localDateKey(e.measured_at);
    const arr = byDay.get(key) ?? [];
    arr.push({
      id: e.id,
      weight: convertWeight(e.weight, e.unit, displayUnit),
      unit: displayUnit,
      measured_at: e.measured_at,
    });
    byDay.set(key, arr);
  }

  // Build groups in ascending date order first so deltaVsPrevDay can look at
  // the chronologically previous day, then reverse to newest-first.
  const ascKeys = [...byDay.keys()].sort();
  const ascGroups: DayGroup[] = [];
  let prevAverage: number | null = null;
  for (const key of ascKeys) {
    const readings = (byDay.get(key) ?? [])
      .slice()
      .sort((a, b) => new Date(a.measured_at).getTime() - new Date(b.measured_at).getTime());
    const weights = readings.map((r) => r.weight);
    const average = weights.reduce((a, b) => a + b, 0) / weights.length;
    const spread = weights.length > 1 ? Math.max(...weights) - Math.min(...weights) : null;
    const deltaVsPrevDay = prevAverage === null ? null : average - prevAverage;
    ascGroups.push({ dateKey: key, readings, average, spread, deltaVsPrevDay });
    prevAverage = average;
  }

  return ascGroups.reverse();
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npm run test -- lib/bodyweight-grouping.test.ts`
Expected: PASS — all `groupByLocalDay` cases green.

- [ ] **Step 5: Commit**

```bash
git add lib/bodyweight-grouping.ts lib/bodyweight-grouping.test.ts
git commit -m "feat(bodyweight): add local-date day-grouping helper for the readings rail"
```

---

## Task 2: `packDayGroupsIntoPages` — whole-day pagination packing

**Files:**
- Modify: `lib/bodyweight-grouping.ts`
- Modify: `lib/bodyweight-grouping.test.ts`

Paginate by **day-group with whole-day packing**: fill a page up to `readingCap` (default 20) readings but **never split a day-group across a page boundary**. A day that would straddle the cap moves wholly to the next page. A single day larger than the cap occupies its own page (it cannot be split). Groups arrive newest-first and stay newest-first, so the most-recent day is at the top of page 1.

- [ ] **Step 1: Write the failing test**

Append to `lib/bodyweight-grouping.test.ts`:

```ts
import { packDayGroupsIntoPages } from "./bodyweight-grouping";
import type { DayGroup } from "./bodyweight-grouping";

function group(dateKey: string, readingCount: number): DayGroup {
  const readings = Array.from({ length: readingCount }, (_, i) => ({
    id: `${dateKey}-${i}`,
    weight: 180,
    unit: "lb" as const,
    measured_at: `${dateKey}T0${i}:00:00`,
  }));
  return { dateKey, readings, average: 180, spread: null, deltaVsPrevDay: null };
}

describe("packDayGroupsIntoPages", () => {
  it("returns no pages for no groups", () => {
    expect(packDayGroupsIntoPages([], 20)).toEqual([]);
  });

  it("keeps everything on one page when under the cap", () => {
    const pages = packDayGroupsIntoPages([group("2026-06-14", 2), group("2026-06-13", 1)], 20);
    expect(pages).toHaveLength(1);
    expect(pages[0].map((g) => g.dateKey)).toEqual(["2026-06-14", "2026-06-13"]);
  });

  it("moves a day that would straddle the cap wholly to the next page", () => {
    // 18 + 5 = 23 > 20, so the 5-reading day starts page 2 intact.
    const pages = packDayGroupsIntoPages([group("2026-06-14", 18), group("2026-06-13", 5)], 20);
    expect(pages).toHaveLength(2);
    expect(pages[0].map((g) => g.dateKey)).toEqual(["2026-06-14"]);
    expect(pages[1].map((g) => g.dateKey)).toEqual(["2026-06-13"]);
  });

  it("never splits a single day's readings across pages", () => {
    const pages = packDayGroupsIntoPages([group("2026-06-14", 12), group("2026-06-13", 12)], 20);
    expect(pages).toHaveLength(2);
    expect(pages[0][0].readings).toHaveLength(12);
    expect(pages[1][0].readings).toHaveLength(12);
  });

  it("gives a day larger than the cap its own page", () => {
    const pages = packDayGroupsIntoPages([group("2026-06-14", 25), group("2026-06-13", 3)], 20);
    expect(pages).toHaveLength(2);
    expect(pages[0].map((g) => g.dateKey)).toEqual(["2026-06-14"]);
    expect(pages[1].map((g) => g.dateKey)).toEqual(["2026-06-13"]);
  });

  it("preserves newest-first ordering across pages", () => {
    const pages = packDayGroupsIntoPages(
      [group("2026-06-14", 15), group("2026-06-13", 10), group("2026-06-12", 10)],
      20,
    );
    expect(pages).toHaveLength(2);
    expect(pages[0].map((g) => g.dateKey)).toEqual(["2026-06-14"]);
    expect(pages[1].map((g) => g.dateKey)).toEqual(["2026-06-13", "2026-06-12"]);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npm run test -- lib/bodyweight-grouping.test.ts`
Expected: FAIL — `packDayGroupsIntoPages` is not exported.

- [ ] **Step 3: Write the implementation**

Append to `lib/bodyweight-grouping.ts`:

```ts
/**
 * Pack day-groups into pages of up to `readingCap` readings each, never
 * splitting a day-group across a page boundary. A day that would push the
 * current page over the cap starts the next page intact; a single day larger
 * than the cap occupies its own page. Input order (newest-first) is preserved,
 * so the most-recent day stays at the top of page 1.
 */
export function packDayGroupsIntoPages(groups: DayGroup[], readingCap: number): DayGroup[][] {
  const pages: DayGroup[][] = [];
  let current: DayGroup[] = [];
  let currentCount = 0;
  for (const g of groups) {
    const n = g.readings.length;
    if (current.length > 0 && currentCount + n > readingCap) {
      pages.push(current);
      current = [];
      currentCount = 0;
    }
    current.push(g);
    currentCount += n;
  }
  if (current.length > 0) pages.push(current);
  return pages;
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npm run test -- lib/bodyweight-grouping.test.ts`
Expected: PASS — both `groupByLocalDay` and `packDayGroupsIntoPages` suites green.

- [ ] **Step 5: Commit**

```bash
git add lib/bodyweight-grouping.ts lib/bodyweight-grouping.test.ts
git commit -m "feat(bodyweight): add whole-day pagination packing for the readings rail"
```

---

## Task 3: `BodyweightReadingsTimeline` component

**Files:**
- Create: `components/bodyweight/bodyweight-readings-timeline.tsx`
- Test: `components/bodyweight/bodyweight-readings-timeline.test.tsx`

A presentational client component over already-fetched, already-range-filtered data. It groups the entries, packs them into pages, and renders the rail. Edit/delete intents are emitted back up via callbacks; mobile day/bead taps emit a single `onTapReading` (the page opens the existing `BodyweightActionSheet`). The toolbar, modals, and action sheet all live in `page.tsx` — this component renders **only** the rail region (nodes + beads + connectors + pagination + empty state).

**Props interface (exact):**

```ts
export type BodyweightReadingsTimelineProps = {
  entries: BodyweightEntry[]; // already range-filtered by the page
  displayUnit: "lb" | "kg";
  onEdit: (entry: BodyweightEntry) => void; // desktop pencil
  onDelete: (entry: BodyweightEntry) => void; // desktop trash
  onTapReading: (entry: BodyweightEntry) => void; // mobile tap → action sheet
};
```

**Rendering spec:**

- Root `<section className="@container">` so descendants can use Tailwind v4 **container-query** variants (`@[NNNpx]:`), making the layout respond to the region's own width, not the viewport — satisfying "both breakpoints via container queries."
- **Empty state:** when `entries.length === 0`, render a single hairline-panel message. The page distinguishes "no readings at all" from "none in range," so this component shows the in-range fallback text: `"No readings in this range — try widening the time range above."` (The page renders the true first-run "Tap Log to add your first reading" copy itself before mounting the timeline — see Task 4. The timeline is only mounted when `entries.length > 0`, but it still renders a safe empty state defensively.)
- **Rail + nodes:** a `<ol>` of day-nodes, newest first. Each node has a left **rail column** (fixed width, e.g. `w-7 @[420px]:w-9`) containing a hairline vertical line (`bg-[var(--border-strong)]`) and a node dot, and a **content column** with the day header and beads.
- **Node dot:** a small ring on the rail. The newest node (index 0 of page 1) reads as active with an `--accent` ring (`border-[var(--accent)]`); other dots are `border-[var(--border-strong)]` on `--surface`.
- **Day header:** the local date label (`weekday, Mon D` from `dateKey`) in `--muted`, and the day **average** + unit as the headline figure (`tabular-nums`, `--foreground`, medium weight — not an oversized hero numeral). When the day has >1 reading, show a quiet `N readings` count.
- **Beads:**
  - Single-reading day → one bead row: the reading value + unit and its time (`h:mm AM/PM`), with desktop edit/delete affordances.
  - Multi-reading day → one bead row per reading (value + time + affordances), and the **intra-day spread** rendered as a bracketed marker beside/between the cluster, e.g. `↕ {spread} {unit}`, in `--muted`. The spread must remain visible at ≤360px — render it as its own line under the cluster on narrow containers (`@[420px]:` may inline it). Never truncate it.
- **Connector / delta:** between a node and the next (older) one, a short connector segment on the rail plus a small delta label. Use `deltaVsPrevDay`:
  - `delta < 0` (down, toward goal) → `--accent` (`text-[var(--accent)]`, connector `bg-[var(--accent-line)]`), label `↓ {abs delta} {unit}`.
  - `delta > 0` (up) → `--muted` slate (`text-[var(--muted)]`, connector `bg-[var(--border-strong)]`), label `↑ {abs delta} {unit}`.
  - `delta === 0` → `--muted`, label `± 0 {unit}`.
  - `delta === null` (oldest node) → no connector label; the rail line still renders for structure.
  The delta label must remain visible at ≤360px (own line, never clipped).
- **Desktop edit/delete:** per-reading `Edit` / `Delete` icon buttons that are visually revealed on hover/focus of the bead row (`opacity-0 group-hover:opacity-100 focus-within:opacity-100`, plus `focus:opacity-100` on the buttons themselves for keyboard access) at container widths `@[420px]:` and up; hidden below that width. Wire to `onEdit(entry)` / `onDelete(entry)`. Reuse the pencil/trash SVG idiom already in the page (inline these small SVGs in this file; they are tiny and local — do not add a shared module for two icons).
- **Mobile tap:** below `@[420px]:` the whole bead row is a button firing `onTapReading(entry)`; the page opens `BodyweightActionSheet`. (Desktop bead rows are not buttons — they expose the icon buttons instead — to avoid a button-in-button.)
- **Pagination:** internal `page` state. Compute `pages = packDayGroupsIntoPages(groups, 20)`, `totalPages = Math.max(1, pages.length)`, clamp the shown page to `totalPages`, and render the current page's nodes. Reset to page 1 when `entries` changes (`useEffect(() => setPage(1), [entries])`). Render page controls **below** the rail, only when `totalPages > 1`. Keep the controls simple: `Page X of Y` plus Prev/Next buttons (First/Last optional — match the existing footer's token styling: hairline border, `--surface`, disabled at the ends).
- **Number formatting:** add a local `formatNumber` (at most one decimal, no trailing zero, em-dash for non-finite) — identical contract to `lib/format.ts`'s `formatNumber`; import it from `@/lib/format` rather than re-defining (DRY).
- **Time/date formatting:** local helpers `formatBeadTime(iso)` (`h:mm AM/PM`) and `formatNodeDate(dateKey)` (parse the `YYYY-MM-DD` as a local date and format `weekday, Mon D`). Parse `dateKey` with `new Date(year, monthIndex, day)` (not `new Date("YYYY-MM-DD")`, which is UTC) so the label matches the local bucket.

- [ ] **Step 1: Write the failing test**

Create `components/bodyweight/bodyweight-readings-timeline.test.tsx`:

```tsx
/// <reference types="vitest/globals" />

import { render, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import type { BodyweightEntry } from "@/lib/api";
import { BodyweightReadingsTimeline } from "./bodyweight-readings-timeline";

let nextId = 0;
function entry(weight: number, measured_at: string, unit: "lb" | "kg" = "lb"): BodyweightEntry {
  return { id: `e${nextId++}`, weight, unit, measured_at, created_at: measured_at };
}

function renderTimeline(entries: BodyweightEntry[], over: Partial<Parameters<typeof BodyweightReadingsTimeline>[0]> = {}) {
  const props = {
    entries,
    displayUnit: "lb" as const,
    onEdit: vi.fn(),
    onDelete: vi.fn(),
    onTapReading: vi.fn(),
    ...over,
  };
  render(<BodyweightReadingsTimeline {...props} />);
  return props;
}

describe("BodyweightReadingsTimeline", () => {
  it("renders an empty state when there are no entries", () => {
    renderTimeline([]);
    expect(screen.getByText(/no readings in this range/i)).toBeInTheDocument();
  });

  it("renders one node per local calendar day, newest first", () => {
    renderTimeline([
      entry(180, "2026-06-12T08:00:00"),
      entry(181, "2026-06-14T08:00:00"),
      entry(179, "2026-06-13T08:00:00"),
    ]);
    const nodes = screen.getAllByRole("listitem");
    expect(nodes).toHaveLength(3);
    // newest first: Jun 14 appears before Jun 12
    expect(within(nodes[0]).getByText(/jun 14/i)).toBeInTheDocument();
    expect(within(nodes[2]).getByText(/jun 12/i)).toBeInTheDocument();
  });

  it("shows a multi-reading day's spread", () => {
    renderTimeline([
      entry(180, "2026-06-15T07:30:00"),
      entry(183, "2026-06-15T22:15:00"),
    ]);
    // spread = 3 lb, shown somewhere in the (single) node
    expect(screen.getByText(/3 lb/)).toBeInTheDocument();
    // both bead times render
    expect(screen.getByText(/7:30/i)).toBeInTheDocument();
    expect(screen.getByText(/10:15/i)).toBeInTheDocument();
  });

  it("labels a down move with the down delta and an up move with the up delta", () => {
    // ascending 180 -> 178 (down) -> 179 (up); newest first node is the up move.
    renderTimeline([
      entry(180, "2026-06-12T08:00:00"),
      entry(178, "2026-06-13T08:00:00"),
      entry(179, "2026-06-14T08:00:00"),
    ]);
    expect(screen.getByText(/↑\s*1 lb/)).toBeInTheDocument();
    expect(screen.getByText(/↓\s*2 lb/)).toBeInTheDocument();
  });

  it("fires onEdit and onDelete from the desktop affordances", async () => {
    const user = userEvent.setup();
    const props = renderTimeline([entry(180, "2026-06-15T07:30:00")]);
    await user.click(screen.getByRole("button", { name: /edit reading/i }));
    expect(props.onEdit).toHaveBeenCalledTimes(1);
    await user.click(screen.getByRole("button", { name: /delete reading/i }));
    expect(props.onDelete).toHaveBeenCalledTimes(1);
  });

  it("fires onTapReading from the mobile bead button", async () => {
    const user = userEvent.setup();
    const props = renderTimeline([entry(180, "2026-06-15T07:30:00")]);
    // The mobile tap target carries an accessible name with the weight.
    await user.click(screen.getByRole("button", { name: /180 lb on/i }));
    expect(props.onTapReading).toHaveBeenCalledTimes(1);
  });

  it("paginates by whole days without splitting a day, controls below", () => {
    // Two 12-reading days => 2 pages at the 20 cap; day not split.
    const day1 = Array.from({ length: 12 }, (_, i) =>
      entry(180, `2026-06-14T0${i % 10}:0${i % 6}:00`),
    );
    const day2 = Array.from({ length: 12 }, (_, i) =>
      entry(179, `2026-06-13T0${i % 10}:0${i % 6}:00`),
    );
    renderTimeline([...day1, ...day2]);
    expect(screen.getByText(/page 1 of 2/i)).toBeInTheDocument();
    // page 1 shows only the newest day node
    expect(screen.getAllByRole("listitem")).toHaveLength(1);
    expect(screen.getByText(/jun 14/i)).toBeInTheDocument();
  });

  it("renders mixed-unit days in the display unit", () => {
    renderTimeline([entry(81.65, "2026-06-15T07:00:00", "kg")], { displayUnit: "lb" });
    // ~180 lb headline, not 81.65
    expect(screen.getByText(/180 lb/)).toBeInTheDocument();
  });
});
```

> Note: `@testing-library/user-event` is available transitively via `@testing-library/react`; if the import fails, fall back to `fireEvent` from `@testing-library/react` (`fireEvent.click(...)`). Verify the package resolves during Step 2 and adjust the test imports accordingly — do **not** add a new dependency.

- [ ] **Step 2: Run the test to verify it fails**

Run: `npm run test -- components/bodyweight/bodyweight-readings-timeline.test.tsx`
Expected: FAIL — component module not found.

- [ ] **Step 3: Write the implementation**

Create `components/bodyweight/bodyweight-readings-timeline.tsx` implementing the rendering spec above. Use only existing CSS-variable tokens. Structure it roughly as:

```tsx
"use client";

import { useEffect, useState } from "react";
import type { BodyweightEntry } from "@/lib/api";
import { formatNumber } from "@/lib/format";
import {
  groupByLocalDay,
  packDayGroupsIntoPages,
  type DayGroup,
  type DayReading,
} from "@/lib/bodyweight-grouping";

const READING_CAP = 20;

export type BodyweightReadingsTimelineProps = {
  entries: BodyweightEntry[];
  displayUnit: "lb" | "kg";
  onEdit: (entry: BodyweightEntry) => void;
  onDelete: (entry: BodyweightEntry) => void;
  onTapReading: (entry: BodyweightEntry) => void;
};

export function BodyweightReadingsTimeline({
  entries,
  displayUnit,
  onEdit,
  onDelete,
  onTapReading,
}: BodyweightReadingsTimelineProps) {
  const [page, setPage] = useState(1);
  useEffect(() => setPage(1), [entries]);

  if (entries.length === 0) {
    return (
      <p className="rounded-[14px] border border-[var(--border)] bg-[var(--surface)] px-4 py-6 text-center text-sm text-[var(--muted)]">
        No readings in this range — try widening the time range above.
      </p>
    );
  }

  const groups = groupByLocalDay(entries, displayUnit);
  const pages = packDayGroupsIntoPages(groups, READING_CAP);
  const totalPages = Math.max(1, pages.length);
  const current = Math.min(page, totalPages);
  const pageGroups = pages[current - 1] ?? [];

  return (
    <section className="@container">
      <ol className="flex flex-col">
        {pageGroups.map((group, i) => (
          <DayNode
            key={group.dateKey}
            group={group}
            isNewest={current === 1 && i === 0}
            onEdit={onEdit}
            onDelete={onDelete}
            onTapReading={onTapReading}
            entriesById={entries}
          />
        ))}
      </ol>
      {totalPages > 1 && (
        <TimelinePagination page={current} totalPages={totalPages} onPageChange={setPage} />
      )}
    </section>
  );
}
```

Then implement:

- `DayNode` — renders the rail column (line + dot, accent ring when `isNewest`), the day header (`formatNodeDate(group.dateKey)`, `formatNumber(group.average)` + unit, optional `N readings`), the bead cluster, the spread (when `group.spread !== null`), and the connector/delta block (when `group.deltaVsPrevDay !== null`). The reading's display unit is `group.readings[0].unit`.
- `Bead` — one reading row. To call `onEdit/onDelete/onTapReading` with the original `BodyweightEntry`, resolve it from the `entries` array by `id` (pass a `Map<string, BodyweightEntry>` built once in the parent, or look up with `entries.find`). Provide the desktop pencil/trash `IconButton`s (`aria-label="Edit reading"` / `"Delete reading"`, revealed on hover/focus, shown only `@[420px]:`), and the mobile full-row button (`aria-label={`${formatNumber(weight)} ${unit} on ${date} at ${time} — tap to edit or delete`}`, shown only below `@[420px]:` via `@[420px]:hidden`).
- Inline `PencilIcon`, `TrashIcon` SVGs (copy the 16px stroke idiom from `bodyweight-action-sheet.tsx`).
- `DeltaConnector` — the `↓/↑/±` label colored per the sign rules above.
- `SpreadMarker` — `↕ {formatNumber(spread)} {unit}` in `--muted`, on its own line at narrow widths.
- `TimelinePagination` — `Page X of Y` + Prev/Next buttons (token-styled, disabled at the ends), `onPageChange`.
- `formatBeadTime(iso)` and `formatNodeDate(dateKey)` helpers as specified (local parsing for `dateKey`).

Build the `id → entry` lookup once (`new Map(entries.map((e) => [e.id, e]))`) and thread it down, so a bead can emit the original entry without re-deriving.

- [ ] **Step 4: Run the test to verify it passes**

Run: `npm run test -- components/bodyweight/bodyweight-readings-timeline.test.tsx`
Expected: PASS — all component cases green. Iterate on markup/labels until green (adjust the test's text matchers only if the chosen copy differs trivially, e.g. time-format spacing — keep the asserted *behavior* intact).

- [ ] **Step 5: Typecheck + lint the new files**

Run: `npm run typecheck && npm run lint`
Expected: PASS — no type errors, no new lint errors/warnings.

- [ ] **Step 6: Commit**

```bash
git add components/bodyweight/bodyweight-readings-timeline.tsx components/bodyweight/bodyweight-readings-timeline.test.tsx
git commit -m "feat(bodyweight): add timeline-rail readings component"
```

---

## Task 4: Swap the timeline into `/bodyweight` page

**Files:**
- Modify: `app/(app)/bodyweight/page.tsx`

Replace the old log region (table + mobile cards + shared pagination) with the new component, keeping every handler, modal, the toolbar, and the action sheet. The page still owns fetch / mutation / goal state and passes `entriesInRange` straight through.

- [ ] **Step 1: Wire in the component**

In `app/(app)/bodyweight/page.tsx`:

1. Add the import near the other component imports:

```tsx
import { BodyweightReadingsTimeline } from "@/components/bodyweight/bodyweight-readings-timeline";
```

2. Remove the client-side pagination plumbing that is now internal to the component:
   - the `const PAGE_SIZE = 20;` constant,
   - the `const [page, setPage] = useState(1);` state,
   - the `totalPages` + `pageEntries` `useMemo`s,
   - the `useEffect(() => { setPage(1); }, [range]);` effect.

3. Replace the `{entries && entriesInRange.length > 0 && (<BodyweightTable ... />)}` block with:

```tsx
{entries && entriesInRange.length > 0 && (
  <BodyweightReadingsTimeline
    entries={entriesInRange}
    displayUnit={displayUnit}
    onEdit={(entry) => setEditingEntry(entry)}
    onDelete={(entry) => setDeletingEntry(entry)}
    onTapReading={(entry) => setActionTarget(entry)}
  />
)}
```

   Keep the preceding loading (`entries === null`) and empty (`entriesInRange.length === 0`) branches exactly as they are — the page's "No readings yet. Tap Log to add your first reading." vs "No readings in this range…" distinction stays in the page.

4. Delete the now-unused `BodyweightTable`, `Pagination`, and `PaginationBtn` function components.

5. Delete now-unused helpers **only if they have no remaining caller**: `formatRowDate`, `formatMobileRowDate`, `formatRowTime` are used by the delete modal (`formatRowDate` + `formatRowTime`) — keep those; `formatMobileRowDate` was table-only — remove it if nothing else references it. Verify with a grep before deleting.

6. Keep `isMobile` / `useSyncExternalStore` only if still referenced. After the swap, `isMobile` is still passed to `<TrendSection>` — leave it. (Do not remove the media-query plumbing.)

- [ ] **Step 2: Verify nothing dangling**

Run:
```bash
grep -n "BodyweightTable\|PAGE_SIZE\|pageEntries\|formatMobileRowDate\|\bsetPage\b" "app/(app)/bodyweight/page.tsx"
```
Expected: no matches for `BodyweightTable`, `PAGE_SIZE`, `pageEntries`, `formatMobileRowDate`, or `setPage` (the table-era ones). If `formatMobileRowDate` still has a caller, leave it; otherwise it must be gone.

- [ ] **Step 3: Typecheck, lint, test, build**

Run: `npm run typecheck && npm run lint && npm run test && npm run build`
Expected: all PASS. Fix any unused-import / unused-var fallout from the deletions.

- [ ] **Step 4: Commit**

```bash
git add "app/(app)/bodyweight/page.tsx"
git commit -m "feat(bodyweight): swap the readings table for the timeline-rail"
```

---

## Task 5: Full-suite gate + final review

**Files:** none (verification only)

- [ ] **Step 1: Run the complete CI gate**

Run:
```bash
npm run typecheck && npm run lint && npm run test && npm run format:check && npm run build
```
Expected: all PASS, no Prettier diffs, no new lint warnings. If `format:check` flags files, run `npm run format` and re-commit.

- [ ] **Step 2: Self-review against the SOW state matrix**

Confirm each required state is exercised by a test or visible in the component: multi-per-day (spread), single-reading day, up vs down (connector color/label), cross-midnight day boundary (grouping test), no goal set (toolbar unchanged, no per-node goal), pagination boundary (whole-day packing), empty/first reading (page empty branch + component empty state), unit lb/kg (mixed-unit test). Note any gap and add a test.

- [ ] **Step 3: Commit any fixups**

```bash
git add -A && git commit -m "test(bodyweight): cover remaining timeline-rail states"
```
(Skip if nothing changed.)

---

## Self-Review (plan author)

- **Spec coverage:** day-grouping (T1), spread/average/delta (T1), whole-day packing (T2), rail/beads/connector/both-breakpoints/edit-delete/create-preserved (T3+T4), all required states (T1–T3 tests + T5), design-system tokens (T3), tests incl. cross-midnight + mixed units + packing (T1–T3). ✅
- **Non-goals respected:** no API/chart/stat-tile/token changes; toolbar + goal behavior preserved; no import from `app/design-explore`. ✅
- **Type consistency:** `DayGroup` / `DayReading` shapes and `groupByLocalDay` / `packDayGroupsIntoPages` / `BodyweightReadingsTimelineProps` signatures are referenced identically across tasks. ✅
- **No placeholders:** every code step shows real code or real commands. ✅
