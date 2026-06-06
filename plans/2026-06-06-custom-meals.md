# Custom Meals — Implementation Plan

> **For agentic workers:** Implement task-by-task. Each task is implemented by a
> subagent, then spec-reviewed and code-quality-reviewed before moving on.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let users log one-off meals that aren't backed by a saved pantry item
or recipe. A nullable `custom_meal_name` column on `nutrition_log_entries`, a
three-way XOR over `pantry_item_id` / `recipe_id` / `custom_meal_name`, a new
`POST /nutrition-log/custom` endpoint plus an extended `PUT /nutrition-log/{id}`,
a new MCP `log_custom_meal` tool, an agent prompt rule, and the web Quick Add /
edit-modal / log-row surfaces.

**SOW:** `sows/custom-meals.md`

**Additive, coordinated dispatch.** The new CHECK admits every existing row; no
data backfill. Merge order: `prog-strength-api` → `prog-strength-mcp` →
(`prog-strength-agent` ∥ `prog-strength-web`) → `prog-strength-docs`.

Module path (Go): `github.com/jwallace145/progressive-overload-fitness-tracker`,
Go 1.25.3. MCP & agent: Python 3.12, `uv`. Web: Next.js 16, no test runner.

---

## Repo: prog-strength-api

### Task A1 — Migration 012 + migration test

Files: `internal/db/migrations/012_nutrition_log_custom_meals.sql` (new),
`internal/db/migrate_test.go` or `internal/db/migrations_test.go` (test).

Current table (after 006 + the 007 `meal` ALTER) has columns in this order:
`id, user_id, consumed_at, pantry_item_id, recipe_id, quantity, calories,
protein_g, fat_g, carbs_g, created_at, deleted_at, meal`. The active index is
`idx_nutrition_log_user_consumed ON nutrition_log_entries(user_id, consumed_at
DESC) WHERE deleted_at IS NULL` (from 006).

SQLite has no `DROP CONSTRAINT`, so the migration **recreates** the table.
Nothing else references `nutrition_log_entries`, so a drop/recreate is FK-safe
even with foreign_keys on (the rows being copied reference still-existing
pantry_items / recipes). (Note: migrations 002 and 005 do NOT recreate tables —
this is the repo's first recreation; follow the standard SQLite create →
INSERT…SELECT → DROP → RENAME sequence below, all inside the migration's
transaction.)

- [ ] Create `012_nutrition_log_custom_meals.sql` with a header comment in the
  style of 006/007, then:
  1. `CREATE TABLE nutrition_log_entries_new (...)` — copy 006's full column
     definitions **including** the `meal TEXT NOT NULL DEFAULT 'snack'
     CHECK(meal IN ('breakfast','lunch','dinner','snack'))` column from 007,
     add `custom_meal_name TEXT` (nullable, no default), and replace the 2-way
     CHECK with the 3-way XOR:
     ```sql
     CHECK (
         (pantry_item_id IS NOT NULL AND recipe_id IS NULL AND custom_meal_name IS NULL) OR
         (pantry_item_id IS NULL AND recipe_id IS NOT NULL AND custom_meal_name IS NULL) OR
         (pantry_item_id IS NULL AND recipe_id IS NULL AND custom_meal_name IS NOT NULL)
     )
     ```
     Keep the `REFERENCES pantry_items(id)` / `REFERENCES recipes(id)` and every
     existing `CHECK` (quantity>0, macros>=0).
  2. `INSERT INTO nutrition_log_entries_new (id, user_id, consumed_at,
     pantry_item_id, recipe_id, quantity, calories, protein_g, fat_g, carbs_g,
     created_at, deleted_at, meal) SELECT <those same 13 columns> FROM
     nutrition_log_entries;` — `custom_meal_name` is omitted so existing rows get
     NULL.
  3. `DROP TABLE nutrition_log_entries;`
  4. `ALTER TABLE nutrition_log_entries_new RENAME TO nutrition_log_entries;`
  5. Recreate the index exactly: `CREATE INDEX idx_nutrition_log_user_consumed
     ON nutrition_log_entries(user_id, consumed_at DESC) WHERE deleted_at IS NULL;`
