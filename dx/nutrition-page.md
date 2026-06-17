---
type: dx
status: awaiting_selection
surface: nutrition-page
idioms:
  - dashboard-hero
  - dense-tracker-table
  - macro-budget-bars
  - timeline-feed
  - unified-canvas
references:
  - MyFitnessPal
  - MacroFactor
  - Cronometer
  - Whoop
  - Linear
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Nutrition Page

**Status**: Awaiting selection · **Last updated**: 2026-06-17

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

The nutrition page (`/nutrition`) is where a Prog Strength user lands every time
they eat — log a meal, check whether they've hit protein, see how much of the
day's budget is left. It's one of the highest-frequency surfaces in the product:
a user opens it three or four times a day, far more than they open the calendar.
The **functionality is solid and not in question here** — the four macro rings
compute against goals, the day strip navigates dates, entries bucket into
breakfast/lunch/dinner/snack with per-meal subtotals, and Quick Add, Edit
Macros, Pantry, and Recipes all work. What's unsatisfying is the **look and the
composition**.

Three things specifically drag it down today:

- **The log reads spreadsheet-y.** Each meal is a bare data grid — `ITEM / SERV /
  CAL / P / F / C` columns of small text with pencil/trash glyphs on the right.
  It's a database table, not a food log you enjoy keeping. It's the densest,
  most-looked-at part of the page and the least considered.
- **The hero summary lacks punch.** The four rings plus the day strip are
  competent but flat — they don't deliver the "how's my day going?" hit that the
  nutrition apps this user lives in nail in the first half-second.
- **The tabs and actions feel bolted on.** `Log · Pantry · Recipes` plus
  `+ Quick Add` and `✎ Edit Macros` read as five separate features stapled into
  one toolbar rather than one coherent surface with a clear primary job.

This DX exists to find the *direction* for a nutrition page that feels
purpose-built for daily tracking — before committing engineering effort to
building one. `scope: in-system`: the foundation is already decided (dark slate,
the violet accent, Nunito, and the macro tints the design system defines
explicitly "for nutrition macro chips"), so the variants do **not** re-litigate
palette, accent, or type. They diverge on **layout, structure, density, and
composition** — which element is the hero, how dense the log is, and how the
logging/catalog actions cohere. No production nutrition code changes here.

## The surface

The `/nutrition` daily view. Anatomy of what's on screen today (a variant should
account for all of it, though it may reorganize, demote, or fold chrome together
in service of its idiom):

- **Day strip** — a horizontal week of date tiles (`WED 10 … TUE 16`) with
  prev/next chevrons and a "Today" jump; the selected day is accent-filled.
- **Macro summary ("TODAY")** — four rings: **Calories** (`2560 / 3400 kcal`,
  75%), **Protein** (`140 / 185 g`, 76%), **Carbs** (`264 / 400 g`, 66%), **Fat**
  (`105.5 / 80 g`, 132% — *over goal*, called out in amber). Each ring shows a
  percentage in its center and the consumed/goal figure beneath.
- **Action row** — `+ Quick Add` and `✎ Edit Macros` on the left; the
  `Log · Pantry · Recipes` tab group on the right.
- **The log** — entries grouped by **meal** (Breakfast / Lunch / Dinner /
  Snacks), each section showing a subtotal header (`670 cal · P 38g · F 30g ·
  C 63g`) and a table of item rows: **Item**, **Serv**, **Cal**, **P**, **F**,
  **C**, plus edit/delete affordances. A custom free-text entry (e.g. the long
  Chipotle bowl line) wraps to multiple lines.
- **Pantry / Recipes tabs** — catalog grids of saved foods and recipes (search,
  letter-bucketed, paginated); each card logs into the day or opens an edit form.

The data shapes (from `prog-strength-web/lib/api.ts`):

```ts
type MacroGoals = {
  protein_g: number; carbs_g: number; fat_g: number; calories: number;
  created_at: string | null; updated_at: string | null; // null ⇒ never set ⇒ render empty-ring outline
};

type MealType = "breakfast" | "lunch" | "dinner" | "snack";

type NutritionLogEntry = {
  id: string; consumed_at: string;
  pantry_item_id?: string | null; recipe_id?: string | null;
  custom_meal_name: string | null;          // non-null ⇒ free-text entry; stores totals, not a per-serving formula
  quantity: number;                          // the "Serv" column
  calories: number; protein_g: number; fat_g: number; carbs_g: number; // denormalized at log time, immutable
  meal: MealType; created_at: string;
};

type PantryItem = {
  id: string; name: string;
  calories: number; protein_g: number; fat_g: number; carbs_g: number; // per serving
  serving_size: number; serving_unit: string;                          // "1 egg", "100 g"
  created_at: string; updated_at: string;
};

type Recipe = {
  id: string; name: string;
  components: { id: string; pantry_item_id: string; quantity: number; position: number }[];
  macros: { calories: number; protein_g: number; fat_g: number; carbs_g: number }; // computed on read
  created_at: string; updated_at: string;
};
```

