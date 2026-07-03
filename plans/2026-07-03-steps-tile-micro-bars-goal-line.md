# Dashboard Steps Tile — Micro-Bars + Goal Line Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the min/max-normalized sparkline in the dashboard **Steps** mini-card with a goal-relative seven-bar micro-chart (dashed goal line + fainter average line, goal-state coloring, today accented), production-quality against the real `StepsView` data.

**Architecture:** A pure derivation helper (`buildStepsGoalBars`) turns `{ spark, avg, goal }` into a fixed seven-slot render model (bar heights as % of a linearly-scaled ceiling, goal/avg line positions, per-bar tone). A thin presentational component (`StepsGoalBars`) renders that model with divs (bars + absolutely-positioned reference lines), reading v0.4 CSS vars (`--success`, `--muted`, `--accent`, `--faint`, `--border`). `StepsCard` in `page.tsx` swaps `<Spark …>` 1:1 for `<StepsGoalBars …>`. No backend/API/token change.

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript, Tailwind v4 (CSS-var theme tokens), Vitest + Testing Library (jsdom).

---

## File Structure

- **Create** `app/(app)/dashboard/_components/steps-goal-bars.tsx` — the `StepsGoalBars` component **and** the co-located pure helper `buildStepsGoalBars` + its exported types. One responsibility: turn the three steps values into a small goal-relative bar chart. Pure logic and rendering live in one file (small, mirrors how `spark.tsx` co-locates its math with its render) so the derivation is unit-testable via a named export.
- **Create** `app/(app)/dashboard/_components/steps-goal-bars.test.tsx` — unit tests for `buildStepsGoalBars` (scaling, goal-state, degraded states) + render assertions for `StepsGoalBars`.
- **Modify** `app/(app)/dashboard/page.tsx` — in `StepsCard` (currently `page.tsx:299-312`), replace the `<Spark …>` line with `<StepsGoalBars …>` and add the import. Nothing else in the file changes.
- **Modify** `app/(app)/dashboard/page.test.tsx` — extend the Steps assertions (bars present-state, "set a goal" no-goal meta). The existing empty-state test already covers `present: false`.

### Design decisions locked in (from the SOW + Open Questions leans)

- **Hand-rolled divs**, not recharts (Open Q2 lean) — matches the sibling tiles' lightweight primitives, no tooltips/axis needed.
- **Headroom factor `1.1`** above `max(goal, maxDay)` (Open Q4 lean) so over-goal bars visibly crest the goal line and the tallest bar isn't pinned to the frame top.
- **Goal line dominant, average quiet** (Open Q3 lean): goal line is a **dashed** rule in `--success`, drawn **in front of** the bars; average is a **thin solid** rule in a quiet ink (`--muted`), drawn **behind** the bars. If they coincide, the goal line wins the pixel (it's on top).
- **`hasGoal = goal !== null && goal > 0`** — null-safe, mirrors `steps-stats.ts`'s `goal > 0` semantics. No goal line and no over/under split unless `hasGoal`. (The tile's `MetaRow` keeps its existing `goal !== null ? compact(goal) : "set a goal"` — unchanged.)
- **Seven fixed slots, right-aligned**: `spark` (oldest→newest) fills from the right so the last slot is always **today** (the accent bar); a short history left-pads with **empty** slots (a faint track, no colored bar) — distinct from a logged **0**, which is a real (muted) **floor** bar.
- **Today always wins the accent** — today's bar is `--accent` whether it's over or under goal (the over/under split colors the other six).
- **Chart band height** ≈ 48px (`h-12`), a calm step up from the old 28px sparkline that still fits the ~180px tile budget and reads beside the other tiles.

---

## Task 1: Pure derivation helper `buildStepsGoalBars`

**Files:**
- Create: `app/(app)/dashboard/_components/steps-goal-bars.tsx` (helper + types only in this task; the component shell is added in Task 2)
- Test: `app/(app)/dashboard/_components/steps-goal-bars.test.tsx`