- [ ] Migration test: apply all migrations against a fresh DB seeded (after
  migration, or seed then this is moot — simplest is to apply migrations, insert
  3 pantry-backed + 3 recipe-backed rows directly via SQL, then assert all 6
  survive and that the new CHECK accepts a `custom_meal_name`-only row and
  rejects a 0-source and a 2-source row). Check how existing migrate tests are
  structured first (`internal/db/*_test.go`); mirror that. If no migration test
  harness exists, add a focused test that opens an in-memory DB, runs `Migrate`,
  and exercises the three CHECK shapes with raw `INSERT`s.
- [ ] Verify: `go test ./internal/db/...`, `gofmt -l internal/db`.

### Task A2 — Struct, `exactlyOneSource`, repository INSERT/UPDATE, XOR unit tests

Files: `internal/nutrition/log_entry.go`, `internal/nutrition/memory_repository.go`,
`internal/nutrition/sqlite_repository.go`, `internal/nutrition/nutrition_test.go`.

- [ ] `log_entry.go`: add `CustomMealName *string` to `NutritionLogEntry` (place
  it right after `RecipeID`, with a `// Non-nil iff this is a custom-typed meal.`
  comment).
- [ ] `memory_repository.go`: rename `exactlyOneRef` → `exactlyOneSource` and
  rewrite its body to the 3-way count (treat an empty-string pointer as unset, to
  match the existing `*PantryItemID != ""` guard style):
  ```go
  func exactlyOneSource(e *NutritionLogEntry) bool {
      n := 0
      if e.PantryItemID != nil && *e.PantryItemID != "" { n++ }
      if e.RecipeID != nil && *e.RecipeID != "" { n++ }
      if e.CustomMealName != nil && *e.CustomMealName != "" { n++ }
      return n == 1
  }
  ```
  Keep the doc comment, updated to mention all three sources.
- [ ] Update all 4 call sites (`memory_repository.go` Create ~132 / Update ~191,
  `sqlite_repository.go` Create ~201 / Update ~287) from `exactlyOneRef` →
  `exactlyOneSource`. The `ErrLogEntryReferenceRequired` sentinel stays; the
  validation flow is unchanged.
- [ ] `sqlite_repository.go` `CreateNutritionLogEntry`: add `custom_meal_name` to
  the INSERT column list and a matching `nullableString(e.CustomMealName)`
  parameter (insert it adjacent to `recipe_id` in both the column list and the
  values, keeping positions aligned).
- [ ] `sqlite_repository.go` `UpdateNutritionLogEntry`: add `custom_meal_name = ?`
  to the SET clause with `nullableString(e.CustomMealName)` in the matching arg
  position.
- [ ] `memory_repository.go`: the Create/Update already `stored := *e` (full
  struct copy) so `CustomMealName` is carried automatically — no field-copy
  change needed; just confirm.
- [ ] `nutrition_test.go`: rewrite `TestLogEntry_ExactlyOneReferenceEnforced` (or
  add a sibling) to cover the 3-way XOR via `CreateNutritionLogEntry` on the
  memory repo: all three single-source shapes succeed; 0-set, all three 2-set
  pairs, and the 3-set shape each return `ErrLogEntryReferenceRequired`. A
  custom-only row needs `CustomMealName: &name` plus the macros and `Quantity: 1`.
- [ ] Verify: `go build ./...`, `go test ./internal/nutrition/...`, `gofmt -l internal/nutrition`.

### Task A3 — Handler: DTO, `POST /custom`, `PUT` extension, handler tests

Files: `internal/nutrition/handler.go`, `internal/nutrition/handler_test.go`,
`internal/nutrition/errors.go` (maybe a new sentinel).

