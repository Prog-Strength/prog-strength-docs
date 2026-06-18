# Bodyweight Page Redesign (trend-band-analyst) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebuild the `/bodyweight` chart-and-stats region in `prog-strength-web` into the `trend-band-analyst` composition — a smoothed violet EWMA trend curve as the hero, wrapped in a faint ±spread band, with demoted slate raw dots, a rate-of-change readout, and a compact caption strip — leaving the log table and all modals functionally intact.

**Architecture:** Add two pure `lib/` modules (a smoothing/stats module and a colocated chart-color constants module), extract the trend visualization region into one focused page-private component (`_components/trend-section.tsx`), then re-wire `page.tsx` to use it and restyle the range tabs. No backend change: the trend, band, and weekly rate are derived client-side from the readings the page already fetches via `listBodyweight` / `getBodyweightGoal`. The design conforms to `prog-strength-docs/design-system.md` (slate ramp, single violet accent, Nunito + the condensed `var(--font-display)` face for the hero numeral); because Recharts writes stroke/fill as SVG attributes where CSS `var(--token)` does not resolve, the system tokens are mirrored as literal constants in one colocated module.

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript, Tailwind v4, Recharts 3.x, Vitest + Testing Library.

---

## File Structure

- **Create** `lib/bodyweight-trend.ts` — pure numeric derivations: `convertWeight`, `buildDayPoints` (group readings by local calendar day → `{ t, avg, spread }`), `computeStats` (Average/Δ/Min+date/Max+date), `ewmaTrend` (α 0.25 smoothing + ±band), `trendSummary` (current value + time-normalized weekly rate). One responsibility: the bodyweight math the chart and caption read from.
- **Create** `lib/bodyweight-trend.test.ts` — unit tests for the above.
- **Create** `lib/bodyweight-chart-colors.ts` — literal hex/rgba constants mirroring the design-system tokens, used for Recharts SVG attributes only.
- **Create** `app/(app)/bodyweight/_components/trend-section.tsx` — the trend visualization region: trend hero readout (overline, condensed-display current value, rate pill, legend), the Recharts `ComposedChart` (band `Area` → slate `Scatter` → violet `Line` → goal `ReferenceLine`) with a custom tooltip, and the caption strip. Exposes `TrendSection` (default region) and `TrendTooltip` (named, for the regression test).
- **Create** `app/(app)/bodyweight/_components/trend-section.test.tsx` — component + tooltip tests.
- **Modify** `app/(app)/bodyweight/page.tsx` — replace `ChartCard` usage with `<TrendSection>`, restyle `TimeRangeTabs` to quiet pills + a `"{n} readings · ending {date}"` caption, retone the `TargetIcon` off `#10b981`, and delete the now-dead `ChartCard` / `AccentStatTile` / chart `Legend` / `COLOR_AVG` / `COLOR_RAW` / `convertWeight` / `computeStats` / `mean` / `Stats` / `formatShortDate` and the now-unused `recharts` import.

---

## Task 1: Smoothing & stats library

**Files:**
- Create: `lib/bodyweight-trend.ts`
- Test: `lib/bodyweight-trend.test.ts`

Background: today `page.tsx` groups readings by local calendar day, converts each to the display unit, and means them, timestamping the day point at local noon (`dayStart + 12h`). This task lifts that grouping into a pure, tested module and adds the EWMA smoothing layer and the weekly rate, ported faithfully from the DX `_data.ts`.

- [ ] **Step 1: Write the failing tests**

Create `lib/bodyweight-trend.test.ts`:

```ts
/// <reference types="vitest/globals" />

import { describe, expect, it } from "vitest";
import type { BodyweightEntry } from "./api";
import {
  buildDayPoints,
  computeStats,
  convertWeight,
  ewmaTrend,
  trendSummary,
  type DayPoint,
} from "./bodyweight-trend";

const DAY = 24 * 60 * 60 * 1000;

// Local-noon day point factory: `day` is days offset from an arbitrary
// epoch base, kept tz-independent by working in raw ms.
function dp(day: number, avg: number, spread = 0): DayPoint {
  return { t: day * DAY + 12 * 60 * 60 * 1000, avg, spread };
}

// Minimal BodyweightEntry factory — only weight/unit/measured_at matter.
let nextId = 0;
function entry(weight: number, measured_at: string, unit: "lb" | "kg" = "lb"): BodyweightEntry {
  return { id: `e${nextId++}`, weight, unit, measured_at, created_at: measured_at };
}

describe("convertWeight", () => {
  it("is identity when units match", () => {
    expect(convertWeight(180, "lb", "lb")).toBe(180);
  });
  it("converts kg → lb", () => {
    expect(convertWeight(100, "kg", "lb")).toBeCloseTo(220.462, 2);
  });
});

describe("buildDayPoints", () => {
  it("groups multi-per-day readings into one mean point with intra-day spread", () => {
    const points = buildDayPoints(
      [
        entry(180, "2026-06-01T07:00:00"),
        entry(182, "2026-06-01T19:00:00"),
        entry(181, "2026-06-02T07:00:00"),
      ],
      "lb",
    );
    expect(points).toHaveLength(2);
    expect(points[0].avg).toBeCloseTo(181, 5); // (180+182)/2
    expect(points[0].spread).toBeCloseTo(2, 5); // 182-180
    expect(points[1].avg).toBeCloseTo(181, 5);
    expect(points[1].spread).toBe(0); // single reading
    expect(points[0].t).toBeLessThan(points[1].t); // sorted ascending
  });
});

describe("ewmaTrend", () => {
  it("seeds at the first day's mean and weights by alpha", () => {
    const out = ewmaTrend([dp(0, 200), dp(1, 210)], 0.25);
    expect(out[0].trend).toBeCloseTo(200, 5); // seed = first mean
    expect(out[1].trend).toBeCloseTo(0.25 * 210 + 0.75 * 200, 5); // 202.5
  });
  it("floors the band at 0.6 and adds half the intra-day spread", () => {
    const out = ewmaTrend([dp(0, 200, 4)], 0.25);
    // first point: dev=0 → variance 0 → sqrt floored to 0.6; +spread/2 = 2
    expect(out[0].hi - out[0].trend).toBeCloseTo(0.6 + 2, 5);
    expect(out[0].trend - out[0].lo).toBeCloseTo(0.6 + 2, 5);
  });
  it("handles single-day input: trend equals that mean, minimal band", () => {
    const out = ewmaTrend([dp(0, 175)], 0.25);
    expect(out).toHaveLength(1);
    expect(out[0].trend).toBeCloseTo(175, 5);
    expect(out[0].hi - out[0].trend).toBeCloseTo(0.6, 5); // spread 0
  });
  it("returns [] for empty input", () => {
    expect(ewmaTrend([], 0.25)).toEqual([]);
  });
});

describe("trendSummary", () => {
  it("computes the weekly rate over a clean 7-day window", () => {
    const trend = ewmaTrend([dp(0, 200), dp(7, 193)], 0.25);
    const s = trendSummary(trend);
    expect(s.hasRate).toBe(true);
    expect(s.current).toBeCloseTo(trend[trend.length - 1].trend, 5);
    // ref is the point exactly 7 days back; rate = (last-ref)/7 * 7 = last-ref
    expect(s.ratePerWeek).toBeCloseTo(trend[1].trend - trend[0].trend, 5);
  });
  it("normalizes a sparse/gapped window by true days, not index", () => {
    // points at day 0 and day 10 only; ref = day-0 point (>= 7 days back)
    const trend = ewmaTrend([dp(0, 200), dp(10, 190)], 0.25);
    const s = trendSummary(trend);
    expect(s.hasRate).toBe(true);
    const expected = ((trend[1].trend - trend[0].trend) / 10) * 7;
    expect(s.ratePerWeek).toBeCloseTo(expected, 5);
  });
  it("hides the rate for a single point", () => {
    const s = trendSummary(ewmaTrend([dp(0, 200)], 0.25));
    expect(s.hasRate).toBe(false);
    expect(s.ratePerWeek).toBeNull();
    expect(s.current).toBeCloseTo(200, 5);
  });
  it("hides the rate when the range spans under a week", () => {
    const s = trendSummary(ewmaTrend([dp(0, 200), dp(3, 199)], 0.25));
    expect(s.hasRate).toBe(false);
    expect(s.ratePerWeek).toBeNull();
  });
  it("returns nulls for empty input", () => {
    expect(trendSummary([])).toEqual({ current: null, ratePerWeek: null, hasRate: false });
  });
});

describe("computeStats", () => {
  it("computes average, range delta, and dated min/max", () => {
    const stats = computeStats(
      [
        entry(200, "2026-06-01T08:00:00"),
        entry(196, "2026-06-08T08:00:00"),
        entry(198, "2026-06-15T08:00:00"),
      ],
      "lb",
    );
    expect(stats.count).toBe(3);
    expect(stats.avg).toBeCloseTo(198, 5);
    expect(stats.min?.weight).toBeCloseTo(196, 5);
    expect(stats.max?.weight).toBeCloseTo(200, 5);
    expect(stats.delta).toBeCloseTo(-2, 5); // last day mean 198 - first day mean 200
  });
  it("returns nulls for empty input", () => {
    const stats = computeStats([], "lb");
    expect(stats.avg).toBeNull();
    expect(stats.min).toBeNull();
    expect(stats.delta).toBeNull();
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm run test -- lib/bodyweight-trend.test.ts`
Expected: FAIL — module `./bodyweight-trend` not found.

- [ ] **Step 3: Write the implementation**

Create `lib/bodyweight-trend.ts`:

```ts
import type { BodyweightEntry } from "./api";

const LB_PER_KG = 2.20462;
const DAY_MS = 24 * 60 * 60 * 1000;

/** Convert a weight between lb and kg. Identity when units match. */
export function convertWeight(weight: number, from: "lb" | "kg", to: "lb" | "kg"): number {
  if (from === to) return weight;
  return from === "kg" ? weight * LB_PER_KG : weight / LB_PER_KG;
}

/** One per-day aggregate: local-noon timestamp, daily mean (in display
 * unit), and the intra-day max−min spread (0 for a single reading). */
export type DayPoint = { t: number; avg: number; spread: number };

/** Group readings by LOCAL calendar day, mean them in the display unit,
 * and timestamp each day at local noon — the same grouping the chart has
 * always used. Returns points sorted ascending by time. */
export function buildDayPoints(entries: BodyweightEntry[], displayUnit: "lb" | "kg"): DayPoint[] {
  const byDay = new Map<number, number[]>();
  for (const e of entries) {
    const d = new Date(e.measured_at);
    const dayStart = new Date(d.getFullYear(), d.getMonth(), d.getDate()).getTime();
    const arr = byDay.get(dayStart) ?? [];
    arr.push(convertWeight(e.weight, e.unit, displayUnit));
    byDay.set(dayStart, arr);
  }
  const points: DayPoint[] = [];
  for (const [dayStart, weights] of byDay) {
    const avg = weights.reduce((a, b) => a + b, 0) / weights.length;
    const spread = Math.max(...weights) - Math.min(...weights);
    points.push({ t: dayStart + 12 * 60 * 60 * 1000, avg, spread });
  }
  points.sort((a, b) => a.t - b.t);
  return points;
}

/** One point on the smoothed trend: the EWMA value plus the ±band edges. */
export type TrendPoint = { t: number; trend: number; lo: number; hi: number };

/** TrendWeight-style exponentially-weighted moving average over the daily
 * means (α 0.25 by default). Seeds `s` at the first day's mean, then
 * `s = α·avg + (1−α)·s`; tracks an EW variance of the residual and emits a
 * ±band of `max(√v, 0.6) + spread/2`. Ported from the DX `_data.ts`. */
export function ewmaTrend(dayPoints: DayPoint[], alpha = 0.25): TrendPoint[] {
  if (dayPoints.length === 0) return [];
  let s = dayPoints[0].avg;
  let v = 0;
  const out: TrendPoint[] = [];
  for (const p of dayPoints) {
    s = alpha * p.avg + (1 - alpha) * s;
    const dev = p.avg - s;
    v = alpha * dev * dev + (1 - alpha) * v;
    const band = Math.max(Math.sqrt(v), 0.6) + p.spread / 2;
    out.push({ t: p.t, trend: s, lo: s - band, hi: s + band });
  }
  return out;
}

/** The hero readout: the latest trend value, plus a time-normalized weekly
 * rate of change. `hasRate` is false (and `ratePerWeek` null) when there is
 * under ~a week of data or only one point, so the hero can hide the pill
 * instead of showing a noisy/zero rate. */
export type TrendSummary = {
  current: number | null;
  ratePerWeek: number | null;
  hasRate: boolean;
};

export function trendSummary(trend: TrendPoint[]): TrendSummary {
  if (trend.length === 0) return { current: null, ratePerWeek: null, hasRate: false };
  const last = trend[trend.length - 1];
  const current = last.trend;
  // Reference point: the nearest trend point that is at least 7 days before
  // the latest. `trend` is sorted ascending, so the last qualifying point is
  // the one closest to the 7-day mark. Time-normalize instead of stepping a
  // fixed number of indices back, so gaps / multi-per-day don't skew it.
  const cutoff = last.t - 7 * DAY_MS;
  let ref: TrendPoint | null = null;
  for (const p of trend) {
    if (p.t <= cutoff) ref = p;
  }
  if (!ref) return { current, ratePerWeek: null, hasRate: false };
  const deltaDays = (last.t - ref.t) / DAY_MS;
  if (deltaDays <= 0) return { current, ratePerWeek: null, hasRate: false };
  const ratePerWeek = ((last.trend - ref.trend) / deltaDays) * 7;
  return { current, ratePerWeek, hasRate: true };
}

/** Derived summary stats for the caption strip: count, average, dated
 * min/max, and the range delta (last day mean − first day mean). */
export type Stats = {
  count: number;
  avg: number | null;
  min: { weight: number; date: string } | null;
  max: { weight: number; date: string } | null;
  delta: number | null;
};

export function computeStats(entries: BodyweightEntry[], displayUnit: "lb" | "kg"): Stats {
  if (entries.length === 0) {
    return { count: 0, avg: null, min: null, max: null, delta: null };
  }
  const normalized = entries.map((e) => ({
    weight: convertWeight(e.weight, e.unit, displayUnit),
    measured_at: e.measured_at,
  }));
  const sum = normalized.reduce((a, b) => a + b.weight, 0);
  const avg = sum / normalized.length;
  const min = normalized.reduce((acc, w) => (w.weight < acc.weight ? w : acc), normalized[0]);
  const max = normalized.reduce((acc, w) => (w.weight > acc.weight ? w : acc), normalized[0]);

  const byDay = new Map<number, number[]>();
  for (const w of normalized) {
    const d = new Date(w.measured_at);
    const dayStart = new Date(d.getFullYear(), d.getMonth(), d.getDate()).getTime();
    const arr = byDay.get(dayStart) ?? [];
    arr.push(w.weight);
    byDay.set(dayStart, arr);
  }
  const dayStartTimes = [...byDay.keys()].sort((a, b) => a - b);
  let delta: number | null = null;
  if (dayStartTimes.length >= 2) {
    const firstAvg = mean(byDay.get(dayStartTimes[0]) ?? []);
    const lastAvg = mean(byDay.get(dayStartTimes[dayStartTimes.length - 1]) ?? []);
    delta = lastAvg - firstAvg;
  }

  return {
    count: entries.length,
    avg,
    min: { weight: min.weight, date: min.measured_at },
    max: { weight: max.weight, date: max.measured_at },
    delta,
  };
}

function mean(xs: number[]): number {
  if (xs.length === 0) return 0;
  return xs.reduce((a, b) => a + b, 0) / xs.length;
}
```