- [ ] **Step 1: Write the failing tests**

```tsx
/// <reference types="vitest/globals" />

import { buildStepsGoalBars } from "./steps-goal-bars";

describe("buildStepsGoalBars — scaling", () => {
  it("scales bars linearly against max(goal, maxDay) × headroom, not min/max-normalized", () => {
    // A 16k day must NOT flatten a 6k day: their heights keep the ~16:6 ratio.
    const m = buildStepsGoalBars([6000, 16000], 11000, 10000);
    const big = m.slots[6]; // today (last), 16000
    const small = m.slots[5]; // 6000
    expect(big.heightPct).toBeGreaterThan(small.heightPct);
    // ceiling = 16000 * 1.1 = 17600 → 16000/17600 ≈ 90.9, 6000/17600 ≈ 34.1
    expect(big.heightPct).toBeCloseTo((16000 / 17600) * 100, 1);
    expect(small.heightPct).toBeCloseTo((6000 / 17600) * 100, 1);
    // Not equal, not both pinned high — the shape of the week survives.
    expect(big.heightPct - small.heightPct).toBeGreaterThan(40);
  });

  it("gives over-goal bars headroom to crest the goal line", () => {
    const m = buildStepsGoalBars([12000], 12000, 10000);
    // goal line below the over-goal bar
    expect(m.goalPct).not.toBeNull();
    expect(m.slots[6].heightPct).toBeGreaterThan(m.goalPct as number);
  });
});

describe("buildStepsGoalBars — seven-slot axis", () => {
  it("always returns seven slots, right-aligned with today last", () => {
    const m = buildStepsGoalBars([9000], 9000, 10000);
    expect(m.slots).toHaveLength(7);
    // one real day → six empty left slots, today (real) last
    expect(m.slots.slice(0, 6).every((s) => s.steps === null)).toBe(true);
    expect(m.slots[6].steps).toBe(9000);
    expect(m.slots[6].isToday).toBe(true);
    expect(m.slots[0].tone).toBe("empty");
  });

  it("takes the last seven when given more than seven days", () => {
    const m = buildStepsGoalBars([1, 2, 3, 4, 5, 6, 7, 8], 5, 10);
    expect(m.slots).toHaveLength(7);
    expect(m.slots[0].steps).toBe(2);
    expect(m.slots[6].steps).toBe(8);
  });
});

describe("buildStepsGoalBars — goal state coloring", () => {
  it("colors over-goal success, under-goal muted, today accent", () => {
    //            under   under  over    today(over)
    const m = buildStepsGoalBars([8000, 9000, 12000, 11500], 10125, 10000);
    expect(m.hasGoal).toBe(true);
    expect(m.goalPct).not.toBeNull();
    expect(m.slots[3].tone).toBe("muted"); // 8000 under
    expect(m.slots[4].tone).toBe("muted"); // 9000 under
    expect(m.slots[5].tone).toBe("success"); // 12000 over
    expect(m.slots[6].tone).toBe("accent"); // today
  });

  it("keeps today the accent even when today is under goal", () => {
    const m = buildStepsGoalBars([12000, 4000], 8000, 10000);
    expect(m.slots[6].tone).toBe("accent"); // today under goal, still accent
    expect(m.slots[5].tone).toBe("success"); // 12000 over
  });
});

describe("buildStepsGoalBars — no goal", () => {
  it("draws no goal line and falls back to neutral + accent-today", () => {
    const m = buildStepsGoalBars([8000, 9000, 12000, 11500], 10125, null);
    expect(m.hasGoal).toBe(false);
    expect(m.goalPct).toBeNull();
    // no over/under split without a goal — the six non-today real bars are muted
    expect(m.slots[3].tone).toBe("muted");
    expect(m.slots[5].tone).toBe("muted"); // 12000, but no goal to clear
    expect(m.slots[6].tone).toBe("accent"); // today still accent
    // bars scale to the max day (12000), no NaN
    expect(m.slots.every((s) => Number.isFinite(s.heightPct))).toBe(true);
  });
});

describe("buildStepsGoalBars — degraded states", () => {
  it("renders an all-zero week as a flat floor with no NaN and no divide-by-zero", () => {
    const m = buildStepsGoalBars([0, 0, 0, 0, 0, 0, 0], 0, 10000);
    expect(m.slots.every((s) => s.heightPct === 0)).toBe(true);
    expect(m.slots.every((s) => Number.isFinite(s.heightPct))).toBe(true);
    // goal set → goal line still drawn (you're under all week), no NaN
    expect(m.goalPct).not.toBeNull();
    expect(Number.isFinite(m.goalPct as number)).toBe(true);
  });

  it("renders a logged 0 as an honest floor bar (0 height), not a mid-height artifact", () => {
    const m = buildStepsGoalBars([16000, 0], 8000, 10000);
    expect(m.slots[5].steps).toBe(0);
    expect(m.slots[5].heightPct).toBe(0); // floor, not normalized to mid-height
    expect(m.slots[5].tone).toBe("muted"); // logged, under goal
    expect(m.slots[6].heightPct).toBeGreaterThan(0); // 16000 tall
  });

  it("handles an empty spark without NaN or a broken frame", () => {
    const m = buildStepsGoalBars([], 0, 10000);
    expect(m.slots).toHaveLength(7);
    expect(m.slots.every((s) => s.tone === "empty" && s.heightPct === 0)).toBe(true);
  });

  it("omits the goal line when goal is null or non-positive", () => {
    expect(buildStepsGoalBars([9000], 9000, null).goalPct).toBeNull();
    expect(buildStepsGoalBars([9000], 9000, 0).goalPct).toBeNull();
  });

  it("positions the average line and omits it only when unresolvable", () => {
    const m = buildStepsGoalBars([8000, 12000], 10000, 10000);
    expect(m.avgPct).not.toBeNull();
    // avg 10000 vs ceiling max(10000,12000)*1.1 = 13200 → ~75.8
    expect(m.avgPct as number).toBeCloseTo((10000 / 13200) * 100, 1);
    // all-zero, no goal → nothing to scale against, no average line
    expect(buildStepsGoalBars([0, 0], 0, null).avgPct).toBeNull();
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd /workspace/prog-strength-web && npx vitest run app/\(app\)/dashboard/_components/steps-goal-bars.test.tsx`
Expected: FAIL — `buildStepsGoalBars` is not exported / file has no such export.

