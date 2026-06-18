# Calendar Event Detail & Activity-Type Colors Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make calendar events click-to-detail (planned pill → read-only modal, logged pill → detail page) and make *activity type* own the pill/banner color everywhere, centralized behind a single `lib/activity-colors.ts` resolver, retiring the hardcoded `emerald-*` completed color and the violet "planned" status color.

**Architecture:** A new `lib/activity-colors.ts` is the single source of truth mapping a `Discipline` (`run`/`lift`, from the existing `disciplineOf` helper) to its CSS-variable color tokens (`{ dot, bg, fg }`, raw `var(--discipline-*)` strings) — modeled on `lib/macro-colors.ts`. Tokens are consumed as inline `style` (so a future per-user color source swaps in behind `activityColors()` with no call-site changes — Tailwind class scanning can't express runtime-chosen colors, inline style can). The one pseudo-class case, the banner focus ring, uses a companion `activityRingClass()` returning a literal Tailwind `focus-visible:ring-[…]` class kept in the same module. Status (planned/completed/skipped) is conveyed by shape + badge + glyph, never by stealing the color slot. Click routing changes so a grid pill opens its single event directly.

**Tech Stack:** Next.js 16 (App Router), React 19, TypeScript, Tailwind v4 (CSS-first, no config file), Vitest + Testing Library (jsdom).

**Repo path:** `/workspace/prog-strength-web` (branch `feat/calendar-event-detail-and-activity-colors`, already created and checked out; `npm ci` already run).

**Source spec:** `prog-strength-docs/sows/calendar-event-detail-and-activity-colors.md`

**Conventions (from `prog-strength-web/AGENTS.md`):**
- Tailwind utility classes + CSS variables for theme colors. Match neighboring surfaces; no new color tokens.
- Tests: Vitest + Testing Library, co-located `*.test.ts(x)`. New behavior with logic ships with a test.
- Conventional Commits enforced (Husky `commit-msg` + `pre-commit` lint-staged/typecheck). **Never `--no-verify`.** Format `type(scope): subject`, e.g. `feat(calendar): …`.
- Gate before claiming done: `npm run typecheck && npm run lint && npm run test`; run `npm run build` if structural.

**Resolved design decisions (from the SOW's Open Questions):**
1. **Logged grid-pill click → navigate directly** to the session detail page (the SOW's recommendation and its Goals wording "workout/run pills … navigate to /workouts/{id} / /running/{id}"). This makes the grid model "a pill opens its event" uniform. Consequence: the old grid-pill *select-day-and-auto-expand-banner* path becomes dead and is removed (Task 4).
2. **Completed-planned color source → the logged session's discipline** (`disciplineOf` already keys off `event.logged.kind`). No change needed; just route it through the resolver.
3. **Skipped legibility → muted + strikethrough + `Skipped` badge** at the activity-type color reduced to muted; no stronger cue added.

---

## File Structure

- **Create** `lib/activity-colors.ts` — the resolver (single source of truth, type → color tokens + focus-ring class).
- **Create** `lib/activity-colors.test.ts` — unit tests for the resolver.
- **Create** `components/calendar/completed-planned-banner.test.tsx` — color-parity + de-emerald tests (none exists today).
- **Modify** `components/calendar/day-cell.tsx` — pills source color from `activityColors` via inline style; retire `emerald-*`; planned pill → `onOpenPlanned`; logged/completed pills → navigate.
- **Modify** `components/calendar/day-cell.test.tsx` — update color + click-routing assertions.
- **Modify** `components/calendar/workout-banner.tsx`, `run-banner.tsx` — rail + focus ring from the resolver.
- **Modify** `components/calendar/planned-banner.tsx` — rail = type color (not status); retire emerald + violet-as-status; keep dashed shape + badge.
- **Modify** `components/calendar/planned-banner.test.tsx` — add rail/badge color assertions.
- **Modify** `components/calendar/completed-planned-banner.tsx` — rail/check/badge de-emeralded to type color/neutral.
- **Modify** `components/calendar/day-digest.tsx` — drop `autoExpandId` threading + key open/closed suffixes (now-dead after Task 4 routing).
- **Modify** `app/(app)/calendar/page.tsx` — wire `onOpenPlanned` → `openViewPlan` (+ select day); logged/completed pills → `router.push`; remove `selectDayAndExpand`/`autoExpandId`.
- **Modify** `app/(app)/calendar/page.test.tsx` — update the pill-click test to assert navigation/modal.

---

## Task 1: `lib/activity-colors.ts` — the activity-color resolver

**Files:**
- Create: `lib/activity-colors.ts`
- Test: `lib/activity-colors.test.ts`

Context: This is the single source of truth the rest of the plan routes through. `Discipline` (`"run" | "lift" | "mobility" | "core"`) is already defined in `components/calendar/derivations.ts` and `disciplineOf(event)` already maps any `CalendarEvent` to it. Only `run`/`lift` are live today; `mobility`/`core` are reserved (tokens exist in `globals.css` but no data produces them) and must fall back to a neutral token set. The CSS variables already exist in `app/globals.css`: `--discipline-run-{bg,fg,dot}`, `--discipline-lift-{bg,fg,dot}`, and neutral `--border`, `--surface-2`, `--muted`.

- [ ] **Step 1: Write the failing test**

```ts
// lib/activity-colors.test.ts
/// <reference types="vitest/globals" />
import { activityColors, activityRingClass, ACTIVITY_COLORS } from "./activity-colors";

describe("activityColors", () => {
  it("resolves run to the run discipline tokens", () => {
    expect(activityColors("run")).toEqual({
      dot: "var(--discipline-run-dot)",
      bg: "var(--discipline-run-bg)",
      fg: "var(--discipline-run-fg)",
    });
  });

  it("resolves lift to the lift discipline tokens", () => {
    expect(activityColors("lift")).toEqual({
      dot: "var(--discipline-lift-dot)",
      bg: "var(--discipline-lift-bg)",
      fg: "var(--discipline-lift-fg)",
    });
  });

  it("never resolves to the violet accent or a hardcoded emerald", () => {
    for (const type of ["run", "lift"] as const) {
      const tokens = activityColors(type);
      const joined = `${tokens.dot} ${tokens.bg} ${tokens.fg}`;
      expect(joined).not.toMatch(/accent/);
      expect(joined).not.toMatch(/emerald/);
    }
  });

  it("falls back to a neutral token set for reserved/unmapped disciplines", () => {
    const neutral = { dot: "var(--border)", bg: "var(--surface-2)", fg: "var(--muted)" };
    expect(activityColors("mobility")).toEqual(neutral);
    expect(activityColors("core")).toEqual(neutral);
  });

  it("exposes the static map as the present resolver source", () => {
    expect(ACTIVITY_COLORS.run).toBeDefined();
    expect(ACTIVITY_COLORS.lift).toBeDefined();
  });
});

describe("activityRingClass", () => {
  it("returns the discipline-toned focus ring class for run and lift", () => {
    expect(activityRingClass("run")).toBe("focus-visible:ring-[var(--discipline-run-dot)]");
    expect(activityRingClass("lift")).toBe("focus-visible:ring-[var(--discipline-lift-dot)]");
  });

  it("falls back to a neutral ring for reserved disciplines", () => {
    expect(activityRingClass("mobility")).toBe("focus-visible:ring-[var(--border)]");
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run lib/activity-colors.test.ts`
Expected: FAIL — `Cannot find module './activity-colors'`.

- [ ] **Step 3: Write the implementation**

```ts
// lib/activity-colors.ts
import type { Discipline } from "@/components/calendar/derivations";

/**
 * Single source of truth mapping an activity type to its theme color tokens.
 * The indirection is deliberate: today this is a static map onto the
 * `--discipline-*` CSS variables, but a future "let the user pick their
 * calendar colors" feature swaps the resolver's source (e.g. user-chosen
 * values) without touching any call site. Modeled on `lib/macro-colors.ts`.
 *
 * Tokens are raw `var(--…)` strings, consumed as inline `style` (fills, text,
 * rails). A Tailwind arbitrary-value class can't be built from a runtime
 * string — only inline style can carry a value chosen at runtime — so inline
 * style is what keeps the future user-color swap a one-place change.
 *
 * `disciplineOf` (the one place event → type is decided) returns the full
 * `Discipline` union; only `run`/`lift` are live today. Reserved disciplines
 * (`mobility`/`core`) and any unmapped value fall back to a neutral set.
 */
export type ActivityColorTokens = { dot: string; bg: string; fg: string };

/** The activity types with a live color mapping today. */
export type ActivityType = Extract<Discipline, "run" | "lift">;

const NEUTRAL: ActivityColorTokens = {
  dot: "var(--border)",
  bg: "var(--surface-2)",
  fg: "var(--muted)",
};

export const ACTIVITY_COLORS: Record<ActivityType, ActivityColorTokens> = {
  run: { dot: "var(--discipline-run-dot)", bg: "var(--discipline-run-bg)", fg: "var(--discipline-run-fg)" },
  lift: { dot: "var(--discipline-lift-dot)", bg: "var(--discipline-lift-bg)", fg: "var(--discipline-lift-fg)" },
};

/** Resolve an activity type to its color tokens; neutral for reserved/unmapped. */
export function activityColors(type: Discipline): ActivityColorTokens {
  return (ACTIVITY_COLORS as Partial<Record<Discipline, ActivityColorTokens>>)[type] ?? NEUTRAL;
}

/**
 * The discipline-toned focus-ring class. A `:focus-visible` ring can't be an
 * inline style, so this is a literal Tailwind class — kept here so the ring
 * color stays sourced from the same module as the fills. Literal strings (not
 * interpolated) so Tailwind's scanner generates the utilities.
 */
const ACTIVITY_RING: Record<ActivityType, string> = {
  run: "focus-visible:ring-[var(--discipline-run-dot)]",
  lift: "focus-visible:ring-[var(--discipline-lift-dot)]",
};

export function activityRingClass(type: Discipline): string {
  return (ACTIVITY_RING as Partial<Record<Discipline, string>>)[type] ?? "focus-visible:ring-[var(--border)]";
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run lib/activity-colors.test.ts`
Expected: PASS (all cases).

- [ ] **Step 5: Typecheck**

Run: `npm run typecheck`
Expected: no errors.

- [ ] **Step 6: Commit**

```bash
git add lib/activity-colors.ts lib/activity-colors.test.ts
git commit -m "feat(calendar): add activity-colors resolver as single color source"
```

---

## Task 2: `day-cell.tsx` — pills source color from the resolver; planned → modal, logged → navigate

**Files:**
- Modify: `components/calendar/day-cell.tsx`
- Test: `components/calendar/day-cell.test.tsx`

Context: `DayCell` renders the month-grid pills. Today each pill's color comes from inline `var(--discipline-*)` Tailwind classes via `doneToneClasses`, and `PlannedPill` has a hardcoded `border-emerald-500/70 text-emerald-300` completed branch. Click handlers currently all "select the day (+ expand)". After this task: every pill color comes from `activityColors(...)` applied as inline `style`; the emerald completed branch is gone (completed = type color + check); a planned pill opens the read-only modal via a new `onOpenPlanned(plan)` prop; logged + completed-planned pills navigate via renamed `onNavigateWorkout(id)` / `onNavigateRun(id)` props; `+N more` keeps calling `onSelectDay`.

**Prop signature change** (DayCell):
- Remove: `onSelectWorkout`, `onSelectRun`, `onSelectPlanned`.
- Add: `onNavigateWorkout: (id: string) => void`, `onNavigateRun: (id: string) => void`, `onOpenPlanned: (plan: PlannedWorkout) => void`.
- Keep: `onSelectDay`.

The color cues: pill fill `bg`, text `fg`, left-bar/border `dot` — all from `activityColors(discipline)`. Status still drives shape only: `WorkoutPill`/`RunPill` (logged) are solid fills; `PlannedPill` is dashed outline + clock (planned) / check (completed) / strikethrough (skipped); `CompletedPlannedPill` is a solid fill + check.

- [ ] **Step 1: Update tests first (TDD) — color via style + new routing**

Replace the body of `components/calendar/day-cell.test.tsx` with the version below. Key changes vs today: render with the new prop names; assert pill color via inline `style` (jsdom preserves the raw `var(--…)` string) instead of className; assert planned-pill click fires `onOpenPlanned` with the plan, logged-pill click fires `onNavigateWorkout`/`onNavigateRun`, and the completed planned branch is no longer emerald.

```tsx
import { render, screen, fireEvent } from "@testing-library/react";
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
function plan(id: string, name: string, overrides: Partial<PlannedWorkout> = {}): PlannedWorkout {
  return {
    id,
    name,
    activity_kind: "lift",
    scheduled_start: "2026-06-16T17:00:00Z",
    status: "planned",
    google_sync_status: null,
    ...overrides,
  } as unknown as PlannedWorkout;
}

function noop() {}

function renderCell(
  events: CalendarEvent[],
  overrides: Partial<React.ComponentProps<typeof DayCell>> = {},
) {
  return render(
    <DayCell
      day={DAY}
      inMonth
      isToday={false}
      isSelected={false}
      events={events}
      onSelectDay={noop}
      onNavigateWorkout={noop}
      onNavigateRun={noop}
      onOpenPlanned={noop}
      {...overrides}
    />,
  );
}

describe("DayCell", () => {
  it("renders the date number", () => {
    renderCell([]);
    expect(screen.getByText("16")).toBeInTheDocument();
  });

  it("renders a done lift chip and navigates on click", () => {
    const onNavigateWorkout = vi.fn();
    renderCell([{ kind: "workout", startMs: 1, workout: workout("w1", "Upper 1") }], {
      onNavigateWorkout,
    });
    fireEvent.click(screen.getByRole("button", { name: "Upper 1" }));
    expect(onNavigateWorkout).toHaveBeenCalledWith("w1");
  });

  it("renders a run chip and navigates on click", () => {
    const onNavigateRun = vi.fn();
    renderCell([{ kind: "run", startMs: 1, run: run("r1", "Morning Run") }], { onNavigateRun });
    fireEvent.click(screen.getByRole("button", { name: "Morning Run" }));
    expect(onNavigateRun).toHaveBeenCalledWith("r1");
  });

  it("opens the planned modal (not navigate) when a planned pill is clicked", () => {
    const onOpenPlanned = vi.fn();
    const p = plan("p1", "Planned Lift");
    renderCell([{ kind: "planned", startMs: 1, planned: p }], { onOpenPlanned });
    fireEvent.click(screen.getByTestId("planned-pill"));
    expect(onOpenPlanned).toHaveBeenCalledWith(p);
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

  it("tones run and lift chips with their activity-type tokens (via style)", () => {
    renderCell([
      { kind: "workout", startMs: 1, workout: workout("w1", "Lift") },
      { kind: "run", startMs: 2, run: run("r1", "Run") },
    ]);
    expect(screen.getByRole("button", { name: "Lift" }).style.backgroundColor).toContain(
      "discipline-lift-bg",
    );
    expect(screen.getByRole("button", { name: "Run" }).style.backgroundColor).toContain(
      "discipline-run-bg",
    );
  });

  it("colors a completed planned pill in its activity type, not emerald", () => {
    const p = plan("p1", "Done Lift", { status: "completed" });
    renderCell([{ kind: "planned", startMs: 1, planned: p }]);
    const pill = screen.getByTestId("planned-pill");
    // Type color (lift), not green.
    expect(pill.style.color).toContain("discipline-lift-fg");
    expect(pill.style.borderColor).toContain("discipline-lift-dot");
    expect(pill.className).not.toMatch(/emerald/);
    expect(pill.style.color).not.toContain("emerald");
  });

  it("mutes and strikes a skipped planned pill", () => {
    const p = plan("p1", "Missed Lift", { status: "skipped" });
    renderCell([{ kind: "planned", startMs: 1, planned: p }]);
    const pill = screen.getByTestId("planned-pill");
    expect(pill.className).toMatch(/line-through/);
  });

  it("rolls events beyond the visible max into a +N more control", () => {
    const onSelectDay = vi.fn();
    renderCell(
      [
        { kind: "workout", startMs: 1, workout: workout("w1", "A") },
        { kind: "workout", startMs: 2, workout: workout("w2", "B") },
        { kind: "workout", startMs: 3, workout: workout("w3", "C") },
        { kind: "workout", startMs: 4, workout: workout("w4", "D") },
      ],
      { onSelectDay },
    );
    fireEvent.click(screen.getByRole("button", { name: /\+1 more/ }));
    expect(onSelectDay).toHaveBeenCalled();
  });

  it("exposes an accessible date+activity label", () => {
    renderCell([{ kind: "run", startMs: 1, run: run("r1", "Run") }]);
    expect(screen.getByLabelText(/Tuesday, June 16, 2026, 1 run/)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run components/calendar/day-cell.test.tsx`
Expected: FAIL — props renamed / colors still on className not style / emerald still present.

- [ ] **Step 3: Implement `day-cell.tsx`**

Make these edits:

(a) Add the import at the top (after the `derivations` import):

```ts
import { activityColors } from "@/lib/activity-colors";
```

(b) Remove the `doneToneClasses` helper entirely. The shared `CHIP_BASE` stays but drop any color from it (it currently has none beyond layout — leave layout classes). Keep `CHIP_BASE` as is (no color classes are in it).

(c) Change the prop signature of `DayCell` from:

```ts
  onSelectWorkout,
  onSelectRun,
  onSelectPlanned,
}: {
  …
  onSelectWorkout: (id: string) => void;
  onSelectRun: (id: string) => void;
  onSelectPlanned: (id: string) => void;
}) {
```

to:

```ts
  onNavigateWorkout,
  onNavigateRun,
  onOpenPlanned,
}: {
  …
  onNavigateWorkout: (id: string) => void;
  onNavigateRun: (id: string) => void;
  onOpenPlanned: (plan: PlannedWorkout) => void;
}) {
```

(d) Update the pill render switch in `DayCell`'s JSX:
- `WorkoutPill … onClick={() => onSelectWorkout(ev.workout.id)}` → `onClick={() => onNavigateWorkout(ev.workout.id)}`
- `RunPill … onClick={() => onSelectRun(ev.run.id)}` → `onClick={() => onNavigateRun(ev.run.id)}`
- `CompletedPlannedPill` onClick: `ev.logged.kind === "workout" ? onSelectWorkout(ev.logged.workout.id) : onSelectRun(ev.logged.run.id)` → use `onNavigateWorkout` / `onNavigateRun`.
- `PlannedPill … discipline={disciplineOf(ev)} onClick={() => onSelectPlanned(ev.planned.id)}` → `onClick={() => onOpenPlanned(ev.planned)}`.

(e) `WorkoutPill` — replace `className={`${CHIP_BASE} ${doneToneClasses("lift")}`}` with the base class plus inline style. The chip is a solid fill in the lift tone:

```tsx
function WorkoutPill({ workout, onClick }: { workout: Workout; onClick: () => void }) {
  const time = new Date(workout.performed_at).toLocaleTimeString("en-US", {
    hour: "numeric",
    minute: "2-digit",
  });
  const named = hasMeaningfulName(workout.name);
  const label = named ? (workout.name as string) : time;
  const c = activityColors("lift");
  return (
    <button
      type="button"
      onClick={(e) => {
        e.stopPropagation();
        onClick();
      }}
      title={named ? `${time} · ${workout.name}` : time}
      className={`${CHIP_BASE} hover:opacity-90`}
      style={{ backgroundColor: c.bg, color: c.fg, borderColor: c.dot }}
    >
      <CheckGlyph />
      <span className="truncate">{label}</span>
    </button>
  );
}
```

(f) `RunPill` — same shape, run tone:

```tsx
function RunPill({ run, onClick }: { run: RunningSession; onClick: () => void }) {
  const time = new Date(run.start_time).toLocaleTimeString("en-US", {
    hour: "numeric",
    minute: "2-digit",
  });
  const label = run.name?.trim() ? run.name : time;
  const c = activityColors("run");
  return (
    <button
      type="button"
      onClick={(e) => {
        e.stopPropagation();
        onClick();
      }}
      title={`${time} · Run${run.name ? ` · ${run.name}` : ""}`}
      className={`${CHIP_BASE} hover:opacity-90`}
      style={{ backgroundColor: c.bg, color: c.fg, borderColor: c.dot }}
    >
      <CheckGlyph />
      <span className="truncate">{label}</span>
    </button>
  );
}
```

(g) `PlannedPill` — retire the emerald completed branch. Color (border + text) is always the activity-type tone from `activityColors(discipline)`; status drives only shape (dashed always), glyph (clock planned / check completed / none skipped), and skipped's strikethrough + muting. Note: dashed pills have transparent fill, so apply only `color` + `borderColor` via style; skipped overrides to muted/neutral.

```tsx
function PlannedPill({
  planned,
  discipline,
  onClick,
}: {
  planned: PlannedWorkout;
  discipline: Discipline;
  onClick: () => void;
}) {
  const time = new Date(planned.scheduled_start).toLocaleTimeString("en-US", {
    hour: "numeric",
    minute: "2-digit",
  });
  const label = planned.name?.trim() ? planned.name : time;
  const completed = planned.status === "completed";
  const skipped = planned.status === "skipped";
  const synced = planned.google_sync_status === "synced";
  const c = activityColors(discipline);

  // Activity type owns the color. Status is shape + glyph only: dashed
  // outline marks "planned/forward-looking"; a check marks completed; a
  // skipped plan is muted + struck through (a deliberate de-emphasis, the
  // one place the type color yields — to "missed", not to another status
  // color).
  const style: React.CSSProperties = skipped
    ? { color: "var(--muted)", borderColor: "var(--border)" }
    : { color: c.fg, borderColor: c.dot };

  return (
    <button
      type="button"
      data-testid="planned-pill"
      onClick={(e) => {
        e.stopPropagation();
        onClick();
      }}
      title={`${time} · Planned${planned.name ? ` · ${planned.name}` : ""}${
        completed ? " (completed)" : skipped ? " (skipped)" : ""
      }`}
      className={`${CHIP_BASE} border border-dashed bg-transparent hover:bg-[var(--surface-2)] ${
        skipped ? "line-through" : ""
      }`}
      style={style}
    >
      {completed ? <CheckGlyph /> : skipped ? null : <ClockGlyph />}
      <span className="truncate">{label}</span>
      {synced && <SyncGlyph />}
    </button>
  );
}
```

(h) `CompletedPlannedPill` — replace `const tone = doneToneClasses(disciplineOf(event));` and `className={`${CHIP_BASE} ${tone}`}` with inline style from the resolver (solid fill in the logged discipline's tone):

```tsx
  const c = activityColors(disciplineOf(event));
  return (
    <button
      type="button"
      data-testid="completed-planned-pill"
      onClick={(e) => {
        e.stopPropagation();
        onClick();
      }}
      title={`${time} · ${label} (completed planned ${isRun ? "run" : "workout"})`}
      className={`${CHIP_BASE} hover:opacity-90`}
      style={{ backgroundColor: c.bg, color: c.fg, borderColor: c.dot }}
    >
      <CheckGlyph />
      <span className="truncate">{label}</span>
    </button>
  );
```

Add `import type React from "react";` only if needed for `React.CSSProperties` — this file already has `"use client"` and JSX; `React.CSSProperties` resolves via the existing React types without an explicit import in this codebase's tsconfig (verify with typecheck; if it errors, add `import type { CSSProperties } from "react";` and use `CSSProperties`).

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npx vitest run components/calendar/day-cell.test.tsx`
Expected: PASS.

- [ ] **Step 5: Typecheck + lint**

Run: `npm run typecheck && npm run lint`
Expected: no errors. (Note: `page.tsx` still passes the OLD prop names to `DayCell` at this point and WILL fail typecheck — that's expected and fixed in Task 4. If typecheck fails ONLY on `app/(app)/calendar/page.tsx` prop names, that is acceptable for this task's commit; note it in your report. Lint of the touched file should be clean.)

- [ ] **Step 6: Commit**

```bash
git add components/calendar/day-cell.tsx components/calendar/day-cell.test.tsx
git commit -m "feat(calendar): route grid pills through activity-colors; planned pill opens modal"
```

---

## Task 3: Timeline banners — rail + ring from the resolver; retire emerald/violet-as-status

**Files:**
- Modify: `components/calendar/workout-banner.tsx`
- Modify: `components/calendar/run-banner.tsx`
- Modify: `components/calendar/planned-banner.tsx`
- Modify: `components/calendar/completed-planned-banner.tsx`
- Test: `components/calendar/planned-banner.test.tsx` (extend), `components/calendar/completed-planned-banner.test.tsx` (create)

Context: The four banner rows in the day digest each draw a left accent rail and a focus ring. `workout-banner`/`run-banner` already use `--discipline-lift-dot`/`--discipline-run-dot` inline — route them through the resolver. `planned-banner` colors the rail by **status** (`--accent` planned / `emerald-500` completed / `--muted` skipped) and uses a violet dashed border + violet/emerald badges — change the rail to the **activity-type** color and de-violet/de-emerald the status cues (status = dashed shape + neutral badge + strikethrough). `completed-planned-banner` uses an always-`emerald-500` rail, emerald badge, and emerald check — change rail to the **logged** discipline's type color and make the completion cue neutral (check glyph + "Completed" text), not green.

Each banner's discipline is statically known: workout → `"lift"`, run → `"run"`, planned → from `planned.activity_kind`, completed-planned → from `logged.kind`. The rail is a `<span>` background (inline `style={{ backgroundColor: activityColors(d).dot }}`); the focus ring is `activityRingClass(d)` (literal class). Keep `focus-visible:ring-2 focus-visible:ring-inset` width/position utilities already present.

- [ ] **Step 1: workout-banner.tsx**

Add import:
```ts
import { activityColors, activityRingClass } from "@/lib/activity-colors";
```
Inside the component, before `return`:
```ts
  const c = activityColors("lift");
  const ring = activityRingClass("lift");
```
Replace the body `<button>` className's literal `focus-visible:ring-[var(--discipline-lift-dot)]` with `${ring}`, the chevron `<button>` className's `focus-visible:ring-[var(--discipline-lift-dot)]` with `${ring}`, and the rail `<span>`:
```tsx
          <span
            aria-hidden="true"
            className="h-7 w-1 shrink-0 rounded-full md:h-8 md:w-1.5"
            style={{ backgroundColor: c.dot }}
          />
```
(Both buttons use template literals; e.g. `className={`flex … focus-visible:ring-2 ${ring} focus-visible:ring-inset md:gap-3 …`}`.)

- [ ] **Step 2: run-banner.tsx**

Same as workout-banner but with `activityColors("run")` / `activityRingClass("run")`, replacing the two `focus-visible:ring-[var(--discipline-run-dot)]` occurrences and the rail `<span>`'s `bg-[var(--discipline-run-dot)]` with `style={{ backgroundColor: c.dot }}`.

- [ ] **Step 3: planned-banner.tsx — rail = type color, de-violet/de-emerald status**

Add import:
```ts
import { activityColors, activityRingClass } from "@/lib/activity-colors";
```
Compute discipline + tokens near the top of `PlannedBanner` (after `const isRun = …`):
```ts
  const discipline = isRun ? "run" : "lift";
  const c = activityColors(discipline);
  const ring = activityRingClass(discipline);
```
Remove the status-driven `rail` const:
```ts
  // DELETE: const rail = completed ? "bg-emerald-500" : skipped ? "bg-[var(--muted)]" : "bg-[var(--accent)]";
```
Container `<div>` className — change the dashed border from the violet accent to a neutral hairline (status is the dashed *shape*, not a color):
```tsx
    className="overflow-hidden rounded-2xl border border-dashed border-[var(--border)] bg-[var(--surface)]"
```
Body `<button>` — replace `focus-visible:ring-[var(--accent)]` with `${ring}` (keep `focus-visible:ring-2 … focus-visible:ring-inset`).
Rail `<span>` — replace `${rail}` class with the type color via style:
```tsx
          <span
            aria-hidden="true"
            className="h-7 w-1 shrink-0 rounded-full md:h-8 md:w-1.5"
            style={{ backgroundColor: skipped ? "var(--muted)" : c.dot }}
          />
```
(Skipped rail mutes — consistent with the skipped pill de-emphasis. Completed and planned rails are the activity-type color.)

`StatusBadge` — de-violet/de-emerald. Replace the `map` with neutral, shape-carried badges (the word + the container's dashed/solid shape carry status):
```tsx
function StatusBadge({ status }: { status: PlannedWorkout["status"] }) {
  const map = {
    planned: { label: "Planned", cls: "border-[var(--border)] text-[var(--muted)]" },
    completed: { label: "Completed", cls: "border-[var(--border)] text-[var(--foreground)]" },
    skipped: { label: "Skipped", cls: "border-[var(--border)] text-[var(--muted)]" },
  } as const;
  const { label, cls } = map[status];
  return (
    <span
      className={`shrink-0 rounded-full border px-1.5 py-px text-[9px] font-semibold uppercase tracking-wide ${cls}`}
    >
      {label}
    </span>
  );
}
```

- [ ] **Step 4: completed-planned-banner.tsx — rail/check/badge to type color/neutral**

Add import:
```ts
import { activityColors, activityRingClass } from "@/lib/activity-colors";
```
Compute discipline + tokens (replace the existing `ringColor` block):
```ts
  const discipline = isRun ? "run" : "lift";
  const c = activityColors(discipline);
  const ring = activityRingClass(discipline);
```
Body `<button>` and chevron `<button>` classNames — replace `${ringColor}` with `${ring}` (these already have `focus-visible:ring-2 … focus-visible:ring-inset`).
Rail `<span>` — replace `bg-emerald-500` with the type color and update the comment:
```tsx
          {/* The rail is the logged session's activity-type color — a
              completed plan reads as the activity it was, with the check +
              "Completed" badge (not green) marking that it closed out a plan. */}
          <span
            aria-hidden="true"
            className="h-7 w-1 shrink-0 rounded-full md:h-8 md:w-1.5"
            style={{ backgroundColor: c.dot }}
          />
```
"Completed" badge `<span>` — de-emerald to neutral:
```tsx
              <span className="shrink-0 rounded-full border border-[var(--border)] px-1.5 py-px text-[9px] font-semibold uppercase tracking-wide text-[var(--foreground)]">
                Completed
              </span>
```
`CheckGlyph` — drop `text-emerald-500`, make it muted so the glyph reads as a quiet completion mark:
```tsx
      className="shrink-0 text-[var(--muted)]"
```

- [ ] **Step 5: Extend `planned-banner.test.tsx`**

Add a new `describe` block asserting the rail is the activity-type color and that no violet/emerald is used for the planned/completed rail. Append before the final closing of the file:

```tsx
describe("PlannedBanner — activity-type color (not status)", () => {
  it("colors a planned run's rail with the run token, not the violet accent", () => {
    const { container } = render(
      <PlannedBanner planned={makePlanned({ activity_kind: "run" })} />,
    );
    const rail = container.querySelector('span[aria-hidden="true"]') as HTMLElement;
    expect(rail.style.backgroundColor).toContain("discipline-run-dot");
    expect(rail.style.backgroundColor).not.toContain("accent");
  });

  it("colors a planned lift's rail with the lift token", () => {
    const { container } = render(
      <PlannedBanner planned={makePlanned({ activity_kind: "lift" })} />,
    );
    const rail = container.querySelector('span[aria-hidden="true"]') as HTMLElement;
    expect(rail.style.backgroundColor).toContain("discipline-lift-dot");
  });

  it("keeps a completed plan's rail in its activity type, not emerald", () => {
    const { container } = render(
      <PlannedBanner
        planned={makePlanned({ activity_kind: "lift", status: "completed" })}
      />,
    );
    const rail = container.querySelector('span[aria-hidden="true"]') as HTMLElement;
    expect(rail.style.backgroundColor).toContain("discipline-lift-dot");
    expect(rail.style.backgroundColor).not.toContain("emerald");
    expect(rail.className).not.toMatch(/emerald/);
  });

  it("mutes a skipped plan's rail", () => {
    const { container } = render(
      <PlannedBanner
        planned={makePlanned({ activity_kind: "run", status: "skipped" })}
      />,
    );
    const rail = container.querySelector('span[aria-hidden="true"]') as HTMLElement;
    expect(rail.style.backgroundColor).toContain("muted");
  });
});
```

- [ ] **Step 6: Create `completed-planned-banner.test.tsx`**

```tsx
/// <reference types="vitest/globals" />
import { render, screen } from "@testing-library/react";
import { DistanceUnitProvider } from "@/lib/distance-unit-context";
import type { Exercise, PlannedWorkout, RunningSession, Workout } from "@/lib/api";
import { CompletedPlannedBanner } from "./completed-planned-banner";

function makePlanned(overrides: Partial<PlannedWorkout> = {}): PlannedWorkout {
  return {
    id: "p-1",
    name: "W7 D1 - Easy Run",
    activity_kind: "run",
    scheduled_start: "2026-06-20T12:00:00Z",
    scheduled_end: "2026-06-20T13:00:00Z",
    timezone: "America/Denver",
    status: "completed",
    notes: null,
    completed_session_id: "act1",
    completed_session_kind: "activity",
    calendar_detail: null,
    google_event_id: null,
    last_sync_error: null,
    google_sync_status: null,
    run_type: "easy",
    run_details: null,
    exercises: [],
    created_at: "2026-06-19T12:00:00Z",
    updated_at: "2026-06-19T12:00:00Z",
    ...overrides,
  };
}
function makeRun(overrides: Partial<RunningSession> = {}): RunningSession {
  return {
    id: "run-1",
    name: "Tempo Run",
    start_time: "2026-06-20T12:00:00Z",
    distance_meters: 8046.72,
    duration_seconds: 2520,
    avg_pace_sec_per_km: 313,
    ...overrides,
  } as unknown as RunningSession;
}
function makeWorkout(overrides: Partial<Workout> = {}): Workout {
  return {
    id: "w-1",
    name: "Upper 1",
    performed_at: "2026-06-20T17:00:00Z",
    exercises: [],
    ...overrides,
  } as unknown as Workout;
}

function renderBanner(
  logged:
    | { kind: "workout"; workout: Workout }
    | { kind: "run"; run: RunningSession },
  planned = makePlanned(),
) {
  return render(
    <DistanceUnitProvider>
      <CompletedPlannedBanner
        planned={planned}
        logged={logged}
        exerciseMap={new Map<string, Exercise>()}
        onNavigate={() => {}}
      />
    </DistanceUnitProvider>,
  );
}

describe("CompletedPlannedBanner", () => {
  it("colors the rail with the logged run's type color, not emerald", () => {
    const { container } = renderBanner({ kind: "run", run: makeRun() });
    const rail = container.querySelector('span[aria-hidden="true"]') as HTMLElement;
    expect(rail.style.backgroundColor).toContain("discipline-run-dot");
    expect(rail.style.backgroundColor).not.toContain("emerald");
    expect(rail.className).not.toMatch(/emerald/);
  });

  it("colors the rail with the logged workout's lift color when a plan was fulfilled by a lift", () => {
    const { container } = renderBanner({ kind: "workout", workout: makeWorkout() });
    const rail = container.querySelector('span[aria-hidden="true"]') as HTMLElement;
    expect(rail.style.backgroundColor).toContain("discipline-lift-dot");
  });

  it("uses no emerald anywhere (completion is the check + badge, not green)", () => {
    const { container } = renderBanner({ kind: "run", run: makeRun() });
    expect(container.innerHTML).not.toMatch(/emerald/);
    expect(screen.getByText("Completed")).toBeInTheDocument();
  });
});
```

- [ ] **Step 7: Run banner tests**

Run: `npx vitest run components/calendar/planned-banner.test.tsx components/calendar/completed-planned-banner.test.tsx components/calendar/workout-banner.test.tsx components/calendar/run-banner.test.tsx`
Expected: PASS.

- [ ] **Step 8: Typecheck + lint touched files**

Run: `npm run typecheck && npm run lint`
Expected: no errors (page.tsx prop mismatch from Task 2 may still surface here — fixed in Task 4; note if so).

- [ ] **Step 9: Commit**

```bash
git add components/calendar/workout-banner.tsx components/calendar/run-banner.tsx components/calendar/planned-banner.tsx components/calendar/planned-banner.test.tsx components/calendar/completed-planned-banner.tsx components/calendar/completed-planned-banner.test.tsx
git commit -m "feat(calendar): timeline banners use activity-type color, retire emerald/violet status"
```

---

## Task 4: `page.tsx` + `day-digest.tsx` — wire new routing, remove dead auto-expand

**Files:**
- Modify: `app/(app)/calendar/page.tsx`
- Modify: `components/calendar/day-digest.tsx`
- Test: `app/(app)/calendar/page.test.tsx`

Context: With logged pills navigating and planned pills opening the modal (Task 2), the old grid-pill "select day + auto-expand the matching banner" path is dead. This task wires `DayCell`'s new props in `page.tsx` and removes the now-unused `autoExpandId` state, `selectDayAndExpand`, and the `autoExpandId` threading through `DayDigest` and the banner keys. The timeline banners keep their own manual expand chevrons (their `defaultOpen` prop defaults to `false`); `+N more` and empty-cell clicks still call `selectDay` (select + scroll into view).

- [ ] **Step 1: Update `page.tsx`**

(a) Remove the `autoExpandId` state declaration:
```ts
  // DELETE: const [autoExpandId, setAutoExpandId] = useState<string | null>(null);
```
(b) In the cursor-change effect, remove `setAutoExpandId(null);` (the `setSelected(...)` line stays).
(c) Replace `selectDay` + `selectDayAndExpand` with a single `selectDay` that no longer touches auto-expand:
```ts
  const selectDay = (key: string) => {
    setSelected(key);
    // defer so the digest has re-rendered for the new day before scrolling
    requestAnimationFrame(() =>
      digestRef.current?.scrollIntoView({ behavior: "smooth", block: "nearest" }),
    );
  };
```
(Delete the entire `selectDayAndExpand` function.)
(d) Add a handler that opens a plan's read-only modal and selects its day:
```ts
  // A grid planned pill opens the read-only modal for that plan directly,
  // selecting its day as a side effect (the modal is the detail surface).
  const openPlannedForDay = (key: string, plan: PlannedWorkout) => {
    setSelected(key);
    openViewPlan(plan);
  };
```
(e) Update the `DayCell` render props (in the week-row map):
```tsx
                    <DayCell
                      key={key}
                      day={day}
                      inMonth={inMonth}
                      isToday={isToday}
                      isSelected={isSelected}
                      events={dayEvents}
                      onSelectDay={() => selectDay(key)}
                      onNavigateWorkout={(id) => router.push(`/workouts/${id}`)}
                      onNavigateRun={(id) => router.push(`/running/${id}`)}
                      onOpenPlanned={(plan) => openPlannedForDay(key, plan)}
                    />
```
(f) In the `<DayDigest … />` render, remove the `autoExpandId={autoExpandId}` prop line. Leave all other props (`onNavigateWorkout`, `onNavigateRun`, `onPlanWorkout`, `onOpenPlanned`, `onResyncPlanned`, `onNavigateSession`) unchanged.

- [ ] **Step 2: Update `day-digest.tsx`**

(a) Remove the `autoExpandId` prop from the destructure and the type:
```ts
  // DELETE from params:   autoExpandId,
  // DELETE from type:     autoExpandId?: string | null;
```
(b) Remove the now-unused doc-comment line about `autoExpandId` (the prop's `//` comment).
(c) `completed-planned` branch — drop the open/closed key suffix and `defaultOpen` (it can no longer auto-open from the grid). Simplify the IIFE to a stable key:
```tsx
              ev.kind === "completed-planned" ? (
                <CompletedPlannedBanner
                  key={`cp-${ev.planned.id}`}
                  planned={ev.planned}
                  logged={ev.logged}
                  exerciseMap={exerciseMap}
                  onNavigate={() =>
                    ev.logged.kind === "workout"
                      ? onNavigateWorkout(ev.logged.workout.id)
                      : onNavigateRun(ev.logged.run.id)
                  }
                />
              ) : ev.kind === "planned" ? (
```
(d) `workout` branch — stable key, drop `defaultOpen`:
```tsx
              ) : ev.kind === "workout" ? (
                <WorkoutBanner
                  key={`w-${ev.workout.id}`}
                  workout={ev.workout}
                  exerciseMap={exerciseMap}
                  onNavigate={() => onNavigateWorkout(ev.workout.id)}
                />
              ) : (
```
(e) `run` branch — stable key, drop `defaultOpen`:
```tsx
                <RunBanner
                  key={`r-${ev.run.id}`}
                  run={ev.run}
                  onNavigate={() => onNavigateRun(ev.run.id)}
                />
              ),
```
(Leave the `planned` branch as-is; it already takes no auto-expand.)

Note: the `defaultOpen` prop stays defined on `WorkoutBanner`/`RunBanner`/`CompletedPlannedBanner` (a legitimate optional with a `false` default); we simply stop passing it. Do not remove it from the banner components.

- [ ] **Step 3: Update `page.test.tsx`**

Replace the test `"auto-expands the digest banner when a grid pill is clicked"` with a test that a logged grid pill **navigates**, and add one that a planned grid pill **opens the modal**. The `pushMock` is already hoisted (line 11) and `useRouter` returns `{ push: pushMock }` (line 19).

Find the existing `it("auto-expands the digest banner when a grid pill is clicked", …)` block and replace it with:

```tsx
  it("navigates to the session detail when a logged grid pill is clicked", async () => {
    renderPage();
    await findDigest(TODAY);

    const cell = screen.getByLabelText(new RegExp(`^${longDate(DISTINCT_DATE)}`));
    const pill = await within(cell).findByRole("button", { name: "Midmonth Lift" });
    fireEvent.click(pill);

    expect(pushMock).toHaveBeenCalledWith("/workouts/w-distinct");
  });

  it("opens the read-only planned modal when a planned grid pill is clicked", async () => {
    renderPage();
    await findDigest(TODAY);

    const cell = screen.getByLabelText(new RegExp(`^${longDate(DISTINCT_DATE)}`));
    const pill = await within(cell).findByTestId("planned-pill");
    fireEvent.click(pill);

    // The planned-workout modal opens in read-only view for that plan.
    expect(await screen.findByRole("dialog")).toBeInTheDocument();
    expect(screen.getByLabelText("Edit planned workout")).toBeInTheDocument();
  });
```

Verify the IDs/names used: confirm the logged workout on `DISTINCT_DATE` has id `w-distinct` and name `"Midmonth Lift"`, and the planned workout on that day has `status: "planned"` (so the modal shows the pencil with `aria-label="Edit planned workout"`). Read the test fixtures near the top of `page.test.tsx` (the `WORKOUTS`/`PLANNED` consts and `makePlanned`) and adjust the expected id/name strings to match the actual fixtures. If the planned fixture's id/name differs, keep the structure and fix the literals. Do not weaken assertions to pass.

- [ ] **Step 4: Run the page + digest tests**

Run: `npx vitest run "app/(app)/calendar/page.test.tsx" components/calendar/day-digest.test.tsx`
Expected: PASS. If `day-digest.test.tsx` referenced `autoExpandId`, update it (it currently passes `onNavigateWorkout`/`onNavigateRun` and no `autoExpandId`, so it should be unaffected).

- [ ] **Step 5: Full gate**

Run: `npm run typecheck && npm run lint && npm run test`
Expected: all green. Then `npm run build`.
Run: `npm run build`
Expected: build succeeds.

- [ ] **Step 6: Commit**

```bash
git add "app/(app)/calendar/page.tsx" components/calendar/day-digest.tsx "app/(app)/calendar/page.test.tsx"
git commit -m "feat(calendar): logged pills navigate, planned pills open modal; drop dead auto-expand"
```

---

## Task 5: Verify the resolver is the only color source (cleanup sweep)

**Files:** (review only; fix if found)
- `components/calendar/day-cell.tsx`, `workout-banner.tsx`, `run-banner.tsx`, `planned-banner.tsx`, `completed-planned-banner.tsx`

Context: The SOW requires the resolver to be the single color source — no scattered inline `var(--discipline-*)` and no hardcoded `emerald-*` left in the calendar pill/banner color paths.

- [ ] **Step 1: Grep for stragglers**

Run:
```bash
grep -rn "emerald" components/calendar/ app/\(app\)/calendar/
grep -rn "discipline-run\|discipline-lift" components/calendar/day-cell.tsx components/calendar/workout-banner.tsx components/calendar/run-banner.tsx components/calendar/planned-banner.tsx components/calendar/completed-planned-banner.tsx
```
Expected after Tasks 2–3: **no `emerald` matches** in calendar components; the only `discipline-run`/`discipline-lift` matches are inside `lib/activity-colors.ts` (the resolver) and `lib/activity-colors.test.ts` / `*.test.tsx` assertions — **not** in the component color paths (component color now comes from `activityColors(...)`/`activityRingClass(...)`). The `activityRingClass` literals (`focus-visible:ring-[var(--discipline-*-dot)]`) live in `lib/activity-colors.ts`, which is correct.

- [ ] **Step 2: If any straggler remains in a component**, route it through `activityColors`/`activityRingClass` the same way as the surrounding code, re-run that component's test, and commit:
```bash
git commit -am "refactor(calendar): route remaining discipline color through activity-colors"
```
If the grep is already clean, skip the commit and note "no stragglers" in your report.

- [ ] **Step 3: Final full gate**

Run: `npm run typecheck && npm run lint && npm run test && npm run build`
Expected: all green.

---

## Self-Review (controller checklist, run after all tasks)

1. **Spec coverage:**
   - `lib/activity-colors.ts` single source of truth → Task 1. ✅
   - All calendar color through it (grid pills + 4 banners), emerald removed → Tasks 2, 3, 5. ✅
   - Activity type owns timeline color (rail = type, not status) → Task 3. ✅
   - Status via shape + badge (dashed/solid/strikethrough + Planned/Completed/Skipped) → Tasks 2, 3. ✅
   - Planned pill opens read-only modal + selects day → Tasks 2, 4. ✅
   - `+N more` / empty cell select day + scroll timeline → unchanged `onSelectDay`/`selectDay`. ✅
   - Logged pills navigate to detail pages → Tasks 2, 4. ✅
   - Violet reserved for selection/today/focus (no longer "planned") → Task 3. ✅
   - Tests for resolver + parity + status + click routing → Tasks 1–4. ✅
   - CI green (lint/format/typecheck/test/build) → Tasks 4, 5. ✅
2. **Non-goals respected:** no backend change; no modal-form/month-grid redesign; no logged-session modal; no color-customization UI; no new activity types; no `--status-*` taxonomy.
3. **Type consistency:** `activityColors(type: Discipline)` accepts what `disciplineOf` returns; prop names `onNavigateWorkout`/`onNavigateRun`/`onOpenPlanned` are consistent across `DayCell` and `page.tsx`.

## Docs reconcile (handled on the `prog-strength-docs` branch, not here)

- Add the `--discipline-*` activity palette enumeration to `design-system.md` (the SOW's "small docs reconcile" — it currently describes activity tonal hues but doesn't list the tokens).
- Flip the SOW `status: shipped` + header per the workflow.
