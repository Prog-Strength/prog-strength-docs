---
status: ready_for_implementation
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Nutrition Log + Pantry/Recipes Catalog Polish

**Status**: Ready for Implementation · **Last updated**: 2026-06-04

## Introduction

The nutrition page on the web app has accumulated enough surface area — daily log, pantry catalog, recipes catalog — that the visual treatment of each view starts to matter. Today the daily log dedicates a `Time` column to a piece of information that is genuinely low-value (the user knows when they ate lunch; the column exists because the database stores `consumed_at`) and the cost is real: item names get truncated mid-word so dishes like "Grilled chicken salad, mixed greens" arrive as "Grilled chicken sa…" in the row. The meal-section header line shows totals as `632 cal · P 41g · F 18g · C 76g`, monochrome muted gray, with no visual link to the colored ring chart sitting twelve pixels above it. The pantry and recipes catalogs render one full-width row per item — fine for ten items, wasteful at fifty, and the only way to edit is to click the entire row (which doubles as the only way to discover that editing is possible). The "+ Add" buttons on those views use a bordered control style that the rest of the page has otherwise moved off of in favor of the unobtrusive ghost-text-button pattern shared by `Quick Add`, `Log`, and `Set Goals`.

This SOW bundles a single coordinated round of web-only polish that closes those gaps. The Time column on the log table comes out, and the reclaimed width goes to the item name. The meal-totals header line picks up the same emerald/amber/pink palette the ring chart uses, applied as letter-only color so the line reads as glanceable data without becoming a decorative noise pattern. The pantry and recipes catalogs move from a single-column list to a two-column grid under each alphabetical letter header, drop their per-row "per X g" / "per batch" suffix (still visible when the user opens the edit modal), and gain explicit pencil and trash icons on each item so the action vocabulary matches the bodyweight table and the log table. The bordered Add button is replaced by a ghost text-button matching the rest of the page.

Nothing changes on the API, the MCP layer, the agent, or the mobile client. The macros stored in the database don't move; the time stored on a log entry doesn't move. This is a one-PR cosmetic + interaction polish pass on `prog-strength-web`.

## Proposed Solution

The macro color palette already lives in `components/macro-goal-rings.tsx:15` as a `COLORS` const — emerald-300 for protein, amber-300 for carbs, pink-300 for fat, the page foreground for calories. The first move is to extract this into a shared `lib/macro-colors.ts` module so the log view, the pantry view, and the recipes view can all import one source of truth. The ring chart re-imports from there; pixel for pixel identical.

The log view (`components/nutrition/nutrition-log-view.tsx`) loses its `Time` column. The `consumed_at` timestamp still lives on the entry, still ships in the API response, and is still editable via the existing `LogEntryEditModal`'s Time input (`log-entry-edit-modal.tsx:155`) — none of that changes. The column just stops being a column. The Item cell's `max-w-0` constraint comes off so the reclaimed width goes to the item name, and the column header letters `P`, `F`, `C` are colored against the shared palette (calories stays muted; calories has no ring color of its own). Per-row P/F/C values stay muted to avoid a Christmas-tree effect; the column header letter is the visual anchor.

The meal-section totals header line — currently `450 cal · P 30g · F 18g · C 35g`, all muted — picks up letter-only color: the `P`, `F`, `C` glyphs render in their palette colors, the numeric values stay muted. The treatment is identical across log meal totals, pantry item lines, and recipe item lines so the page reads as one visual system rather than three near-misses.

The pantry view (`components/nutrition/pantry-view.tsx`) drops the whole-row clickable `<button>` in favor of an inert info card with two icon buttons on the right — a muted-gray pencil that opens the existing `PantryItemModal` in edit mode, and an always-red trash that opens a new `PantryItemDeleteModal` (a thin confirmation shell paralleling `LogEntryDeleteModal`). The rows themselves move from a vertical `flex-col` list under each letter header to a two-column `grid grid-cols-2 gap-2` so each letter group shows twice as many items per scroll. The macro line drops its `g` suffix and its "per X g/oz" trailing chip — the modal already shows serving size, the row doesn't have to. `PAGE_SIZE` bumps from 25 to 40 since each row is now half the visual height and the same scroll budget fits more items. Search is unchanged.