- [ ] **Step 3: Write the helper (and the types) in `steps-goal-bars.tsx`**

Put this at the top of the new file (the component in Task 2 goes below it):

```tsx
/**
 * StepsGoalBars — the dashboard Steps tile's goal-relative bar chart.
 *
 * Replaces the min/max-normalized `Spark` for steps: seven slim daily bars
 * drawn on ONE linear scale (ceiling = max(goal, maxDay) × headroom), a
 * dominant dashed goal line and a fainter average line at their scaled
 * heights, and color used as goal-state (over-goal success, under muted,
 * today the accent) rather than decoration. In-system to design-system v0.4
 * — reads `--success` / `--muted` / `--accent` / `--faint`, never raw hex.
 *
 * `buildStepsGoalBars` is the pure render model; the component only draws it.
 */

const SLOTS = 7;
const HEADROOM = 1.1; // lift the ceiling above the tallest reference so an
// over-goal day crests the goal line and the top bar isn't pinned to the frame.

/** One of the seven day slots. `empty` = no data for that slot (a padded,
 * pre-history day) — a quiet track, distinct from a logged `0` floor bar. */
export type StepsBarTone = "accent" | "success" | "muted" | "empty";

export type StepsBarSlot = {
  steps: number | null; // null = empty (no data); a number (incl. 0) = logged
  heightPct: number; // 0–100, share of the chart band; 0 for a logged-0 floor
  tone: StepsBarTone;
  isToday: boolean;
};

export type StepsGoalBarsModel = {
  slots: StepsBarSlot[]; // always length SLOTS (7), oldest → newest
  goalPct: number | null; // goal line height, null when no goal drawn
  avgPct: number | null; // average line height, null when unresolvable
  hasGoal: boolean;
};

/**
 * Derive the seven-slot render model from the three values the tile already
 * holds. Pure and null-safe: no goal → no goal line and no over/under split;
 * all-zero / empty → a flat floor with no divide-by-zero or NaN.
 */
export function buildStepsGoalBars(
  spark: number[],
  avg: number,
  goal: number | null,
): StepsGoalBarsModel {
  // `spark` is already non-finite-sanitized upstream (adaptSteps); guard anyway.
  const clean = (spark ?? []).map((n) => (Number.isFinite(n) ? n : 0));
  // Right-align into seven slots so the newest day is always the last slot
  // (today, the accent bar); a short history left-pads with empty slots.
  const recent = clean.slice(-SLOTS);
  const pad = SLOTS - recent.length;
  const values: (number | null)[] = [...Array(pad).fill(null), ...recent];

  const hasGoal = goal !== null && goal > 0;
  const maxDay = recent.length ? Math.max(...recent) : 0;
  const base = Math.max(hasGoal ? (goal as number) : 0, maxDay, 0);
  const ceiling = base > 0 ? base * HEADROOM : 0;
  const scale = (v: number) =>
    ceiling > 0 ? Math.min(100, Math.max(0, (v / ceiling) * 100)) : 0;

  const slots: StepsBarSlot[] = values.map((steps, i) => {
    const isToday = i === SLOTS - 1 && steps !== null;
    let tone: StepsBarTone;
    if (steps === null) {
      tone = "empty";
    } else if (isToday) {
      tone = "accent";
    } else if (hasGoal && steps >= (goal as number)) {
      tone = "success";
    } else {
      tone = "muted";
    }
    return { steps, heightPct: steps === null ? 0 : scale(steps), tone, isToday };
  });

  return {
    slots,
    goalPct: hasGoal ? scale(goal as number) : null,
    avgPct: Number.isFinite(avg) && ceiling > 0 ? scale(avg) : null,
    hasGoal,
  };
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd /workspace/prog-strength-web && npx vitest run app/\(app\)/dashboard/_components/steps-goal-bars.test.tsx`
Expected: PASS — all `buildStepsGoalBars` describe blocks green. (The render tests added in Task 2 are not in the file yet.)