- [ ] `logEntryDTO`: add `CustomMealName *string `json:"custom_meal_name"``
  (no `omitempty` — the SOW wants the key present and `null` for pantry/recipe
  rows). `toLogDTO`: copy `e.CustomMealName` through.
- [ ] New request type + handler `createCustomLogEntry`:
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
  Validation order → each 400 via `httpresp.Error(w, http.StatusBadRequest, msg)`:
  1. `name := strings.TrimSpace(req.Name)` — `name == ""` → `"name is required"`;
     `len(name) > 200` → `"name is too long"`.
  2. `calories` in `[0, MaxCalories]` → `"calories out of range"`.
  3. each of `protein_g` / `fat_g` / `carbs_g` in `[0, MaxMacroGrams]` →
     `"protein_g out of range"` / `"fat_g out of range"` / `"carbs_g out of range"`.
  4. `meal` valid via `MealType(req.Meal).Valid()` → else
     `httpresp.Error(... ErrInvalidMeal.Error())`.
  5. `consumed_at`: already decoded as `*time.Time` (json handles RFC3339); if the
     decode of the whole body fails → `"invalid request body"` (same as the other
     handlers). Default `consumedAt := time.Now().UTC()`, override with
     `req.ConsumedAt.UTC()` when non-nil.
  Build `entry := &NutritionLogEntry{UserID: userID, ConsumedAt: consumedAt,
  Quantity: 1, CustomMealName: &name, Calories, ProteinG, FatG, CarbsG, Meal}`,
  call `h.repo.CreateNutritionLogEntry`, return
  `httpresp.Created(w, "logged custom meal", toLogDTO(*entry))`. Mirror the
  auth-context + body-decode boilerplate from `createLogEntry`.
- [ ] Mount inside the existing `/nutrition-log` route block, before `/{id}`:
  `r.Post("/custom", h.createCustomLogEntry)` (place it next to `r.Get("/daily", …)`
  so the literal segment is registered before the `{id}` param route).
- [ ] Extend `logEntryRequest` with the custom-only optional pointers:
  ```go
  Name     *string  `json:"name,omitempty"`
  Calories *float64 `json:"calories,omitempty"`
  ProteinG *float64 `json:"protein_g,omitempty"`
  FatG     *float64 `json:"fat_g,omitempty"`
  CarbsG   *float64 `json:"carbs_g,omitempty"`
  ```
- [ ] `updateLogEntry`: after loading `existing`:
  - If `existing.CustomMealName == nil` (pantry/recipe row) and ANY of
    `Name/Calories/ProteinG/FatG/CarbsG` is non-nil → 400
    `"<field> is only editable on custom meal entries"` (name the first offending
    field, JSON name: `name`, `calories`, `protein_g`, `fat_g`, `carbs_g`).
  - If `existing.CustomMealName != nil` (custom row): the entry being persisted
    must keep `CustomMealName` set. Apply supplied fields with validation:
    `Name` → trim, non-blank, ≤200, assign `entry.CustomMealName = &trimmed`;
    `Calories` → range `[0,MaxCalories]`; each macro → `[0,MaxMacroGrams]`.
    Do NOT re-derive macros from a pantry/recipe source for custom rows (there is
    none) — carry `existing`'s macros forward and overwrite only the supplied
    ones. **Important:** the current `updateLogEntry` re-derives macros by
    branching on PantryItemID/RecipeID; add a leading branch: if the row is
    custom, build the updated entry from `existing` (preserving CustomMealName +
    macros) and skip the pantry/recipe re-derivation entirely.
  - The existing `quantity` / `consumed_at` / `meal` handling continues to apply
    to all three types (custom rows keep `quantity` editable at the API level
    even though the web hides it — quantity stays 1 unless a client sends one;
    don't special-case it server-side beyond the existing `> 0` guard).
  - Persist via `h.repo.UpdateNutritionLogEntry`, return `toLogDTO`.
- [ ] If a new validation sentinel is cleaner for the "only editable on custom"
  case, it's fine to inline the message string instead (the SOW specifies the
  exact wording). Keep `isValidationError` mapping intact for existing sentinels.
