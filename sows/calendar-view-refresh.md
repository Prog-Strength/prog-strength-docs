---
type: sow
status: shipped
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Calendar View Refresh — `true-month-grid`: a month-bounded slate/violet calendar

**Status**: Shipped · **Last updated**: 2026-06-17

## Introduction

The monthly training calendar (`/calendar`) is one of the most-visited surfaces in Prog Strength, and its data and functionality are solid — it correctly renders logged lifts, logged runs, planned sessions, and the merged "planned-then-completed" case; weekly rollups compute; the day digest and the plan view/edit flow all work. What lets it down is the **look**, and after the warm-organic redesign ([`sows/calendar-view-redesign.md`](calendar-view-redesign.md)) it is now the app's conspicuous **"orange holdout"** — it predates the settled design system and never fully re-toned, so it reads foreign next to the slate/violet shell around it. On top of the color drift, three concrete problems have accumulated: the grid renders **phantom next-month rows** (it scrolls well past where the month ends), month switching reads as **browser chrome** (a header utility bar with two ghost chevrons), and the day detail and its modals look **unfinished** (a stack of dashed-outline rows; a centered view/edit modal whose inputs, selects, saturated muscle-group chips, and superset grouping predate every design-system decision).

To find a better direction without guessing, the surface went through a second Design Exploration ([`dx/calendar-view-refresh.md`](../dx/calendar-view-refresh.md), PR [Prog-Strength/prog-strength-web#62](https://github.com/Prog-Strength/prog-strength-web/pull/62)), which put four genuinely distinct **in-system compositions** of the whole surface side by side. The owner picked **`true-month-grid`** — "make it a real calendar, done right": a disciplined, bordered Mon→Sun month grid in the Google Calendar / Fantastical vocabulary, re-toned to slate.

This SOW implements that direction for real, production-quality, against the real data and types. After it ships, `/calendar` is a compact bordered month grid that **stops at the month boundary** (only the current month's weeks; adjacent-month days appear as quiet greyed cells inside the boundary rows, never extra rows); the per-week rollup collapses into a **slim 7th "Week" column** on the right; color is **frugal** — near-monochrome slate, violet spent *only* on today and the selected cell, tonal left-bars/dots to mark run vs lift; the month navigates via a **demoted inline stepper pill** instead of a chrome-y header; the selected day expands into a **finished inline detail panel** beneath the grid; and the event **view** modal and **edit** form become re-toned slate panels with real, intentional form controls. Same data, same interactions, same correctness — the presentation the surface deserves, finally belonging to the app.

## Proposed Solution

This is a **presentation-layer redesign of one surface**. No calendar data, API, routing, or business logic changes — the event model (`CalendarEvent`), the weekly rollups (`WeeklyStat`), the day-selection / plan-a-workout / resync / Google-sync interactions, and the completed-planned merge all stay exactly as they are. What changes is the layout and styling of `app/(app)/calendar/page.tsx`, the components under `components/calendar/`, and the `PlannedWorkoutModal` view/edit surface.

The structural moves, taken from the chosen `true-month-grid` variant:

- **Month-bounded grid (the bug fix, structural).** The grid is built today as a fixed **42-day / six-week** block (`buildMonthGrid` always returns 42 cells), which is what produces the phantom next-month week rows. The fix is structural, not a render-time filter: `buildMonthGrid` returns **only the weeks that contain a day of the current month** (4–6 week rows). Leading days of the prior month and trailing days of the next month still appear — but **only** as greyed cells inside the first and last week rows, never as whole extra rows. The fetch bounds (`gridFetchBounds`, steps bounds) continue to cover exactly the visible grid, whatever its height.

- **Retire per-week panels for one bordered grid.** The current layout is a vertical stack of rounded per-week panels, each a seven-column day row plus a full-width `WeekStreakStrip` footer. `true-month-grid` replaces this with a **single bordered grid** of uniform, compact cells: a weekday header row, then one row per week, columns `repeat(7, 1fr) 84px` — the **8th column is a slim "Week" rollup** carrying that week's `WeeklyStat` data, replacing the full-width strip.

- **Frugal, near-monochrome color.** Violet (`var(--accent)`) is reserved for **today and the selected cell** and the primary action only. Run vs lift is marked by the existing **activity tonal hues** as a left-bar / dot / chip-tint, kept visually distinct from the accent so an activity never reads as "today." Encoding is not color-only (glyphs + fills + outlines + position carry the tri-state too), so completed-lift / completed-run / planned / completed-planned survive for a color-blind eye. Any remaining warm/clay residue is removed.

- **Demoted inline navigation.** The header chevron utility bar gives way to a quiet **inline month-stepper pill** (month label flanked by chevrons), demoted out of the hero position. `Today` and `Plan a workout` remain reachable.

- **Finished inline day detail.** The selected-day digest is restyled from dashed-outline rows into an intentional **inline panel** beneath the grid: rounded rows with a tonal left-bar per discipline, the session name, a `PLANNED`/`done` tag, time, and a one-line summary.

- **Re-toned view + edit modals with real controls.** The two centered modals are the `PlannedWorkoutModal`'s existing **view** and **edit** modes. They become compact slate panels with hairline borders and a violet primary action; muscle-group chips drop the saturated red/blue/yellow for restrained slate tints; the **edit form gets real, intentional form controls** — the Lift/Run toggle, Name/Date/Start/Duration/Notes inputs and selects, and the superset grouping — styled as one coherent set. (The variant's `Field`s are static read-only mockups; the real interactive controls — `PlanScheduleField`, `AgendaEditor`, Google-sync — keep all behavior and adopt the new treatment.)

Color and surfaces draw from the **existing shared tokens** (`app/globals.css`): `var(--accent)` / `--accent-soft` / `--accent-line` / `--accent-fg` for emphasis and primary actions; the slate ramp (`--surface` / `--surface-2` / `--surface-3`, hairline `--border` / `--border-strong`) for panels, cells, and chips; `--foreground` / `--muted` / `--faint` for type; the per-discipline `--discipline-run-*` / `--discipline-lift-*` tonal triples for run vs lift; `--shadow-soft` for elevation. No new accent, and no raw hex that duplicates a token.

Because the winning **edit-form treatment** is the first real proposal for form controls in the app — and the design system explicitly lists input/select/toggle specs as "not yet decided" — this SOW also **codifies that treatment back into `design-system.md`** so future forms inherit it (see Implementation Details → Design-system codification).

## Goals and Non-Goals

### Goals

- Reimplement `/calendar` to match the **`true-month-grid`** variant: a bordered, uniform Mon→Sun month grid with a slim 7th "Week" rollup column, compact even cells, and calendar-native rhythm.
- **Month-bounded grid (the fix):** render only the current month's weeks; no phantom next-month rows; adjacent-month days appear only as greyed cells inside the first/last week rows.
- **Frugal color:** violet reserved for today / selected / primary action; run vs lift via the existing tonal hues; tri-state legibility preserved and not color-only; zero warm/clay residue.
- **Demoted inline navigation:** an inline month-stepper pill replacing the header chevron bar; `Today` and `Plan a workout` remain reachable.
- **Finished day detail:** restyle the selected-day digest into an intentional inline panel (tonal left-bar rows, planned/done tags, time, summary).
- **Re-toned view modal:** the plan view mode becomes a compact slate panel — re-toned agenda, restrained muscle-group tints, violet primary action, the "Synced to Google Calendar" note and "Start workout" preserved.
- **Finished edit form:** the plan edit mode gets real, intentional form controls — Lift/Run toggle, Name/Date/Start/Duration/Notes inputs and selects, derived "Ends at", and the superset grouping — as a coherent set, with all current behavior intact.
- **Codify the form-control specs** (input / select / toggle / superset) from the winning treatment into [`design-system.md`](../design-system.md), under what is currently the "not yet decided" form-control item.
- Preserve **all current functionality**: day selection, plan-a-workout (month-level and per-day), prev/next/Today navigation, the planned/clock/recurring affordances, the completed-planned merge, the day digest, resync, Google-sync, the live-session handoff, and all routing.
- Update the calendar component **tests** to the new structure; keep the suite green and CI's gate (`lint`, `format:check`, `typecheck`, `test`, `build`) passing.

### Non-Goals

- **App-wide re-theme.** The sidebar and every non-calendar page stay unchanged. This is one surface.
- **Any change to calendar data, API, rollup math, merge logic, or interactions.** Presentation only. If a behavior changes, that's a regression, not a goal.
- **New activity disciplines.** Only `run` and `lift` tones are produced; the `mobility`/`core` token slots stay reserved but un-inferred.
- **A different palette or type family.** `scope: in-system` — this conforms to the slate ramp, violet accent, and Nunito; it diverges only on composition.
- **Promoting the DX mockup code.** The variant at `app/design-explore/calendar-view-refresh/_components/VariantTrueMonthGrid.tsx` is the visual north star, built on simplified static fixtures (`fixtures.ts`, `HUE`, `CalEvent`) — **not** code to copy. Reimplement against the real `CalendarEvent` / `WeeklyStat` types and real data. The `app/design-explore/` route is not touched and never ships.
- **Removing the modals in favor of a docked panel.** That was the rejected `panel-driven` idiom. `true-month-grid` keeps centered modals, re-toned.

## Implementation Details

All web paths are in `prog-strength-web`. Lifecycle order: grid bounding → grid/cell layout + 7th column → navigation → day detail → view modal → edit form → design-system codification → tests.

### Month-bounded grid (`app/(app)/calendar/page.tsx`)

- **`buildMonthGrid(year, month)`** — change from a fixed 42-day block to a variable run of **whole weeks that intersect the current month**. Start from the Monday on/before the 1st (the existing `mondayOffset` math is correct — keep Monday-first and the `(getDay() + 6) % 7` shift), then emit complete 7-day weeks until the last week that contains a day of `month` is complete. Result is 4–6 week rows. Update the doc comment that currently promises a "fixed 42-cell shape."
- **`gridFetchBounds` and the steps bounds** — both derive from `buildMonthGrid`'s output (first cell → last cell), so they keep covering exactly the visible grid once it's variable-height. Verify the `since`/`until` window still comfortably brackets the visible cells.
- **Grid iteration** — the page currently maps `Array.from({ length: 6 })` and slices `days` in chunks of seven; iterate over the actual weeks instead (`days.length / 7` rows). Keep `localDateKey`, `eventsByDate`, `todayKey`, `selected`, and the `inMonth` check (`day.getMonth() === cursor.month`) — `inMonth=false` cells render as the greyed boundary cells.

### Grid + cell layout (`page.tsx`, `components/calendar/day-cell.tsx`)

- Replace the per-week rounded-panel stack with a **single bordered grid** wrapped in one rounded container (`var(--border)` hairline, `rounded-xl`). A weekday header row (`repeat(7, 1fr) 84px`), then one row per week with a top hairline between rows.
- **`DayCell`** — restyle to a compact, uniform bordered cell: small even type, day-number hierarchy from weight, today = violet day-number/marker, selected = violet ring/fill. Events render as **compact tonal pills** (left-bar/tint by discipline, dashed outline for planned, solid for done, a state glyph). A **dense day** (4+ stacked — the fixture's Jun 16) must stay legible (cap visible pills with a "+N more" affordance rather than truncating into uselessness). Greyed (`inMonth=false`) cells are quiet but still show their day number. Preserve every interaction prop (`onSelectDay`, `onSelectWorkout`, `onSelectRun`, `onSelectPlanned`, `isSelected`, `inMonth`, `isToday`).

### Weekly rollup → slim 7th column (`components/calendar/weekly-overview.tsx`)

- Replace the full-width `WeekStreakStrip` footer with a **slim "Week" column** cell (≈84px) rendered as the 8th column of each week row. It carries the same `WeeklyStat` data (sessions / lift time / run distance / steps) in a compact stacked form, plus a quiet trained-days indicator. Keep the `WeeklyStat` type and its formatting helpers (`formatTotalDuration`, step formatting) — only the presentation changes. The strip's "trained N of 7 days" framing is demoted from a coaching banner to a compact rollup here (the coaching voice now lives mainly in the day detail and modals).

### Navigation (`page.tsx`)

- Remove the header chevron utility bar (`NavButton` prev/next floating beside the title). Introduce a **demoted inline month-stepper pill**: month label flanked by prev/next chevrons in a single quiet pill control, placed above the grid out of the hero position. Keep `goPrev` / `goNext` / `goToday` wiring unchanged. `Today` and the violet `Plan a workout` primary action stay reachable (a compact action cluster).
- The six month-level stat tiles stay (same computation); present them as a frugal, near-monochrome stat row consistent with the variant rather than the current soft cards.

### Day detail (`components/calendar/day-digest.tsx` and banners)

- Restyle the selected-day read-out from dashed-outline rows into a **finished inline panel**: a dated header (`Tuesday, June 16, 2026 · 2 activities`), then rounded rows each with a **tonal left-bar** by discipline, the session name, a `PLANNED`/`done` tag, scheduled time, and a one-line summary. Empty days read intentionally ("Rest day — nothing logged or planned."), not broken.
- Keep all behavior: `autoExpandId` expansion, navigation to workout/run, plan-a-workout, resync, open-planned. Align `workout-banner` / `run-banner` / `planned-banner` / `completed-planned-banner` styling to the new row language.

### View modal (`components/planned-workout-modal.tsx`, view mode)

- Re-tone the read-only view into a compact slate panel: discipline glyph + title, a type tag, the `PLANNED` status, the "Synced to Google Calendar" note, a WHEN block, and the **AGENDA** — the numbered exercise list with per-exercise **muscle-group chips** (restrained slate tints via `--surface-3` / `--muted`, dropping the saturated red/blue/yellow) and set/rep lines, with the superset grouping shown by a violet-line left-bar + "superset" tag. Footer: `Edit` (pencil) and the violet `Start workout`. Behavior unchanged (`PlannedAgendaDetails`, the start-workout handoff).

### Edit form + form controls (`components/planned-workout-modal.tsx` edit mode, `agenda-editor.tsx`, `plan-schedule-field.tsx`)

This is the roughest surface in the app today and the one the DX exists to fix. Style as one coherent set of **real, interactive** controls (not the variant's static `Field` spans):

- **Lift / Run toggle** — a pill segmented control; active segment = violet fill + accent-fg, inactive = muted.
- **Text / number inputs** (Name, Notes) and **date/time/duration** (`PlanScheduleField`) — rounded slate inputs (`--surface-2`, hairline `--border`), accent focus ring, uppercase faint labels, the derived "Ends at" as a read-only derived field.
- **Selects** (run type, etc.) — matching rounded slate treatment.
- **Agenda editor** (`AgendaEditor`) — exercise rows with per-set detail and the **superset grouping** as a coherent, calm treatment (accent-line left-bar binding grouped exercises). Reps-required / weight-optional semantics and the dumbbell-per-dumbbell weight convention are unchanged.
- Footer: `Cancel` / violet `Save changes`. The save-keeps-modal-open-and-returns-to-view behavior is preserved.

### Design-system codification (`prog-strength-docs/design-system.md`)

Once the edit-form treatment lands, promote it from "not yet decided" to a decided convention. Under **Decided → Form & depth** (or a new "Form controls" subsection), record the input / select / toggle / superset specs the calendar edit form establishes: rounded slate field surface (`--surface-2` + hairline), uppercase faint labels, accent focus, the pill segmented toggle, and the accent-line superset/grouping treatment. Remove "form-control (input/select/toggle) specs" from the **Open / not yet decided** list and add a Provenance line pointing at this SOW. Keep it descriptive and tokens-first — it is the language every future form in the app should adopt.

### States to honor (the redesign must not regress these)

The DX enumerated where lazy designs fall apart — verify each renders well against the representative June 2026 fixture: tri-state legibility without orange; the month-bounded grid (no phantom rows, greyed boundary cells only); the **dense day** (Jun 16, four stacked planned) staying legible; **empty days / sparse and recovery weeks** reading intentionally; **today** clearly marked; the worked example **W5 D1 – Chest & Back** (planned lift, Google-synced, six exercises with Machine Fly + Pull Up as a superset) rendering correctly in the view modal and edit form.

### Tests

The DOM structure changes (single grid replaces panels; a 7th rollup column replaces the strip; an inline stepper replaces the header chevrons; restyled modals), so the calendar tests need updating, not just passing-through:

- `app/(app)/calendar/page.test.tsx` — assert the grid renders **only the current month's weeks** (no phantom next-month rows) for a known cursor, the 7th "Week" column shows the right rollup, the inline stepper navigates, and **all existing interaction tests still hold** (day select, plan-a-workout, digest expand, prev/next/Today).
- `weekly-overview.test.tsx` — update for the slim-column rollup in place of the full-width strip.
- `day-cell.test.tsx` / `day-digest.test.tsx` / banner tests — update queries/snapshots for the new cell, pill, and day-detail row styling; keep behavioral assertions (dense day, planned vs done, boundary cells).
- `planned-agenda-details.test.tsx` / `agenda-editor.test.tsx` — keep behavior; update for the re-toned chips and form controls.
- Keep CI green locally before pushing: `npm run lint`, `npm run format:check`, `npm run typecheck`, `npm run test`, `npm run build` (the gate in `.github/workflows/ci.yml`).

## Rollout

Web-first plus a docs change, no API or migration, no coordination window.

1. **`prog-strength-web`** — grid bounding, grid/cell layout + 7th column, navigation, day detail, view modal, edit form + form controls, test updates. One PR. Vercel opens a preview automatically; verify the redesign and the states above there before merging.
2. **`prog-strength-docs`** — codify the form-control specs into `design-system.md`; flip [`dx/calendar-view-refresh.md`](../dx/calendar-view-refresh.md) to `status: selected` (noting `true-month-grid`) if not already; mark this SOW shipped on merge. The DX branch `dx/calendar-view-refresh` and PR #62 are closed (never merged) once this lands — their job (carrying the decision here) is done.

### Verification after rollout

- Open `/calendar` in the preview: a bordered, month-bounded grid in slate/violet with a slim "Week" rollup column — **no orange anywhere**, no phantom next-month rows, the month stops where the month stops.
- Walk the states: the dense day (Jun 16) stays legible; empty/recovery weeks read intentionally; today and the selected cell are the only violet in the grid; planned vs completed are unmistakable without relying on color alone.
- Navigate with the inline stepper and `Today`; plan a workout from the header and from a day; open the view modal and the edit form for W5 D1 — both re-toned, the form controls intentional, the superset grouping clear, all behavior intact.
- Confirm `design-system.md` now records the form-control specs and the DX's "not yet decided" item is resolved.
- The rest of the app is visually unchanged.

## Open Questions

- **Dense-day overflow affordance** — exact cap before "+N more" in a cell, and whether overflow opens the day detail or a popover. Cheap to settle in implementation; must not truncate a 4-session day into uselessness.
- **Week-column density** — how much of the `WeeklyStat` (sessions / lift / run / steps + trained-days) fits legibly in ~84px vs. what moves to a tooltip. Visual call at implementation.
- **Form-control codification scope** — whether the `design-system.md` entry stays descriptive (tokens + patterns) now and a reusable component primitive is a later DS work item. Recommend descriptive-now; primitives when a second form surface lands.