- [ ] **Step 5: Commit**

```bash
cd /workspace/prog-strength-web
git add "app/(app)/dashboard/_components/steps-goal-bars.tsx" "app/(app)/dashboard/_components/steps-goal-bars.test.tsx"
git commit -m "feat(dashboard): add buildStepsGoalBars derivation for the steps tile"
```

---

## Task 2: `StepsGoalBars` presentational component

**Files:**
- Modify: `app/(app)/dashboard/_components/steps-goal-bars.tsx` (add the component below the helper)
- Test: `app/(app)/dashboard/_components/steps-goal-bars.test.tsx` (add render assertions)

- [ ] **Step 1: Write the failing render tests** (append to the existing test file)

```tsx
import { render, screen } from "@testing-library/react";
import { StepsGoalBars } from "./steps-goal-bars";

describe("StepsGoalBars — render", () => {
  it("renders seven bars, a goal line, and an average line when a goal is set", () => {
    render(<StepsGoalBars spark={[8000, 9000, 12000, 11500]} avg={10125} goal={10000} />);
    expect(screen.getAllByTestId("steps-bar")).toHaveLength(7);
    expect(screen.getByTestId("steps-goal-line")).toBeInTheDocument();
    expect(screen.getByTestId("steps-avg-line")).toBeInTheDocument();
  });

  it("tags each bar with its tone (today accent, over success, under muted)", () => {
    render(<StepsGoalBars spark={[8000, 9000, 12000, 11500]} avg={10125} goal={10000} />);
    const bars = screen.getAllByTestId("steps-bar");
    expect(bars[3]).toHaveAttribute("data-tone", "muted");
    expect(bars[5]).toHaveAttribute("data-tone", "success");
    expect(bars[6]).toHaveAttribute("data-tone", "accent");
  });

  it("draws no goal line and no over/under split when goal is null", () => {
    render(<StepsGoalBars spark={[8000, 9000, 12000, 11500]} avg={10125} goal={null} />);
    expect(screen.queryByTestId("steps-goal-line")).not.toBeInTheDocument();
    const bars = screen.getAllByTestId("steps-bar");
    expect(bars[5]).toHaveAttribute("data-tone", "muted"); // no goal to clear
    expect(bars[6]).toHaveAttribute("data-tone", "accent"); // today still accent
  });

  it("renders degraded fixtures without NaN in the DOM", () => {
    const { container: allZero } = render(
      <StepsGoalBars spark={[0, 0, 0, 0, 0, 0, 0]} avg={0} goal={10000} />,
    );
    expect(allZero.textContent).not.toMatch(/NaN/);
    const { container: single } = render(<StepsGoalBars spark={[9000]} avg={9000} goal={10000} />);
    expect(single.querySelectorAll('[data-testid="steps-bar"]')).toHaveLength(7);
    expect(single.textContent).not.toMatch(/NaN/);
  });
});
```