Daily totals and per-meal subtotals are the same accumulated shape
(`{ calories, protein_g, fat_g, carbs_g }`), summed over the day's entries and
over each meal bucket respectively. Each of the three macros has a **fixed tint**
(implemented in `macro-colors.ts`: protein green, carbs amber, fat pink — note
this is what ships today and what the screenshot shows; `design-system.md` still
records an older carb-blue / fat-amber mapping, a discrepancy to reconcile
out-of-band) and **calories is intentionally uncolored** (it reads in the page
foreground). Variants **reuse the existing macro tints as shipped**; they do not
introduce new macro colors.

**Visual states the variants must handle** (this is where lazy designs fall
apart, so render them all in the mockup):

- **Under, at, and over goal.** Three of four macros are under (calories 75%,
  protein 76%, carbs 66%); **fat is over at 132%** and must read as *over* — not
  just "more ring." The over-goal treatment is the single most important state on
  this surface.
- **Goals never set.** `MacroGoals` with null timestamps means the user has no
  targets yet — the hero should degrade to an inviting "set your goals" empty
  state, not a row of 0% / divide-by-zero rings.
- **A meal with several items** vs **an empty meal bucket.** Breakfast has 2,
  Lunch 3, Dinner 1; Snacks is empty. Empty buckets should feel intentional (or
  collapse), not broken.
