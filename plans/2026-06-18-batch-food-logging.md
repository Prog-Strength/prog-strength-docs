# Batch Food Logging Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the two single-item agent nutrition tools with one
`log_consumption_batch` tool backed by a new best-effort `POST
/nutrition-log/batch` API endpoint, so the agent logs everything in one
message as a single tool call and the web chat shows one count chip.

**Architecture:** API-first. The Go API gains a best-effort batch endpoint
that reuses the single endpoints' build-and-insert logic (extracted into
helpers); the single endpoints stay byte-for-byte compatible. The MCP server
swaps its two single-item tools for one discriminated-union batch tool that
forwards to the new endpoint. The agent streams an `item_count` on
`tool_use_start` and updates its prompt to name the new tool. The web reads
`item_count` and renders one "Log Consumption (N items)" chip.

**Tech Stack:** Go (chi), Python (FastMCP, Pydantic, httpx/respx), Python
(Anthropic streaming agent), Next.js/React/TypeScript (vitest).

Each repo is its own task, its own branch (`feat/batch-food-logging`), its own
PR. Tasks are ordered by dependency (API contract first) but each task's code
is self-contained.

---

## Repo: prog-strength-api

### Task 1: Best-effort `POST /nutrition-log/batch` + extracted build helpers

**Files:**
- Modify: `internal/nutrition/handler.go` (extract helpers; add batch handler + DTOs; mount route)
- Modify: `internal/nutrition/handler_test.go` (batch tests + single-endpoint regression)

**Context:** `createLogEntry` (`handler.go:321-399`) validates a pantry/recipe
request and builds a `NutritionLogEntry` with denormalized macros, then
inserts. `createCustomLogEntry` (`handler.go:405-467`) does the same for a
one-off custom meal. Both write their own HTTP responses. We extract the
*build* logic (validate + derive macros, no insert, no HTTP write) into two
helpers so the single handlers and the new batch loop share one code path. The
response envelope helpers are `httpresp.OK/Created/Error/ServerError`
(`internal/httpresp/response.go`). Tests use the in-package `MemoryRepository`
and `authctx.WithUserID`; constants `MaxCalories`/`MaxMacroGrams` and meal
validation (`MealType.Valid()`) already exist.

- [ ] **Step 1: Add a typed caller-facing build error**

In `handler.go`, near the other helpers at the bottom of the file, add:

```go
// logBuildError is a caller-facing failure from the log-entry build
// helpers (bad input or a missing/unowned source). It carries the HTTP
// status and message the single handlers already return, so extracting
// the build logic doesn't change their 400/404 responses. Storage/system
// failures are returned as plain errors instead and map to 500.
type logBuildError struct {
	status int
	msg    string
}

func (e *logBuildError) Error() string { return e.msg }

// writeLogBuildError maps a build-helper error onto the response: a
// *logBuildError becomes its carried status+message, anything else is a
// 500 logged under op.
func writeLogBuildError(w http.ResponseWriter, r *http.Request, op string, err error) {
	var be *logBuildError
	if errors.As(err, &be) {
		httpresp.Error(w, be.status, be.msg)
		return
	}
	httpresp.ServerError(w, r.Context(), op, err)
}
```

- [ ] **Step 2: Extract `buildLogEntry` (pantry/recipe) from `createLogEntry`**

Add this helper (it is the body of `createLogEntry` minus the auth/decode and
minus the final insert + 201 write). It performs the same validation and macro
derivation and returns the unsaved entry:

```go
// buildLogEntry validates a pantry- or recipe-backed log request and
// returns the unsaved NutritionLogEntry with denormalized macros frozen at
// log time. Caller-facing problems come back as *logBuildError; storage
// failures come back as plain errors. It does not insert.
func (h *Handler) buildLogEntry(ctx context.Context, userID string, req logEntryRequest) (*NutritionLogEntry, error) {
	hasPantry := req.PantryItemID != nil && *req.PantryItemID != ""
	hasRecipe := req.RecipeID != nil && *req.RecipeID != ""
	if hasPantry == hasRecipe {
		return nil, &logBuildError{http.StatusBadRequest, ErrLogEntryReferenceRequired.Error()}
	}
	if req.Quantity <= 0 {
		return nil, &logBuildError{http.StatusBadRequest, ErrQuantityNonPositive.Error()}
	}
	meal := MealType(req.Meal)
	if !meal.Valid() {
		return nil, &logBuildError{http.StatusBadRequest, ErrInvalidMeal.Error()}
	}
	consumedAt := time.Now().UTC()
	if req.ConsumedAt != nil {
		consumedAt = req.ConsumedAt.UTC()
	}
	entry := &NutritionLogEntry{
		UserID:     userID,
		ConsumedAt: consumedAt,
		Quantity:   req.Quantity,
		Meal:       meal,
	}
	if hasPantry {
		pantry, err := h.repo.GetPantryItem(ctx, userID, *req.PantryItemID)
		if err != nil {
			if errors.Is(err, ErrNotFound) {
				return nil, &logBuildError{http.StatusNotFound, "pantry item not found"}
			}
			return nil, err
		}
		entry.PantryItemID = req.PantryItemID
		entry.Calories = req.Quantity * pantry.Calories
		entry.ProteinG = req.Quantity * pantry.ProteinG
		entry.FatG = req.Quantity * pantry.FatG
		entry.CarbsG = req.Quantity * pantry.CarbsG
	} else {
		macros, err := h.repo.ComputeRecipeMacros(ctx, userID, *req.RecipeID)
		if err != nil {
			if errors.Is(err, ErrNotFound) {
				return nil, &logBuildError{http.StatusNotFound, "recipe not found"}
			}
			return nil, err
		}
		scaled := macros.Scale(req.Quantity)
		entry.RecipeID = req.RecipeID
		entry.Calories = scaled.Calories
		entry.ProteinG = scaled.ProteinG
		entry.FatG = scaled.FatG
		entry.CarbsG = scaled.CarbsG
	}
	return entry, nil
}
```

Then rewrite `createLogEntry` to delegate (behavior unchanged):

```go
func (h *Handler) createLogEntry(w http.ResponseWriter, r *http.Request) {
	userID, ok := auth.UserIDFrom(r.Context())
	if !ok {
		httpresp.ServerError(w, r.Context(), "missing user in context", errors.New("auth middleware not applied"))
		return
	}
	var req logEntryRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		httpresp.Error(w, http.StatusBadRequest, "invalid request body")
		return
	}
	entry, err := h.buildLogEntry(r.Context(), userID, req)
	if err != nil {
		writeLogBuildError(w, r, "build log entry", err)
		return
	}
	if err := h.repo.CreateNutritionLogEntry(r.Context(), entry); err != nil {
		httpresp.ServerError(w, r.Context(), "create nutrition log entry", err)
		return
	}
	httpresp.Created(w, "logged consumption", toLogDTO(*entry))
}
```

- [ ] **Step 3: Extract `buildCustomLogEntry` from `createCustomLogEntry`**

```go
// buildCustomLogEntry validates a one-off custom-meal request and returns
// the unsaved NutritionLogEntry (quantity fixed at 1, macros verbatim).
// Caller-facing problems come back as *logBuildError. It does not insert.
func buildCustomLogEntry(userID string, req customLogEntryRequest) (*NutritionLogEntry, error) {
	name := strings.TrimSpace(req.Name)
	if name == "" {
		return nil, &logBuildError{http.StatusBadRequest, "name is required"}
	}
	if len(name) > 200 {
		return nil, &logBuildError{http.StatusBadRequest, "name is too long"}
	}
	if req.Calories < 0 || req.Calories > MaxCalories {
		return nil, &logBuildError{http.StatusBadRequest, "calories out of range"}
	}
	for _, m := range []struct {
		name string
		val  float64
	}{
		{"protein_g", req.ProteinG},
		{"fat_g", req.FatG},
		{"carbs_g", req.CarbsG},
	} {
		if m.val < 0 || m.val > MaxMacroGrams {
			return nil, &logBuildError{http.StatusBadRequest, m.name + " out of range"}
		}
	}
	meal := MealType(req.Meal)
	if !meal.Valid() {
		return nil, &logBuildError{http.StatusBadRequest, ErrInvalidMeal.Error()}
	}
	consumedAt := time.Now().UTC()
	if req.ConsumedAt != nil {
		consumedAt = req.ConsumedAt.UTC()
	}
	return &NutritionLogEntry{
		UserID:         userID,
		ConsumedAt:     consumedAt,
		CustomMealName: &name,
		Quantity:       1,
		Calories:       req.Calories,
		ProteinG:       req.ProteinG,
		FatG:           req.FatG,
		CarbsG:         req.CarbsG,
		Meal:           meal,
	}, nil
}
```

Then rewrite `createCustomLogEntry` to delegate:

```go
func (h *Handler) createCustomLogEntry(w http.ResponseWriter, r *http.Request) {
	userID, ok := auth.UserIDFrom(r.Context())
	if !ok {
		httpresp.ServerError(w, r.Context(), "missing user in context", errors.New("auth middleware not applied"))
		return
	}
	var req customLogEntryRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		httpresp.Error(w, http.StatusBadRequest, "invalid request body")
		return
	}
	entry, err := buildCustomLogEntry(userID, req)
	if err != nil {
		writeLogBuildError(w, r, "build custom log entry", err)
		return
	}
	if err := h.repo.CreateNutritionLogEntry(r.Context(), entry); err != nil {
		httpresp.ServerError(w, r.Context(), "create custom nutrition log entry", err)
		return
	}
	httpresp.Created(w, "logged custom meal", toLogDTO(*entry))
}
```

Add `"context"` to the imports if not present (it is needed by
`buildLogEntry`'s signature).

- [ ] **Step 4: Run existing tests to confirm the refactor is behavior-preserving**

Run: `go test ./internal/nutrition/...`
Expected: PASS (all existing single-endpoint tests still green).

- [ ] **Step 5: Add the batch request/response types + the cap constant**

In `handler.go`, with the other DTOs:

```go
// MaxBatchLogItems caps a single batch request — a runaway-model guard
// well above any real meal. See sows/batch-food-logging.md.
const MaxBatchLogItems = 50

// batchLogItemRequest is one item in a POST /nutrition-log/batch body.
// `kind` is the authoritative selector; the per-kind fields reuse the
// single endpoints' field names.
type batchLogItemRequest struct {
	Kind         string     `json:"kind"`
	PantryItemID *string    `json:"pantry_item_id"`
	RecipeID     *string    `json:"recipe_id"`
	Quantity     float64    `json:"quantity"`
	Name         string     `json:"name"`
	Calories     float64    `json:"calories"`
	ProteinG     float64    `json:"protein_g"`
	FatG         float64    `json:"fat_g"`
	CarbsG       float64    `json:"carbs_g"`
	Meal         string     `json:"meal"`
	ConsumedAt   *time.Time `json:"consumed_at"`
}

type batchLogRequest struct {
	Items []batchLogItemRequest `json:"items"`
}

// batchLogResultDTO is one index-aligned per-item outcome. Entry is set
// when ok; Error is set otherwise. omitempty keeps each object to the
// shape that applies.
type batchLogResultDTO struct {
	Index int          `json:"index"`
	OK    bool         `json:"ok"`
	Entry *logEntryDTO `json:"entry,omitempty"`
	Error string       `json:"error,omitempty"`
}

type batchLogResponseDTO struct {
	Results []batchLogResultDTO `json:"results"`
	Logged  int                 `json:"logged"`
	Failed  int                 `json:"failed"`
}
```

- [ ] **Step 6: Add the batch handler**

```go
// createLogEntriesBatch logs a heterogeneous list of items best-effort:
// each item is validated and inserted independently, a failure on one
// never aborts the loop or rolls back a sibling, and per-item outcomes are
// returned index-aligned to the request. The envelope is 400 only when it
// is itself malformed (no items, or over the cap); otherwise it is 200
// even if every item failed.
func (h *Handler) createLogEntriesBatch(w http.ResponseWriter, r *http.Request) {
	userID, ok := auth.UserIDFrom(r.Context())
	if !ok {
		httpresp.ServerError(w, r.Context(), "missing user in context", errors.New("auth middleware not applied"))
		return
	}
	var req batchLogRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		httpresp.Error(w, http.StatusBadRequest, "invalid request body")
		return
	}
	if len(req.Items) == 0 {
		httpresp.Error(w, http.StatusBadRequest, "items is required")
		return
	}
	if len(req.Items) > MaxBatchLogItems {
		httpresp.Error(w, http.StatusBadRequest, "too many items in one batch")
		return
	}

	results := make([]batchLogResultDTO, 0, len(req.Items))
	logged, failed := 0, 0
	for i, item := range req.Items {
		entry, err := h.buildBatchEntry(r.Context(), userID, item)
		if err == nil {
			err = h.repo.CreateNutritionLogEntry(r.Context(), entry)
		}
		if err != nil {
			failed++
			results = append(results, batchLogResultDTO{
				Index: i,
				OK:    false,
				Error: batchItemErrorMessage(r, err),
			})
			continue
		}
		logged++
		dto := toLogDTO(*entry)
		results = append(results, batchLogResultDTO{Index: i, OK: true, Entry: &dto})
	}
	httpresp.OK(w, "logged consumption batch", batchLogResponseDTO{
		Results: results,
		Logged:  logged,
		Failed:  failed,
	})
}

// buildBatchEntry dispatches one batch item on its kind to the shared
// build helper, mapping the item's fields onto the single endpoints'
// request shapes.
func (h *Handler) buildBatchEntry(ctx context.Context, userID string, item batchLogItemRequest) (*NutritionLogEntry, error) {
	switch item.Kind {
	case "pantry":
		return h.buildLogEntry(ctx, userID, logEntryRequest{
			PantryItemID: item.PantryItemID,
			Quantity:     item.Quantity,
			Meal:         item.Meal,
			ConsumedAt:   item.ConsumedAt,
		})
	case "recipe":
		return h.buildLogEntry(ctx, userID, logEntryRequest{
			RecipeID:   item.RecipeID,
			Quantity:   item.Quantity,
			Meal:       item.Meal,
			ConsumedAt: item.ConsumedAt,
		})
	case "custom":
		return buildCustomLogEntry(userID, customLogEntryRequest{
			Name:       item.Name,
			Calories:   item.Calories,
			ProteinG:   item.ProteinG,
			FatG:       item.FatG,
			CarbsG:     item.CarbsG,
			Meal:       item.Meal,
			ConsumedAt: item.ConsumedAt,
		})
	default:
		return nil, &logBuildError{http.StatusBadRequest, "unknown item kind: " + item.Kind}
	}
}

// batchItemErrorMessage renders a per-item error for the results array.
// Caller-facing build errors carry their own message; an unexpected
// storage failure is logged server-side and reported generically so a raw
// internal error never leaks to the agent.
func batchItemErrorMessage(r *http.Request, err error) string {
	var be *logBuildError
	if errors.As(err, &be) {
		return be.msg
	}
	httpresp.LogServerError(r.Context(), "batch log item insert", err)
	return "internal error"
}
```

NOTE: `httpresp.LogServerError` may not exist. Before writing this, check
`internal/httpresp/response.go` for the logging primitive `ServerError`
uses. If there is no public log-only helper, instead inline the same logging
call `ServerError` makes (e.g. `slog`/the package logger) so the storage
error is still recorded, OR fall back to `log` via the package's existing
pattern. Do not write the response from this helper — the batch always
returns 200. Match whatever logging idiom the package already uses; if the
only available primitive writes a response, add a tiny unexported
`logServerError(ctx, op, err)` in the nutrition package mirroring it. Keep
the generic `"internal error"` string for the per-item `error`.

- [ ] **Step 7: Mount the route**

In `Mount`, inside the `/nutrition-log` route block, register `/batch`
alongside the other literal segments (before `/{id}`):

```go
r.Route("/nutrition-log", func(r chi.Router) {
	r.Get("/daily", h.dailyMacros)
	r.Post("/custom", h.createCustomLogEntry)
	r.Post("/batch", h.createLogEntriesBatch)
	r.Get("/", h.listLogEntries)
	r.Post("/", h.createLogEntry)
	r.Get("/{id}", h.getLogEntry)
	r.Put("/{id}", h.updateLogEntry)
	r.Delete("/{id}", h.deleteLogEntry)
})
```

- [ ] **Step 8: Write the batch tests**

In `handler_test.go`, add a helper to drive the batch handler (mirroring
`postCustom`) and tests. The `MemoryRepository` supports `CreatePantryItem`;
seed a real pantry item so a `kind:"pantry"` item resolves. Cover:

- **All-success mixed batch** (pantry + recipe + custom in one call) → 200,
  `logged==3`, `failed==0`, `results` index-aligned, each `entry` populated,
  and macros denormalized identically to the single endpoint (e.g. a pantry
  item with calories C and quantity q yields `entry.calories == q*C`).
- **Partial failure**: one valid pantry item + one unknown pantry id →
  `logged==1`, `failed==1`; `results[0].ok==true`, `results[1].ok==false`
  with a non-empty `error`; indexes 0 and 1.
- **All-fail still 200**: two unknown pantry ids → HTTP 200, `logged==0`,
  `failed==2`.
- **Empty items** (`{"items":[]}`) → 400.
- **Over-cap** (51 items) → 400.
- **Per-item ownership**: a pantry id owned by another user fails just that
  item (seed a pantry item under user "u2", reference it from a "u1" batch).
- **Custom name maps to `custom_meal_name`**: a `kind:"custom"` item's
  `name` lands on the returned entry's `custom_meal_name` (Open Question 2).
- **Regression**: a focused test that the extracted helpers didn't change
  single-endpoint behavior — e.g. `createLogEntry` with an unknown pantry id
  still returns 404 "pantry item not found", and `createCustomLogEntry` with
  an out-of-range macro still returns 400. (If existing tests already cover
  these, assert it explicitly here too so the refactor is pinned.)

Use the existing envelope-decoding pattern; add a `batchEnvelope` struct
typed as `{ Message string; Data batchLogResponseDTO }`.

- [ ] **Step 9: Run the full gate**

Run from repo root:
```
go test ./...
go vet ./...
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run --timeout=5m
go mod tidy && git diff --exit-code go.mod go.sum
```
Expected: all green, no go.mod/go.sum drift. Fix any lint/gosec finding by
changing the code — never add `//nolint` or silence a rule.

- [ ] **Step 10: Commit**

```bash
git add internal/nutrition/handler.go internal/nutrition/handler_test.go
git commit -m "feat(nutrition): add best-effort POST /nutrition-log/batch"
```

---

## Repo: prog-strength-mcp

### Task 2: Replace the two single-item tools with `log_consumption_batch`

**Files:**
- Modify: `src/prog_strength_mcp/nutrition.py` (remove two tools, add union + batch tool)
- Modify: `src/prog_strength_mcp/api_client.py` (remove `log_consumption`/`log_custom_meal`, add `log_consumption_batch`)
- Modify: `src/prog_strength_mcp/pantry.py` (docstrings: `log_consumption` → `log_consumption_batch`)
- Modify: `tests/test_nutrition_tools.py` (remove old-tool tests, add batch tests)

**Context:** The two tools (`nutrition.py:37-115` `log_consumption`,
`117-174` `log_custom_meal`) are the agent loop's only consumers and their
`api_client` methods (`api_client.py:374-403`, `405-439`) are called only by
them — after removal both are orphaned, so remove them too (clean cutover).
The API endpoint from Task 1 accepts `{"items":[{kind,...}]}` and returns
`{"data":{"results","logged","failed"}}`. Tools register via `@mcp.tool`;
`tool.fn` is the raw coroutine and `tool.run(args)` is the validated path that
raises `pydantic.ValidationError`. `_auth_header_or_raise()` already exists.
respx mocks HTTP in tests. Per-item field descriptions (quantity guidance,
meal-bucket cues, RFC3339 `consumed_at`) carry over verbatim from the current
tools.

- [ ] **Step 1: Add the Pydantic discriminated union + new tool in `nutrition.py`**

Add imports (`BaseModel`, `Literal` already imported via typing) and the
models above `register`, or at module top:

```python
from pydantic import BaseModel, Field

_MEAL = Literal["breakfast", "lunch", "dinner", "snack"]


class PantryItem(BaseModel):
    kind: Literal["pantry"]
    pantry_item_id: str
    quantity: float = Field(gt=0)
    meal: _MEAL
    consumed_at: str | None = None


class RecipeItem(BaseModel):
    kind: Literal["recipe"]
    recipe_id: str
    quantity: float = Field(gt=0)
    meal: _MEAL
    consumed_at: str | None = None


class CustomItem(BaseModel):
    kind: Literal["custom"]
    name: str = Field(min_length=1, max_length=200)
    calories: float = Field(ge=0, le=100_000)
    protein_g: float = Field(ge=0, le=10_000)
    fat_g: float = Field(ge=0, le=10_000)
    carbs_g: float = Field(ge=0, le=10_000)
    meal: _MEAL
    consumed_at: str | None = None


Item = Annotated[PantryItem | RecipeItem | CustomItem, Field(discriminator="kind")]
```

Remove the `log_consumption` and `log_custom_meal` `@mcp.tool` registrations
entirely. Add inside `register`:

```python
    @mcp.tool
    async def log_consumption_batch(
        items: Annotated[
            list[Item],
            Field(
                min_length=1,
                description=(
                    "Every food the user mentioned in their message, as one "
                    "list. A snack of two foods is two items in one call; a "
                    "mixed meal can combine pantry-backed and custom items. "
                    "Set each item's `kind`: use 'pantry'/'recipe' when a "
                    "saved item matches (check the pantry first), otherwise "
                    "'custom' with your best-estimate macros."
                ),
            ),
        ],
    ) -> dict[str, Any]:
        """Log everything the user ate in one message as a single call.

        Collect EVERY food the user mentioned into one `items` list rather
        than calling this tool once per food. Each item carries its own
        `meal` bucket and optional `consumed_at`, so one call can span
        breakfast + a snack or backfill several entries at different times.

        For each item, check `list_pantry_items` first: if a saved pantry
        item or recipe matches, use kind "pantry"/"recipe" with its id and a
        `quantity` of servings/batches (5 eggs against a 1-egg pantry item is
        quantity=5; half a recipe is 0.5). Otherwise use kind "custom" with
        the food's name and your best-estimate total macros as the user ate
        them.

        Logging is best-effort: each item is logged independently and the
        response's `results`/`logged`/`failed` report which items (if any)
        did not log, so you can tell the user. `meal` is one of breakfast,
        lunch, dinner, snack. `consumed_at` is an RFC3339 UTC timestamp;
        omit it to default to now.
        """
        auth = _auth_header_or_raise()
        payload = [item.model_dump(exclude_none=True) for item in items]
        try:
            return await api.log_consumption_batch(auth, items=payload)
        except APIError as e:
            raise RuntimeError(f"API error ({e.status_code}): {e.message}") from e
```

Update the module docstring's "Phase 1 ships log_consumption..." line to name
`log_consumption_batch`.

- [ ] **Step 2: Add `api_client.log_consumption_batch`, remove the two old methods**

Remove `log_consumption` (`api_client.py:374-403`) and `log_custom_meal`
(`405-439`). Add:

```python
    async def log_consumption_batch(
        self,
        auth_header: str,
        *,
        items: list[dict[str, Any]],
    ) -> dict[str, Any]:
        """POST /nutrition-log/batch. `items` is the already-serialized
        list of discriminated-union items ({kind, ...}). Returns the
        `{results, logged, failed}` data payload — best-effort, so a 200
        can still carry per-item failures in `results`.
        """
        resp = await self._client.post(
            "/nutrition-log/batch",
            json={"items": items},
            headers={"Authorization": auth_header},
        )
        _raise_for_status(resp)
        data = resp.json().get("data")
        return data if isinstance(data, dict) else {}
```

- [ ] **Step 3: Update `pantry.py` docstrings**

Replace the three `log_consumption` references in `pantry.py` docstrings
(lines ~53, 105, 110) with `log_consumption_batch` so the model is never told
to call a tool that no longer exists. Pure docstring edits.

- [ ] **Step 4: Rewrite the nutrition tests**

In `tests/test_nutrition_tools.py`:
- Remove the `log_custom_meal` api-client tests (`test_log_custom_meal_*`),
  the `log_custom_meal` tool-boundary tests, and the `_custom_meal_tool`
  fixture and its constraint tests — the tool and api method are gone.
- Remove the `log_custom_meal` method from the `_ExplodingAPI` stub (and add
  a `log_consumption_batch` exploder there only if a test reaches it).
- Add tests:
  - **api client forwards the batch body + auth** (respx): post a
    heterogeneous `items` list via `APIClient.log_consumption_batch`, assert
    the request JSON is `{"items":[...]}` with each item's `kind` and fields
    intact and the `Authorization` header set; assert the returned dict is
    the `data` payload.
  - **tool forwards a heterogeneous list** (respx + patched
    `get_http_headers`): call the registered `log_consumption_batch` via
    `tool.run({"items":[pantry-item, custom-item]})`; assert the API received
    the right body (kinds preserved, `consumed_at` omitted when None).
  - **discriminated union validates per-kind required fields**: `tool.run`
    with a `kind:"pantry"` item missing `pantry_item_id`, and a
    `kind:"custom"` item missing `calories`, each raises
    `pydantic.ValidationError` before any HTTP.
  - **per-item failure surfaced**: mock the API to return
    `{"data":{"results":[{"index":0,"ok":false,"error":"pantry item not found"}],"logged":0,"failed":1}}`
    and assert the tool returns that payload (the agent can read `failed`).
  - **auth guard fires before the call**: with no auth header, `tool.fn`
    (or `tool.run`) raises before HTTP (reuse the `_ExplodingAPI` pattern).
  - **removed tools are gone**: after `nutrition.register`, `mcp.get_tool`
    for `log_consumption` and `log_custom_meal` raises / returns nothing,
    and `log_consumption_batch` is present.

- [ ] **Step 5: Run the gate**

Read `AGENTS.md`/`CONTRIBUTING.md`/`README.md` for the exact commands; the
repo uses `uv` + `ruff` + `pytest` + likely `mypy`/`pyright`. Run the lint,
format-check, type-check, and `pytest` the repo documents. Fix anything red by
changing code (no rule suppression). Expected: green.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(nutrition): replace single-item log tools with log_consumption_batch"
```

---

## Repo: prog-strength-agent

### Task 3: Stream `item_count` on `tool_use_start` + update the prompt

**Files:**
- Modify: `src/prog_strength_agent/model_harness.py` (item_count on tool_use_start)
- Modify: `src/prog_strength_agent/prompt.py` (name `log_consumption_batch`)
- Modify: `tests/test_model_harness.py` (item_count test)
- Modify: `tests/test_prompt.py` (update tool-name assertions)

**Context:** `model_harness.py:203-211` emits the `tool_use_start` SSE event
with only `block.name`, inside the `content_block_start` handler where
`block = event.content_block`. The prompt's nutrition-logging guidance
(`prompt.py` ~125-161) names `log_consumption`/`log_custom_meal`. The intent
name `log_nutrition` (model_router.py, intents.py) is NOT a tool name — leave
it. NOTE on streaming: in Anthropic streaming, `content_block_start` may carry
an empty `input` for a tool_use (the args arrive via subsequent deltas), so
the helper must tolerate a missing/empty `items` and return `None`; the web
chip degrades to a plain "Log Consumption" when the count is absent. Implement
exactly as the SOW specifies (derive from `block.input`).

- [ ] **Step 1: Add an `item_count` helper**

Near the other module helpers in `model_harness.py`:

```python
def _batch_item_count(name: str, block_input: Any) -> int | None:
    """Item count for a log_consumption_batch tool call, for the web chip.
    None for every other tool, or when the input isn't available yet (the
    streamed tool_use input can be empty at content_block_start)."""
    if name != "log_consumption_batch":
        return None
    if not isinstance(block_input, dict):
        return None
    items = block_input.get("items")
    return len(items) if isinstance(items, list) else None
```

- [ ] **Step 2: Emit `item_count` on the event**

Replace the `tool_use_start` yield (`model_harness.py:206-211`) with:

```python
                            if block.type == "tool_use":
                                start_event: dict[str, Any] = {
                                    "type": "tool_use_start",
                                    "name": block.name,
                                }
                                count = _batch_item_count(
                                    block.name, getattr(block, "input", None)
                                )
                                if count is not None:
                                    start_event["item_count"] = count
                                yield _sse(start_event)
```

- [ ] **Step 3: Add a model_harness unit test**

In `tests/test_model_harness.py`, test `_batch_item_count` directly (it is a
pure function, testable without the streaming machinery the suite avoids):

```python
def test_batch_item_count_for_batch_tool():
    from prog_strength_agent.model_harness import _batch_item_count

    assert _batch_item_count("log_consumption_batch", {"items": [1, 2, 3]}) == 3
    assert _batch_item_count("log_consumption_batch", {"items": []}) == 0


def test_batch_item_count_none_for_other_tools_and_missing_input():
    from prog_strength_agent.model_harness import _batch_item_count

    assert _batch_item_count("list_pantry_items", {"items": [1]}) is None
    assert _batch_item_count("log_consumption_batch", None) is None
    assert _batch_item_count("log_consumption_batch", {}) is None
```

- [ ] **Step 4: Update the prompt to name `log_consumption_batch`**

In `prompt.py`, rewrite the nutrition-logging guidance so every
`log_consumption` / `log_custom_meal` mention becomes `log_consumption_batch`,
and state the one-call-for-the-whole-message expectation including the mixed
pantry + custom case. Concretely:
- In the "Logging a meal the user describes in chat" paragraph (~125-146):
  collect every food the user mentioned into ONE `log_consumption_batch` call;
  for each food, prefer a pantry/recipe match (kind pantry/recipe), else a
  custom item with best-estimate macros; a mixed snack (one pantry item + one
  custom food) is still a single call. Keep the `lookup_food_nutrition`
  guidance and the "after logging, offer to save to pantry" ask, just retarget
  the tool name and make it batch-aware.
- In the "Logging meals from a photo" paragraph (~148-161): replace
  `log_custom_meal` mentions with `log_consumption_batch` (a multi-item
  receipt is one batch call after the user's "yes"); keep "propose first, log
  on yes."

Preserve the existing voice and all non-tool guidance. Do not invent new
behavior — only retarget the tool name and add the "one call" expectation.

- [ ] **Step 5: Update `test_prompt.py`**

Lines ~120 and ~154 assert `"log_custom_meal" in SYSTEM_PROMPT` / `in out`.
Update these assertions to `"log_consumption_batch"`. Read the surrounding
test to keep its intent (it checks the meal-logging guidance is present);
adjust any prose-specific assertions to match the rewritten paragraph.

- [ ] **Step 6: Run the gate**

Run the lint/format/type/test commands the repo's `AGENTS.md`/`CONTRIBUTING.md`
document (uv + ruff + pytest, likely a type checker). Expected: green. Fix red
by changing code.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(agent): stream item_count on tool_use_start; point prompt at log_consumption_batch"
```

---

## Repo: prog-strength-web

### Task 4: `ToolCall.itemCount` + the count chip

**Files:**
- Modify: `lib/stream.ts` (item_count on the tool_use_start SSE event type)
- Modify: `components/chat/types.ts` (`itemCount?` on `ToolCall`)
- Modify: `app/(app)/chat/page.tsx` (populate itemCount from the event)
- Modify: `components/chat/tool-pill.tsx` (the count label)
- Create: `components/chat/tool-pill.test.tsx` (vitest)

**Context:** `stream.ts:12` types the `tool_use_start` event as
`{ type; name }`. `page.tsx:620-632` handles `tool_use_start`, pushing
`{ name, state: "running" }` to `toolsLog` and onto the message's `tools`.
`tool-pill.tsx` renders the chip via `humanizeToolName(tool.name)`. `ToolCall`
round-trips through persistence as JSON (`persistedToUI`), so adding
`itemCount` to the type carries through automatically. Tests use vitest +
@testing-library (see `components/chat/macro-card.test.tsx`). This is a
text-label change only — no new design tokens; the chip keeps its existing
`var(--accent)` styling.

- [ ] **Step 1: Extend the SSE event type**

In `lib/stream.ts`, change the tool_use_start variant:

```ts
  | { type: "tool_use_start"; name: string; item_count?: number }
```

- [ ] **Step 2: Extend `ToolCall`**

In `components/chat/types.ts`:

```ts
export type ToolCall = {
  name: string;
  state: "running" | "ok" | "error";
  itemCount?: number;
};
```

- [ ] **Step 3: Populate `itemCount` from the event in `page.tsx`**

In the `tool_use_start` branch (~620-632), thread `ev.item_count` onto both
the `toolsLog.push` and the `setMessages` running-tool object:

```ts
        } else if (ev.type === "tool_use_start") {
          toolsLog.push({ name: ev.name, state: "running", itemCount: ev.item_count });
          setMessages((prev) =>
            replaceLast(prev, (last) => ({
              ...last,
              tools: [
                ...(last.tools ?? []),
                { name: ev.name, state: "running", itemCount: ev.item_count },
              ],
            })),
          );
        }
```

(`itemCount: undefined` when the field is absent is fine — optional property.)

- [ ] **Step 4: Add the count label in `tool-pill.tsx`**

Replace `const name = humanizeToolName(tool.name);` with `const name =
toolLabel(tool);` and add:

```ts
/**
 * Display label for a tool chip. log_consumption_batch reads "Log
 * Consumption" with an optional " (N items)" suffix from the batch's item
 * count (singular "1 item"); every other tool uses humanizeToolName.
 */
function toolLabel(tool: ToolCall): string {
  if (tool.name === "log_consumption_batch") {
    const base = "Log Consumption";
    if (typeof tool.itemCount === "number") {
      return `${base} (${tool.itemCount} ${tool.itemCount === 1 ? "item" : "items"})`;
    }
    return base;
  }
  return humanizeToolName(tool.name);
}
```

- [ ] **Step 5: Write `tool-pill.test.tsx`**

Mirror `macro-card.test.tsx`'s render/assert pattern. Render `<ToolPill>` for:
- a batch call with `itemCount: 2`, state "ok" → text includes "Log
  Consumption (2 items)" (and NOT "Log Consumption Batch").
- `itemCount: 1`, state "ok" → "Log Consumption (1 item)".
- `log_consumption_batch` with no `itemCount`, state "ok" → "Log Consumption"
  (no parenthetical).
- a non-batch tool (e.g. `list_exercises`), state "ok" → "List Exercises"
  (humanizeToolName path unchanged).
- optionally a "running" batch chip → "Calling Log Consumption (2 items)…".

- [ ] **Step 6: Run the gate**

Read `AGENTS.md`/`CONTRIBUTING.md`. Run the documented lint + typecheck + test
(eslint, `tsc`/`next lint`, `vitest run`). Expected: green. Fix red by
changing code.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(chat): render Log Consumption (N items) count chip for batch logging"
```

---

## Self-review notes (planner)

- **Spec coverage:** API best-effort batch (Task 1: handler, helpers, 200/400
  semantics, cap 50, per-item results, custom name→custom_meal_name, ownership,
  regression); single endpoints untouched (Task 1 delegates without behavior
  change); MCP union tool + removal of both old tools + api_client method
  (Task 2); agent item_count + prompt (Task 3); web itemCount + chip (Task 4).
  Testing section items map to Steps 8 (api), 4 (mcp), 3 (agent), 5 (web).
- **Open Questions:** Q1 cap = 50 (Step 5/8 api). Q2 name→custom_meal_name
  verified by a dedicated test (Step 8 api). Q3 chip shows attempted count
  from tool_use_start only — failures are not counted on the chip (matches the
  deferred decision).
- **Streaming caveat:** `block.input` can be empty at `content_block_start`;
  `_batch_item_count` returns None then and the chip degrades gracefully. This
  is the SOW's specified approach (derive from `block.input` on tool_use_start)
  and is implemented as written.
