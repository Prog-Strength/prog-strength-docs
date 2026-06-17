# Implementation Plan — Calendar View Refresh (`true-month-grid`)

**SOW**: [`sows/calendar-view-refresh.md`](../sows/calendar-view-refresh.md) ·
**DX**: [`dx/calendar-view-refresh.md`](../dx/calendar-view-refresh.md) (winner: `true-month-grid`) ·
**Created**: 2026-06-17

> Presentation-layer redesign of `/calendar` in `prog-strength-web`, plus a
> design-system codification + status flips in `prog-strength-docs`. **No data,
> API, routing, rollup-math, merge-logic, or interaction changes** — if a
> behavior changes, that is a regression, not a goal.

## Design decisions pinned up front (so every task agrees)

These remove ambiguity the SOW leaves to "implementation call".

### Colour tokens (all from `app/globals.css`; no raw hex that duplicates a token)

- **Violet `--accent`** (`#8b7cf6`) is spent **only** on: today's day-number/marker,
  the selected cell, and primary actions (`Plan a workout`, `Save changes`,
  `Start workout`). Nowhere else in the grid.
- **Run vs lift** uses the per-discipline tonal triples — **re-toned from warm
  to cool** (the warm values are the "orange holdout"; they are calendar-only,
  see grep below). Pin these exact values in `globals.css`:
  - run (green-teal): `--discipline-run-bg: #16302a`, `--discipline-run-fg: #82d3b8`, `--discipline-run-dot: #46b893`
  - lift (steel-blue): `--discipline-lift-bg: #1a2a3c`, `--discipline-lift-fg: #8cbce8`, `--discipline-lift-dot: #5598d8`
  - `mobility`/`core` slots: **leave as-is, reserved & un-inferred** (Non-Goal).
- **`--warm-accent` / `--warm-accent-fg`** are removed entirely (token defs +
  `@theme inline` mappings) once no calendar file references them. Every former
  `var(--warm-accent)` site becomes `var(--accent)` (violet) for today/selected/
  primary, or a neutral hairline (`--border`/`--border-strong`) for hover.
- **Completed/done semantic green** (`emerald-*`) on planned-status badges and
  the completed-planned rail is **kept** — it is a status colour, not warm
  residue. The tri-state stays glyph+fill+outline+position encoded, not
  colour-only.
- Token usage is calendar-scoped today (verified):
  `grep -rl "discipline-\|warm-accent"` → only `app/globals.css` + `components/calendar/*`
  + `app/(app)/calendar/page.tsx`. So re-toning the discipline triples and
  removing `--warm-accent` does **not** touch any non-calendar surface.

### Scoped re-toning of shared components (Non-Goal: non-calendar pages unchanged)

- `MuscleGroupPill` (`components/muscle-group-pill.tsx`) is shared with the
  exercises catalog + workout detail → **do not change it.**
- `PlannedAgendaDetails` (`components/calendar/planned-agenda-details.tsx`) is
  shared with the non-calendar `app/(app)/planned-workouts/[id]/page.tsx` →
  add an **opt-in prop** (e.g. `chipVariant?: "default" | "slate"`, default
  `"default"`) that swaps the saturated `MuscleGroupPill` for a restrained
  slate chip (`--surface-3` bg, `--muted` text, hairline border). The calendar
  view modal passes `chipVariant="slate"`; the standalone page keeps the
  default. This satisfies "restrained slate tints" in the modal without
  changing the other route.

### Layout specifics

- Single bordered grid in one `rounded-xl` container, `var(--border)` hairline.
- Columns: `repeat(7, 1fr) 84px` — weekday header row (7 day labels + a "Week"
  label over the 8th column), then one row per week with a top hairline.
- Dense-day cap (Open Question resolved): show up to **3** event pills, then a
  `+N more` button that selects the day (existing behavior). Keep
  `MAX_VISIBLE_PILLS = 3`.
- Week column (~84px): compact stacked `WeeklyStat` — sessions count, lift time,
  run distance, steps, plus a quiet trained-days indicator (e.g. `N/7`). The
  "trained N of 7" coaching line is demoted here from a full-width banner.
