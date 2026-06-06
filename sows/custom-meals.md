---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-mcp
  - prog-strength-agent
  - prog-strength-web
  - prog-strength-docs
---

# Custom Meals

**Status**: Shipped · **Last updated**: 2026-06-06

## Introduction

A user opens the Prog Strength chat at 6:42 PM and types "I just had a chicken bowl from Chipotle." The pantry catalog the user has built over a couple of months does not contain "Chipotle bowl" — there's no item to reference, no recipe to reference — and the agent has no way to log the meal. The same gap exists on the web: the Quick Add modal asks the user to pick a saved pantry item or recipe, and if neither exists for what the user just ate, the user has to first go create a pantry item ("Chipotle chicken bowl, 850 cal, 1 serving = 1 meal") and then come back to log it. For one-time meals the round trip is friction that pushes users to skip the log entirely, which compounds — a single missed entry on a tracked day means the daily macros widget on the nutrition page reads as under-eaten and the agent answers "how am I doing today" with a number that disagrees with the user's intuition.

A small data-model widening closes this. The nutrition log already denormalizes macros into every row at log time — the four float columns sit there ready, the soft delete and timestamps already work, the daily aggregation already sums whatever is on the row. The only thing the schema enforces that gets in the way is the CHECK that exactly one of `pantry_item_id` or `recipe_id` is set on every entry. Relaxing that constraint to allow a third option — a row whose source is the user's typing, not a saved item — lets the same log surface absorb restaurant meals, friends-of-friends home cooking, the random protein bar the user bought at the airport, and anything else the user does not want to commit to their pantry.

The agent learns one new tool and one new prompt rule: when the user logs something through chat, the agent searches the pantry first by name and reaches for the new tool only when the pantry comes up empty. After a successful custom-meal log, the agent appends a single short follow-up ask — "Want me to save this to your pantry for next time?" — and pushes the saved-or-not decision to the user. Branded chains a fitness user eats often (Chipotle, Qdoba, Sweetgreen) accumulate into the pantry through that user choice without any heuristic on the agent's side.

The web Quick Add modal grows from a single source dropdown to a three-tab segmented picker — Pantry, Recipes, Custom — and the custom tab is a small form with name + four macro inputs that uses the same emerald / pink / amber palette the rings, log table, and catalog rows already share.

## Proposed Solution

A single additive migration on `nutrition_log_entries` adds a nullable `custom_meal_name TEXT` column and widens the existing two-way exactly-one CHECK to a three-way XOR over `pantry_item_id`, `recipe_id`, and `custom_meal_name`. The Go API gains a new `POST /nutrition-log/custom` endpoint that accepts the user's typed name and macros and inserts a row through the same repository path the existing log endpoints already use. The existing `PUT /nutrition-log/{id}` grows: when the targeted row is a custom entry, the body may also include `name` and `calories`/`protein_g`/`fat_g`/`carbs_g`; when the row is a pantry- or recipe-backed entry, those fields 400 (the macros on those rows are derived from their source and editing them in place would silently break the future-pantry-edit-doesn't-rewrite-history property the log relies on).

The MCP server picks up one new tool, `log_custom_meal`, that wraps the new endpoint with the same parameter validation. `log_consumption` is unchanged — keeping the two tools separate matches the `get_macro_goals` / `set_macro_goals` split the macro-goals SOW already established, and writing one Pydantic field list that describes "pass either pantry_item_id, or recipe_id, or name + four macros" was the alternative that lost. Each tool description stays short, the agent has fewer things to remember per tool, and the API handler keeps two narrow paths instead of one wide one.

The agent gains a paragraph in its system prompt: when the user logs something through chat, call `list_pantry_items` first with the noun extracted from the user's message. If anything matches, use the existing `log_consumption` flow. If nothing matches and the user's wording suggests an external meal — a chain name, a "from $place", a "I bought…" — call `log_custom_meal` with macros estimated from world knowledge. After a successful custom-meal log, the agent appends one short ask to its confirmation reply: "Want me to save \"<name>\" to your pantry so I can find it next time?" If the user agrees, the agent calls the existing `create_pantry_item` tool (`serving_size: 1`, `serving_unit: "meal"`). Both the search-first rule and the post-log ask are prompt-level changes, not new tools.