- [ ] `handler_test.go` extensions (mirror existing helpers — `seedLogEntry`,
  the `authctx.WithUserID` request pattern, `assertBadRequest`):
  - `POST /nutrition-log/custom` happy path → 201, `custom_meal_name` echoed,
    macros stored as-typed, `quantity == 1`.
  - POST validation: empty/blank name → 400 `name`; >200-char name → 400;
    calories > 100000 → 400; each macro > 10000 → 400; invalid meal → 400;
    omitted `consumed_at` defaults to ~now (assert non-zero / close to now).
  - `PUT` on a custom entry with new `name` → updated name returned.
  - `PUT` on a custom entry with new macros → updated macros returned.
  - `PUT` on a pantry-backed entry with `name` → 400
    `"name is only editable on custom meal entries"`.
  - `PUT` on a recipe-backed entry with `calories` → 400.
  - You'll need a helper to seed a custom entry (CustomMealName set) and a
    recipe-backed entry; extend `seedLogEntry` or add `seedCustomLogEntry` /
    `seedRecipeLogEntry`. For the POST/PUT handler tests, drive the handler the
    same way `listLog` does (httptest request + `authctx.WithUserID` + call the
    handler method directly), decoding the `{message,data}` envelope.
- [ ] Verify: `go build ./...`, `go test ./...`, `gofmt -l internal/nutrition`.

---

## Repo: prog-strength-mcp

### Task M1 — `log_custom_meal` tool + api client method + tests

Files: `src/prog_strength_mcp/nutrition.py`, `src/prog_strength_mcp/api_client.py`,
`tests/test_nutrition_tools.py`.

- [ ] `api_client.py`: add `log_custom_meal` mirroring `log_consumption`:
  ```python
  async def log_custom_meal(
      self,
      auth_header: str,
      *,
      name: str,
      calories: float,
      protein_g: float,
      fat_g: float,
      carbs_g: float,
      meal: str,
      consumed_at: str | None = None,
  ) -> dict[str, Any]:
      """POST /nutrition-log/custom. Logs a one-off meal the user typed,
      not backed by a pantry item or recipe."""
      body: dict[str, Any] = {
          "name": name, "calories": calories, "protein_g": protein_g,
          "fat_g": fat_g, "carbs_g": carbs_g, "meal": meal,
      }
      if consumed_at is not None:
          body["consumed_at"] = consumed_at
      resp = await self._client.post(
          "/nutrition-log/custom", json=body,
          headers={"Authorization": auth_header},
      )
      _raise_for_status(resp)
      data = resp.json().get("data")
      return data if isinstance(data, dict) else {}
  ```
- [ ] `nutrition.py`: add the `log_custom_meal` `@mcp.tool` from the SOW
  (§ MCP). Use `Annotated[..., Field(...)]` with `min_length=1, max_length=200`
  on `name`, `ge=0, le=100_000` on calories, `ge=0, le=10_000` on each macro,
  `Literal["breakfast","lunch","dinner","snack"]` on meal, optional
  `consumed_at: str | None = None`. Call `_auth_header_or_raise()`, then
  `api.log_custom_meal(auth, …)` inside the same `try/except APIError` →
  `RuntimeError(f"API error ({e.status_code}): {e.message}")` pattern. Docstring
  from the SOW (mentions checking `list_pantry_items` first). `log_consumption`
  stays untouched. Whatever registration mechanism the module uses
  (`@mcp.tool` at import vs a `register(mcp, api)` fn), match the existing
  `log_consumption` exactly.