The recipes view (`components/nutrition/recipes-view.tsx`) mirrors pantry exactly: two-column grid, inert info card, pencil + trash icons, drop the "per batch" suffix, `PAGE_SIZE = 40`, new `RecipeDeleteModal` paralleling the pantry one. The two views are deliberately built from the same vocabulary because they behave the same way; if a single shared `<CatalogList>` helper falls out of the refactor naturally, take it. Don't force one if it doesn't.

The bordered `+ Add` button on both catalog views is replaced by the same ghost text-button pattern the bodyweight page's `Log` button uses (`app/(app)/bodyweight/page.tsx:464`): `inline-flex items-center gap-1.5 text-sm font-medium text-foreground transition hover:opacity-70`, with a plus icon and "Add" text. Same vocabulary as `Quick Add` and `Set Macros` on the existing nutrition toolbar (`app/(app)/nutrition/page.tsx:233-242`). No outline, no fill, hover-fades for affordance.

The pantry and recipes views currently rely on the entire row being a `<button>` to surface "edit me" — replacing that with explicit pencil affordances makes the action discoverable rather than implicit. The trash icon turns delete from a modal-internal action (currently only reachable by opening the edit modal and clicking the Delete button at the bottom) into a one-click confirm-modal flow, which matches the log table and the bodyweight table.

No data model changes. No API changes. No new endpoints. No backend tests. The work is entirely in `prog-strength-web`.

## Goals and Non-Goals

### Goals

- Extract the macro color palette currently inline in `components/macro-goal-rings.tsx:15` to `lib/macro-colors.ts` and re-import from the rings component. Hex values stay identical; no visual change to the rings. Shared by the log view, pantry view, and recipes view.
- Remove the `Time` column from the nutrition log table (`components/nutrition/nutrition-log-view.tsx`). Drop the `<th>` and the per-row `<td>` and the local `formatLocalTime` helper if it has no other consumers. `consumed_at` continues to ship on the wire and remains editable via the existing `LogEntryEditModal` Time input — no API change.
- Remove the `max-w-0` constraint on the Item cell in the log table so item names occupy the reclaimed width and stop truncating mid-word for plausible meal names.
- Color the log table column headers `P`, `F`, `C` with the shared palette (emerald / amber / pink); leave per-row values muted. Calories column header stays muted (no ring color allocated).
- Color the meal-section totals header line with letter-only treatment: `P`, `F`, `C` glyphs in palette colors, numeric values muted. Same treatment for every meal section (Breakfast, Lunch, Dinner, Snacks).
- Replace the bordered `+ Add` button on both `PantryView` and `RecipesView` with the ghost text-button pattern matching the bodyweight page's `Log` button: plus icon + "Add" text, no border, no fill, hover-fades.
- In `PantryView`: replace the whole-row clickable `<button>` with an inert info card that has the name + macros on the left and a pencil + trash icon group on the right. The pencil opens the existing `PantryItemModal` in edit mode. The trash opens a new `PantryItemDeleteModal`.
- Lay out the pantry rows beneath each letter header as a two-column grid (`grid grid-cols-2 gap-2`) so each letter group shows twice the items per visible height.
- Drop the `g` suffix on the pantry macro line and remove the "per X g/oz" serving suffix from the row. Serving size remains editable in the modal.
- Apply the same letter-only macro color treatment to the pantry row info: `P` emerald, `F` pink, `C` amber, calories muted.
- Bump `PAGE_SIZE` in `PantryView` from 25 to 40. The row is half the height; the scroll budget fits more.
- Create `PantryItemDeleteModal` paralleling `LogEntryDeleteModal`: confirmation modal shell, title "Delete this pantry item?", explanatory body noting that historical log entries that reference the item keep their frozen macros (already the storage behavior), Cancel / Delete (red).
- Apply every pantry change to `RecipesView` identically: two-column grid, inert info card, pencil + trash icons, drop the "per batch" suffix, letter-only macro coloring, `PAGE_SIZE = 40`, new `RecipeDeleteModal`.
- Preserve the pantry and recipes search input. Unchanged behavior.
- Preserve the per-letter alphabetical grouping. The grid sits under each `<h3>` letter header inside its own `<div>`, so the visual rhythm of the page stays the same.
- Preserve all existing modal flows. The pantry add modal (PantryItemModal in create mode), the pantry edit modal (same component in edit mode), the recipe add/edit modal — none of these change shape; they just get reached through new affordances.