The web Quick Add modal replaces its current single source-select with a three-tab segmented control. Pantry and Recipes are now separate tabs, each rendering a dropdown filtered to that one source plus the existing quantity field. The Custom tab renders a small form: a name input and four macro inputs styled the same way the pantry-item form is, with the protein / fat / carbs labels carrying their palette color dots. Meal type and "When" stay shared across the three tabs. The existing `LogEntryEditModal` learns the same trick: when the entry under edit is a custom entry, the modal additionally shows the name + four macro inputs and the API client routes the PUT through the extended path. The log-row name resolver gains a third branch for `entry.custom_meal_name`, rendering it in the row the same way a pantry-name or recipe-name renders today.

No schema work on the agent side. No mobile work in this SOW. The contract is the API + MCP shape; mobile catches up in its own dispatch once the contract is live.

## Goals and Non-Goals

### Goals

- Add a nullable `custom_meal_name TEXT` column to `nutrition_log_entries` (migration `012`) and widen the existing exactly-one CHECK over `pantry_item_id` / `recipe_id` to a three-way XOR that also accepts a row where only `custom_meal_name` is set.
- Add `CustomMealName *string` to the `NutritionLogEntry` struct. Rename the `exactlyOneRef` helper to `exactlyOneSource` and update its body to enforce the new 3-way XOR.
- Add `POST /nutrition-log/custom` with body `{name, calories, protein_g, fat_g, carbs_g, meal, consumed_at?}`. Validation:
  - `name`: non-blank, ≤ 200 characters
  - `calories`: `0 ≤ x ≤ 100000` (mirrors `MaxCalories` from macro-goals)
  - `protein_g`, `fat_g`, `carbs_g`: each `0 ≤ x ≤ 10000` (mirrors `MaxMacroGrams`)
  - `meal`: one of the four valid `MealType` values
  - `consumed_at`: optional RFC3339 UTC; defaults to server `time.Now().UTC()` when omitted
  Inserts through `repo.CreateNutritionLogEntry` with `Quantity = 1`, `CustomMealName = &name`, and the four supplied macros in the existing macro columns. Returns the created entry.
- Extend `PUT /nutrition-log/{id}` so that for a custom entry the body may additionally include `name`, `calories`, `protein_g`, `fat_g`, `carbs_g`. For a pantry- or recipe-backed entry, supplying any of those fields returns 400 with a message naming the offending field. The previously editable fields (`quantity`, `consumed_at`, `meal`) continue to work for all three entry types.
- Add `custom_meal_name string | null` to the log-entry DTO on every read path (`GET /nutrition-log`, `GET /nutrition-log/{id}`, `POST /nutrition-log`, the new `POST /nutrition-log/custom`, `PUT /nutrition-log/{id}`).
- New MCP tool `log_custom_meal(name, calories, protein_g, fat_g, carbs_g, meal, consumed_at?)` that wraps the new endpoint. Mirrors the same validation client-side so misuse fails at the MCP boundary before hitting the API.
- Update the agent system prompt with one paragraph covering: (a) pantry-first lookup when the user logs something via chat — call `list_pantry_items` with the extracted noun; (b) fall back to `log_custom_meal` when the pantry has nothing matching and the wording suggests an external meal; (c) append the "Want me to save this to your pantry?" ask after a successful custom-meal log; (d) on user agreement, call the existing `create_pantry_item` tool with `serving_size: 1`, `serving_unit: "meal"`, and the same name + macros.
- Update the web Quick Add modal (`components/quick-add-modal.tsx`) to use a three-tab segmented control (Pantry / Recipes / Custom). Pantry and Recipes tabs each render their own filtered source dropdown + the existing quantity field. Custom tab renders the form: name input, four macro inputs styled to match the pantry-item form (palette color dots in the labels, 2px colored left border on the inputs for protein / fat / carbs; calories stays neutral). Meal type and "When" inputs stay shared across all three tabs.
- Extend the web `LogEntryEditModal` (`components/nutrition/log-entry-edit-modal.tsx`) so that when the entry's `custom_meal_name` is non-null, the modal additionally renders the name input + four macro inputs (same palette + styling as the QuickAdd Custom tab). The save call routes the extra fields through the extended `PUT /nutrition-log/{id}`. For pantry/recipe entries the form stays exactly as it is today.
- Extend `resolveItemName` in `components/nutrition/nutrition-log-view.tsx` to fall through to `entry.custom_meal_name` when both `pantry_item_id` and `recipe_id` are null. Render in the log row identically to the pantry / recipe name (no chip, no badge — the name is the signal).
- Add API client functions `createCustomNutritionLogEntry(token, payload)` and extend `updateNutritionLogEntry` to accept the extra optional fields. Add `custom_meal_name: string | null` to the `NutritionLogEntry` TypeScript type.
- Toast on save success on both the QuickAdd Custom tab and the extended edit modal uses the existing toast system shipped in PR #9.
- New tests (per repo): see Implementation Details § Tests.