- [ ] **Step 2: Run to verify the new render tests fail**

Run: `cd /workspace/prog-strength-web && npx vitest run app/\(app\)/dashboard/_components/steps-goal-bars.test.tsx`
Expected: FAIL — `StepsGoalBars` is not exported yet.

- [ ] **Step 3: Add the component** (below the helper in `steps-goal-bars.tsx`)

```tsx
/** Per-tone bar fill. `empty` slots render only a faint track (no fill). */
const BAR_COLOR: Record<Exclude<StepsBarTone, "empty">, string> = {
  accent: "var(--accent)",
  success: "var(--success)",
  muted: "var(--muted)",
};

/**
 * The steps tile's goal-relative bar week. Swaps 1:1 for `<Spark>` inside
 * `StepsCard`, keeping the tile's rhythm and ~180px budget. Presentational
 * and pure — all layout math is in `buildStepsGoalBars`.
 */
export function StepsGoalBars({
  spark,
  avg,
  goal,
}: {
  spark: number[];
  avg: number;
  goal: number | null;
}) {
  const model = buildStepsGoalBars(spark, avg, goal);

  return (
    <div
      className="relative h-12 w-full"
      role="img"
      aria-label="Daily steps versus goal, last seven days"
    >
      {/* Average line — quiet, thin, behind the bars. */}
      {model.avgPct !== null && (
        <div
          data-testid="steps-avg-line"
          aria-hidden="true"
          className="pointer-events-none absolute inset-x-0 z-0 border-t border-[var(--muted)]/60"
          style={{ bottom: `${model.avgPct}%` }}
        />
      )}

      {/* Bars — a seven-column floor-anchored row. */}
      <div className="absolute inset-0 z-10 flex items-end gap-[3px]">
        {model.slots.map((slot, i) => (
          <div key={i} className="flex h-full flex-1 items-end">
            {slot.tone === "empty" ? (
              // Quiet empty track — a pre-history slot, not a logged 0.
              <div
                data-testid="steps-bar"
                data-tone="empty"
                aria-hidden="true"
                className="w-full rounded-sm bg-[var(--border)]"
                style={{ height: "2px" }}
              />
            ) : (
              <div
                data-testid="steps-bar"
                data-tone={slot.tone}
                aria-hidden="true"
                className="w-full rounded-sm"
                style={{
                  height: `${slot.heightPct}%`,
                  minHeight: "2px", // a logged day always shows a floor stub
                  backgroundColor: BAR_COLOR[slot.tone],
                }}
              />
            )}
          </div>
        ))}
      </div>

      {/* Goal line — dominant, dashed, in front of the bars. */}
      {model.goalPct !== null && (
        <div
          data-testid="steps-goal-line"
          aria-hidden="true"
          className="pointer-events-none absolute inset-x-0 z-20 border-t border-dashed border-[var(--success)]"
          style={{ bottom: `${model.goalPct}%` }}
        />
      )}
    </div>
  );
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd /workspace/prog-strength-web && npx vitest run app/\(app\)/dashboard/_components/steps-goal-bars.test.tsx`
Expected: PASS — helper tests and render tests all green.

