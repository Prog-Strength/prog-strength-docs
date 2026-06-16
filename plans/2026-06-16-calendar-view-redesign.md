# Calendar View Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reimplement the `/calendar` surface in `prog-strength-web` to match the `warm-organic-coaching` design direction — per-week rounded panels, a per-week streak strip, soft tonal activity chips, soft rounded hero-stat cards, and a coaching header — re-toned onto the existing dark theme, with zero change to calendar data, API, rollup math, merge logic, or interactions.

**Architecture:** Presentation-layer only. Introduce dark-appropriate warm design tokens (CSS variables) in `app/globals.css`; centralize discipline/chip derivations in a new pure module `components/calendar/derivations.ts`; replace the right-hand weekly column with a `WeekStreakStrip`; restyle `day-cell`, `day-digest`, and the planned/logged banners to the new chip+token language; and recompose `app/(app)/calendar/page.tsx` from an 8-column grid into a vertical stack of per-week rounded panels with a coaching header. All existing data wiring (`buildEventsByDate`, `eventsByDate`, `weeklyStats`, `monthStats`, `localDateKey`, `todayKey`, `selected`) and every interaction prop is preserved.

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript, Tailwind v4 (utility classes + CSS-variable theme tokens), Vitest + Testing Library (jsdom), co-located `*.test.tsx`.

---

## File Structure

**Created:**
- `prog-strength-web/components/calendar/derivations.ts` — pure helpers: `Discipline` type, `disciplineOf`, `ChipState` type, `chipStateOf`, `isTrainedDay`, plus honest coaching-copy helpers `weekStreakCopy` and `monthConsistencyCopy`. One responsibility: classify events and produce streak/coaching values. No React, no I/O.
- `prog-strength-web/components/calendar/derivations.test.ts` — unit tests for the above.
- `prog-strength-web/components/calendar/day-cell.test.tsx` — new behavioral + styling tests for the restyled cell (none exists today).

**Modified:**
- `prog-strength-web/app/globals.css` — add warm accent + per-discipline tonal tokens (run, lift produced; mobility, core reserved) to `:root` and `@theme inline`.
- `prog-strength-web/components/calendar/weekly-overview.tsx` — replace `WeeklyTile`/`WeeklyChip`/`WeeklyOverviewColumn` with `WeekStreakStrip`; extend `WeeklyStat` with a per-day `days: DayMark[]` field; keep `formatTotalDuration` + step formatting.
- `prog-strength-web/components/calendar/weekly-overview.test.tsx` — rewrite for `WeekStreakStrip`.
- `prog-strength-web/components/calendar/day-cell.tsx` — rounded dark cell + tonal chips, token-driven, behavior preserved.
- `prog-strength-web/components/calendar/day-digest.tsx` — restyle to rounded warm-on-dark cards / tonal accents; behavior preserved.
- `prog-strength-web/components/calendar/planned-banner.tsx`, `run-banner.tsx`, `workout-banner.tsx`, `completed-planned-banner.tsx` — align rails/accents to the discipline-token language; behavior preserved.
- `prog-strength-web/app/(app)/calendar/page.tsx` — coaching header (`useProfile()`), soft hero-stat cards, per-week rounded panels (DayCell row + WeekStreakStrip footer) replacing the 8-column grid; compute `days: DayMark[]` per week and the month-consistency count.
- `prog-strength-web/app/(app)/calendar/page.test.tsx` — mock `useProfile`, assert panels/strip/header, keep all interaction tests.

