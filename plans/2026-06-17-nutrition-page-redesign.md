# Implementation Plan — Nutrition Page Redesign (Dashboard-Hero)

Derived from `sows/nutrition-page-redesign.md`. Web-only presentation rebuild
(plus docs status flip). No backend change. Conforms to `design-system.md`
(slate / single violet accent / Nunito / rounded-2xl–3xl / full-pill) using the
existing CSS-variable tokens and the shipped macro tints in
`lib/macro-colors.ts` (protein emerald, carbs amber, fat pink; calories
deliberately uncolored — the ring uses violet `var(--accent)`).

## Goal

Rebuild `/nutrition`'s daily view into the **dashboard-hero** composition:
quiet day strip → dominant hero band (composite calories ring + three big macro
numerals each with a budget bar) → action row → demoted meal-bucketed log that
keeps full edit/delete. Make the over-goal state unmistakable (numeral + bar +
caption flip to `var(--warning)` + "· OVER"; bar caps at 100%). Reuse the
existing data layer, CRUD, modals, and timezone logic unchanged — this is a
presentation rebuild over the existing `NutritionPageInner` orchestration.

## Constraints / fixed points (must not regress)

- Data layer unchanged: `listNutritionLog`, client-side `totals` `useMemo`,
  `getMacroGoals`/`putMacroGoals`, full entry CRUD, `listPantryItems`/`listRecipes`.
- Full entry edit/delete inside the demoted log (desktop pencil/trash + mobile
  `EntryActionSheet`), with the existing optimistic updates and toasts.