### Non-Goals

- **API or schema changes.** `consumed_at` stays in the database and ships on the wire; the time isn't displayed on the log row but the storage and edit paths are untouched. No new endpoints. No migrations.
- **MCP / agent / mobile.** Cosmetic web polish; the agent doesn't render any of these surfaces and the mobile client has its own treatment.
- **Pagination redesign.** `PAGE_SIZE` bumps from 25 to 40 in pantry and recipes; the existing prev/next pagination footer stays. No infinite scroll, no virtualization. The catalog isn't large enough to need either.
- **Search redesign.** The existing search input stays as-is. Don't expand it into a filter bar or add tag filters.
- **Modal redesign.** `PantryItemModal`, `RecipeModal`, and `LogEntryEditModal` keep their existing shape. We're adding two new thin delete-confirm modals (`PantryItemDeleteModal`, `RecipeDeleteModal`) and otherwise touching nothing.
- **Forcing a shared catalog component.** Pantry and recipes share enough structure that a `<CatalogList items={...} renderItem={...} />` helper might fall out naturally; if it does, take it. Don't force an extraction that adds props-juggling for the sake of DRY — three repetitions is the threshold, and this is the second.
- **Recoloring calories.** Calories has no ring color (the rings render it in the page foreground, not in the palette). Calorie values and column header stay muted.
- **Coloring per-row macro values.** Only column headers and meal-totals letters get color. Per-row numeric values (the `38g` in the log row, the `6` in the pantry macro line) stay muted to avoid a Christmas-tree effect.
- **Bodyweight or other-page polish.** The bodyweight page got its own SOW (`bodyweight-goal-and-page-polish.md`). This SOW is strictly nutrition / pantry / recipes.

## Implementation Details

### Shared macro color palette

Extract `lib/macro-colors.ts`:

```ts
/**
 * Single source of truth for the macro palette. The rings chart, the
 * log view's meal totals + column headers, and the pantry / recipe
 * catalog item lines all import from here so a one-line tweak retints
 * every surface uniformly.
 *
 * Colors mirror Tailwind's pastel-300 ramp so the values read well
 * against both surface tones used on the nutrition page.
 *
 * Calories has no ring color: it renders in the page foreground in the
 * rings chart, and column headers / labels for calories stay muted in
 * the catalog and log views. Don't add a calorie color here without a
 * design review — it's a deliberate non-allocation.
 */
export const MACRO_COLORS = {
  protein: "#6ee7b7", // emerald-300
  carbs: "#fcd34d", // amber-300
  fat: "#f9a8d4", // pink-300
} as const;
```

Re-import from `components/macro-goal-rings.tsx`: replace the inline `COLORS` literal with the named import; the `calories: var(--foreground)` half stays inline at the component since it's a CSS variable, not a value.

### Log view (`components/nutrition/nutrition-log-view.tsx`)

Remove the Time column entirely:

- Delete the `Time` `<th>` (`:160`) and the per-row `<td>` (`:194`).
- Delete the `formatLocalTime` helper (`:244`) if no other component imports it. (Grep first; if anything else uses it, leave it in.)
- Drop `max-w-0` from the Item cell so the name column grows into the freed width.

Color the column header letters. The current `<th>` for P / F / C is plain text; wrap each letter in a span keyed to `MACRO_COLORS`:

```tsx
<th scope="col" className="py-1.5 pr-3 text-right font-semibold">
  <span style={{ color: MACRO_COLORS.protein }}>P</span>
</th>
```