- [ ] **Step 5: Commit**

```bash
cd /workspace/prog-strength-web
git add "app/(app)/dashboard/_components/steps-goal-bars.tsx" "app/(app)/dashboard/_components/steps-goal-bars.test.tsx"
git commit -m "feat(dashboard): render StepsGoalBars bars, goal line, and average line"
```

---

## Task 3: Rewire `StepsCard` and extend the dashboard test

**Files:**
- Modify: `app/(app)/dashboard/page.tsx` (import + the one `<Spark>` line inside `StepsCard`, ~`page.tsx:299-312`)
- Test: `app/(app)/dashboard/page.test.tsx`

- [ ] **Step 1: Add the failing dashboard assertions**

In `page.test.tsx`, inside `describe("DashboardPage — full payload", …)`, add a test (the `FULL_SUMMARY` steps fixture is `{ avg: 9200, today: 11500, goal: 10000, daily_spark: [8000, 9000, 12000, 11500] }`):

```tsx
it("renders the steps tile as goal-relative bars with the today headline and avg · goal meta", async () => {
  render(<DashboardPage />);
  const stepsCard = await screen.findByRole("link", { name: /Steps/i });
  // Seven goal-relative bars replace the sparkline.
  expect(within(stepsCard).getAllByTestId("steps-bar")).toHaveLength(7);
  // A goal is set → the dashed goal line is drawn.
  expect(within(stepsCard).getByTestId("steps-goal-line")).toBeInTheDocument();
  // Headline "today" figure and avg · goal meta still lead.
  expect(within(stepsCard).getByText("11.5k")).toBeInTheDocument(); // compact(11500)
  expect(within(stepsCard).getByText("9,200")).toBeInTheDocument(); // compact(9200) avg
  expect(within(stepsCard).getByText("10k")).toBeInTheDocument(); // compact(10000) goal
});
```

Add a no-goal test in the same describe block (clone the fixture, null the goal):

```tsx
it("shows 'set a goal' and no goal line when the steps goal is null", async () => {
  summaryToReturn = {
    ...FULL_SUMMARY,
    steps: { ...FULL_SUMMARY.steps!, goal: null },
  };
  render(<DashboardPage />);
  const stepsCard = await screen.findByRole("link", { name: /Steps/i });
  expect(within(stepsCard).getByText("set a goal")).toBeInTheDocument();
  expect(within(stepsCard).queryByTestId("steps-goal-line")).not.toBeInTheDocument();
  expect(within(stepsCard).getAllByTestId("steps-bar")).toHaveLength(7);
});
```

Ensure `within` is imported at the top of the test file:

```tsx
import { render, screen, fireEvent, waitFor, within } from "@testing-library/react";
```

> Note: `getByText("10k")` — if any other card also renders "10k", scope with `within(stepsCard)` (already done above) so it stays unambiguous. The existing empty-state test already asserts the `MiniCardEmpty` CTA for `present: false`, so no new empty-state test is needed here.