> Note: EWMA runs over the **filtered** range the caller passes (so the curve matches the selected tab). The earliest points of a short range are less "warmed up" — acceptable and consistent with the range semantics (see SOW Open Questions #3). `BodyweightEntry` (`id`, `weight`, `unit`, `measured_at`, `created_at`) is defined in `lib/api.ts`; confirm the import resolves.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm run test -- lib/bodyweight-trend.test.ts`
Expected: PASS (all describe blocks green).

- [ ] **Step 5: Commit**

```bash
git add lib/bodyweight-trend.ts lib/bodyweight-trend.test.ts
git commit -m "feat(bodyweight): add EWMA trend + stats derivation module"
```

---

## Task 2: Chart color constants

**Files:**
- Create: `lib/bodyweight-chart-colors.ts`

Background: Recharts writes `stroke`/`fill` as SVG attributes, where CSS `var(--token)` does not resolve. The design-system tokens are therefore mirrored as literal constants here — the same values as `globals.css`, not a new palette.

- [ ] **Step 1: Write the implementation** (no separate test — these are constants; Task 3's tests exercise them)

Create `lib/bodyweight-chart-colors.ts`:

```ts
/**
 * Chart SVG colors for the bodyweight trend chart.
 *
 * Recharts writes stroke/fill as SVG attributes, where CSS `var(--token)`
 * does NOT resolve — so these literals MIRROR the design-system tokens in
 * `app/globals.css` / `prog-strength-docs/design-system.md`. They are not a
 * new palette: keep them in sync with those tokens. They exist only because
 * SVG attributes can't read CSS variables.
 */
export const CHART_COLORS = {
  trend: "#8b7cf6", // --accent (violet) — the EWMA trend line, the subject
  bandTop: "rgba(139, 124, 246, 0.22)", // accent fill, gradient top
  bandBottom: "rgba(139, 124, 246, 0.04)", // accent fill, gradient bottom
  bandLegend: "rgba(139, 124, 246, 0.18)", // flat swatch for the legend
  rawDot: "#6b7280", // --faint slate — demoted raw readings
  goal: "#9aa1ad", // --muted slate — quiet dashed goal marker
  grid: "rgba(255, 255, 255, 0.06)", // hairline cartesian grid
  axis: "#9aa1ad", // --muted — axis ticks + text
  tooltipSurface: "#1e2128", // --surface — tooltip background
  tooltipBorder: "rgba(255, 255, 255, 0.1)", // --border-strong — tooltip border
} as const;
```

- [ ] **Step 2: Verify it typechecks**

Run: `npm run typecheck`
Expected: PASS (no errors).

- [ ] **Step 3: Commit**

```bash
git add lib/bodyweight-chart-colors.ts
git commit -m "feat(bodyweight): mirror design-system chart colors as SVG literals"
```

---

## Task 3: TrendSection component (hero + chart + caption + tooltip)

**Files:**
- Create: `app/(app)/bodyweight/_components/trend-section.tsx`
- Test: `app/(app)/bodyweight/_components/trend-section.test.tsx`

Background: this is the visual heart of the redesign. It replaces the old `ChartCard` (chart + four stat tiles). Top to bottom: a trend hero readout, the layered `ComposedChart` (the hero), and the caption strip. It conforms to the design system — slate panel, violet accent, the condensed `var(--font-display)` face for the big trend numeral, full-pill rate chip. The legacy palette (`#3b82f6` / `#fcd34d` / `#10b981`) appears nowhere; all SVG colors come from `CHART_COLORS`. The tooltip is a custom component that formats the hovered **weight** (never the raw millisecond timestamp) and excludes the band series — fixing the DX-flagged `Daily avg : 1780150948286 lb` bug.

- [ ] **Step 1: Write the failing tests**

Create `app/(app)/bodyweight/_components/trend-section.test.tsx`:

```tsx
/// <reference types="vitest/globals" />

import { render, screen } from "@testing-library/react";
import type { BodyweightEntry, BodyweightGoal } from "@/lib/api";
import { TrendSection, TrendTooltip } from "./trend-section";

// Recharts renders poorly in jsdom (no layout). We only assert on our own
// DOM — the hero, legend, and caption strip render outside the SVG — so we
// stub recharts to passthrough <div>s that still mount the children.
vi.mock("recharts", () => {
  const Pass =
    (name: string) =>
    ({ children }: { children?: React.ReactNode }) => <div data-recharts={name}>{children}</div>;
  const Leaf =
    (name: string) =>
    (props: Record<string, unknown>) => (
      <div data-recharts={name} data-datakey={String(props.dataKey ?? "")} />
    );
  return {
    ResponsiveContainer: Pass("ResponsiveContainer"),
    ComposedChart: Pass("ComposedChart"),
    Area: Leaf("Area"),
    Scatter: Leaf("Scatter"),
    Line: Leaf("Line"),
    ReferenceLine: Leaf("ReferenceLine"),
    CartesianGrid: Leaf("CartesianGrid"),
    XAxis: Leaf("XAxis"),
    YAxis: Leaf("YAxis"),
    Tooltip: Leaf("Tooltip"),
  };
});

let nextId = 0;
function entry(weight: number, measured_at: string, unit: "lb" | "kg" = "lb"): BodyweightEntry {
  return { id: `e${nextId++}`, weight, unit, measured_at, created_at: measured_at };
}

// A clearly trending-DOWN series spanning >1 week (so hasRate is true).
function downTrend(): BodyweightEntry[] {
  return [
    entry(200, "2026-06-01T08:00:00"),
    entry(199, "2026-06-04T08:00:00"),
    entry(197, "2026-06-08T08:00:00"),
    entry(195, "2026-06-12T08:00:00"),
    entry(193, "2026-06-15T08:00:00"),
  ];
}

function upTrend(): BodyweightEntry[] {
  return [
    entry(190, "2026-06-01T08:00:00"),
    entry(192, "2026-06-04T08:00:00"),
    entry(194, "2026-06-08T08:00:00"),
    entry(196, "2026-06-12T08:00:00"),
    entry(198, "2026-06-15T08:00:00"),
  ];
}

const GOAL: BodyweightGoal = {
  weight: 185,
  unit: "lb",
  created_at: "2026-01-01T00:00:00Z",
  updated_at: "2026-01-01T00:00:00Z",
};

describe("TrendSection", () => {
  it("shows the TREND NOW overline and a down arrow for a down-trending fixture", () => {
    render(<TrendSection entries={downTrend()} displayUnit="lb" goal={GOAL} isMobile={false} />);
    expect(screen.getByText(/trend now/i)).toBeInTheDocument();
    expect(screen.getByText(/lb\/wk/i)).toHaveTextContent("↓");
  });

  it("shows an up arrow for an up-trending fixture", () => {
    render(<TrendSection entries={upTrend()} displayUnit="lb" goal={GOAL} isMobile={false} />);
    expect(screen.getByText(/lb\/wk/i)).toHaveTextContent("↑");
  });

  it("hides the rate pill when there is under a week of data", () => {
    const sparse = [entry(200, "2026-06-14T08:00:00"), entry(199, "2026-06-15T08:00:00")];
    render(<TrendSection entries={sparse} displayUnit="lb" goal={null} isMobile={false} />);
    expect(screen.queryByText(/lb\/wk/i)).not.toBeInTheDocument();
  });

  it("renders the caption strip labels", () => {
    render(<TrendSection entries={downTrend()} displayUnit="lb" goal={GOAL} isMobile={false} />);
    expect(screen.getByText("Average")).toBeInTheDocument();
    expect(screen.getByText("Range Δ")).toBeInTheDocument();
    expect(screen.getByText("Low")).toBeInTheDocument();
    expect(screen.getByText("High")).toBeInTheDocument();
    expect(screen.getByText("To goal")).toBeInTheDocument();
  });

  it("omits the goal legend and To-goal caption when no goal is set", () => {
    render(<TrendSection entries={downTrend()} displayUnit="lb" goal={null} isMobile={false} />);
    expect(screen.queryByText("To goal")).not.toBeInTheDocument();
    expect(screen.queryByText(/^Goal /)).not.toBeInTheDocument();
  });

  it("renders the empty state without a chart", () => {
    render(<TrendSection entries={[]} displayUnit="lb" goal={null} isMobile={false} />);
    expect(screen.getByText(/no readings in this range/i)).toBeInTheDocument();
    expect(screen.queryByText(/trend now/i)).not.toBeInTheDocument();
  });

  it("renders a single reading without breaking and hides the rate", () => {
    render(
      <TrendSection
        entries={[entry(184, "2026-06-15T08:00:00")]}
        displayUnit="lb"
        goal={null}
        isMobile={false}
      />,
    );
    expect(screen.getByText(/trend now/i)).toBeInTheDocument();
    expect(screen.queryByText(/lb\/wk/i)).not.toBeInTheDocument();
  });
});

describe("TrendTooltip", () => {
  const payload = [
    { dataKey: "trend", value: 182.4 },
    { dataKey: "band", value: [180.1, 184.7] },
  ];

  it("formats the hovered value as a weight, never a raw timestamp", () => {
    render(
      <TrendTooltip active payload={payload} label={1780150948286} displayUnit="lb" />,
    );
    expect(screen.getByText(/182\.4 lb/)).toBeInTheDocument();
    // The DX-flagged bug: the raw millisecond timestamp must never appear.
    expect(screen.queryByText(/1780150948286/)).not.toBeInTheDocument();
  });

  it("contributes no row for the band series", () => {
    const { container } = render(
      <TrendTooltip active payload={payload} label={1780150948286} displayUnit="lb" />,
    );
    expect(container.textContent).not.toMatch(/180\.1/);
    expect(container.textContent).not.toMatch(/184\.7/);
  });

  it("renders nothing when inactive", () => {
    const { container } = render(
      <TrendTooltip active={false} payload={payload} label={1780150948286} displayUnit="lb" />,
    );
    expect(container).toBeEmptyDOMElement();
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm run test -- app/(app)/bodyweight/_components/trend-section.test.tsx`
Expected: FAIL — module `./trend-section` not found.

- [ ] **Step 3: Write the implementation**

Create `app/(app)/bodyweight/_components/trend-section.tsx`:

```tsx
"use client";

import { useMemo } from "react";
import {
  Area,
  CartesianGrid,
  ComposedChart,
  Line,
  ReferenceLine,
  ResponsiveContainer,
  Scatter,
  Tooltip,
  XAxis,
  YAxis,
} from "recharts";
import type { BodyweightEntry, BodyweightGoal } from "@/lib/api";
import { formatNumber } from "@/lib/format";
import {
  buildDayPoints,
  computeStats,
  convertWeight,
  ewmaTrend,
  trendSummary,
  type Stats,
  type TrendSummary,
} from "@/lib/bodyweight-trend";
import { CHART_COLORS } from "@/lib/bodyweight-chart-colors";

const EWMA_ALPHA = 0.25;

/**
 * The trend-band-analyst visualization region: a hero readout (TREND NOW
 * overline → condensed-display current value → rate pill → legend), the
 * layered Recharts ComposedChart (±spread band → demoted slate readings →
 * violet EWMA trend → quiet dashed goal), and a compact caption strip. All
 * SVG colors come from CHART_COLORS (the design-system tokens mirrored for
 * SVG); page chrome uses the CSS-variable tokens directly.
 */
export function TrendSection({
  entries,
  displayUnit,
  goal,
  isMobile,
}: {
  entries: BodyweightEntry[];
  displayUnit: "lb" | "kg";
  goal: BodyweightGoal | null;
  isMobile: boolean;
}) {
  const derived = useMemo(() => {
    const dayPoints = buildDayPoints(entries, displayUnit);
    const trend = ewmaTrend(dayPoints, EWMA_ALPHA);
    const summary = trendSummary(trend);
    const stats = computeStats(entries, displayUnit);
    const rawPoints = entries.map((e) => ({
      t: new Date(e.measured_at).getTime(),
      weight: convertWeight(e.weight, e.unit, displayUnit),
    }));
    // The Area consumes `band: [lo, hi]` as a tuple to render a range.
    const trendData = trend.map((p) => ({
      t: p.t,
      trend: p.trend,
      band: [p.lo, p.hi] as [number, number],
    }));
    return { dayCount: dayPoints.length, summary, stats, rawPoints, trendData };
  }, [entries, displayUnit]);

  // Goal projected into the display unit, so the y-domain and the goal line
  // read off the same number. Null when no goal is set.
  const goalInDisplayUnit =
    goal && goal.weight > 0 ? convertWeight(goal.weight, goal.unit, displayUnit) : null;

  if (entries.length === 0) {
    return (
      <div className="rounded-2xl border border-[var(--border)] bg-[var(--surface)] px-4 py-10 text-center text-sm text-[var(--muted)]">
        No readings in this range.
      </div>
    );
  }

  const { summary, stats, rawPoints, trendData, dayCount } = derived;

  return (
    <div className="flex flex-col gap-4 rounded-2xl border border-[var(--border)] bg-[var(--surface)] p-4 sm:p-5">
      {/* Trend hero readout */}
      <div className="flex flex-wrap items-end justify-between gap-3">
        <div className="flex flex-col gap-1">
          <span className="text-[11px] font-semibold uppercase tracking-wider text-[var(--faint)]">
            Trend now
          </span>
          <div className="flex items-end gap-2">
            <span className="font-display text-5xl font-bold leading-none tabular-nums text-[var(--foreground)]">
              {summary.current !== null ? formatNumber(summary.current) : "—"}
            </span>
            <span className="pb-1 text-sm font-semibold text-[var(--muted)]">{displayUnit}</span>
          </div>
        </div>
        {summary.hasRate && summary.ratePerWeek !== null && (
          <RatePill rate={summary.ratePerWeek} unit={displayUnit} />
        )}
      </div>

      <TrendLegend
        goalLabel={goalInDisplayUnit !== null ? formatNumber(goalInDisplayUnit) : null}
        unit={displayUnit}
      />

      {/* The chart — the hero. Layered back to front: ±band, raw readings,
          EWMA trend, goal. */}
      <div className="h-[300px] w-full sm:h-[400px]">
        <ResponsiveContainer width="100%" height="100%">
          <ComposedChart margin={{ top: 12, right: isMobile ? 8 : 16, bottom: 8, left: 0 }}>
            <defs>
              <linearGradient id="bw-band-gradient" x1="0" y1="0" x2="0" y2="1">
                <stop offset="0%" stopColor={CHART_COLORS.bandTop} />
                <stop offset="100%" stopColor={CHART_COLORS.bandBottom} />
              </linearGradient>
            </defs>
            <CartesianGrid stroke={CHART_COLORS.grid} strokeDasharray="3 3" />
            <XAxis
              dataKey="t"
              type="number"
              domain={["dataMin", "dataMax"]}
              stroke={CHART_COLORS.axis}
              tick={{ fill: CHART_COLORS.axis, fontSize: 11 }}
              minTickGap={isMobile ? 24 : 12}
              tickFormatter={(v: number) => {
                const d = new Date(v);
                if (isMobile) return `${d.getMonth() + 1}/${d.getDate()}`;
                return d.toLocaleDateString("en-US", { month: "short", day: "numeric" });
              }}
            />
            <YAxis
              stroke={CHART_COLORS.axis}
              tick={{ fill: CHART_COLORS.axis, fontSize: 11 }}
              // Force the goal into the visible domain; ±2 padding clears the
              // dashed goal line and its label off the chart edges.
              domain={[
                (dataMin: number) =>
                  Math.floor(
                    goalInDisplayUnit !== null
                      ? Math.min(dataMin, goalInDisplayUnit) - 2
                      : dataMin - 2,
                  ),
                (dataMax: number) =>
                  Math.ceil(
                    goalInDisplayUnit !== null
                      ? Math.max(dataMax, goalInDisplayUnit) + 2
                      : dataMax + 2,
                  ),
              ]}
              width={isMobile ? 32 : 48}
              tickFormatter={(v: number) => `${Math.round(v)}`}
            />
            <Tooltip
              cursor={{ stroke: CHART_COLORS.axis, strokeWidth: 1 }}
              wrapperStyle={{ outline: "none" }}
              content={<TrendTooltip displayUnit={displayUnit} />}
            />
            <Area
              data={trendData}
              dataKey="band"
              stroke="none"
              fill="url(#bw-band-gradient)"
              isAnimationActive={false}
              activeDot={false}
            />
            <Scatter
              data={rawPoints}
              dataKey="weight"
              fill={CHART_COLORS.rawDot}
              fillOpacity={0.5}
              isAnimationActive={false}
            />
            <Line
              data={trendData}
              type="monotone"
              dataKey="trend"
              stroke={CHART_COLORS.trend}
              strokeWidth={3.5}
              dot={false}
              isAnimationActive={false}
            />
            {goalInDisplayUnit !== null && (
              <ReferenceLine
                y={goalInDisplayUnit}
                stroke={CHART_COLORS.goal}
                strokeDasharray="6 4"
                strokeWidth={1.5}
                label={{
                  value: isMobile
                    ? `${formatNumber(goalInDisplayUnit)} ${displayUnit}`
                    : `Goal ${formatNumber(goalInDisplayUnit)} ${displayUnit}`,
                  position: isMobile ? "insideTopRight" : "right",
                  fill: CHART_COLORS.goal,
                  fontSize: 10,
                }}
              />
            )}
          </ComposedChart>
        </ResponsiveContainer>
      </div>

      <CaptionStrip
        stats={stats}
        goalInDisplayUnit={goalInDisplayUnit}
        displayUnit={displayUnit}
        dayCount={dayCount}
      />
    </div>
  );
}

// --- Hero bits ----------------------------------------------------

// Rate-of-change chip. The arrow AND the sign carry direction — not color
// alone — so it reads without relying on the accent fill.
function RatePill({ rate, unit }: { rate: number; unit: string }) {
  const down = rate < 0;
  const arrow = down ? "↓" : "↑";
  const sign = down ? "−" : "+";
  return (
    <span className="inline-flex items-center gap-1.5 rounded-full bg-[var(--accent-soft)] px-3 py-1 text-sm font-semibold text-[var(--accent)]">
      <span aria-hidden>{arrow}</span>
      <span className="tabular-nums">
        {sign}
        {formatNumber(Math.abs(rate))} {unit}/wk
      </span>
    </span>
  );
}

function TrendLegend({ goalLabel, unit }: { goalLabel: string | null; unit: string }) {
  return (
    <div className="flex flex-wrap items-center gap-x-4 gap-y-1.5 text-xs text-[var(--muted)]">
      <LegendItem
        swatch={
          <span
            className="inline-block h-[3px] w-5 rounded-full"
            style={{ background: CHART_COLORS.trend }}
          />
        }
        label="EWMA trend"
      />
      <LegendItem
        swatch={
          <span
            className="inline-block h-2.5 w-5 rounded-sm"
            style={{ background: CHART_COLORS.bandLegend }}
          />
        }
        label="±spread band"
      />
      <LegendItem
        swatch={
          <span
            className="inline-block h-2 w-2 rounded-full"
            style={{ backgroundColor: CHART_COLORS.rawDot, opacity: 0.6 }}
          />
        }
        label="Readings"
      />
      {goalLabel !== null && (
        <LegendItem
          swatch={
            <span
              className="inline-block w-5 border-t-2 border-dashed"
              style={{ borderColor: CHART_COLORS.goal }}
            />
          }
          label={`Goal ${goalLabel} ${unit}`}
        />
      )}
    </div>
  );
}

function LegendItem({ swatch, label }: { swatch: React.ReactNode; label: string }) {
  return (
    <span className="inline-flex items-center gap-1.5">
      <span aria-hidden>{swatch}</span>
      {label}
    </span>
  );
}

// --- Caption strip ------------------------------------------------

function CaptionStrip({
  stats,
  goalInDisplayUnit,
  displayUnit,
  dayCount,
}: {
  stats: Stats;
  goalInDisplayUnit: number | null;
  displayUnit: "lb" | "kg";
  dayCount: number;
}) {
  return (
    <div className="flex flex-wrap items-baseline gap-x-5 gap-y-2 border-t border-[var(--border)] pt-3 text-xs">
      <Caption
        label="Average"
        value={stats.avg !== null ? `${formatNumber(stats.avg)} ${displayUnit}` : "—"}
      />
      <Caption
        label="Range Δ"
        accent
        value={
          stats.delta !== null
            ? `${stats.delta >= 0 ? "+" : "−"}${formatNumber(Math.abs(stats.delta))} ${displayUnit}`
            : "—"
        }
      />
      <Caption
        label="Low"
        value={
          stats.min !== null
            ? `${formatNumber(stats.min.weight)} ${displayUnit} · ${formatShortDate(stats.min.date)}`
            : "—"
        }
      />
      <Caption
        label="High"
        value={
          stats.max !== null
            ? `${formatNumber(stats.max.weight)} ${displayUnit} · ${formatShortDate(stats.max.date)}`
            : "—"
        }
      />
      {goalInDisplayUnit !== null && stats.avg !== null && (
        <Caption
          label="To goal"
          value={`${formatNumber(Math.abs(goalInDisplayUnit - stats.avg))} ${displayUnit}`}
        />
      )}
      <span className="ml-auto self-center text-[10px] text-[var(--faint)]">
        {dayCount} day{dayCount === 1 ? "" : "s"} · trend smoothed (EWMA α0.25)
      </span>
    </div>
  );
}

function Caption({ label, value, accent }: { label: string; value: string; accent?: boolean }) {
  return (
    <span className="inline-flex flex-col">
      <span className="text-[10px] font-semibold uppercase tracking-wider text-[var(--faint)]">
        {label}
      </span>
      <span
        className={`tabular-nums font-semibold ${
          accent ? "text-[var(--accent)]" : "text-[var(--foreground)]"
        }`}
      >
        {value}
      </span>
    </span>
  );
}

// --- Tooltip ------------------------------------------------------

type TooltipPayloadItem = { dataKey?: string | number; value?: number | number[] | string };

/**
 * Custom chart tooltip. Formats the hovered WEIGHT (via formatNumber),
 * labels the day with a day-long formatter, and EXCLUDES the band series —
 * so the band's noon timestamp can never be rendered as a value. This is the
 * fix for the DX-flagged `Daily avg : 1780150948286 lb` bug. Exported so the
 * regression test can target it directly without recharts layout.
 */
export function TrendTooltip({
  active,
  payload,
  label,
  displayUnit,
}: {
  active?: boolean;
  payload?: TooltipPayloadItem[];
  label?: number | string;
  displayUnit: "lb" | "kg";
}) {
  if (!active || !payload || payload.length === 0) return null;
  const rows = payload
    .filter((p) => p.dataKey === "trend" || p.dataKey === "weight")
    .map((p) => ({
      name: p.dataKey === "trend" ? "Trend" : "Reading",
      value: typeof p.value === "number" ? p.value : Number(p.value),
    }))
    .filter((r) => Number.isFinite(r.value));
  if (rows.length === 0) return null;

  return (
    <div
      style={{
        backgroundColor: CHART_COLORS.tooltipSurface,
        border: `1px solid ${CHART_COLORS.tooltipBorder}`,
        borderRadius: "0.5rem",
        padding: "8px 10px",
        fontSize: "12px",
      }}
    >
      <p className="mb-1 text-[var(--muted)]">{formatTooltipDay(label)}</p>
      {rows.map((r) => (
        <p key={r.name} className="tabular-nums text-[var(--foreground)]">
          <span className="text-[var(--muted)]">{r.name}: </span>
          {formatNumber(r.value)} {displayUnit}
        </p>
      ))}
    </div>
  );
}

function formatTooltipDay(label: number | string | undefined): string {
  const v = typeof label === "number" ? label : Number(label);
  if (!Number.isFinite(v)) return "";
  return new Date(v).toLocaleDateString("en-US", {
    weekday: "short",
    month: "short",
    day: "numeric",
    year: "numeric",
  });
}

function formatShortDate(iso: string): string {
  return new Date(iso).toLocaleDateString("en-US", { month: "short", day: "numeric" });
}
```

> Notes for the implementer:
> - `BodyweightGoal` (`weight`, `unit`, `created_at`) and `BodyweightEntry` come from `lib/api.ts` — confirm the import path/shape before writing. If `BodyweightGoal.created_at` is nullable in the type, the `goal` prop handling here only reads `goal.weight`/`goal.unit`, which is fine.
> - Recharts `Area` with a two-element-array `dataKey` value renders a band between the two values — this is how the ±spread band is drawn. Keep `stroke="none"`.
> - Do NOT add the band series to the tooltip. The custom `TrendTooltip` filters to `trend`/`weight` dataKeys, which is the guard the regression test checks.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm run test -- app/(app)/bodyweight/_components/trend-section.test.tsx`
Expected: PASS (TrendSection + TrendTooltip blocks green).

- [ ] **Step 5: Typecheck + lint the new file**

Run: `npm run typecheck && npm run lint`
Expected: PASS (no errors, no new warnings).

- [ ] **Step 6: Commit**

```bash
git add "app/(app)/bodyweight/_components/trend-section.tsx" "app/(app)/bodyweight/_components/trend-section.test.tsx"
git commit -m "feat(bodyweight): add trend-band-analyst chart section"
```

---

## Task 4: Wire TrendSection into the page and retire the legacy palette

**Files:**
- Modify: `app/(app)/bodyweight/page.tsx`

Background: the page keeps all its data plumbing, modals, the log table, pagination, the goal affordance, and the mobile action sheet — unchanged. This task swaps the old `ChartCard` for `<TrendSection>`, restyles `TimeRangeTabs` into quiet pills with a `"{n} readings · ending {date}"` caption, retones the goal `TargetIcon` off the legacy emerald, and deletes the now-dead chart code. After this task NO `#3b82f6` / `#fcd34d` / `#10b981` (and no `#f59e0b` / `#94a3b8` from the old stat tiles) remain in the file, and `recharts` is no longer imported by `page.tsx`.

- [ ] **Step 1: Replace the recharts import block and add the TrendSection import**

In `app/(app)/bodyweight/page.tsx`, DELETE the entire recharts import (lines importing `CartesianGrid, ComposedChart, Line, ReferenceLine, ResponsiveContainer, Scatter, Tooltip, XAxis, YAxis` from `"recharts"`). Then add, next to the existing `BodyweightActionSheet` import:

```tsx
import { TrendSection } from "./_components/trend-section";
```

Leave the `clearToken/getToken`, `lib/api`, and `BodyweightActionSheet` imports as they are.

- [ ] **Step 2: Delete the two legacy color constants**

Delete these lines (the `COLOR_AVG` / `COLOR_RAW` block and their leading comment) near the top of the file:

```tsx
// Recharts series colors. Picked for clear hue contrast ...
const COLOR_AVG = "#3b82f6"; // blue-500 — trend line
const COLOR_RAW = "#fcd34d"; // amber-300 — raw readings scatter
```

- [ ] **Step 3: Compute the range caption inputs and swap ChartCard → TrendSection**

In the `BodyweightPage` component body, the render currently has:

```tsx
          <TimeRangeTabs value={range} onChange={setRange} />

          <ChartCard
            entries={entriesInRange}
            displayUnit={displayUnit}
            goal={hasGoal ? goal : null}
            isMobile={isMobile}
          />
```

Replace that block with:

```tsx
          <TimeRangeTabs
            value={range}
            onChange={setRange}
            count={entriesInRange.length}
            endingLabel={
              entriesInRange.length > 0
                ? new Date(entriesInRange[0].measured_at).toLocaleDateString("en-US", {
                    month: "short",
                    day: "numeric",
                  })
                : null
            }
          />

          <TrendSection
            entries={entriesInRange}
            displayUnit={displayUnit}
            goal={hasGoal ? goal : null}
            isMobile={isMobile}
          />
```

(`entriesInRange` is already sorted descending by `measured_at`, so `[0]` is the latest reading — the range's ending date.)

- [ ] **Step 4: Restyle TimeRangeTabs into quiet pills with the caption**

Replace the entire `TimeRangeTabs` function with:

```tsx
function TimeRangeTabs({
  value,
  onChange,
  count,
  endingLabel,
}: {
  value: RangeKey;
  onChange: (v: RangeKey) => void;
  count: number;
  endingLabel: string | null;
}) {
  return (
    <div className="flex flex-wrap items-center justify-between gap-2 border-b border-[var(--border)] pb-3">
      <div className="flex flex-wrap items-center gap-1.5">
        {RANGES.map((r) => {
          const selected = r.key === value;
          // Quiet pills — the chart is the headline. Active = accent-soft
          // fill + accent-line border + accent text, per the design system.
          const stateClasses = selected
            ? "bg-[var(--accent-soft)] border border-[var(--accent-line)] text-[var(--accent)]"
            : "border border-transparent text-[var(--muted)] hover:bg-[var(--surface-2)] hover:text-[var(--foreground)]";
          return (
            <button
              key={r.key}
              type="button"
              onClick={() => onChange(r.key)}
              className={`rounded-full px-3 py-1 text-xs font-semibold transition ${stateClasses}`}
            >
              {r.label}
            </button>
          );
        })}
      </div>
      {count > 0 && endingLabel !== null && (
        <span className="text-xs text-[var(--muted)]">
          {count} reading{count === 1 ? "" : "s"} · ending {endingLabel}
        </span>
      )}
    </div>
  );
}
```

- [ ] **Step 5: Delete the dead chart code**

Delete the following now-unused declarations from `page.tsx` (all were only used by the old chart region):
- the entire `ChartCard` function,
- the `Tone` type and the entire `AccentStatTile` function,
- the entire `Legend` function (the chart legend — NOT the table, NOT the modals),
- in `helpers`: the `convertWeight` function, the `Stats` type, the `computeStats` function, the `mean` function, and the `formatShortDate` function.

Verify each is unreferenced before deleting (e.g. `formatShortDate` was only used inside `AccentStatTile`; `convertWeight`/`computeStats`/`mean`/`Stats` only inside `ChartCard`/`AccentStatTile`).

- [ ] **Step 6: Retone the goal TargetIcon off the legacy emerald**

In the `TargetIcon` function, change the SVG className from:

```tsx
      className="h-4 w-4 text-emerald-500"
```

to:

```tsx
      className="h-4 w-4 text-[var(--muted)]"
```

(The goal is now a quiet neutral marker, conforming to the design system — this removes the last `#10b981`-family color from the page.)

- [ ] **Step 7: Verify no legacy palette remains and the build is clean**

Run:
```bash
grep -nE "#3b82f6|#fcd34d|#10b981|#f59e0b|#94a3b8|emerald-500|from \"recharts\"" "app/(app)/bodyweight/page.tsx"
```
Expected: no matches (empty output).

Then run the full gate:
```bash
npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build
```
Expected: all PASS. (If `format:check` flags the edited files, run `npm run format` and re-stage.)

- [ ] **Step 8: Commit**

```bash
git add "app/(app)/bodyweight/page.tsx"
git commit -m "feat(bodyweight): rebuild chart region into trend-band-analyst"
```

---

## Task 5: Full gate + responsive sanity

**Files:** none (verification only)

- [ ] **Step 1: Run the complete CI gate locally (the exact CI order)**

```bash
npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build
```
Expected: all five PASS. Fix any failure in the code (never `--no-verify`, never `//nolint`/rule-disable, never skip a test).

- [ ] **Step 2: Confirm the bodyweight test files pass in isolation**

```bash
npm run test -- bodyweight
```
Expected: `lib/bodyweight-trend.test.ts` and `app/(app)/bodyweight/_components/trend-section.test.tsx` green.

- [ ] **Step 3: Responsive reflow check (manual reasoning / preview)**

The hero (`flex-wrap`), legend (`flex-wrap`), and caption strip (`flex-wrap` + `ml-auto` footnote) all wrap; the chart container is `h-[300px]` mobile / `sm:h-[400px]` desktop. Confirm by reading the markup that no row uses a fixed width that would overflow a 320px viewport. The Vercel preview deploy on the PR is the real-browser confirmation.

---

## Self-Review (run after writing the plan)

**1. Spec coverage** — checked against the SOW:
- Rebuild chart-and-stats region into trend-band-analyst → Tasks 3 + 4.
- `ewmaTrend` (α 0.25, seed, EW-variance band floored 0.6 + spread/2) → Task 1.
- Weekly rate over a true ~7-day window, hidden under a week → Task 1 (`trendSummary`).
- Trend beats noise (violet line, demoted slate dots, faint band) → Task 3.
- Reuse data layer unchanged (`listBodyweight`/`getBodyweightGoal`, daily grouping, units, range filter) → Task 4 keeps all plumbing; `buildDayPoints` mirrors the existing grouping.
- Preserve CRUD + table + pagination + modals + action sheet → Task 4 touches only the chart region + tabs.
- Retire legacy palette (`#3b82f6`/`#fcd34d`/`#10b981`), mirror tokens as literals → Tasks 2 + 4 (Step 7 grep guard).
- Fix tooltip timestamp bug (weight-formatted, band excluded) → Task 3 (`TrendTooltip` + regression test).
- Conform to design system (slate, violet, Nunito + `var(--font-display)`, rounded cards, full-pill chips) → Tasks 3 + 4.
- Tests: `ewmaTrend`/rate unit tests + tooltip regression + component states → Tasks 1 + 3.
- CI green → Tasks 4 + 5.

**2. Placeholder scan** — every code step contains complete code; no TBD/TODO.

**3. Type consistency** — `DayPoint`/`TrendPoint`/`TrendSummary`/`Stats` defined in Task 1 are used with the same shapes in Task 3; `ewmaTrend`/`trendSummary`/`computeStats`/`buildDayPoints`/`convertWeight` signatures match across tasks; `CHART_COLORS` keys defined in Task 2 are exactly those consumed in Task 3.