### Non-Goals

- **Renaming `log_consumption`.** The tool name is stable in the agent's history and the rename adds churn for no signal. `log_consumption` keeps its current behavior (pantry XOR recipe) and the new path is `log_custom_meal`.
- **Mobile client.** Out of scope. The API + MCP contracts are stable from day one of this SOW shipping so the mobile dispatch picks them up without renegotiation.
- **Search-based pantry / recipe pickers in the QuickAdd modal tabs.** The existing dropdown shape per tab is fine for v1. If the catalog grows large enough that a search field beats a dropdown, file a follow-up.
- **Auto-promote-to-pantry.** The agent never silently saves a custom meal as a pantry item; the user must agree to the "Want me to save…" ask. Auto-promotion is tempting for repeated branded meals but pushes a decision the user might disagree with onto the agent.
- **Storing the agent's "I estimated these macros" provenance.** Once a row is written, the macros are the user's data. The agent's source (world knowledge vs the back of a Chipotle receipt) doesn't get a column. If a "verify these" follow-up flow comes up, file a separate SOW.
- **Refactoring `log_consumption` to merge into the new endpoint.** Locked in brainstorming: separate tools, separate endpoints.
- **Changing the `quantity` model for custom meals.** Locked: custom rows always have `quantity = 1` and the user types the totals they ate. A per-serving × quantity flow for custom meals is a follow-up if a user reports needing it.
- **A persistent "branded restaurants" catalog seeded into the pantry.** Useful for onboarding but expands the scope into a content / curation problem that's orthogonal to this SOW.
- **Recipe components keyed to a custom meal.** Recipes still reference only pantry items. Treating a custom-meal entry as a recipe component would re-introduce the "macros change after log time" problem the row-level denormalization exists to prevent.
- **Server-side promotion endpoint (`POST /pantry-items/from-log/:id`).** The agent's promotion flow uses the existing `create_pantry_item` tool with the same name + macros; a dedicated promotion endpoint would save one tool call at the cost of an extra API surface.
- **Bodyweight / macro-goals / workout surfaces.** Untouched.

## Implementation Details

### Data Model

One new migration: `internal/db/migrations/012_nutrition_log_custom_meals.sql`.

```sql
-- Add the nullable custom_meal_name column for user-typed meals that
-- aren't backed by a pantry item or recipe. Macros are still
-- denormalized in the existing four columns; quantity is always 1 for
-- custom rows (the user types the totals they ate).
ALTER TABLE nutrition_log_entries ADD COLUMN custom_meal_name TEXT;

-- Drop the existing 2-way XOR CHECK and re-add as a 3-way XOR over
-- pantry_item_id, recipe_id, and custom_meal_name. SQLite doesn't
-- support DROP CHECK directly, so the migration recreates the table.
-- (See migrations 002, 005 for the same pattern.)
... -- table recreation with the new CHECK, preserving every existing row.
```

The new CHECK:

```sql
CHECK (
    (pantry_item_id IS NOT NULL AND recipe_id IS NULL AND custom_meal_name IS NULL) OR
    (pantry_item_id IS NULL AND recipe_id IS NOT NULL AND custom_meal_name IS NULL) OR
    (pantry_item_id IS NULL AND recipe_id IS NULL AND custom_meal_name IS NOT NULL)
)
```