- [ ] **Step 2: Run to verify the new assertions fail**

Run: `cd /workspace/prog-strength-web && npx vitest run app/\(app\)/dashboard/page.test.tsx`
Expected: FAIL — no `steps-bar` testids yet (the card still renders `<Spark>`).

- [ ] **Step 3: Rewire `StepsCard`**

Add the import near the other `_components` imports in `page.tsx` (by `import { Spark } from "./_components/spark";`):

```tsx
import { StepsGoalBars } from "./_components/steps-goal-bars";
```

In `StepsCard`, replace this line:

```tsx
      <Spark points={v.spark} className="h-7 w-full text-[var(--accent-2)]" />
```

with:

```tsx
      <StepsGoalBars spark={v.spark} avg={v.avg} goal={v.goal} />
```

Leave the `MiniCard`, the `BigNum` today headline, the `MetaRow` (`avg · goal`, `goal → "set a goal"` when null), and the `present: false` `MiniCardEmpty` branch exactly as they are. Do not touch the other five cards. `Spark` stays imported (Running/Lifting/Bodyweight still use it).

- [ ] **Step 4: Run the dashboard tests to verify they pass**

Run: `cd /workspace/prog-strength-web && npx vitest run app/\(app\)/dashboard/page.test.tsx`
Expected: PASS — steps present-state, no-goal, and the untouched empty-state all green.

- [ ] **Step 5: Commit**

```bash
cd /workspace/prog-strength-web
git add "app/(app)/dashboard/page.tsx" "app/(app)/dashboard/page.test.tsx"
git commit -m "feat(dashboard): swap the steps sparkline for goal-relative micro-bars"
```

---

## Task 4: Full local CI gate

**Files:** none (verification only)

- [ ] **Step 1: Run the full gate CI runs**

Run: `cd /workspace/prog-strength-web && npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build`
Expected: all five green. `npm run test` runs the whole suite (the new files plus the existing dashboard tests), matching CI's `checks` job order (lint → format:check → typecheck → test → build).

- [ ] **Step 2: Fix any failure at its source**

If `format:check` flags the new files, run `npx prettier --write` on them and re-commit. If lint/typecheck/build fail, fix the code (never `--no-verify`, never add `//eslint-disable`, never weaken a test). Re-run the gate until green.

- [ ] **Step 3: Commit any fixups**

```bash
cd /workspace/prog-strength-web
git add -A
git commit -m "chore(dashboard): satisfy lint/format for steps goal bars"  # only if there were fixups
```

---

## Self-Review (run against the SOW before handing off)

- **Bars linearly scaled, 16k doesn't flatten 6k** → Task 1 test "scales bars linearly…". ✅
- **Goal line present when goal set, absent when null** → Task 1 "omits the goal line…" + Task 2/3 render tests. ✅
- **Average line drawn** → `avgPct` + `steps-avg-line`. ✅
- **Over-goal success / under muted / today accent; no-goal → neutral + accent-today** → Task 1 + Task 2 tone tests. ✅
- **Logged 0 = floor bar (0 height, muted), not mid-height** → Task 1 "logged 0…". ✅
- **All-zero week = flat floor, no NaN / divide-by-zero** → Task 1 "all-zero week…". ✅
- **Single-day / <7 days holds the seven-slot axis** → Task 1 "seven-slot axis" + Task 2 degraded render. ✅
- **Tokens only, no raw hex / new hue** → `--accent` / `--success` / `--muted` / `--faint` / `--border` in the component; no hex. ✅
- **`StepsCard` keeps BigNum + `avg · goal` (+ "set a goal") + MiniCardEmpty; other five tiles untouched** → Task 3 (single-line swap) + page tests. ✅
- **No backend/API/token/design-system change** → only two web files created, two modified; no `lib/api.ts`, no `globals.css`, no `design-system.md`. ✅
- **CI green (lint/format/typecheck/test/build)** → Task 4. ✅