Apply the same treatment to the meal-section totals header (`MealSection`, around `:148-153`):

```tsx
<p className="text-xs text-[var(--muted)] tabular-nums">
  {formatNumber(subtotal.calories)} cal ·{" "}
  <span style={{ color: MACRO_COLORS.protein }} className="font-semibold">P</span>{" "}
  {formatNumber(subtotal.protein_g)}g · …
</p>
```

Values stay in `text-[var(--muted)]` via the parent; the letter spans override color and weight. Use `font-semibold` on the letter so the color reads even at 11px.

### Pantry view (`components/nutrition/pantry-view.tsx`)

Add button (`:76-83`) → replace the bordered button with the ghost text-button pattern from the bodyweight page (`app/(app)/bodyweight/page.tsx:464`). Extract `<ToolbarButton>` into a shared `components/toolbar-button.tsx` if neither the bodyweight page nor the nutrition page has done so already; if they have local copies, mirror locally. Plus icon component already exists at `app/(app)/nutrition/page.tsx:390` (`PlusIcon`); extract to `components/icons/plus.tsx` along the way, or duplicate locally — pick the smaller change.

```tsx
<ToolbarButton onClick={() => setModal({ mode: "create" })} icon={<PlusIcon />} label="Add" />
```

Rows layout — replace the `<ul className="flex flex-col gap-2">` (`:115-121`) under each letter header with a CSS grid:

```tsx
<div className="grid grid-cols-2 gap-2">
  {g.items.map((p) => (
    <PantryRow key={p.id} item={p} onEdit={() => setModal({ mode: "edit", id: p.id })} onDelete={() => setDeleting(p)} />
  ))}
</div>
```

Rewrite `PantryRow` (`:160`):

```tsx
function PantryRow({
  item,
  onEdit,
  onDelete,
}: {
  item: PantryItem;
  onEdit: () => void;
  onDelete: () => void;
}) {
  return (
    <div className="flex items-center justify-between gap-2 rounded-md border border-[var(--border)] bg-[var(--surface)] px-3 py-2 min-w-0">
      <div className="flex min-w-0 flex-1 flex-col gap-0.5 overflow-hidden">
        <p className="truncate text-sm font-medium">{item.name}</p>
        <p className="text-xs text-[var(--muted)] tabular-nums">
          {formatNumber(item.calories)} cal ·{" "}
          <span style={{ color: MACRO_COLORS.protein }} className="font-semibold">P</span>{" "}
          {formatNumber(item.protein_g)} ·{" "}
          <span style={{ color: MACRO_COLORS.fat }} className="font-semibold">F</span>{" "}
          {formatNumber(item.fat_g)} ·{" "}
          <span style={{ color: MACRO_COLORS.carbs }} className="font-semibold">C</span>{" "}
          {formatNumber(item.carbs_g)}
        </p>
      </div>
      <div className="flex shrink-0 items-center gap-0.5">
        <IconButton aria-label="Edit" onClick={onEdit} tone="muted"><PencilIcon /></IconButton>
        <IconButton aria-label="Delete" onClick={onDelete} tone="danger"><TrashIcon /></IconButton>
      </div>
    </div>
  );
}
```

`IconButton` should match the bodyweight page's pattern (transparent background, `tone="muted"` paints muted at rest + foreground on hover, `tone="danger"` paints danger at rest); reuse if extracted, otherwise mirror locally. The row body is no longer interactive — only the two icons fire callbacks.

Page state change: add a `deleting: PantryItem | null` state and render `<PantryItemDeleteModal>` when it's non-null. The existing `modal` state (edit/create) stays as-is.

Bump `PAGE_SIZE` (`:8`) from 25 to 40.

Drop the `g` suffix and the "per X g/oz" trailing chip from the macro line — both already absent in the snippet above.

### Recipes view (`components/nutrition/recipes-view.tsx`)

Mirror every change above to `RecipesView`:

- Same ghost-text Add button.
- Same 2-column grid under each letter header.
- Rewrite `RecipeRow` to the same inert-info-card shape with pencil + trash. Substitute `recipe.macros.calories` etc. for the pantry's flat fields.
- Same `PAGE_SIZE = 40`.
- Same letter-only macro coloring.
- Drop the "per batch" suffix.
- New `RecipeDeleteModal`.

### New delete-confirm modals

Two thin modals, mirroring `LogEntryDeleteModal`:

`components/nutrition/pantry-item-delete-modal.tsx`:

```tsx
export function PantryItemDeleteModal({
  token,
  item,
  onDeleted,
  onClose,
}: {
  token: string;
  item: PantryItem;
  onDeleted: () => void;
  onClose: () => void;
}) {
  // Modal shell identical to LogEntryDeleteModal: backdrop, centered
  // card, escape-to-close, body-scroll-lock, busy state.
  // Body copy: "Delete \"<item.name>\"? Log entries that already used
  // this item keep their numbers — macros were frozen at log time."
  // Buttons: Cancel | Delete (red, busy → "Deleting…").
  // On Delete: call deletePantryItem(token, item.id), then onDeleted +
  // onClose. Surface 4xx errors inline.
}
```

`components/nutrition/recipe-delete-modal.tsx`: same shape, body copy "Delete the recipe \"<name>\"? Log entries that used this recipe keep their numbers — macros were frozen at log time." Calls `deleteRecipe`.

The existing `deletePantryItem` and `deleteRecipe` API helpers are already in `lib/api.ts`. No new client calls.

### Page-level wiring

The page (`app/(app)/nutrition/page.tsx`) already owns the pantry, recipes, and log-entry refetch callbacks. The two new delete modals' `onDeleted` callbacks bubble up to the existing `onChanged` handler (the same one the edit modals use), which triggers `fetchPantry()` + `fetchRecipes()` for pantry mutations and `fetchRecipes()` for recipe mutations. No new page state.

### Behavior the user might check for

- Time of a log entry is still editable via the pencil icon → opens `LogEntryEditModal` → Time field stays at the entry's current local time, change submits as `consumed_at` ISO update.
- Pantry edit modal still shows serving size + serving unit (it always did; this SOW removes them from the row, not the modal).
- Pantry item delete from inside the edit modal continues to work (the Delete button at the modal footer). The new icon-driven flow is additive; the in-modal Delete stays.
- Catalog search still filters case-insensitively against the item / recipe name; clearing the search resets the page to 1 (already the behavior).

### Tests

Web has no test runner today. Verification is:

- `npx tsc --noEmit` — clean
- `npx eslint app/\(app\)/nutrition components/nutrition components/macro-goal-rings.tsx lib/macro-colors.ts` — clean
- `npx next build` — all pages generate (specifically `/nutrition`)
- Hand-test checklist:
  - **Log row:** open `/nutrition` for a day with at least one logged entry. Confirm no Time column. Confirm long item names ("Grilled chicken salad, mixed greens") render in full. Confirm the column header letters `P`, `F`, `C` are colored emerald / pink / amber. Confirm per-row `P`, `F`, `C` values remain muted.
  - **Meal totals:** confirm each meal section's header (Breakfast / Lunch / Dinner / Snacks) shows `<n> cal · P <n>g · F <n>g · C <n>g` with only the letters colored.
  - **Edit log entry:** click the pencil on a row. Confirm the edit modal still shows + lets you change the Time. Save. Confirm `consumed_at` updates correctly.
  - **Pantry view:** switch to `/nutrition?view=pantry`. Confirm the items render in a 2-column grid under each letter header. Confirm the macro line shows letter-only coloring. Confirm the row body is not clickable (cursor doesn't change to pointer over name). Click pencil → edit modal. Click trash → confirm modal. Confirm Add button is ghost text + plus icon, no border.
  - **Recipes view:** same checklist as pantry against `/nutrition?view=recipes`.
  - **Delete confirm modal:** confirm cancel closes without effect; delete removes the item and refetches.

### Rollout

Single PR on `prog-strength-web`. No API dependency, no migration, no coordination window. Merge whenever. Hand-test in deployed environment after merge.