- [ ] `tests/test_nutrition_tools.py`: add
  - api-client happy path (respx-mocked `POST /nutrition-log/custom`): asserts all
    7 fields land in the JSON body + the Authorization header, and that
    `consumed_at` is omitted when None.
  - tool-boundary validation: a `name=""` (or too-long / out-of-range macro) call
    raises at the Pydantic layer **before** any HTTP — reuse the `_ExplodingAPI`
    sentinel pattern + `await mcp.get_tool("log_custom_meal")` then `.fn`, and
    `pytest.raises`. Note: Pydantic `Field` constraints on a `@mcp.tool` raise a
    `pydantic.ValidationError` when the tool is invoked through the tool wrapper;
    if calling `.fn` directly bypasses validation, instead assert via the tool's
    validated call path the existing tests use. Check how existing tests trigger
    Pydantic validation and match that; if they don't, assert the happy-path
    forwarding and the auth-guard, and cover range/length validation at the
    api-client/contract level.
- [ ] Verify: `uv run pytest`, `uv run ruff check`.

---

## Repo: prog-strength-agent

### Task AG1 — Prompt paragraph + prompt-content tests

Files: `src/prog_strength_agent/prompt.py`, `tests/test_prompt.py`.

- [ ] `prompt.py`: insert a new bolded paragraph after the "Use get_daily_macros
  for daily totals." block and before the `## Tone` section, matching the
  existing `**Keyword.** …` backslash-continued style. Content (from SOW § Agent):
  pantry-first lookup (call `list_pantry_items` with the extracted noun);
  fall back to `log_custom_meal` when nothing matches and the wording suggests an
  external meal (chain name / "from <place>" / "I bought…" / "I ordered…");
  after a successful `log_custom_meal`, append the one-line ask `Want me to save
  "<name>" to your pantry so I can find it next time?`; on user agreement call
  `create_pantry_item` with `serving_size: 1`, `serving_unit: "meal"`, same
  name + macros; never silently auto-save.
- [ ] `tests/test_prompt.py`: add assertions (substring style, matching the
  existing tests) that the prompt mentions `list_pantry_items`, `log_custom_meal`,
  `create_pantry_item`, the save-to-pantry ask, and `serving_unit` / `"meal"`.
  Assert against both `SYSTEM_PROMPT` and `build_chat_system_prompt(...)` output
  as the existing tests do.
