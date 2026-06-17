---
type: sow
status: ready_for_implementation
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Nutrition Page Redesign — Dashboard-Hero

**Status**: Ready for implementation · **Last updated**: 2026-06-17

## Introduction

`/nutrition` is the highest-frequency surface in the product — a user opens it 3–4× a day to answer one question: **"how's my day going?"** Today it doesn't answer that question fast. The page (`app/(app)/nutrition/page.tsx`) stacks a date strip, a row of four equal-weight `MacroGoalRings`, a toolbar, and the log — and the rings give calories, protein, carbs, and fat exactly the same visual weight, so nothing leads. The over-goal state is whisper-quiet (a small amber "132% of goal" caption under one ring), and the log reads like a spreadsheet. There's no first-half-second hit that tells you where you stand.

The Design Exploration `dx/nutrition-page.md` (PR Prog-Strength/prog-strength-web#68) explored that surface across five in-system variants on a representative day (Tue Jun 16: a partially-eaten log, **fat over goal at 132%**, the long custom Chipotle bowl entry, and an **empty Snacks bucket** — the states where lazy designs fall apart). The owner selected **`dashboard-hero`** (drawing on Whoop's today screen): a big, confident **hero band** — a composite calories ring plus three oversized macro numerals (Nunito-800, 5xl–6xl) each with a budget bar — that dominates the top of the page and answers "how's my day?" instantly, with the **log demoted** to a calm, small-type supporting scroll beneath it.

This is a **web-only redesign** (plus docs bookkeeping). It rebuilds the `/nutrition` daily view against the **existing** data layer — `listNutritionLog`, the client-side macro summation, `getMacroGoals` / `putMacroGoals`, the full entry CRUD (`createNutritionLogEntry` / `createCustomNutritionLogEntry` / `updateNutritionLogEntry` / `deleteNutritionLogEntry`), and the Pantry / Recipes views — restyling the composition, not the plumbing. **There is no backend change.** The SOW inherits the DX's `scope: in-system`: dark slate, the single violet accent, Nunito, and the macro tints **as shipped** (`lib/macro-colors.ts` — protein emerald, carbs amber, fat pink; calories deliberately uncolored), per [`../design-system.md`](../design-system.md).

## Proposed Solution

One shippable change: rebuild the `/nutrition` daily view into the dashboard-hero composition. The variant on the DX branch (`app/design-explore/nutrition-page/_components/variant-dashboard-hero.tsx` + `_fixtures.ts`) is the **visual spec**, not code to promote — it's reimplemented production-quality against the real route, wired to the real data and the real CRUD instead of fixtures.

The composition, top to bottom:

- **Day strip** — the week of date tiles, restyled to the quieter pill treatment (selected day = violet pill, others faint) but keeping the real `DateTileStrip` behavior (7 tiles desktop / 3 mobile, prev-next chevrons, the "Today" snap).
- **Hero band** (`rounded-3xl` surface, soft shadow, generous padding; column on mobile → row at `lg`) — the visual dominant:
  - a **composite calories ring** (large SVG donut, violet `var(--accent)` fill over a `var(--surface-2)` track) with the big calorie numeral (`text-6xl font-extrabold tabular-nums`), an "of {goal} kcal" sublabel, and a "{n} left" remaining badge in the center;
  - **three macro numerals** (Protein · Carbs · Fat) in a `sm:grid-cols-3` grid, each a `HeroMacro`: small uppercase label, a huge numeral colored by its **macro tint**, the "/{goal}g", a thin budget bar, and a "{pct}%" caption.
- **Over-goal encoding** — when a macro exceeds its goal, its numeral, budget bar, **and** percentage caption all flip from the macro tint to `var(--warning)` (amber) and the caption gains a "· OVER" tag (fat on the representative day → amber "132% · OVER"). The bar fills to 100% and stops (it doesn't overflow its track). This makes the over-state unmistakable at a glance — the thing the current page fails at.
- **Action row** — the primary **Quick Add** (violet pill), the secondary **Edit Macros** (outlined), and the **Log / Pantry / Recipes** tab group, restyled to the pill tab-group treatment.
- **Demoted log** — meal sections (Breakfast · Lunch · Dinner · Snacks) as calm neutral cards: a meal header with a small subtotal (`"670 cal · P 38g · C 63g · F 30g"`), then entry rows in small muted type (name · ×quantity · calories). The empty Snacks bucket renders its header with "nothing logged"; the long custom Chipotle entry wraps safely (`min-w-0 flex-1`) while its calorie figure stays right-aligned (`shrink-0`).

Crucially, the demoted log in the mock is **display-only** — production must keep the full **entry edit/delete** affordances (desktop pencil/trash, the mobile `EntryActionSheet`) inside the calmer visual. The mock shows the resting state, not the interactions.

## Goals and Non-Goals

### Goals

- **Rebuild the `/nutrition` daily view** (`app/(app)/nutrition/page.tsx` and the components it renders) into the dashboard-hero composition: quiet day strip → dominant hero band → action row → demoted meal-bucketed log.
- **Build the hero band** as the page's visual lead: a composite calories ring (violet accent fill, big calorie numeral + "of {goal} kcal" + "{n} left" badge) and three big macro numerals (Protein/Carbs/Fat) each with a budget bar and percentage, at the DX's dramatic type-scale contrast (hero numerals Nunito-800 5xl–6xl; the log demoted to small neutral type).
- **Make the over-goal state unmistakable**: a macro over its goal flips its numeral, budget bar, and caption to `var(--warning)` and appends "· OVER"; the bar fills to 100% and stops. Replaces today's quiet amber caption-only treatment, applied consistently for any macro (and calories) that goes over.
- **Reuse the existing data layer unchanged** — `listNutritionLog(token, {date, timezone})`; the **client-side** macro summation (sum entries in a `useMemo`, no daily-macros endpoint — per the standing decision); `getMacroGoals` / `putMacroGoals`; entry CRUD (`createNutritionLogEntry`, `createCustomNutritionLogEntry`, `updateNutritionLogEntry`, `deleteNutritionLogEntry`); `listPantryItems` / `listRecipes`. No new endpoints, no SDK changes.
- **Preserve the full entry CRUD inside the demoted log**: edit and delete each entry (desktop pencil/trash, mobile `EntryActionSheet`), with the existing optimistic state updates and toasts. The calmer visual must not lose the interactions.
- **Preserve every fixed point** from the DX/current page: meal-bucketed log grouped by the entry `meal` field (`breakfast`/`lunch`/`dinner`/`snack`, labels Breakfast/Lunch/Dinner/**Snacks**) with per-meal subtotals; the empty-bucket "nothing logged" state; long custom-entry name wrapping with right-aligned calories; the Quick Add and Edit Macros actions; the Log/Pantry/Recipes tabs (`?view=`); timezone-aware day selection (`toLocalYMD`, `startOfLocalDay`, the `timezone` param); the date strip's prev/next/Today navigation.
- **Preserve the goals-never-set empty state**: when `MacroGoals.created_at === null` (or a goal is 0), the hero degrades gracefully — show consumed figures with an inviting "set your goals" affordance instead of a broken ring/over-state, matching today's graceful degradation.
- **Use the shipped macro tints** from `lib/macro-colors.ts` (`MACRO_COLORS.protein` emerald, `.carbs` amber, `.fat` pink; **calories uncolored** — the ring uses violet `var(--accent)`, not a macro tint). No new colors.
- **Conform to the design system** ([`../design-system.md`](../design-system.md)): dark slate ramp, the single violet accent, Nunito, `rounded-2xl`/`rounded-3xl` cards, full-pill chips/buttons, soft shadows — via the existing CSS-variable tokens (`var(--surface)`, `var(--surface-2)`, `var(--border)`, `var(--border-strong)`, `var(--muted)`, `var(--faint)`, `var(--accent)`, `var(--accent-fg)`, `var(--warning)`), not new hex.
- **Intentional at every state**: the representative day (partial log, fat 132% over, the long Chipotle entry, empty Snacks); a clean under-goal day; the goals-never-set day; and a **narrow/mobile** width where the hero stacks (ring above the macro grid) and the log reflows without breaking.
- Keep web CI green (lint/format/typecheck/test/build), adding tests for the new hero and over-goal logic (the daily view currently has no dedicated test).

### Non-Goals

- **Any backend change.** Every endpoint already exists; `prog-strength-api` is not in `repos:`. No schema, route, or SDK changes; macro totals continue to be summed **client-side** from loaded entries (the daily-macros endpoint is for surfaces that don't load entries, not this one).
- **Redesigning Pantry and Recipes.** The Log/Pantry/Recipes tab group is restyled, but `PantryView` and `RecipesView` (and their search/pagination/CRUD) keep their current internals; this SOW redesigns the **daily Log view** and the summary, not the catalogs. (A separate polish SOW already covers catalogs.)
- **Changing the Quick Add / Edit Macros / Log-item / Edit / Delete modal *flows*.** They're re-skinned to match but keep their behavior, validation (the 4-4-9 calorie hint, the per-macro caps), meal inference, and timezone logic.
- **Adding photo meal logging** or any new logging modality to this page — out of scope.
- **Changing meal bucketing.** Entries are grouped by the existing `entry.meal` field as today; no client-side time-of-day re-bucketing.
- **Promoting the DX mockup code.** `variant-dashboard-hero.tsx` + `_fixtures.ts` are the visual spec, not code to copy; the `design-explore/nutrition-page` route stays gated (`designExploreEnabled` / `NEXT_PUBLIC_DESIGN_EXPLORE`), untouched, and never ships to production.
- **The other four variants.** dense-tracker-table, macro-budget-bars, timeline-feed, and unified-canvas are not built; the DX PR is closed (never merged).

## Implementation Details

### Web: the nutrition daily view (`prog-strength-web`)

Rebuild `app/(app)/nutrition/page.tsx` and its daily-view components into the dashboard-hero composition, rendered inside the app shell. Token conformance first: reuse the existing CSS-variable tokens and `lib/macro-colors.ts` — reference tokens, not new hex. Keep `NutritionPageInner`'s data orchestration (the `date`/`entries`/`goals`/`pantry`/`recipes` state, the `?view=` tab routing, the timezone resolution, the `totals` `useMemo`); this is a presentation rebuild over that orchestration.

- **HeroBand** (`_components`) — replaces the current four-ring `MacroGoalRings` block at the top of the daily view. Takes `totals`, `goals`, and renders:
  - **CaloriesRing** — large SVG donut (violet `var(--accent)` fill over `var(--surface-2)` track, capped at 100% sweep), centered calorie numeral (`text-6xl font-extrabold tabular-nums`), "of {goal} kcal" sublabel, "{n} left" badge. Calories is intentionally uncolored (no macro tint).
  - **HeroMacro** ×3 (Protein/Carbs/Fat) — uppercase label, big numeral (`text-5xl font-extrabold`) colored by `MACRO_COLORS[macro]`, "/{goal}g", a thin budget bar (`bg-[var(--surface-2)]` track, tint fill capped at 100% width), and a "{pct}%" caption. **Over-goal** (`pct > 100`): numeral, bar fill, and caption all use `var(--warning)`, and the caption appends " · OVER". Factor the over/under color decision into one helper so the three macros and the ring stay consistent.
  - **Goals-never-set** path: when `goals.created_at == null` (or a goal is 0), render consumed figures without an over-state and surface an inviting "Set your goals" prompt wired to the existing `MacroGoalsModal` — match the current empty-state copy/intent.
- **DayStrip** — keep `DateTileStrip`'s behavior (7/3 tiles, chevrons, Today snap, `useSyncExternalStore` breakpoint) and restyle the tiles to the quieter pill treatment (selected = `bg-[var(--accent)] text-[var(--accent-fg)]` pill; others `text-[var(--faint)]`).
- **ActionRow** — Quick Add (violet pill, opens `QuickAddModal`), Edit Macros (outlined, opens `MacroGoalsModal`), and the Log/Pantry/Recipes pill tab group bound to `?view=`. Keep the mobile icon-only/desktop-labeled `ToolbarButton` ergonomics.
- **Demoted log** — rebuild `components/nutrition/nutrition-log-view.tsx`'s presentation into the calm meal-section cards: header (meal label + subtotal `"{cal} cal · P {p}g · C {c}g · F {f}g"`, or "nothing logged" when empty), then small-type entry rows (`resolveItemName` for the display name, ×quantity, right-aligned calories). **Keep the edit/delete affordances**: the desktop pencil/trash and the mobile `EntryActionSheet`, wired to the existing edit/delete modals and optimistic state updates. Long names wrap (`min-w-0 flex-1`); calories stay `shrink-0`.
- **Modals** — `QuickAddModal`, `MacroGoalsModal`, `LogItemModal`, `LogEntryEditModal`, `LogEntryDeleteModal` keep their flows; restyle only for token/visual consistency with the new page.

The DX `_fixtures.ts` (the representative-day numbers, the entry/meal shapes) is the source of the visual spec and the test fixture; reimplement against the real `NutritionLogEntry` / `MacroGoals` types from `lib/api.ts` rather than importing from the gated `design-explore` tree.

### Tests

- **Web** (new — the daily view currently has none): hero tests — the calories ring renders the consumed/goal numerals and "{n} left"; each `HeroMacro` renders numeral, goal, bar, and percentage from a known total/goal fixture; the **over-goal** macro (fat at 132% on the representative day) renders the numeral/bar/caption in the warning color with the "· OVER" tag while under-goal macros stay tinted; the goals-never-set fixture renders the consumed figures + "set your goals" prompt and no over-state. Log tests — entries group into Breakfast/Lunch/Dinner/Snacks by `meal`; the empty Snacks bucket shows "nothing logged"; the long custom entry wraps with calories right-aligned; **edit and delete still fire** their mutations and optimistically update from a row. Keep the existing API/toast mocks. lint/format/typecheck/test/build green.

## Rollout

1. **`prog-strength-web`** — the nutrition daily-view rebuild. Vercel preview to verify the hero band leads the page, the over-goal (fat 132%) reads as unmistakably amber + "OVER", the demoted log keeps edit/delete, the empty Snacks bucket and long Chipotle entry render cleanly, the goals-never-set state degrades gracefully, and the narrow/mobile reflow — then confirm a real add/edit/delete persists end-to-end against the API.
2. **`prog-strength-docs`** — flip `dx/nutrition-page.md` to `status: selected` (idiom `dashboard-hero`); mark this SOW shipped on merge.

### Verification after rollout

- `/nutrition` (signed in): a quiet day strip, then a dominant hero band — composite calories ring (calories number, "of {goal} kcal", "{n} left") and three big tinted macro numerals each with a budget bar and percentage — answering "how's my day?" in the first glance, on the app's slate/violet/Nunito theme with the shipped macro tints.
- A macro over goal (e.g. fat) shows its numeral, bar, and caption in amber with a "· OVER" tag and the bar pinned at 100% — unmistakable, unlike the old quiet caption.
- The log below is demoted to calm small type, grouped Breakfast/Lunch/Dinner/Snacks with per-meal subtotals; the empty Snacks bucket shows "nothing logged"; the long custom Chipotle entry wraps with calories right-aligned.
- Adding (Quick Add), editing, and deleting entries still works from the demoted log and persists; macro totals (summed client-side) and the hero update immediately.
- Quick Add, Edit Macros, and the Log/Pantry/Recipes tabs all work; Pantry and Recipes views are unchanged apart from the restyled tab group.
- With goals never set, the hero shows consumed figures and an inviting "set your goals" prompt rather than a broken ring/over-state.
- At a narrow/mobile width the hero stacks (ring above the macro grid) and the log reflows — nothing breaks.
- The `design-explore/nutrition-page` route remains gated/404 in production; no DX mockup code shipped.

## Open Questions

- **Calories over-goal treatment** — the macros flip to amber + "OVER"; should the calories ring do the same when consumed > goal (the ring sweep is already capped at 100%)? Lean: yes, flip the ring fill and "{n} left" badge to the warning color and show "over by {n}" so calories isn't the one metric that stays calm when blown — keeps the over-encoding consistent. Confirm at implementation.
- **Demoted-log macro detail on desktop** — the DX log row shows only name · quantity · calories, dropping the current desktop table's per-row P/F/C columns. Lean: keep the row minimal per the chosen variant (the hero carries the macro story; per-meal subtotals carry the breakdown), and surface per-entry macros in the edit modal — but if losing the at-a-glance per-row macros is missed, a compact "P/C/F" line under the name is the fallback. Visual call on the preview.
- **Ring vs. bars for calories** — dashboard-hero uses a ring for calories and bars for macros (a deliberate two-encoding hero). Lean: keep as the variant specifies (the asymmetry is the point — calories is the headline, macros are the budget); flagged only so a reviewer can veto the mixed encoding before build.
</content>