- **A long custom entry.** The Chipotle line ("double-steak bowl with brown rice,
  black beans, queso, pico de gallo, cilantro lime sauce, and nacho chips")
  wraps; the row must stay legible and keep its macros aligned.
- **A near-empty day.** Early in the morning the log is empty and the rings sit
  near 0% — the page should still feel like a place to start, not a void.
- **Branded item names.** Real entries are long and brand-heavy ("1st Phorm
  Birthday Cake Protein Powder", "Elev8 - Creamy Rice (Peanut Butter Cup)") —
  the item treatment can't assume short names.

**Representative fixture** (use something like the screenshot so variants render
realistically — a partially-eaten day, one macro over, a long custom entry, and
an empty bucket):

- Day: **Tue, Jun 16**. Goals: 3400 kcal · P 185g · C 400g · F 80g.
- **Breakfast** — *670 cal · P 38g · F 30g · C 63g*: Egg ×5 (350 / 30 / 25 / 5);
  La Favorita Tortillas ×2 (320 / 8 / 5 / 58).
- **Lunch** — *510 cal · P 35g · F 16g · C 60g*: PBJ Grape Uncrustables ×1 (260 /
  8 / 13 / 31); Elev8 - Creamy Rice (Peanut Butter Cup) ×1 (130 / 3 / 1 / 27);
  1st Phorm Birthday Cake Protein Powder ×1 (120 / 24 / 2 / 2).
- **Dinner** — *1380 cal · P 67g · F 59.5g · C 141g*: the long custom Chipotle
  double-steak bowl ×1 (1380 / 67 / 59.5 / 141).
- **Snacks** — empty.
- Day total: **2560 / 3400 kcal · P 140/185 · C 264/400 · F 105.5/80 (over)**.

These are mockups: **static fixtures that look real are preferred** — do not wire
the variants to live nutrition services.

## Idioms

Five genuinely distinct compositions of the *same* dark-slate / violet / Nunito /
macro-tint surface. Because `scope: in-system`, none of them re-decides palette,
accent, or type — they diverge along **type scale** (within Nunito's 600/700/800
range and how hard they lean on size vs weight for hierarchy), **color logic**
(how aggressively the fixed macro tints and the violet accent are deployed vs
held back), and **spacing rhythm** (airy hero ↔ dense ledger). Each makes a
*different element the hero* and leans on a different reference app. The Fixed
Points hold: the "P" identity and the dark theme stay.

- **dashboard-hero** — The day's macro state is the star. An **oversized hero
  band** dominates the top: large rings (or one large composite ring) and **big,
  confident macro numerals** in Nunito 800, with the log demoted to a calmer
  scroll beneath. Dramatic type-scale contrast — hero numerals are huge, log text
  small. Color logic: the **macro tints carry the hero** (they're the loudest
  thing on the page) while the log stays mostly neutral. Spacing rhythm: generous
  and breathable up top, the page opens with a "how's my day?" hit à la
  **Whoop's** today screen. → answers *punchless hero*.

- **dense-tracker-table** — The **log is the star**. A tight, scannable food
  **diary ledger** in the **MyFitnessPal / Cronometer** tradition: compact rows,
  running daily totals and **goal-remaining math inline** ("Protein 45g left"),
  the four rings shrunk to a slim summary strip so the entries get the screen.
  Tight, uniform type scale; hierarchy from weight and column alignment, not size.
  Color logic: **restrained** — macro tints reduced to thin column accents or
  small figures, never filled blocks; violet only on the active add affordance.
  Spacing rhythm: dense, gridded, maximally informative per square inch — the
  power-user end of the spread. → owns the *density* end and makes the log feel
  *designed*, not tabular-by-default.

- **macro-budget-bars** — Reframes the day as a **budget you spend down**.
  Horizontal **consumed-vs-remaining bars** replace the rings as the hero (à la
  **MacroFactor's** energy budget): each macro is a full-width track, the filled
  portion in that macro's tint, the remainder a faint slate track, with the
  *remaining* number called out rather than the consumed one. Over-goal (fat) is
  the bar overflowing its track / flipping to the amber over-state — a naturally
  legible encoding of the 132% case. Mid type scale, mid density. Color logic:
  **each macro tint owns its bar**; the page is otherwise neutral. Spacing rhythm:
  calm horizontal rhythm, the bars stacked and even. → a *different mental model*
  of the same numbers, and the cleanest take on the over-goal state.

- **timeline-feed** — The day as a **chronological feed of meals**, ordered by
  `consumed_at` rather than bucketed into a four-section table. Each meal is a
  **card** — a heading, a small per-card macro summary chip-row, and its items as
  soft rows rather than table cells — so the day reads like a **journal** you're
  keeping, not a database you're filling. Looser, editorial leading; comfortable
  Nunito scale with more air between entries. Color logic: macro tints appear as
  **small per-card chips**, the day total as a sticky compact summary. Spacing
  rhythm: card-based and generous. → the most direct answer to *spreadsheet-y*;
  trades raw density for warmth and narrative.

- **unified-canvas** — Attacks the **bolted-on tabs** head-on by dissolving
  `Log / Pantry / Recipes` and the Quick Add modal into **one two-pane working
  surface** (a **Linear**-style composer): the **day log lives in one column**
  and a **live search-and-add picker** (pantry + recipes + custom, unified) lives
  in the other, so logging is an inline act on the page rather than a tab switch
  and a modal. The macro summary compresses to a persistent header rail across
  the top. Crisp, functional type; keyboard-first density. Color logic:
  **restrained and structural** — violet marks the active add target and
  selection, macro tints stay as small figures. Spacing rhythm: two-pane,
  efficient, composable. → re-thinks the *information architecture* itself, not
  just the skin.

## References

In-system, so "what to take" from each is **structural** — composition, density,
and information design, not their palettes or type:

- **MyFitnessPal** (food diary) — take its **meal-bucketed running ledger** and
  the **fixed daily macro header** that's always in view while you scroll the day.
  Drives `dense-tracker-table`.
- **MacroFactor** (dashboard) — take its **energy/macro *budget* framing**:
  consumed-vs-remaining bars, "remaining" as the headline number, and its calm,
  uncluttered hero. Drives `macro-budget-bars`.
- **Cronometer** (diary + targets) — take its **per-entry density** and its
  at-a-glance **targets panel** where you instantly see what's met and what's
  short. Informs `dense-tracker-table` and the goal-remaining math.
- **Whoop** (today screen) — take its **big, confident hero metric** and the
  first-half-second "here's your day" hit. Drives `dashboard-hero`.
- **Linear** (composer / two-pane) — take its **two-pane composability,
  keyboard-first density, and single-accent restraint** — logging as an inline
  act, not a modal. Drives `unified-canvas`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I'm trying to decide:

- **Does it answer "how's my day going?" in the first half-second?** This is the
  page I open most — protein-to-go and calories-left should land instantly,
  before I read anything.
- **Is the over-goal state unmistakable?** Fat at 132% has to *read as over* at a
  glance — that's the signal that changes what I eat next, and the current ring
  barely whispers it.
- **Does the log finally stop feeling like a spreadsheet** while staying fast to
  scan and quick to add to? Density and warmth are in tension here; I want to feel
  which trade I actually prefer day to day.
- **Do Pantry/Recipes and logging feel like one surface** rather than stapled
  tabs — is adding a food a smooth inline act?
- **Does it still read as Prog Strength** — native to the dark slate / violet /
  Nunito system, not a re-skin — while clearly beating the nutrition apps I
  actually use?
- The long custom Chipotle entry and the empty Snacks bucket should both survive
  the treatment without looking broken.

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/nutrition-page` branch as it opens the PR; the owner sets the
> terminal value when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the
> draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy, pick
> one, tick its box, set `status: selected` (noting the winning idiom), and
> **close the PR — never merge it.** Then I open a SOW: *"implement nutrition-page
> per the `<chosen-idiom>` variant from `dx/nutrition-page`, production-quality,
> conforming to the design system."*