- [ ] **Note on the SOW's harness tests.** The SOW's § Agent Tests describe
  model-behavior assertions ("harness sees a `list_pantry_items` call, then
  `log_custom_meal`"). The existing `tests/test_model_harness.py` deliberately
  does NOT drive the real Anthropic stream + MCP tool loop (see its own comment),
  and these described behaviors depend on live-model decisions, so they aren't a
  deterministic unit test in this repo's harness. Cover the agent change with the
  prompt-content tests above (the actual code delta is prompt-only) and call this
  out in the PR body. Do not fabricate a passing model-behavior test.
- [ ] Verify: `uv run pytest`, `uv run ruff check`.

---

## Repo: prog-strength-web

### Task W1 — API client: types + functions

Files: `lib/api.ts`.

- [ ] `NutritionLogEntry`: add `custom_meal_name: string | null;`.
- [ ] Add:
  ```ts
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
  ): Promise<NutritionLogEntry> { /* POST /nutrition-log/custom, mirror createNutritionLogEntry */ }
  ```
- [ ] `UpdateLogEntryPayload`: add optional `name?`, `calories?`, `protein_g?`,
  `fat_g?`, `carbs_g?`. `updateNutritionLogEntry` already forwards the whole
  payload, so no body change beyond the type.
- [ ] Verify: `npm run typecheck`, `npm run lint`.

### Task W2 — QuickAddModal 3-tab + page routing + toast

Files: `components/quick-add-modal.tsx`, `app/(app)/nutrition/page.tsx`.

- [ ] `quick-add-modal.tsx`: replace the single source `<select>` with a 3-tab
  segmented control (`type Tab = "pantry" | "recipe" | "custom"`, `useState`).
  Use the markup shape from the SOW § Web QuickAddModal (`grid grid-cols-3`
  segmented control + a small `TabButton` helper). Per-tab body:
  - **Pantry**: pantry-only dropdown + the existing quantity input.
  - **Recipes**: recipe-only dropdown + the existing quantity input.
  - **Custom**: name input + a 4-column macro row reusing the `MacroField`
    component / palette treatment from `pantry-item-form.tsx` (import
    `MACRO_COLORS` from `@/lib/macro-colors`; protein=emerald, fat=pink,
    carbs=amber dots + 2px left border; calories neutral). No quantity input.
    If `MacroField` isn't exported, either export it from `pantry-item-form.tsx`
    or lift it into a small shared module — prefer exporting it to avoid a
    second copy.
  - Shared below the per-tab body: the existing meal selector + "When" input.
  - Inline form guard: blank custom name → inline error, no `onLog` call.
- [ ] Widen `onLog` to the discriminated union from the SOW:
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
  (Note this moves `quantity` and `consumedAt` into the `onLog` contract; update
  the page accordingly. Keep the modal's existing consumed-at derivation if it
  computes one, otherwise pass the page's computed value — match whatever the
  current `handleLog` does for `consumed_at`.)
- [ ] `app/(app)/nutrition/page.tsx` `handleLog`: switch on `source.kind`. Pantry
  / recipe → `createNutritionLogEntry` (as today). Custom →
  `createCustomNutritionLogEntry(token, { name, calories, protein_g, fat_g,
  carbs_g, meal, consumed_at })`. Both branches keep the existing optimistic
  `setEntries((prev) => prev ? [entry, ...prev] : [entry])` append and fire
  `toast.success(...)` on resolve. Import `useToast` from `@/components/toast`
  (see `pantry-item-modal.tsx` for the pattern). Preserve the 401-redirect
  catch. Update `handleLog`'s signature to the new `onLog` shape.
- [ ] Verify: `npm run typecheck`, `npm run lint`, `npm run build`.

### Task W3 — LogEntryEditModal custom branch + resolveItemName

Files: `components/nutrition/log-entry-edit-modal.tsx`,
`components/nutrition/nutrition-log-view.tsx`.

- [ ] `resolveItemName`: add a third branch returning `entry.custom_meal_name`
  when both `pantry_item_id` and `recipe_id` are null (before the
  `"Untitled entry"` fallback). Log row rendering unchanged (no chip/badge for
  custom — the recipe pill stays gated on `entry.recipe_id`).
- [ ] `log-entry-edit-modal.tsx`: when `entry.custom_meal_name != null`, render
  the name input + a 4-column `MacroField` macro row (same palette as W2's custom
  tab), and **hide the quantity input** (custom rows store totals, not
  per-serving × multiplier). Seed local state from `entry.custom_meal_name` and
  `entry.calories/protein_g/fat_g/carbs_g`. The save call for a custom entry
  sends `{ consumed_at, meal, name, calories, protein_g, fat_g, carbs_g }` (no
  `quantity`); pantry/recipe entries keep sending `{ quantity, consumed_at, meal }`
  exactly as today. Fire `toast.success` on save (import `useToast`). Keep the
  existing time/meal handling + 401 redirect.
- [ ] Verify: `npm run typecheck`, `npm run lint`, `npm run build`.

---

## Repo: prog-strength-docs

### Task D1 — Mark SOW shipped

File: `sows/custom-meals.md` on branch `feat/custom-meals` (this plan file is
also committed on that branch).

- [ ] Frontmatter `status: shipped`; body `**Status**: Shipped · …`;
  `**Last updated**: 2026-06-06`. Commit `docs: mark custom-meals as shipped`.

---

## PRs

Push `feat/custom-meals` and open a PR against `main` in each modified repo plus
`prog-strength-docs`. Titles follow conventional-commit `feat(<scope>): …` style;
bodies follow the Summary + Test plan format used in recent merged PRs. The MCP
PR notes it depends on the API endpoint; the docs PR body notes that merging it
marks the SOW shipped.