The migration recreates the table, preserves every existing row (which all have one of pantry_item_id or recipe_id set, satisfying the new check), and re-applies the soft-delete index from migration 006.

### Go API: types + validation

`internal/nutrition/log_entry.go`:

```go
type NutritionLogEntry struct {
    ID              string
    UserID          string
    ConsumedAt      time.Time
    PantryItemID    *string
    RecipeID        *string
    CustomMealName  *string  // NEW. Non-nil iff this is a custom-typed meal.
    Quantity        float64
    Calories        float64
    ProteinG        float64
    FatG            float64
    CarbsG          float64
    Meal            MealType
    CreatedAt       time.Time
    DeletedAt       *time.Time
}
```

Rename and rewrite the existing helper:

```go
// exactlyOneSource returns true when the entry has exactly one of
// PantryItemID, RecipeID, or CustomMealName set. The CHECK on the
// underlying table enforces the same; this is the Go-side echo so
// callers get an ErrLogEntryReferenceRequired instead of an opaque
// CHECK violation.
func exactlyOneSource(e *NutritionLogEntry) bool {
    n := 0
    if e.PantryItemID != nil { n++ }
    if e.RecipeID != nil { n++ }
    if e.CustomMealName != nil { n++ }
    return n == 1
}
```

Update every call site (`sqlite_repository.go:201, :287`, `memory_repository.go:132, :191`) to the new name. The error sentinel `ErrLogEntryReferenceRequired` keeps its current message — "exactly one of pantry_item_id, recipe_id, or custom_meal_name is required" — which the handler maps to 400.

### Go API: `POST /nutrition-log/custom`

New handler `createCustomLogEntry`:

```go
type customLogEntryRequest struct {
    Name       string     `json:"name"`
    Calories   float64    `json:"calories"`
    ProteinG   float64    `json:"protein_g"`
    FatG       float64    `json:"fat_g"`
    CarbsG     float64    `json:"carbs_g"`
    Meal       string     `json:"meal"`
    ConsumedAt *time.Time `json:"consumed_at,omitempty"`
}
```

Validation order:

1. `name`: `strings.TrimSpace(name)` is non-empty and `len(trimmed) ≤ 200` → otherwise 400 with `"name is required"` / `"name is too long"`.
2. `calories` in `[0, 100000]` → 400 `"calories out of range"`.
3. each of `protein_g`, `fat_g`, `carbs_g` in `[0, 10000]` → 400 `"<macro> out of range"`.
4. `meal` valid → 400 `"invalid meal"` (existing message from `ErrInvalidMeal`).
5. `consumed_at` if supplied parses as RFC3339 → 400 `"invalid consumed_at"`.

On success: build an entry with `Quantity: 1`, `CustomMealName: &trimmedName`, the four macros copied in, `ConsumedAt` defaulted to `time.Now().UTC()` if nil. Pass through `repo.CreateNutritionLogEntry`, return the created row through `httpresp.Created(w, "logged custom meal", toLogDTO(*entry))`.

Mount: `r.Post("/custom", h.createCustomLogEntry)` inside the existing `/nutrition-log` Route (`handler.go:36-45`).

### Go API: extend `PUT /nutrition-log/{id}`

`logEntryRequest` (currently used by `updateLogEntry`) gains optional fields:

```go
type logEntryRequest struct {
    Quantity   float64    `json:"quantity"`
    ConsumedAt *time.Time `json:"consumed_at,omitempty"`
    Meal       string     `json:"meal,omitempty"`
    // Custom-only fields. The handler rejects any of these with 400 if
    // the targeted row is pantry- or recipe-backed.
    Name      *string  `json:"name,omitempty"`
    Calories  *float64 `json:"calories,omitempty"`
    ProteinG  *float64 `json:"protein_g,omitempty"`
    FatG      *float64 `json:"fat_g,omitempty"`
    CarbsG    *float64 `json:"carbs_g,omitempty"`
}
```

Handler change in `updateLogEntry`:

1. Look up the existing row (same as today).
2. If the row has `CustomMealName == nil` and any of the custom-only fields are supplied → 400 `"<field> is only editable on custom meal entries"`.
3. If the row has `CustomMealName != nil`:
   - When `Name` is supplied, validate non-blank + length ≤ 200, assign `entry.CustomMealName = &trimmed`.
   - When `Calories` is supplied, range-check 0..100000, assign.
   - When `ProteinG` / `FatG` / `CarbsG` are supplied, range-check 0..10000 each, assign.
