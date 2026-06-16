---
type: sow
status: draft
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Calendar View Redesign — Warm-Organic Coaching Structure on the Dark Theme

**Status**: Draft · **Last updated**: 2026-06-16

## Introduction

The monthly training calendar (`/calendar`) is one of the most-visited surfaces in Prog Strength, and its functionality is solid — it correctly renders logged lifts, logged runs, planned sessions, and the merged "planned-then-completed" case; it rolls up weekly stats; the day digest works. What lets it down is the *look*: a low-contrast grid of small text pills with the weekly summary stranded in a narrow right-hand column. It reads like a generic admin calendar, not a training tool you're glad to open every morning.

To find a better direction without guessing, this surface went through a Design Exploration (`dx/calendar-view.md`, PR Prog-Strength/prog-strength-web#56), which put five distinct visual directions side by side. The owner picked **`warm-organic-coaching`** — the human, encouraging idiom: rounded cards, soft tonal activity hues, a per-week "streak strip" that frames the month as consistency ("you trained 4 of 7 days this week") rather than a wall of numbers, and a warm coaching tone up top.

This SOW implements that direction for real. One deliberate adaptation, decided at the selection gate: the chosen mockup is a warm **light** theme, but the rest of the app (sidebar, every other page) is dark, and shipping a light-themed island inside a dark app would read as broken. So this SOW **adopts the variant's structure, layout, and coaching feel, re-toned onto the existing dark palette** — not its light background. The user gets the rounded, encouraging, streak-forward calendar they picked, and the app stays visually coherent. A future, app-wide warmth shift (if ever wanted) is a separate design-system decision, explicitly out of scope here.

After this ships, the calendar is a stack of rounded per-week panels on the existing dark surface: each week is its own card holding seven day cells plus a streak-strip footer; activities are soft tonal chips (warm-but-dark hues for runs vs lifts) instead of flat pills; the header greets the user and summarizes the month's consistency; and the day digest below is restyled to match. Same data, same interactions, same correctness — a presentation the surface deserves.

## Proposed Solution

This is a **presentation-layer redesign of one surface**. No calendar data, API, routing, or business logic changes — the event model (`CalendarEvent`), the weekly rollups (`WeeklyStat`), the day-selection and plan-a-workout interactions, and the merge logic all stay exactly as they are. What changes is the layout and styling of `app/(app)/calendar/page.tsx` and the components under `components/calendar/`.

The structural move, taken from the chosen variant, is to **retire the "7 days + a narrow Week column" grid in favor of per-week rounded panels**. Today the calendar is one CSS grid of eight columns (seven days plus a `WeeklyTile`/`WeeklyChip`/`WeeklyOverviewColumn` summary column). The redesign makes each week a self-contained rounded panel: a seven-column day row on top, and a **streak-strip footer** beneath it that carries the same weekly data (sessions, lift time, run distance, steps) but reframed as a coaching line — seven dots for the seven days (filled when trained), "You trained N of M days" (or a gentle rest-week message when the week is empty), and compact metric labels.

Activities become **soft tonal chips**. The variant defines per-discipline tonal hues; re-toned for dark, runs and lifts each get a desaturated warm hue (extensible to mobility/core later, but only run and lift are produced today — the real data is lifts and runs). Completed sessions are filled tonal chips with a ✓; planned sessions keep their existing dashed/clock/recurring affordances, re-skinned. A **coaching header** greets the user by name and states the month's consistency, computed from real events — no fabricated copy.

The warm accent and tonal activity hues are introduced as **named design tokens** (CSS variables) in the app's existing token system, dark-appropriate, so they're reusable and a future broader adoption composes cleanly rather than hard-coding hex values into the calendar.

## Goals and Non-Goals

### Goals

- Reimplement `/calendar` to match the **`warm-organic-coaching`** variant's structure and feel: per-week rounded panels, a streak-strip footer per week, soft tonal activity chips, soft rounded hero-stat cards, and a coaching header.
- **Re-tone to the existing dark palette**, not the variant's light background, so the calendar stays coherent with the rest of the app.
- Preserve **all current functionality**: day selection, plan-a-workout (month-level and per-day), prev/next/Today navigation, the planned/clock/recurring affordances, the completed-planned merge, the day digest, and all routing.
- Reframe the weekly summary as a **streak strip** (seven trained/untrained dots + "trained N of M days" + lift/run/steps labels), carrying the same `WeeklyStat` data the right column carries today.
- A **coaching header**: greet by the user's real `display_name`, and summarize the month's training consistency from real event data.
- Map activity → **tonal hue by discipline** (run vs lift today), via an extensible `Discipline` system, drawing on the app's **shared design tokens** — the violet accent for emphasis plus the per-discipline tones on the slate ramp (see Design tokens). No separate warm/clay accent.
- Update the calendar component **tests** to the new structure; keep the suite green and CI's gate (`lint`, `format`, `typecheck`, `test`, `build`) passing.

### Non-Goals

- **App-wide re-theme.** The sidebar and every non-calendar page stay dark and unchanged. Whether warm-organic becomes the global direction is a separate design-system decision, not this SOW.
- **The warm *light* palette.** The selected variant's light background is intentionally not adopted (see Introduction). Structure and feel, yes; light theme, no.
- **New activity disciplines.** Only run and lift tones are produced. `mobility`/`core` exist in the type system as extensible slots but are not inferred from names today (that would mis-tag free-text workout names).
- **Any change to calendar data, API, rollup math, merge logic, or interactions.** This is presentation only. If a behavior changes, that's a regression, not a goal.
- **Promoting the DX mockup code.** The variant at `app/design-explore/calendar-view/_variants/warm-organic-coaching.tsx` is a reference spec built on simplified static fixtures (`CalEvent`, `Discipline`, `fixtures.ts`). It is the visual north star, **not** code to copy — reimplement against the real `CalendarEvent`/`WeeklyStat` types and real data. The `app/design-explore/` route is not touched and never ships.

## Implementation Details

All paths are in `prog-strength-web`. Lifecycle order: tokens → data derivations → components → page composition → tests.

### Design tokens (conform to the app's soft-modern-messenger system)

> **Palette update (2026-06-16).** This section originally introduced a warm **clay** accent on dark. Since then, [`sows/chat-and-app-shell-redesign.md`](chat-and-app-shell-redesign.md) shipped the **soft-modern-messenger** system as the app's foundational design tokens — a slate neutral ramp and a single **violet accent** (`#8b7cf6`, replacing the old blue) — which every page now inherits. To keep the app coherent (one accent, not two), the calendar **conforms to those shared tokens** rather than introducing clay. The *structure and coaching feel* (rounded per-week panels, the streak strip, the coaching header) are unchanged; only the palette is re-pointed.

The calendar draws its accent and surfaces from the **existing shared tokens** (in `app/globals.css`) — it adds no new accent:

- Use the shared **violet accent** (`var(--accent)` / `var(--accent-soft)` / `var(--accent-line)`) for actions, "today," and the streak emphasis — the same accent the shell and chat now use. Do **not** introduce a clay/warm accent; the app has one accent and it is violet.
- Panels, day cells, and chips sit on the shared **slate ramp** (`var(--surface)` / `var(--surface-2)` / `var(--surface-3)`, hairline `var(--border)`), `var(--radius-card)` / `var(--radius-card-lg)` radii, and `var(--shadow-soft)` depth.
- **Per-discipline tonal hues** for `run` and `lift` remain a `{ bg, fg, dot }` triple, but **re-toned to sit on the slate ramp** and hold WCAG-AA contrast for chip text. The app already ships dark-surface discipline tokens (`--discipline-run-*`, `--discipline-lift-*`, with reserved `--discipline-mobility-*` / `--discipline-core-*` slots) — reuse those rather than defining a parallel warm set. Keep `run` and `lift` visually distinct from the violet accent so activity chips don't read as "today"/action emphasis.

Hard-coded hex in the calendar components is a smell — reference the shared tokens.

### Data derivations (compute from real events; no fabrication)

The coaching copy and streak strip need values the page doesn't compute yet. Derive them from the day buckets the grid already builds (`eventsByDate`):

- **Discipline of an event** — a helper `disciplineOf(event): Discipline`. `run` and `completed-planned` whose `logged` is a run → `run`; `workout` and `completed-planned` whose `logged` is a workout → `lift`; `planned` → derive from the planned session's type (run vs lift). Centralize this so chips, dots, and any future surface agree.
- **Chip state** — `done` for `workout`/`run`/`completed-planned`; `planned` for `planned`. (Completed-planned is a "done" session — it already merged.)
- **Per-week trained days** — count of in-month days in that week carrying at least one `done` event. Drives the seven dots and "N of M."
- **Month consistency** — count of distinct in-month days with a `done` event, for the header line ("You've trained 9 days this month…"). Keep the copy templated and honest; a rest-heavy month should read encouragingly, not scold.

### Components

- **`components/calendar/weekly-overview.tsx`** — replace the right-column model. `WeeklyTile`, `WeeklyChip`, and `WeeklyOverviewColumn` (the desktop tile, mobile chip, and column) give way to a **`WeekStreakStrip`** component: the seven trained/untrained dots, the "trained N of M days" / rest-week line, and compact `🏋 lift · 🏃 run · 👟 steps` labels derived from `WeeklyStat`. Keep the `WeeklyStat` type and its formatting helpers (`formatTotalDuration`, step formatting) — only the presentation changes. (Emoji vs inline icon is a small call; match whatever the app already does elsewhere for activity glyphs if it has them.)
- **`components/calendar/day-cell.tsx`** — restyle to a rounded dark cell (`rounded-3xl`-equivalent, generous min-height, soft border; today = warm-accent border + accent-filled date badge). Render events as **tonal chips**: done = filled discipline tone + ✓ dot; planned = dashed outline keeping the existing clock/recurring affordances, re-skinned. Preserve every interaction prop (`onSelectDay`, `onSelectWorkout`, `onSelectRun`, `onSelectPlanned`, `isSelected`, `inMonth`, `isToday`).
- **`components/calendar/day-digest.tsx`** — restyle the selected-day read-out to match (rounded warm-on-dark cards, tonal accents per discipline). No change to its data, navigation, or plan/resync actions.
- **Planned affordances** (`planned-banner.tsx` / related) — keep behavior; align their styling to the new chip/token language so a planned session looks consistent in the cell and the digest.

### Page composition (`app/(app)/calendar/page.tsx`)

- **Header** — add a coaching greeting using the real profile name (`useProfile()` → `profile.display_name`, with a graceful fallback when null) and the month-consistency line. Keep `Today`, `+ Plan a workout`, and prev/next exactly as wired; restyle to rounded warm-accent buttons.
- **Hero stats** — keep the six month stats (Lift Time, Run Time, Run Distance, Avg Pace, Longest Run, Activities) and their existing computation; present them as soft rounded cards instead of flat tiles.
- **Grid → per-week panels** — replace the eight-column `grid-cols-[repeat(7,…)_minmax(140px,180px)]` with a vertical stack of **per-week rounded panels**. Each panel renders its seven `DayCell`s in a seven-column row, then a `WeekStreakStrip` footer. This removes the desktop "Week" column and the mobile week-chip-above-row entirely; the streak strip is the single responsive home for weekly data at every breakpoint (simpler than today's `md:contents` / `md:hidden` split). Keep the six-week iteration and the existing day-bucketing (`days.slice`, `eventsByDate`, `localDateKey`, `todayKey`, `selected`).
- **Day digest** — unchanged wiring; sits below the panels as today.

### States to honor (the redesign must not regress these)

Completed-lift vs completed-run vs planned must stay instantly distinguishable; a **dense day** (4+ stacked sessions — e.g. the fixture's June 16) must not break; **empty days and sparse/rest weeks** should read intentional (the rest-week streak line); **leading/trailing** month days stay de-emphasized; **today** is clearly marked. These are exactly the states the DX brief enumerated — verify each renders well.

### Tests

The DOM structure changes (panels replace the column; chips replace pills; a streak strip appears), so the calendar component tests need updating, not just passing-through:

- `page.test.tsx` — assert the per-week panels render, the streak strip shows the right "N of M" for a known fixture, the coaching header shows the profile name, and **all existing interaction tests still hold** (day select, plan-a-workout, digest expand, navigation).
- `weekly-overview.test.tsx` — rewrite for `WeekStreakStrip` (dots count, trained/total, rest-week copy, metric labels) in place of the tile/chip/column tests.
- `day-cell` / `day-digest` / planned-banner tests — update snapshots/queries for the new chip and token styling; keep behavioral assertions.
- Keep CI green locally before pushing: `npm run lint`, `format:check`, `typecheck`, `test`, `build` (the gate in `.github/workflows/ci.yml`).

## Rollout

Web-only, no API or migration, no coordination window.

1. **`prog-strength-web`** — tokens, derivations, component rework, page recomposition, test updates. One PR. Vercel opens a preview automatically; verify the redesign there against the states above before merging.
2. **`prog-strength-docs`** — flip `dx/calendar-view.md` to `status: selected` (noting `warm-organic-coaching`) if not already, and mark this SOW shipped on merge. The DX branch `dx/calendar-view` and PR #56 can be closed/deleted once this lands — their job (carrying the decision here) is done.

### Verification after rollout

- Open `/calendar` in the preview: per-week rounded panels on the dark theme, tonal run/lift chips, a streak strip per week, a coaching header with your name and an honest month-consistency line.
- Walk the states: a dense day stays legible; a rest week shows the gentle line, not a broken empty row; today is clearly marked; planned vs completed are unmistakable.
- Confirm interactions are intact: select a day, expand the digest, plan a workout (from the header and from a day), navigate months, hit Today.
- The rest of the app is visually unchanged (still dark) — no light-theme bleed.

## Open Questions

- **Streak metric labels** — emoji (🏋/🏃/👟, as the mockup) vs the app's existing activity icons if it has a set. Match the app for consistency; cheap to settle in implementation.
- **Month-consistency copy** — exact wording and thresholds for the encouraging line (e.g. how a low-volume month reads). Keep it honest and non-scolding; the precise sentence is an implementation detail, not a contract.
- **Accent — resolved (2026-06-16): violet, not clay.** This SOW originally proposed a warm clay accent; the app has since standardized on the shared **violet** accent (`var(--accent)`) via `sows/chat-and-app-shell-redesign.md`. The calendar uses that shared accent for "today"/streak emphasis (see Design tokens). No clay token is introduced.
- **Discipline hues vs. the accent** — ensure the re-toned `run`/`lift` chip tones stay visually distinct from the violet accent so an activity chip never reads as the "today"/action emphasis. Visual call at implementation, using the shared `--discipline-*` tokens.
- **Future app-wide warmth** — if a warmer direction is ever wanted, that's a design-system conversation (and likely a DS work item), not a reopen of this SOW.