- Inline month-stepper pill: `‹  June 2026  ›` as one quiet pill
  (`--surface`/`--border`), demoted out of the hero; `Today` + violet
  `Plan a workout` in a compact cluster. Keep `goPrev`/`goNext`/`goToday` wiring.

## Execution order (subagent-driven; each task = implement → spec-review → code-quality-review)

All web paths under `prog-strength-web`. Tasks are sequential (shared working
tree / shared files). After every task: `npm run typecheck && npm run lint &&
npm run test` must pass before the next.

- **W1 — Colour tokens (`globals.css`).** Re-tone the run/lift discipline
  triples to the cool values above; update the stale "Warm-organic coaching
  tokens" comment. Leave `--warm-accent` defs in place for now (removed in W8).
  Appearance-only; no test asserts these hex, so the suite stays green.

- **W2 — Month-bounded grid logic (`page.tsx` + `page.test.tsx`).** Rewrite
  `buildMonthGrid(year, month)` to emit only whole weeks that intersect the
  month (4–6 rows) — Monday-on/before the 1st, then full 7-day weeks until the
  last week containing a day of `month` completes. Update its doc comment (drop
  "fixed 42-cell"). Verify `gridFetchBounds` + steps bounds still bracket the
  visible grid (they derive from `buildMonthGrid`). Change grid iteration from
  `Array.from({length: 6})` to `days.length / 7`. `weeklyStats` already chunks
  by 7 — confirm it tracks variable length. Update `weekContainsToday` doc.
  **Tests**: replace `strips.length === 6` with the correct count for the
  current month, and assert the **last** week row contains an in-month day
  (no phantom next-month rows). Keep all other page tests green. (Per-week panel
  rendering stays until W3 — this task isolates the structural bug fix.)

- **W3 — Single bordered grid + cell + 7th Week column
  (`page.tsx`, `components/calendar/day-cell.tsx`,
  `components/calendar/weekly-overview.tsx`, their tests).** Replace the
  per-week rounded-panel stack with one bordered `rounded-xl` grid:
  `repeat(7,1fr) 84px`, weekday+"Week" header row, one row per week with top
  hairline. Restyle `DayCell` to compact uniform bordered cells (violet only for
  today/selected; tonal left-bar/tint pills by discipline; dashed=planned,
  solid=done; state glyphs; greyed `inMonth=false` cells still show their
  number; keep `+N more`). Convert `WeekStreakStrip` (full-width) → a slim
  `WeekColumn` cell carrying the same `WeeklyStat` (keep `formatTotalDuration`,
  step formatting, the `WeeklyStat`/`DayMark` types). Replace `--warm-accent`
  in these files with `--accent`/neutral. **Preserve every `DayCell` prop and
  click path.** Update `day-cell.test.tsx` + `weekly-overview.test.tsx` for the
  new structure (keep behavioral assertions: dense day, planned vs done,
  boundary cells, the rollup data); update `page.test.tsx` streak-strip
  assertions to the new Week-column testid/structure.

- **W4 — Navigation + stat row (`page.tsx`, `page.test.tsx`).** Remove the
  header chevron utility bar + `NavButton`; add the demoted inline month-stepper
  pill (label flanked by chevrons) above the grid; keep `Today` + violet
  `Plan a workout` reachable. Re-tone the six month stat tiles into a frugal
  near-monochrome row (same computation). Remove remaining `--warm-accent` from
  the header. Update `page.test.tsx` so the stepper's prev/next still drive
  month change (the existing "Previous month"/"Next month" aria-labels should be
  preserved on the stepper chevrons so those tests keep working).

- **W5 — Day detail panel + banners
  (`components/calendar/day-digest.tsx`, `workout-banner.tsx`,
  `run-banner.tsx`, `planned-banner.tsx`, `completed-planned-banner.tsx`, and
  their tests).** Restyle the digest from dashed-outline rows into a finished
  inline panel: dated header + rounded rows each with a tonal left-bar by
  discipline, session name, `PLANNED`/`done` tag, time, one-line summary; empty
  day reads "Rest day — nothing logged or planned." (keep the plan-a-workout
  CTA). Align the four banners to the new row language (tonal left-bars already
  use discipline-dot tokens — now cool). Preserve ALL behavior: `autoExpandId`
  expansion + remount-key trick, navigation, plan-a-workout, resync, open-planned,
  view-plan disclosure. Update banner tests + `day-digest.test.tsx` (keep
  behavioral assertions).