4. Continue with the existing `quantity` / `consumed_at` / `meal` handling.
5. Persist through `repo.UpdateNutritionLogEntry`.

The repository's `UpdateNutritionLogEntry` already writes every field on the entry; the only change there is the SQL `UPDATE` statement adds `custom_meal_name = ?` to the SET clause and the matching `*string` parameter.

### Go API: DTOs

`logEntryDTO` (in `handler.go`) gains:

```go
CustomMealName *string `json:"custom_meal_name"` // null when pantry/recipe-backed
```

`toLogDTO` copies it through.

### MCP: `log_custom_meal`

In `prog-strength-mcp/src/prog_strength_mcp/nutrition.py`:

```python
@mcp.tool
async def log_custom_meal(
    name: Annotated[
        str,
        Field(min_length=1, max_length=200, description=(
            "What the user ate. Free-form text the user types or you "
            "extract from their message — \"Chipotle chicken bowl\", "
            "\"Sweetgreen Harvest Bowl\", \"airport protein bar\". "
            "Stored as-is on the log entry; appears in the user's "
            "nutrition log under that exact name."
        )),
    ],
    calories: Annotated[float, Field(ge=0, le=100_000, description="Total calories for the meal as the user ate it.")],
    protein_g: Annotated[float, Field(ge=0, le=10_000, description="Total protein in grams.")],
    fat_g:     Annotated[float, Field(ge=0, le=10_000, description="Total fat in grams.")],
    carbs_g:   Annotated[float, Field(ge=0, le=10_000, description="Total carbohydrates in grams.")],
    meal: Annotated[
        Literal["breakfast", "lunch", "dinner", "snack"],
        Field(description="Meal bucket on the user's nutrition page."),
    ],
    consumed_at: Annotated[
        str | None,
        Field(default=None, description="RFC3339 UTC timestamp; omit for now."),
    ] = None,
) -> dict[str, Any]:
    """Log a one-off meal that isn't backed by a pantry item or recipe.

    Use this when the user describes eating something they don't have
    saved — restaurant meals, foods bought outside, anything one-off.
    Always check `list_pantry_items` first; if there's a match, use
    `log_consumption` against that match instead.

    Returns the created log entry with the user-typed name and macros
    frozen on the row.
    """
    auth = _auth_header_or_raise()
    return await api.log_custom_meal(
        auth,
        name=name,
        calories=calories,
        protein_g=protein_g,
        fat_g=fat_g,
        carbs_g=carbs_g,
        meal=meal,
        consumed_at=consumed_at,
    )
```

Add `api.log_custom_meal` in `api_client.py` matching the pattern used by `log_consumption`.

`log_consumption` is unchanged.

### Agent: prompt update

In `prog-strength-agent/src/prog_strength_agent/prompt.py`, add a paragraph adjacent to the existing nutrition guidance:

```text
Logging meals through chat. When the user says they ate something:

1. Call list_pantry_items first with the noun extracted from their
   message ("chipotle bowl" -> query="chipotle"). Match generously —
   if anything plausible comes back, prefer log_consumption against
   that pantry item or recipe.

2. If nothing matches and the user's wording suggests an external
   meal — a chain name, "from <place>", "I bought…", "I ordered…" —
   call log_custom_meal with your best estimate of the macros. Be
   conservative on calories for restaurant meals (bowls trend higher
   than menu calorie cards because of oils + serving size).

3. After a successful log_custom_meal call, append one short ask to
   your reply: 'Want me to save "<name>" to your pantry so I can
   find it next time?' If the user agrees in the next turn, call
   create_pantry_item with the same name and macros, serving_size: 1,
   serving_unit: "meal".

Do NOT silently save a custom meal to the pantry; the ask is the
user's decision.
```

The agent's existing tool-routing logic doesn't need code changes — the prompt change drives the new behavior. New agent tests (`test_model_harness.py`) verify:

- A chat where the user says "I had a chipotle bowl" with no matching pantry item → the harness sees a `list_pantry_items` call, then a `log_custom_meal` call.
- A follow-up "yes" turn → the harness sees a `create_pantry_item` call with the same name + macros.
- A chat where the user says "I had three eggs" with a matching "Egg" pantry item → the harness sees `list_pantry_items` then `log_consumption` (no `log_custom_meal`).

### Web: API client

`lib/api.ts`:

```ts
export type NutritionLogEntry = {
  id: string;
  consumed_at: string;
  pantry_item_id: string | null;
  recipe_id: string | null;
  custom_meal_name: string | null;  // NEW
  quantity: number;
  calories: number;
  protein_g: number;
  fat_g: number;
  carbs_g: number;
  meal: MealType;
  created_at: string;
};

export type CreateCustomLogEntryPayload = {
  name: string;
  calories: number;
  protein_g: number;
  fat_g: number;
  carbs_g: number;
  meal: MealType;
  consumed_at?: string;
};

export async function createCustomNutritionLogEntry(
  token: string,
  payload: CreateCustomLogEntryPayload,
): Promise<NutritionLogEntry>;
```

`UpdateLogEntryPayload` gains optional `name`, `calories`, `protein_g`, `fat_g`, `carbs_g`.

### Web: QuickAddModal

`components/quick-add-modal.tsx` replaces the single source select with a 3-tab segmented control at the top:

```tsx
type Tab = "pantry" | "recipe" | "custom";
const [tab, setTab] = useState<Tab>("pantry");
```

Render the segmented control (matching the visual reviewed in brainstorming):

```tsx
<div className="grid grid-cols-3 rounded-md border border-[var(--border)] bg-[var(--surface)] overflow-hidden">
  <TabButton active={tab === "pantry"} onClick={() => setTab("pantry")}>Pantry</TabButton>
  <TabButton active={tab === "recipe"} onClick={() => setTab("recipe")}>Recipes</TabButton>
  <TabButton active={tab === "custom"} onClick={() => setTab("custom")}>Custom</TabButton>
</div>
```

Below the tabs, render the per-tab body:

- **Pantry tab**: source dropdown filtered to pantry items + a quantity input.
- **Recipes tab**: same shape filtered to recipes + a quantity input.
- **Custom tab**: a name input + a four-column macro row using the same `MacroField` component from `pantry-item-form.tsx` (color-coded protein / fat / carbs labels + 2px left border, calories neutral). No quantity input — the user types totals.

Below the per-tab body, the shared meal selector + the existing "When" input stay rendered for all three tabs.

The `onLog` prop's signature widens to a discriminated union:

```ts
onLog: (
  source:
    | { kind: "pantry"; id: string; quantity: number }
    | { kind: "recipe"; id: string; quantity: number }
    | { kind: "custom"; name: string; calories: number; protein_g: number; fat_g: number; carbs_g: number },
  meal: MealType,
  consumedAt: string,
) => Promise<void>;
```