- Meal bucketing by `entry.meal` (Breakfast/Lunch/Dinner/**Snacks**), per-meal
  subtotals, empty-bucket "nothing logged", long-name wrap with right-aligned
  calories.
- `?view=` Log/Pantry/Recipes tabs; Pantry/Recipes internals untouched.
- Timezone-aware day selection; `DateTileStrip` behavior (7/3 tiles, chevrons,
  Today snap) preserved — only the tile visuals get the quieter pill treatment.
- Goals-never-set (`goals.created_at == null` or a goal is 0): degrade
  gracefully — show consumed figures, no over-state, inviting "Set your goals".
- Web CI green: lint / format / typecheck / test / build.

## Tasks

### Task 1 — Macro status helper + HeroBand components (new files)

New file `app/(app)/nutrition/_components/hero-band.tsx` exporting `HeroBand`
plus internal `CaloriesRing`, `HeroMacro`, and a single shared helper that
decides over/under colour so the ring and the three macros stay consistent.

- `macroStatus(value, goal)` helper → `{ pct, over, hasGoal }` where
  `hasGoal = goal > 0`; `pct = hasGoal ? round(value/goal*100) : 0`;
  `over = hasGoal && value > goal`. One helper, reused by ring + macros.
- `CaloriesRing` — large SVG donut (scaled from the shipped ring geometry),
  violet `var(--accent)` fill over a `var(--surface-2)` track, sweep capped at
  100%. Centered `text-6xl font-extrabold tabular-nums` calorie numeral, "of
  {goal} kcal" sublabel, "{n} left" badge. Per Open-Question lean (a):
  **calories over goal flips** the ring fill + badge to `var(--warning)` and the
  badge reads "over by {n}". When goals not set: render the consumed numeral,
  no track-fill/over-state, badge omitted.
- `HeroMacro` ×3 (Protein/Carbs/Fat): uppercase `var(--faint)` label, big
  `text-5xl font-extrabold tabular-nums` numeral coloured by
  `MACRO_COLORS[macro]`, "/{goal}g", thin budget bar (`bg-[var(--surface-2)]`
  track, tint fill capped at 100% width), "{pct}%" caption. Over-goal: numeral,
  bar fill, caption → `var(--warning)`, caption appends " · OVER". Goals-not-set:
  show consumed grams, neutral, no bar fill / "%" / over-state.
- `HeroBand` — `rounded-3xl` surface, hairline border, `shadow-[var(--shadow-soft)]`,
  generous padding; column on mobile → row at `lg`. Composite ring + a
  `sm:grid-cols-3` macro grid. When goals-never-set, render an inviting "Set
  your goals" affordance (button) wired via an `onEditGoals` prop to the page's
  existing `MacroGoalsModal`; match the current empty-state intent/copy.
- Props: `totals` (calories/protein_g/fat_g/carbs_g), `goals: MacroGoals`,
  `onEditGoals: () => void`. Reimplement against real `lib/api.ts` types — do
  NOT import from the gated `design-explore` tree.

### Task 2 — Demoted log presentation (`components/nutrition/nutrition-log-view.tsx`)

Rebuild the presentation to calm neutral `rounded-2xl bg-[var(--surface)]`
meal-section cards while keeping ALL behavior:

- Meal header: label + subtotal `"{cal} cal · P {p}g · C {c}g · F {f}g"`, or
  "nothing logged" when the bucket is empty (render the empty header — don't drop
  the section, and drop the page-level "Nothing logged on this day yet." early
  return so empty Snacks renders its header).
- Entry rows in small muted type: `resolveItemName` name (`min-w-0 flex-1`,
  wraps), `×{quantity}`, right-aligned calories (`shrink-0`).
- **Keep edit/delete**: desktop per-row pencil/trash (revealed on hover/focus,
  calm), mobile tap → `EntryActionSheet`. Same `onEdit`/`onDelete` callback
  shape so the page's modals + optimistic updates are unchanged. Keep the
  `recipe` tag.
- Per Open-Question lean (b): drop the desktop per-row P/F/C columns (hero +
  per-meal subtotals carry the macro story); keep `resolveItemName` exported.

### Task 3 — Page composition (`app/(app)/nutrition/page.tsx` + DayStrip restyle)

- Replace the `<MacroGoalRings>` block with `<HeroBand totals goals
  onEditGoals={() => setShowGoalsModal(true)} />`. Render the hero whenever
  `goals` is loaded (HeroBand handles the not-set degrade internally). Remove the
  now-unused `MacroGoalRings` import; delete `components/macro-goal-rings.tsx`
  (only consumer was this page).
- **ActionRow**: Quick Add (violet `var(--accent)` pill), Edit Macros (outlined
  `border-[var(--border-strong)]`), and the Log/Pantry/Recipes **pill tab group**
  (`?view=` bound) restyled to the pill segmented treatment. Keep mobile
  icon-only / desktop-labeled ergonomics (the existing `ToolbarButton` math, or
  a small local pill-tab component — keep aria-labels).
- **DayStrip**: restyle `DateTileStrip` tiles to the quieter pill treatment
  (selected = `bg-[var(--accent)] text-[var(--accent-fg)]` pill; others
  `text-[var(--faint)]`) while preserving ALL behavior (window state, chevrons,
  Today snap, `useSyncExternalStore` breakpoint, a11y roles). Restyle in place
  (only consumer is this page).
- Keep all orchestration (`date`/`entries`/`goals`/`pantry`/`recipes`,
  `?view=`, timezone, `totals` `useMemo`), the modals, the 401 handling.

### Task 4 — Tests

New `app/(app)/nutrition/page.test.tsx` (or co-located component tests) using
the existing api/auth/toast mock pattern (see `progress/page.test.tsx`):

- Hero: calories ring shows consumed/goal numerals + "{n} left"; each
  `HeroMacro` renders numeral, goal, bar, percentage from a known total/goal
  fixture; the **over-goal** macro (fat 132% on the representative day) renders
  numeral/bar/caption in the warning colour with "· OVER" while under-goal
  macros stay tinted; goals-never-set fixture shows consumed figures + "set your
  goals" and no over-state.
- Log: entries group into Breakfast/Lunch/Dinner/Snacks by `meal`; empty Snacks
  shows "nothing logged"; long custom Chipotle entry wraps with calories
  right-aligned; **edit and delete still fire** their mutations and optimistically
  update from a row. Reuse the representative-day numbers from the DX fixtures as
  the test fixture (reimplemented against `NutritionLogEntry`/`MacroGoals`).

### Gate before push

`npm run lint && npm run format:check && npm run typecheck && npm run test &&
npm run build` all green. No `--no-verify`, no `nolint`, no skipped/weakened
tests.

## Out of scope (per SOW non-goals)

Backend/SDK; Pantry/Recipes internals; modal flows/validation; photo logging;
meal re-bucketing; promoting DX mockup code (design-explore stays gated).