- **W6 — View modal (`components/planned-workout-modal.tsx` view mode;
  `planned-agenda-details.tsx` + its test; `planned-workout-modal.test.tsx`).**
  Re-tone the read-only view into a compact slate panel (hairline borders,
  violet primary). Add the `chipVariant` prop to `PlannedAgendaDetails` (default
  unchanged) and pass `"slate"` from the modal so muscle chips become restrained
  slate tints. Keep the type tag, `PLANNED` status, "Synced to Google Calendar"
  note, WHEN block, AGENDA (numbered, set/rep lines, superset shown via a
  violet-line left-bar + "superset" tag), `Edit` (pencil) + violet
  `Start workout`. Behavior unchanged (`PlannedAgendaDetails`, start-workout
  handoff).

- **W7 — Edit form + form controls
  (`components/planned-workout-modal.tsx` edit mode, `agenda-editor.tsx`,
  `plan-schedule-field.tsx`, and `agenda-editor.test.tsx` /
  `planned-workout-modal.test.tsx`).** Make one coherent set of real interactive
  controls: pill segmented Lift/Run toggle (active = violet fill + accent-fg);
  rounded slate inputs (`--surface-2`, hairline `--border`, accent focus ring,
  uppercase faint labels) for Name/Notes; matching date/time/duration in
  `PlanScheduleField` with the derived read-only "Ends at"; matching selects;
  `AgendaEditor` superset grouping as a calm accent-line left-bar treatment.
  Footer `Cancel` / violet `Save changes`. **Preserve** all behavior:
  reps-required/weight-optional, AMRAP, dumbbell-per-dumbbell convention,
  superset normalize/toggle helpers, save-keeps-modal-open-returns-to-view.

- **W8 — Cleanup + full local gate (`globals.css`, repo-wide).** Remove the now
  unused `--warm-accent`/`--warm-accent-fg` token defs + `@theme inline`
  mappings. `grep -ri "warm\|orange\|clay\|amber\|#e08a5e\|#d6ab54"` across
  `app/(app)/calendar`, `components/calendar`, `components/planned-workout-modal.tsx`
  → no residue (amber only acceptable if it's an unrelated macro token, which it
  is not in calendar). Run the full CI gate locally: `npm run lint`,
  `npm run format:check`, `npm run typecheck`, `npm run test`, `npm run build`.
  Fix anything red — never `//nolint`, never `--no-verify`, never weaken a test.

## prog-strength-docs tasks

- **D1 — Design-system codification (`design-system.md`).** Under
  **Decided → Form & depth** add a "Form controls" subsection recording the
  specs the calendar edit form establishes: rounded slate field surface
  (`--surface-2` + hairline `--border`), uppercase faint labels, accent focus
  ring, the pill segmented toggle (active = `--accent` fill + `--accent-fg`),
  and the accent-line superset/grouping left-bar. Remove
  "form-control (input/select/toggle) specs" from **Open / not yet decided**.
  Add a Provenance line pointing at this SOW. Bump the version + changelog.
  Descriptive + tokens-first (component primitive deferred — Open Question
  recommendation).
- **D2 — DX status flip (`dx/calendar-view-refresh.md`).** `status: draft` →
  `status: selected`; note the winning idiom `true-month-grid`; update the
  body `**Status**:` line + `**Last updated**: 2026-06-17`.
- **D3 — SOW status flip (`sows/calendar-view-refresh.md`).** frontmatter
  `status: shipped`; body `**Status**: Shipped`; `**Last updated**: 2026-06-17`.

## Verification (matches SOW Rollout)

- `/calendar`: bordered, month-bounded slate/violet grid with a slim "Week"
  rollup column — no orange, no phantom next-month rows.
- Dense day stays legible; empty/recovery weeks intentional; today + selected
  cell are the only violet in the grid; planned vs done unmistakable without
  colour alone.
- Inline stepper + `Today` navigate; plan from header and from a day; view
  modal + edit form re-toned, form controls intentional, superset clear, all
  behavior intact.
- `design-system.md` records the form-control specs; the "not yet decided" item
  is resolved.
- Local gate green: lint, format:check, typecheck, test, build.
</content>