The page (`app/(app)/nutrition/page.tsx`) routes the source to either `createNutritionLogEntry` (pantry / recipe) or `createCustomNutritionLogEntry` (custom). Both fire `toast.success` on resolve (using the system shipped in PR #9). The page's existing optimistic `setEntries` update appends the returned entry exactly the same way.

### Web: LogEntryEditModal

`components/nutrition/log-entry-edit-modal.tsx` checks `entry.custom_meal_name`:

- If non-null (custom entry): hide the existing `quantity` input — the row's stored macros are the totals the user typed, not a per-serving × multiplier pair, so editing quantity would be a no-op against the daily aggregation and reads as a UX trap. Render `meal` and `consumed_at` as today, plus a new name input and a four-column macro row using `MacroField` with the same color treatment as the QuickAdd Custom tab.
- If null (pantry/recipe entry): render exactly as today.

The save call sends `{quantity, consumed_at, meal}` for pantry/recipe entries and `{consumed_at, meal, name, calories, protein_g, fat_g, carbs_g}` for custom entries (no `quantity` field on the custom path). The page's existing `LogEntryEditModal` integration unchanged.

### Web: log row name resolution

`resolveItemName` in `components/nutrition/nutrition-log-view.tsx`:

```ts
export function resolveItemName(
  entry: NutritionLogEntry,
  pantryByID: Map<string, PantryItem>,
  recipeByID: Map<string, Recipe>,
): string {
  if (entry.pantry_item_id) {
    return pantryByID.get(entry.pantry_item_id)?.name ?? "Unknown item";
  }
  if (entry.recipe_id) {
    return recipeByID.get(entry.recipe_id)?.name ?? "Unknown recipe";
  }
  if (entry.custom_meal_name) {
    return entry.custom_meal_name;
  }
  return "Untitled entry";
}
```

The log row renders the resolved name exactly as it does today — no chip, no badge for custom entries. The recipe pill stays for recipe entries.

### Tests

**Go API**:

- Migration test (`internal/db/migrations_test.go` or equivalent): apply 012 against a database seeded with 6 entries (3 pantry-backed, 3 recipe-backed). All 6 survive; their CHECK constraints pass under the new shape.
- `internal/nutrition/handler_test.go` extensions:
  - `POST /nutrition-log/custom` happy path: returns 201 with the created row, `custom_meal_name` echoed back, macros stored as-typed.
  - POST validation: empty name → 400 with `name`; name > 200 chars → 400; calories > 100000 → 400; each macro > 10000 → 400; invalid meal → 400; missing consumed_at defaults to now.
  - `PUT /nutrition-log/{id}` on a custom entry with new name → returns the updated entry with the new name.
  - `PUT /nutrition-log/{id}` on a custom entry with new macros → returns updated macros.
  - `PUT /nutrition-log/{id}` on a pantry-backed entry with `name` → 400 with `"name is only editable on custom meal entries"`.
  - `PUT /nutrition-log/{id}` on a recipe-backed entry with `calories` → 400.
- `internal/nutrition/nutrition_test.go`: rename `TestLogEntry_ExactlyOneReferenceEnforced` cases to cover the 3-way XOR — all three valid shapes pass, all 0-set / 2-set / 3-set combinations fail.

**MCP**:

- Extend `test_nutrition.py`: `log_custom_meal` happy path forwards all 7 params + auth header to the API client; min-length name (empty string) raises at the Pydantic layer before the HTTP call; macros out of range raise locally.

**Agent**:

- `test_model_harness.py`:
  - Chat with `client_timezone="America/Denver"` and a user turn "I had a chipotle chicken bowl just now" with no pantry match → harness sees a `list_pantry_items` call, then `log_custom_meal`.
  - Follow-up user turn "yes please save it" → harness sees `create_pantry_item` with the same name and macros.
  - Chat with a user turn "I had three eggs" and a pantry item named "Egg" → harness sees `list_pantry_items` then `log_consumption` (no `log_custom_meal`).

**Web**:

No test runner. Verification:

- `npx tsc --noEmit`, `npx eslint`, `npx next build` all clean.
- Hand-test checklist:
  - QuickAdd → Pantry tab works exactly as before.
  - QuickAdd → Recipes tab works exactly as before.
  - QuickAdd → Custom tab: name + 4 macros + meal + when → Log → row appears in the log with the custom name; success toast fires.
  - QuickAdd → Custom tab: blank name → inline form error, no API call.
  - Log row → pencil → custom entry → edit modal shows name + macros editable → save → row updates.
  - Log row → pencil → pantry entry → edit modal shows the existing fields only (no name input, no macro inputs).

### Rollout

Coordinated cross-repo dispatch. Merge order:

1. `prog-strength-api` (migration 012 + new endpoint + edit extension + DTO field).
2. `prog-strength-mcp` (new tool + API client method).
3. In parallel: `prog-strength-agent` (prompt + tests) and `prog-strength-web` (API client + QuickAdd + edit modal + name resolver).
4. Hand-test in the deployed environment with a real account: log a custom meal via web, log a custom meal via chat, edit a custom entry, decline + accept the agent's promote-to-pantry ask.

The API change is additive — the new CHECK admits every existing row. No data backfill. Each PR is independently revertable until the web + agent PRs deploy and start emitting custom-meal rows; once those land, reverting the API PR would re-narrow the CHECK and reject custom rows the clients have written. Roll forward (fix the clients) rather than back.