**Untouched (verify, don't change):** `merge-events.ts`, `types.ts`, `run-digest.tsx`, `workout-digest.tsx`, `agenda-editor.tsx`, `plan-schedule-field.tsx`, `planned-agenda-details.tsx`, `lib/api.ts`, `lib/profile-context.tsx`, all routing. The `app/design-explore/` route is not in this repo checkout and is never touched.

---

## Color tokens — exact values (used across multiple tasks)

Re-toned for the dark surface (`--background: #0a0a0a`, `--surface: #18181b`). The variant's clay `#cd6f3e` is lightened to `#e08a5e` for AA on dark; chip text is a light warm tone on a dark warm fill (comfortably > 4.5:1).

```
--warm-accent: #e08a5e;        /* clay, lightened for AA on dark — actions, today, streak emphasis */
--warm-accent-fg: #1b1206;     /* near-black warm — text/icon on a filled warm-accent surface */

--discipline-run-bg: #3a241a;  /* dark warm brown — run chip fill */
--discipline-run-fg: #f0b48f;  /* light clay — run chip text (AA on run-bg) */
--discipline-run-dot: #e08a5e; /* run streak dot / accent */

--discipline-lift-bg: #332a16; /* dark warm amber — lift chip fill */
--discipline-lift-fg: #e8c878; /* light gold — lift chip text (AA on lift-bg) */
--discipline-lift-dot: #d6ab54;/* lift streak dot / accent */

/* Reserved, unused today (extensible Discipline slots): */
--discipline-mobility-bg: #1f3330;
--discipline-mobility-fg: #8fd6c4;
--discipline-mobility-dot: #4fbfa3;
--discipline-core-bg: #2c2440;
--discipline-core-fg: #c2adf0;
--discipline-core-dot: #9a7fe0;
```

---

### Task 1: Design tokens (dark-toned warm system)

**Files:**
- Modify: `prog-strength-web/app/globals.css`

- [ ] **Step 1: Add the warm + discipline CSS variables to `:root`**

In `app/globals.css`, inside the existing `:root { … }` block, after the `--success:` line and before the closing `}`, add:

```css
  /* Warm-organic coaching tokens (calendar redesign), re-toned for the
     dark surface. The DX variant's light triples are the hue reference;
     these dark-surface equivalents hold WCAG-AA contrast for chip text. */
  --warm-accent: #e08a5e; /* clay, lightened from #cd6f3e for AA on dark */
  --warm-accent-fg: #1b1206; /* near-black warm text on a filled warm-accent */

  --discipline-run-bg: #3a241a;
  --discipline-run-fg: #f0b48f;
  --discipline-run-dot: #e08a5e;

  --discipline-lift-bg: #332a16;
  --discipline-lift-fg: #e8c878;
  --discipline-lift-dot: #d6ab54;

  /* Reserved discipline slots — not produced from data today (only run and
     lift are inferred); kept so a future mobility/core surface composes. */
  --discipline-mobility-bg: #1f3330;
  --discipline-mobility-fg: #8fd6c4;
  --discipline-mobility-dot: #4fbfa3;
  --discipline-core-bg: #2c2440;
  --discipline-core-fg: #c2adf0;
  --discipline-core-dot: #9a7fe0;
```

- [ ] **Step 2: Mirror the new tokens into `@theme inline`**

Inside the existing `@theme inline { … }` block, after the `--color-success:` line, add (keeps the file's convention of mapping every var into the theme, so `bg-warm-accent` / `text-discipline-run-fg` resolve if ever used):

```css
  --color-warm-accent: var(--warm-accent);
  --color-warm-accent-fg: var(--warm-accent-fg);
  --color-discipline-run-bg: var(--discipline-run-bg);
  --color-discipline-run-fg: var(--discipline-run-fg);
  --color-discipline-run-dot: var(--discipline-run-dot);
  --color-discipline-lift-bg: var(--discipline-lift-bg);
  --color-discipline-lift-fg: var(--discipline-lift-fg);
  --color-discipline-lift-dot: var(--discipline-lift-dot);
  --color-discipline-mobility-bg: var(--discipline-mobility-bg);
  --color-discipline-mobility-fg: var(--discipline-mobility-fg);
  --color-discipline-mobility-dot: var(--discipline-mobility-dot);
  --color-discipline-core-bg: var(--discipline-core-bg);
  --color-discipline-core-fg: var(--discipline-core-fg);
  --color-discipline-core-dot: var(--discipline-core-dot);
```

- [ ] **Step 3: Verify the app still builds and formats clean**

Run: `npm run format:check && npm run build`
Expected: format check passes for `app/globals.css`; build succeeds. (No runtime change yet — tokens are only defined.)

- [ ] **Step 4: Commit**

```bash
git add app/globals.css
git commit -m "feat(calendar): add dark-toned warm + per-discipline design tokens"
```

---

### Task 2: Discipline & coaching derivations module

**Files:**
- Create: `prog-strength-web/components/calendar/derivations.ts`
- Test: `prog-strength-web/components/calendar/derivations.test.ts`

Centralizes how an event maps to a discipline (run vs lift), whether it's a "done" vs "planned" chip, whether a day counts as trained, and the honest coaching copy. Chips, streak dots, and the header all consume these so they agree.

- [ ] **Step 1: Write the failing test**

Create `components/calendar/derivations.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import type { CalendarEvent } from "@/components/calendar/types";
import type { PlannedWorkout, RunningSession, Workout } from "@/lib/api";
import {
  disciplineOf,
  chipStateOf,
  isTrainedDay,
  weekStreakCopy,
  monthConsistencyCopy,
} from "./derivations";

const workout = { id: "w1", performed_at: "2026-06-15T08:00:00Z" } as unknown as Workout;
const run = { id: "r1", start_time: "2026-06-15T07:00:00Z" } as unknown as RunningSession;
const liftPlan = { id: "p1", activity_kind: "lift", scheduled_start: "2026-06-15T17:00:00Z" } as unknown as PlannedWorkout;
const runPlan = { id: "p2", activity_kind: "run", scheduled_start: "2026-06-15T17:00:00Z" } as unknown as PlannedWorkout;

const evWorkout: CalendarEvent = { kind: "workout", startMs: 1, workout };
const evRun: CalendarEvent = { kind: "run", startMs: 1, run };
const evPlannedLift: CalendarEvent = { kind: "planned", startMs: 1, planned: liftPlan };
const evPlannedRun: CalendarEvent = { kind: "planned", startMs: 1, planned: runPlan };
const evCompletedLift: CalendarEvent = {
  kind: "completed-planned", startMs: 1, planned: liftPlan, logged: { kind: "workout", workout },
};
const evCompletedRun: CalendarEvent = {
  kind: "completed-planned", startMs: 1, planned: runPlan, logged: { kind: "run", run },
};

describe("disciplineOf", () => {
  it("maps logged workouts and run-fulfilled plans to the right discipline", () => {
    expect(disciplineOf(evWorkout)).toBe("lift");
    expect(disciplineOf(evRun)).toBe("run");
    expect(disciplineOf(evPlannedLift)).toBe("lift");
    expect(disciplineOf(evPlannedRun)).toBe("run");
    expect(disciplineOf(evCompletedLift)).toBe("lift");
    expect(disciplineOf(evCompletedRun)).toBe("run");
  });
});

describe("chipStateOf", () => {
  it("treats planned as planned and everything else (including completed-planned) as done", () => {
    expect(chipStateOf(evPlannedLift)).toBe("planned");
    expect(chipStateOf(evPlannedRun)).toBe("planned");
    expect(chipStateOf(evWorkout)).toBe("done");
    expect(chipStateOf(evRun)).toBe("done");
    expect(chipStateOf(evCompletedLift)).toBe("done");
    expect(chipStateOf(evCompletedRun)).toBe("done");
  });
});

describe("isTrainedDay", () => {
  it("is true when any event is done, false for empty or planned-only days", () => {
    expect(isTrainedDay([])).toBe(false);
    expect(isTrainedDay([evPlannedLift])).toBe(false);
    expect(isTrainedDay([evPlannedLift, evWorkout])).toBe(true);
    expect(isTrainedDay([evCompletedRun])).toBe(true);
  });
});

describe("weekStreakCopy", () => {
  it("frames a trained week as N of M", () => {
    expect(weekStreakCopy(4, 7)).toBe("You trained 4 of 7 days");
    expect(weekStreakCopy(1, 5)).toBe("You trained 1 of 5 days");
  });
  it("uses a gentle rest-week line when nothing was trained", () => {
    expect(weekStreakCopy(0, 7)).toBe("Rest week — recovery counts too");
    expect(weekStreakCopy(0, 3)).toBe("Rest week — recovery counts too");
  });
});

describe("monthConsistencyCopy", () => {
  it("greets by name and states trained days honestly", () => {
    expect(monthConsistencyCopy("Sam", 9)).toBe("Nice work, Sam — you've trained 9 days this month.");
    expect(monthConsistencyCopy("Sam", 1)).toBe("Nice work, Sam — you've trained 1 day this month.");
  });
  it("reads encouragingly, not scolding, for a zero-day month", () => {
    expect(monthConsistencyCopy("Sam", 0)).toBe("Welcome back, Sam — a fresh month to build on.");
  });
  it("falls back to a neutral greeting when no name is available", () => {
    expect(monthConsistencyCopy(null, 5)).toBe("Nice work — you've trained 5 days this month.");
    expect(monthConsistencyCopy(null, 0)).toBe("Welcome back — a fresh month to build on.");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run components/calendar/derivations.test.ts`
Expected: FAIL — cannot resolve `./derivations` (module not created yet).

- [ ] **Step 3: Write the implementation**

Create `components/calendar/derivations.ts`:

```ts
import type { CalendarEvent } from "@/components/calendar/types";

/**
 * Discipline of an activity, used to pick its tonal hue (run vs lift today;
 * mobility/core are reserved, extensible slots not inferred from data yet).
 * Centralized so chips, streak dots, and any future surface agree.
 */
export type Discipline = "run" | "lift" | "mobility" | "core";

/** Whether a chip reads as completed ("done") or forward-looking ("planned"). */
export type ChipState = "done" | "planned";

/**
 * Map a calendar event to its discipline. A logged run — or a planned/
 * completed-planned session whose activity is a run — is `run`; the lift
 * equivalents are `lift`. Planned discipline comes from the plan's
 * `activity_kind`; completed-planned from the logged session it merged with.
 */
export function disciplineOf(event: CalendarEvent): Discipline {
  switch (event.kind) {
    case "run":
      return "run";
    case "workout":
      return "lift";
    case "completed-planned":
      return event.logged.kind === "run" ? "run" : "lift";
    case "planned":
      return event.planned.activity_kind === "run" ? "run" : "lift";
  }
}

/**
 * A planned (not-yet-done) session is `planned`; logged and completed-planned
 * sessions are `done` (a completed-planned session already merged — it
 * happened).
 */
export function chipStateOf(event: CalendarEvent): ChipState {
  return event.kind === "planned" ? "planned" : "done";
}

/** True when a day carries at least one done event — drives streak dots/counts. */
export function isTrainedDay(events: CalendarEvent[]): boolean {
  return events.some((e) => chipStateOf(e) === "done");
}

/**
 * Coaching line for a single week's streak strip. `trained` is in-month days
 * with a done event; `total` is the in-month days in that week. A zero-trained
 * week reads as intentional rest, never as a failure.
 */
export function weekStreakCopy(trained: number, total: number): string {
  if (trained === 0) return "Rest week — recovery counts too";
  return `You trained ${trained} of ${total} days`;
}

/**
 * Coaching greeting for the month. Greets by `name` when known (graceful
 * neutral fallback when null), and states the month's trained-day count
 * honestly — a zero-day month reads as a fresh start, not a scold.
 */
export function monthConsistencyCopy(name: string | null, trainedDays: number): string {
  const who = name?.trim() ? `, ${name.trim()}` : "";
  if (trainedDays === 0) return `Welcome back${who} — a fresh month to build on.`;
  const dayWord = trainedDays === 1 ? "day" : "days";
  return `Nice work${who} — you've trained ${trainedDays} ${dayWord} this month.`;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run components/calendar/derivations.test.ts`
Expected: PASS — all assertions green.

- [ ] **Step 5: Commit**

```bash
git add components/calendar/derivations.ts components/calendar/derivations.test.ts
git commit -m "feat(calendar): add discipline + coaching-copy derivations"
```

---

### Task 3: WeekStreakStrip (replace the weekly column)

**Files:**
- Modify: `prog-strength-web/components/calendar/weekly-overview.tsx`
- Modify (rewrite): `prog-strength-web/components/calendar/weekly-overview.test.tsx`

Replace `WeeklyTile`/`WeeklyChip`/`WeeklyOverviewColumn` with a single responsive `WeekStreakStrip`: seven trained/untrained dots, a "trained N of M days" / rest-week line, and compact `🏋 lift · 🏃 run · 👟 steps` metric labels. Keep the `WeeklyStat` type (extended with per-day marks) and its `formatTotalDuration` + step formatting helpers.

- [ ] **Step 1: Write the failing test (rewrite the file)**

Replace the entire contents of `components/calendar/weekly-overview.test.tsx` with:

```tsx
import { render, screen, within } from "@testing-library/react";
import { DistanceUnitProvider } from "@/lib/distance-unit-context";
import { WeekStreakStrip, type WeeklyStat, type DayMark } from "./weekly-overview";

function marks(pattern: Array<[boolean, boolean]>): DayMark[] {
  // pattern entries are [inMonth, trained]
  return pattern.map(([inMonth, trained]) => ({ inMonth, trained }));
}

function makeWeek(overrides: Partial<WeeklyStat> = {}): WeeklyStat {
  return {
    weekStart: new Date(2026, 5, 1),
    activities: 0,
    liftMinutes: 0,
    runMeters: 0,
    steps: 0,
    days: marks([
      [true, false], [true, false], [true, false], [true, false],
      [true, false], [true, false], [true, false],
    ]),
    ...overrides,
  };
}

function renderStrip(week: WeeklyStat, isCurrent = false) {
  return render(
    <DistanceUnitProvider>
      <WeekStreakStrip week={week} isCurrent={isCurrent} />
    </DistanceUnitProvider>,
  );
}

describe("WeekStreakStrip", () => {
  it("renders seven day dots", () => {
    renderStrip(makeWeek());
    const strip = screen.getByTestId("week-streak-strip");
    expect(within(strip).getAllByTestId("streak-dot")).toHaveLength(7);
  });

  it("frames a trained week as N of M in-month days", () => {
    const week = makeWeek({
      activities: 4,
      days: marks([
        [true, true], [true, false], [true, true], [true, true],
        [true, false], [true, true], [true, false],
      ]),
    });
    renderStrip(week);
    expect(screen.getByTestId("week-streak-strip")).toHaveTextContent("You trained 4 of 7 days");
  });

  it("counts only in-month days toward the N of M total", () => {
    // Leading week: first two days belong to the previous month.
    const week = makeWeek({
      activities: 1,
      days: marks([
        [false, false], [false, false], [true, true], [true, false],
        [true, false], [true, false], [true, false],
      ]),
    });
    renderStrip(week);
    expect(screen.getByTestId("week-streak-strip")).toHaveTextContent("You trained 1 of 5 days");
  });

  it("marks trained dots distinctly from untrained ones", () => {
    const week = makeWeek({
      days: marks([
        [true, true], [true, false], [true, false], [true, false],
        [true, false], [true, false], [true, false],
      ]),
    });
    renderStrip(week);
    const dots = screen.getAllByTestId("streak-dot");
    expect(dots[0]).toHaveAttribute("data-trained", "true");
    expect(dots[1]).not.toHaveAttribute("data-trained", "true");
  });

  it("shows a gentle rest-week line when no in-month day was trained", () => {
    renderStrip(makeWeek({ activities: 0 }));
    expect(screen.getByTestId("week-streak-strip")).toHaveTextContent("Rest week — recovery counts too");
  });

  it("renders metric labels for lift time, run distance, and steps when present", () => {
    const week = makeWeek({ activities: 3, liftMinutes: 90, runMeters: 5000, steps: 8200 });
    const strip = renderStrip(week).getByTestId
      ? screen.getByTestId("week-streak-strip")
      : screen.getByTestId("week-streak-strip");
    expect(strip).toHaveTextContent("1h 30m"); // lift time
    expect(strip).toHaveTextContent("8,200"); // steps, thousands-separated
  });

  it("omits zero metric labels", () => {
    const week = makeWeek({ activities: 1, liftMinutes: 0, runMeters: 0, steps: 0 });
    const strip = screen.getByTestId("week-streak-strip");
    // Only the streak line; no lift/steps metric values.
    expect(strip).not.toHaveTextContent("1h");
  });

  it("emphasizes the current week", () => {
    renderStrip(makeWeek(), true);
    const strip = screen.getByTestId("week-streak-strip");
    expect(strip).toHaveAttribute("data-current", "true");
  });
});
```

Note: the last two `it()` blocks each call `renderStrip` once; the `getByTestId` lookups read from the most recent render. Keep one `render` per test (Testing Library auto-cleans between tests).

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run components/calendar/weekly-overview.test.tsx`
Expected: FAIL — `WeekStreakStrip` / `DayMark` not exported.

- [ ] **Step 3: Rewrite the implementation**

Replace the entire contents of `components/calendar/weekly-overview.tsx` with:

```tsx
"use client";

import { useDistanceUnit } from "@/lib/distance-unit-context";
import { weekStreakCopy } from "@/components/calendar/derivations";

/**
 * Per-week consistency summary shown as the footer of each week panel. The
 * month grid is six Monday-started weeks; one streak strip sits beneath each
 * week's seven day cells, reframing the old right-column rollup as a coaching
 * line: seven trained/untrained dots, "You trained N of M days" (or a gentle
 * rest-week line), and compact lift/run/steps metric labels.
 */

/** One day's marks within a week: whether it's in the cursor month and trained. */
export type DayMark = { inMonth: boolean; trained: boolean };

export type WeeklyStat = {
  weekStart: Date;
  activities: number;
  liftMinutes: number;
  runMeters: number;
  steps: number;
  /** Seven entries, Monday→Sunday, aligned with the week's day cells. */
  days: DayMark[];
};

/** Thousands-separated step count, e.g. 52340 → "52,340". */
function formatSteps(steps: number): string {
  return steps.toLocaleString("en-US");
}

/**
 * Total duration as `Xh Ym`, `Xh`, or `Ym`; "0h" for non-positive. Kept in
 * sync with page.tsx's formatTotalDuration (page.tsx doesn't export it).
 */
export function formatTotalDuration(minutes: number): string {
  if (minutes <= 0) return "0h";
  if (minutes < 60) return `${minutes}m`;
  const h = Math.floor(minutes / 60);
  const m = minutes % 60;
  return m > 0 ? `${h}h ${m}m` : `${h}h`;
}

/**
 * One week's streak strip. Trained/total are derived from the in-month days
 * in `week.days` (out-of-month leading/trailing days don't count toward the
 * total and read de-emphasized). The current week gets a warm-accent border.
 */
export function WeekStreakStrip({ week, isCurrent }: { week: WeeklyStat; isCurrent: boolean }) {
  const { formatDistance, unitLabel } = useDistanceUnit();
  const trained = week.days.filter((d) => d.inMonth && d.trained).length;
  const total = week.days.filter((d) => d.inMonth).length;

  const metrics: string[] = [];
  if (week.liftMinutes > 0) metrics.push(`🏋 ${formatTotalDuration(week.liftMinutes)}`);
  if (week.runMeters > 0) metrics.push(`🏃 ${formatDistance(week.runMeters)} ${unitLabel}`);
  if (week.steps > 0) metrics.push(`👟 ${formatSteps(week.steps)}`);

  const borderClass = isCurrent ? "border-[var(--warm-accent)]" : "border-[var(--border)]";

  return (
    <div
      data-testid="week-streak-strip"
      data-current={isCurrent ? "true" : undefined}
      className={`flex flex-wrap items-center gap-x-4 gap-y-2 rounded-2xl border ${borderClass} bg-[var(--surface)]/60 px-3 py-2.5`}
    >
      <div className="flex items-center gap-1.5" aria-hidden="true">
        {week.days.map((d, i) => (
          <span
            key={i}
            data-testid="streak-dot"
            data-trained={d.inMonth && d.trained ? "true" : undefined}
            className={`h-2.5 w-2.5 rounded-full ${
              d.inMonth && d.trained
                ? "bg-[var(--warm-accent)]"
                : d.inMonth
                  ? "border border-[var(--border)] bg-transparent"
                  : "border border-[var(--border)]/40 bg-transparent opacity-40"
            }`}
          />
        ))}
      </div>
      <span
        className={`text-xs font-medium ${
          trained > 0 ? "text-[var(--foreground)]" : "text-[var(--muted)]"
        }`}
      >
        {weekStreakCopy(trained, total)}
      </span>
      {metrics.length > 0 && (
        <span className="ml-auto text-xs tabular-nums text-[var(--muted)]">
          {metrics.join("  ·  ")}
        </span>
      )}
    </div>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run components/calendar/weekly-overview.test.tsx`
Expected: PASS. (The page.tsx import of `WeeklyChip`/`WeeklyTile` is now broken — that's fixed in Task 6; typecheck/build for the whole app is deferred to Task 6 and the final gate.)

- [ ] **Step 5: Commit**

```bash
git add components/calendar/weekly-overview.tsx components/calendar/weekly-overview.test.tsx
git commit -m "feat(calendar): replace weekly column with WeekStreakStrip"
```

---

### Task 4: DayCell — rounded dark cell with tonal chips

**Files:**
- Modify: `prog-strength-web/components/calendar/day-cell.tsx`
- Create: `prog-strength-web/components/calendar/day-cell.test.tsx`

Restyle the cell to a rounded dark surface (generous min-height, soft border; today = warm-accent border + accent-filled date badge) and render events as tonal chips driven by the discipline tokens: done = filled discipline tone + ✓; planned = dashed outline keeping the existing clock/recurring/sync affordances, re-skinned. **Every interaction prop and the `+N more`/`MAX_VISIBLE_PILLS` behavior is preserved.**

- [ ] **Step 1: Write the failing test**

Create `components/calendar/day-cell.test.tsx`:

```tsx
import { render, screen, fireEvent, within } from "@testing-library/react";
import type { CalendarEvent } from "@/components/calendar/types";
import type { PlannedWorkout, RunningSession, Workout } from "@/lib/api";
import { DayCell } from "./day-cell";

const DAY = new Date(2026, 5, 16);

function workout(id: string, name: string): Workout {
  return { id, name, performed_at: "2026-06-16T08:00:00Z", exercises: [] } as unknown as Workout;
}
function run(id: string, name: string): RunningSession {
  return { id, name, start_time: "2026-06-16T07:00:00Z" } as unknown as RunningSession;
}
function plan(id: string, name: string): PlannedWorkout {
  return {
    id, name, activity_kind: "lift", scheduled_start: "2026-06-16T17:00:00Z",
    status: "planned", google_sync_status: null,
  } as unknown as PlannedWorkout;
}

function noop() {}

function renderCell(events: CalendarEvent[], overrides: Partial<React.ComponentProps<typeof DayCell>> = {}) {
  return render(
    <DayCell
      day={DAY}
      inMonth
      isToday={false}
      isSelected={false}
      events={events}
      onSelectDay={overrides.onSelectDay ?? noop}
      onSelectWorkout={overrides.onSelectWorkout ?? noop}
      onSelectRun={overrides.onSelectRun ?? noop}
      onSelectPlanned={overrides.onSelectPlanned ?? noop}
      {...overrides}
    />,
  );
}

describe("DayCell", () => {
  it("renders the date number", () => {
    renderCell([]);
    expect(screen.getByText("16")).toBeInTheDocument();
  });

  it("renders a done lift chip and fires onSelectWorkout", () => {
    const onSelectWorkout = vi.fn();
    renderCell([{ kind: "workout", startMs: 1, workout: workout("w1", "Upper 1") }], { onSelectWorkout });
    const chip = screen.getByRole("button", { name: "Upper 1" });
    fireEvent.click(chip);
    expect(onSelectWorkout).toHaveBeenCalledWith("w1");
  });

  it("renders a planned chip with a dashed outline, distinct from a done chip", () => {
    renderCell([
      { kind: "workout", startMs: 1, workout: workout("w1", "Logged Lift") },
      { kind: "planned", startMs: 2, planned: plan("p1", "Planned Lift") },
    ]);
    const done = screen.getByRole("button", { name: "Logged Lift" });
    const planned = screen.getByTestId("planned-pill");
    expect(planned).toHaveTextContent("Planned Lift");
    expect(planned.className).toMatch(/border-dashed/);
    expect(done.className).not.toMatch(/border-dashed/);
  });

  it("tones run and lift chips differently", () => {
    renderCell([
      { kind: "workout", startMs: 1, workout: workout("w1", "Lift") },
      { kind: "run", startMs: 2, run: run("r1", "Run") },
    ]);
    const lift = screen.getByRole("button", { name: "Lift" });
    const runChip = screen.getByRole("button", { name: "Run" });
    expect(lift.className).toMatch(/discipline-lift/);
    expect(runChip.className).toMatch(/discipline-run/);
  });

  it("rolls events beyond the visible max into a +N more control", () => {
    const onSelectDay = vi.fn();
    const events: CalendarEvent[] = [
      { kind: "workout", startMs: 1, workout: workout("w1", "A") },
      { kind: "workout", startMs: 2, workout: workout("w2", "B") },
      { kind: "workout", startMs: 3, workout: workout("w3", "C") },
      { kind: "workout", startMs: 4, workout: workout("w4", "D") },
    ];
    renderCell(events, { onSelectDay });
    const more = screen.getByRole("button", { name: /\+1 more/ });
    fireEvent.click(more);
    expect(onSelectDay).toHaveBeenCalled();
  });

  it("exposes an accessible date+activity label", () => {
    renderCell([{ kind: "run", startMs: 1, run: run("r1", "Run") }]);
    expect(screen.getByLabelText(/Tuesday, June 16, 2026, 1 run/)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run components/calendar/day-cell.test.tsx`
Expected: FAIL on the tonal-class assertions (`discipline-lift` / `discipline-run` not present) — the chips still use `bg-[var(--accent)]` / teal. (Other assertions may pass against today's markup.)

- [ ] **Step 3: Restyle the cell skeleton (token-driven, rounded, today badge)**

In `components/calendar/day-cell.tsx`, replace the cell-skeleton class block (the `baseClasses` / `borderClasses` / `fillClasses` / `labelClasses` constants and the date-number `<div>`) so the cell is rounded with a generous min-height and today gets a warm-accent border + filled date badge. Replace:

```tsx
  const baseClasses =
    "flex min-h-[60px] cursor-pointer flex-col gap-0.5 rounded-md border p-1 transition hover:border-[var(--accent)]/50 md:min-h-[88px] md:gap-1 md:p-1.5";
  const borderClasses = isToday ? "border-[var(--accent)]" : "border-[var(--border)]";
  const fillClasses = isSelected ? "bg-[var(--accent)]/10" : "bg-[var(--surface)]";
  const labelClasses = inMonth ? "text-[var(--foreground)]" : "text-[var(--muted)] opacity-60";

  return (
    <div
      className={`${baseClasses} ${borderClasses} ${fillClasses}`}
      aria-label={ariaLabelFor(day, events)}
      onClick={onSelectDay}
    >
      <div
        className={`px-0.5 text-[11px] font-medium leading-none md:px-1 md:text-xs ${labelClasses}`}
      >
        {day.getDate()}
      </div>
```

with:

```tsx
  const baseClasses =
    "flex min-h-[72px] cursor-pointer flex-col gap-1 rounded-2xl border p-1.5 transition hover:border-[var(--warm-accent)]/50 md:min-h-[104px] md:gap-1.5 md:p-2";
  const borderClasses = isToday ? "border-[var(--warm-accent)]" : "border-[var(--border)]";
  const fillClasses = isSelected ? "bg-[var(--warm-accent)]/10" : "bg-[var(--surface)]";
  const labelClasses = inMonth ? "text-[var(--foreground)]" : "text-[var(--muted)] opacity-50";

  return (
    <div
      className={`${baseClasses} ${borderClasses} ${fillClasses}`}
      aria-label={ariaLabelFor(day, events)}
      onClick={onSelectDay}
    >
      <div className="px-0.5 leading-none md:px-1">
        {isToday ? (
          <span className="inline-flex h-5 w-5 items-center justify-center rounded-full bg-[var(--warm-accent)] text-[11px] font-semibold text-[var(--warm-accent-fg)] md:h-6 md:w-6 md:text-xs">
            {day.getDate()}
          </span>
        ) : (
          <span className={`text-[11px] font-medium md:text-xs ${labelClasses}`}>
            {day.getDate()}
          </span>
        )}
      </div>
```

- [ ] **Step 4: Add a shared tonal-chip class helper and re-skin WorkoutPill/RunPill**

Near the top of `day-cell.tsx` (after `MAX_VISIBLE_PILLS`), add a helper that turns a discipline into the done-chip classes, importing the discipline type:

Add to the imports at the top:

```tsx
import { disciplineOf, type Discipline } from "@/components/calendar/derivations";
```

Add the helper:

```tsx
// Shared chip skeleton + per-discipline "done" tone, sourced from the
// design tokens so run vs lift read as distinct warm hues on dark.
const CHIP_BASE =
  "flex items-center gap-1 truncate rounded-lg px-1.5 py-0.5 text-left text-[9px] font-medium leading-tight transition md:px-2 md:py-0.5 md:text-[10px] md:leading-normal";

function doneToneClasses(discipline: Discipline): string {
  return discipline === "run"
    ? "bg-[var(--discipline-run-bg)] text-[var(--discipline-run-fg)] hover:opacity-90"
    : "bg-[var(--discipline-lift-bg)] text-[var(--discipline-lift-fg)] hover:opacity-90";
}
```

Replace `WorkoutPill`'s `className` (the long `className="truncate rounded bg-[var(--accent)] …"`) so it uses the lift tone and a leading ✓ done-dot. Replace the whole `<button>` body of `WorkoutPill` with:

```tsx
  return (
    <button
      type="button"
      onClick={(e) => {
        e.stopPropagation();
        onClick();
      }}
      title={named ? `${time} · ${workout.name}` : time}
      className={`${CHIP_BASE} ${doneToneClasses("lift")}`}
    >
      <CheckGlyph />
      <span className="truncate">{label}</span>
    </button>
  );
```

Replace `RunPill`'s `<button>` body with the run tone:

```tsx
  return (
    <button
      type="button"
      onClick={(e) => {
        e.stopPropagation();
        onClick();
      }}
      title={`${time} · Run${run.name ? ` · ${run.name}` : ""}`}
      className={`${CHIP_BASE} ${doneToneClasses("run")}`}
    >
      <CheckGlyph />
      <span className="truncate">{label}</span>
    </button>
  );
```

- [ ] **Step 5: Re-skin CompletedPlannedPill and PlannedPill to the tokens**

In `CompletedPlannedPill`, replace the `tone` computation and the `<button>` className. Replace:

```tsx
  const tone = isRun
    ? "bg-teal-500/20 text-teal-300 hover:bg-teal-500/30"
    : "bg-[var(--accent)] text-[var(--accent-fg)] hover:opacity-90";
```

with:

```tsx
  const tone = doneToneClasses(isRun ? "run" : "lift");
```

and replace its `<button>`'s `className={`flex items-center gap-1 truncate rounded px-1 py-px …`}` with `className={`${CHIP_BASE} ${tone}`}`. Keep the leading `<CheckGlyph />` and `<span className="truncate">{label}</span>`.

In `PlannedPill`, keep the dashed-outline + clock/check/sync affordances but source the planned tone from the discipline. Replace the `tone` block:

```tsx
  const tone = completed
    ? "border-emerald-500/60 text-emerald-300"
    : skipped
      ? "border-[var(--border)] text-[var(--muted)] line-through"
      : "border-[var(--accent)]/60 text-[var(--accent)]";
```

with:

```tsx
  const discipline = planned.activity_kind === "run" ? "run" : "lift";
  const plannedTone =
    discipline === "run"
      ? "border-[var(--discipline-run-dot)]/60 text-[var(--discipline-run-fg)]"
      : "border-[var(--discipline-lift-dot)]/60 text-[var(--discipline-lift-fg)]";
  const tone = completed
    ? "border-emerald-500/60 text-emerald-300"
    : skipped
      ? "border-[var(--border)] text-[var(--muted)] line-through"
      : plannedTone;
```

and replace the `PlannedPill` `<button>` className (the `flex items-center gap-1 truncate rounded border border-dashed …` string) with:

```tsx
      className={`${CHIP_BASE} border border-dashed bg-transparent hover:bg-[var(--surface-2)] ${tone}`}
```

Keep `data-testid="planned-pill"`, the `{completed ? <CheckGlyph /> : skipped ? null : <ClockGlyph />}`, the `<span className="truncate">{label}</span>`, and `{synced && <SyncGlyph />}` exactly as they are. Leave `MAX_VISIBLE_PILLS`, the `+N more` button, `ariaLabelFor`, `ClockGlyph`, `CheckGlyph`, and `SyncGlyph` unchanged.

- [ ] **Step 6: Run the tests to verify they pass**

Run: `npx vitest run components/calendar/day-cell.test.tsx`
Expected: PASS — all six tests green.

- [ ] **Step 7: Commit**

```bash
git add components/calendar/day-cell.tsx components/calendar/day-cell.test.tsx
git commit -m "feat(calendar): tonal activity chips and rounded day cell"
```

---

### Task 5: Restyle the day digest and its banners to the token language

**Files:**
- Modify: `prog-strength-web/components/calendar/day-digest.tsx`
- Modify: `prog-strength-web/components/calendar/run-banner.tsx`
- Modify: `prog-strength-web/components/calendar/workout-banner.tsx`
- Modify: `prog-strength-web/components/calendar/completed-planned-banner.tsx`
- Modify: `prog-strength-web/components/calendar/planned-banner.tsx`

Restyle only. **No change to any banner's data, navigation, expand/collapse, plan/resync/unlink, or `defaultOpen` behavior** — the existing `day-digest.test.tsx` and `planned-banner.test.tsx` behavioral assertions must keep passing untouched.

- [ ] **Step 1: Soften the digest containers to rounded warm-on-dark**

In `day-digest.tsx`, the `StepsBanner` wrapper uses `rounded-lg`. Change its container className from `rounded-lg border border-[var(--border)]` to `rounded-2xl border border-[var(--border)]` to match the new card radius. No other change in this file.

- [ ] **Step 2: Re-skin the logged banners' rounding and rails**

In `run-banner.tsx`, `workout-banner.tsx`, and `completed-planned-banner.tsx`, change the outer container radius from `rounded-lg` to `rounded-2xl` (the `className="overflow-hidden rounded-lg border border-[var(--border)] bg-[var(--surface)]"` on each). Then align the colored rail to the discipline tone:

- `run-banner.tsx`: change the rail `bg-teal-500 dark:bg-teal-300` → `bg-[var(--discipline-run-dot)]` and the focus rings `ring-teal-500` → `ring-[var(--discipline-run-dot)]` (two occurrences).
- `workout-banner.tsx`: change the rail `bg-[var(--accent)]` → `bg-[var(--discipline-lift-dot)]` and the focus rings `ring-[var(--accent)]` → `ring-[var(--discipline-lift-dot)]` (two occurrences).
- `completed-planned-banner.tsx`: keep the emerald "completed" rail (it signals completion, not discipline — preserve the existing meaning), but change `ringColor` from `isRun ? "ring-teal-500" : "ring-[var(--accent)]"` → `isRun ? "ring-[var(--discipline-run-dot)]" : "ring-[var(--discipline-lift-dot)]"`.

- [ ] **Step 3: Re-skin the planned banner to the planned/token language**

In `planned-banner.tsx`: change the outer container radius `rounded-lg` → `rounded-2xl`. Leave the dashed border, the status-driven rail (`bg-emerald-500` / `bg-[var(--muted)]` / `bg-[var(--accent)]`), the `StatusBadge`, and the `SyncIndicator` behavior unchanged — the dashed outline + status rail is the established "planned" affordance and Task 4's chip already aligns to it. (Touching status colors risks regressing the planned-banner tests; radius is the only safe visual change here.)

- [ ] **Step 4: Run the existing banner/digest tests to confirm no behavioral regression**

Run: `npx vitest run components/calendar/day-digest.test.tsx components/calendar/planned-banner.test.tsx components/calendar/run-banner.test.tsx components/calendar/workout-banner.test.tsx`
Expected: PASS — all behavioral assertions unchanged and green.

- [ ] **Step 5: Commit**

```bash
git add components/calendar/day-digest.tsx components/calendar/run-banner.tsx components/calendar/workout-banner.tsx components/calendar/completed-planned-banner.tsx components/calendar/planned-banner.tsx
git commit -m "feat(calendar): align day digest banners to warm token language"
```

---

### Task 6: Page composition — coaching header, hero cards, per-week panels

**Files:**
- Modify: `prog-strength-web/app/(app)/calendar/page.tsx`
- Modify: `prog-strength-web/app/(app)/calendar/page.test.tsx`

Recompose the page: a coaching header greeting the user by `profile.display_name` with an honest month-consistency line; soft rounded hero-stat cards; and a vertical stack of per-week rounded panels (a seven-column DayCell row + a `WeekStreakStrip` footer) replacing the 8-column grid. Extend `weeklyStats` with the per-day `DayMark[]` and compute the month trained-day count. **All data wiring and interactions are preserved.**

- [ ] **Step 1: Update the page test first (mock useProfile, assert the new structure)**

In `app/(app)/calendar/page.test.tsx`, add a `useProfile` mock alongside the other `vi.mock` calls (after the `@/lib/active-workout-session` mock, before the `@/lib/api` mock):

```tsx
// The redesigned header greets the user by display_name via useProfile.
vi.mock("@/lib/profile-context", () => ({
  useProfile: () => ({ profile: { display_name: "Sam" } }),
}));
```

Then update the two weekly-tile tests and add header/panel assertions. Replace the test `"shows weekly tiles whose activity counts sum to the visible fixtures"` with:

```tsx
  it("shows one streak strip per week summarizing trained days", async () => {
    renderPage();
    await findDigest(TODAY);

    const strips = await screen.findAllByTestId("week-streak-strip");
    expect(strips.length).toBe(6); // six week rows
    // At least one week reads as a trained week ("You trained N of M days"),
    // since every fixture lands in the cursor month.
    expect(strips.some((s) => /You trained \d+ of \d+ days/.test(s.textContent ?? ""))).toBe(true);
  });
```

Replace the test `"rolls weekly steps into the weekly tiles"` with:

```tsx
  it("rolls weekly steps into the streak strip metric labels", async () => {
    renderPage();
    await findDigest(TODAY);

    const strips = await screen.findAllByTestId("week-streak-strip");
    // The seeded step totals (8,000 + 12,345) land in the cursor month, so at
    // least one strip carries a steps metric label (👟 with a separated total).
    expect(strips.some((s) => /👟/.test(s.textContent ?? ""))).toBe(true);
  });
```

Add a new test after those (asserts the coaching header):

```tsx
  it("greets the user by name with a month-consistency line", async () => {
    renderPage();
    await findDigest(TODAY);
    // The header greets by the mocked display_name and states trained days.
    expect(await screen.findByText(/Sam/)).toBeInTheDocument();
    expect(screen.getByText(/trained \d+ days? this month|fresh month to build on/)).toBeInTheDocument();
  });
```

Leave every other test (defaults-to-today, day-select, re-anchor, auto-expand, steps-in-digest, Avg Pace/Longest Run show/hide, planned-pill distinct, completed-planned collapse, planned-workouts window) exactly as is.

- [ ] **Step 2: Run the page test to verify it fails**

Run: `npx vitest run "app/(app)/calendar/page.test.tsx"`
Expected: FAIL — `week-streak-strip` not found / page still imports removed `WeeklyChip`/`WeeklyTile` (and the header greeting isn't rendered yet).

- [ ] **Step 3: Update imports and the weeklyStats derivation in page.tsx**

In `app/(app)/calendar/page.tsx`:

Replace the weekly-overview import:

```tsx
import { WeeklyChip, WeeklyTile, type WeeklyStat } from "@/components/calendar/weekly-overview";
```

with:

```tsx
import { WeekStreakStrip, type WeeklyStat } from "@/components/calendar/weekly-overview";
```

Add two imports (after the existing `@/components/calendar/...` imports):

```tsx
import { isTrainedDay, monthConsistencyCopy } from "@/components/calendar/derivations";
import { useProfile } from "@/lib/profile-context";
```

Read the profile near the other hooks at the top of `CalendarPage` (after `const { start } = useActiveWorkoutSession();`):

```tsx
  const { profile } = useProfile();
```

In the `weeklyStats` useMemo, build the per-day `days` array and include it in each pushed week. Replace the loop body's accumulation + push. Specifically, inside `for (let i = 0; i < days.length; i += 7) { … }`, after computing `activities/liftMinutes/runMeters/weekSteps`, replace the final `weeks.push({ … })` with:

```tsx
      const dayMarks = weekDays.map((d) => ({
        inMonth: d.getMonth() === cursor.month,
        trained: isTrainedDay(eventsByDate.get(localDateKey(d)) ?? []),
      }));
      weeks.push({
        weekStart: weekDays[0],
        activities,
        liftMinutes,
        runMeters,
        steps: weekSteps,
        days: dayMarks,
      });
```

Add `eventsByDate` and `cursor.month` to the `weeklyStats` dependency array. Change:

```tsx
  }, [days, workouts, runs, steps]);
```

to:

```tsx
  }, [days, workouts, runs, steps, eventsByDate, cursor.month]);
```

Add a memo for the month trained-day count (in-month days with a done event), right after `weeklyStats`:

```tsx
  // Distinct in-month days carrying a done event — the header's consistency
  // line. Weeks partition the 42-day grid, so summing per-week trained
  // in-month days counts each day once.
  const monthTrainedDays = useMemo(
    () => weeklyStats.reduce((sum, w) => sum + w.days.filter((d) => d.inMonth && d.trained).length, 0),
    [weeklyStats],
  );
```

- [ ] **Step 4: Add the coaching greeting to the header**

In the `<header>` block, the left side currently renders `<h1>{monthLabel}</h1>` and the Today button. Wrap the title in a small column that adds the coaching line beneath it. Replace:

```tsx
        <div className="flex items-center gap-3">
          <h1 className="text-lg font-semibold tracking-tight">{monthLabel}</h1>
          <button
            type="button"
            onClick={goToday}
            className="rounded-md border border-[var(--border)] bg-[var(--surface)] px-2 py-1 text-xs text-[var(--muted)] hover:text-[var(--foreground)]"
          >
            Today
          </button>
        </div>
```

with:

```tsx
        <div className="flex items-center gap-3">
          <div className="flex flex-col">
            <h1 className="text-lg font-semibold tracking-tight">{monthLabel}</h1>
            <p className="text-xs text-[var(--muted)]">
              {monthConsistencyCopy(profile?.display_name ?? null, monthTrainedDays)}
            </p>
          </div>
          <button
            type="button"
            onClick={goToday}
            className="rounded-full border border-[var(--border)] bg-[var(--surface)] px-3 py-1 text-xs text-[var(--muted)] transition hover:border-[var(--warm-accent)] hover:text-[var(--foreground)]"
          >
            Today
          </button>
        </div>
```

Re-skin the "Plan a workout" and nav buttons to warm-accent rounded. Change the "Plan a workout" button className from `rounded-md bg-[var(--accent)] …` to `rounded-full bg-[var(--warm-accent)] px-3.5 py-1.5 text-xs font-medium text-[var(--warm-accent-fg)] transition hover:opacity-90`, and in `NavButton` change `rounded-md` → `rounded-full` and `hover:text-[var(--foreground)]` stays; add `hover:border-[var(--warm-accent)]` to its className.

- [ ] **Step 5: Soften the hero StatTile to a rounded card**

Change the `StatTile` container className from:

```tsx
    <div className="rounded-lg border border-[var(--border)] bg-[var(--surface)] px-4 py-3">
```

to:

```tsx
    <div className="rounded-2xl border border-[var(--border)] bg-[var(--surface)] px-4 py-3.5">
```

Leave the six tiles, their values, and the `hasRuns` show/hide wrappers unchanged.

- [ ] **Step 6: Replace the 8-column grid with per-week rounded panels**

Replace the entire weekday-header grid + the `grid grid-cols-7 … md:grid-cols-[repeat(7,…)_minmax(140px,180px)]` block (from the `{/* Calendar grid… */}` comment through the closing of the six-week `Array.from({ length: 6 })` map, i.e. the two sibling `<div className="…grid…">` blocks) with a vertical stack of panels:

```tsx
          {/* Per-week rounded panels: a seven-column day row on top, a
              WeekStreakStrip footer beneath carrying that week's consistency.
              Replaces the old "7 days + Week column" grid — the strip is the
              single responsive home for weekly data at every breakpoint. */}
          <div className="flex flex-col gap-3">
            <div className="grid grid-cols-7 gap-1 px-1">
              {WEEKDAYS.map((d) => (
                <div key={d} className="px-2 text-center text-xs font-medium text-[var(--muted)]">
                  {d}
                </div>
              ))}
            </div>
            {Array.from({ length: 6 }).map((_, w) => (
              <div
                key={`week-${w}`}
                className="flex flex-col gap-2 rounded-3xl border border-[var(--border)] bg-[var(--surface)]/40 p-2 md:p-3"
              >
                <div className="grid grid-cols-7 gap-1 md:gap-1.5">
                  {days.slice(w * 7, w * 7 + 7).map((day) => {
                    const inMonth = day.getMonth() === cursor.month;
                    const key = localDateKey(day);
                    const dayEvents = eventsByDate.get(key) ?? [];
                    const isToday = key === todayKey;
                    const isSelected = key === selected;
                    return (
                      <DayCell
                        key={key}
                        day={day}
                        inMonth={inMonth}
                        isToday={isToday}
                        isSelected={isSelected}
                        events={dayEvents}
                        onSelectDay={() => selectDay(key)}
                        onSelectWorkout={(id) => selectDayAndExpand(key, id)}
                        onSelectRun={(id) => selectDayAndExpand(key, id)}
                        onSelectPlanned={(id) => selectDayAndExpand(key, id)}
                      />
                    );
                  })}
                </div>
                <WeekStreakStrip
                  week={weeklyStats[w]}
                  isCurrent={weekContainsToday(weeklyStats[w], todayKey)}
                />
              </div>
            ))}
          </div>
```

Remove the now-unused `Fragment` import if nothing else uses it (check: `Fragment` was only used by the old grid). Update the React import line `import { Fragment, useCallback, useEffect, useMemo, useRef, useState } from "react";` → drop `Fragment` if unused: `import { useCallback, useEffect, useMemo, useRef, useState } from "react";`. (`weekContainsToday` stays — it's still used for `isCurrent`.)

- [ ] **Step 7: Run the page test to verify it passes**

Run: `npx vitest run "app/(app)/calendar/page.test.tsx"`
Expected: PASS — strip-per-week, steps-in-strip, header greeting, and all preserved interaction tests green.

- [ ] **Step 8: Typecheck the whole app and run the full calendar suite**

Run: `npm run typecheck && npx vitest run components/calendar "app/(app)/calendar"`
Expected: typecheck clean (no dangling `WeeklyTile`/`WeeklyChip`/`Fragment` references); all calendar tests pass.

- [ ] **Step 9: Commit**

```bash
git add "app/(app)/calendar/page.tsx" "app/(app)/calendar/page.test.tsx"
git commit -m "feat(calendar): coaching header and per-week streak panels"
```

---

### Task 7: Full CI gate green locally

**Files:** none (verification + any fixups).

- [ ] **Step 1: Run the exact CI gate**

Run, in order:

```bash
npm run lint
npm run format:check
npm run typecheck
npm run test
npm run build
```

Expected: `lint` reports **0 errors** (pre-existing React-19 `react-hooks` warnings are acceptable per AGENTS.md, but introduce **no new warnings** — in particular don't add a `useProfile`/effect pattern that trips `set-state-in-effect`); `format:check` clean; `typecheck` clean; all tests pass; `build` succeeds.

- [ ] **Step 2: Fix any failures at the source**

If `format:check` fails, run `npm run format` and re-commit. If lint/typecheck/test/build fail, fix the underlying code (never `// eslint-disable`, never skip a test, never `--no-verify`). Re-run the full gate until green.

- [ ] **Step 3: Commit any fixups**

```bash
git add -A
git commit -m "chore(calendar): satisfy lint/format gate for redesign"
```

(Skip if there was nothing to fix.)

---

## Self-Review (completed by plan author)

**Spec coverage:**
- Per-week rounded panels + streak-strip footer → Task 6 (panels), Task 3 (strip). ✓
- Re-tone to dark palette, no light background → Task 1 tokens are all dark-surface; non-calendar pages untouched. ✓
- Preserve all functionality (day select, plan-a-workout month+day, prev/next/Today, clock/recurring affordances, completed-planned merge, digest, routing) → Tasks 4–6 preserve every prop and `merge-events.ts` is untouched; page tests assert interactions. ✓
- Streak strip = 7 dots + "trained N of M" + lift/run/steps labels from `WeeklyStat` → Task 3. ✓
- Coaching header greets by real `display_name` + honest month consistency → Task 2 copy + Task 6 header (graceful null fallback). ✓
- Activity→tonal hue by discipline via extensible `Discipline`, as named dark tokens → Task 1 (tokens, mobility/core reserved) + Task 2 (`disciplineOf`). ✓
- Update tests, keep suite + gate green → Tasks 3,4,6 update tests; Task 7 runs the full gate. ✓
- Non-goals respected: no app-wide re-theme, no light palette, no new disciplines inferred, no data/API/rollup/merge/interaction change, design-explore untouched (and absent from checkout). ✓
- States honored: dense day (`MAX_VISIBLE_PILLS` + `+N more` preserved, Task 4), rest week (rest-week copy, Tasks 2–3), leading/trailing de-emphasized (DayMark `inMonth` opacity, Tasks 3–4), today marked (warm badge/border, Task 4). ✓

**Placeholder scan:** No TBD/TODO; every code step has concrete code; commands have expected output. ✓

**Type consistency:** `Discipline`, `ChipState`, `DayMark`, `WeeklyStat.days`, `disciplineOf`, `chipStateOf`, `isTrainedDay`, `weekStreakCopy`, `monthConsistencyCopy`, `WeekStreakStrip`, `doneToneClasses`, `CHIP_BASE` are named identically wherever referenced across Tasks 2–6. ✓
