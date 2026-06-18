---
status: draft
repos:
  - prog-strength-api
  - prog-strength-mcp
  - prog-strength-agent
  - prog-strength-web
---

# Batch Food Logging: One Agent Tool Call Per Meal

**Status**: Draft · **Last updated**: 2026-06-17

## Introduction

When a user tells the **Prog Strength** agent "I had some dried mango and a rice
krispy treat as a snack just now," the agent logs each food with a **separate
tool call** — two `log_consumption` (or `log_custom_meal`) invocations for two
foods, three for three. Every call is exposed to the web chat as its own
tool-call chip, so a single snack renders as a row of repeated "Log
Consumption" badges.

The two logging tools today are both single-item:

- `log_consumption(quantity, meal, pantry_item_id | recipe_id, consumed_at?)`
  — logs one consumption event against a saved pantry item or recipe
  (`prog-strength-mcp/src/prog_strength_mcp/nutrition.py:37-115`).
- `log_custom_meal(name, calories, protein_g, fat_g, carbs_g, meal,
  consumed_at?)` — logs one one-off meal with agent-estimated macros
  (`nutrition.py:117-174`).

Logging multiple foods at once is the **common case**, not the exception — a
meal or a snack is usually more than one item. Forcing one tool call per item
is the wrong default: it is slower (sequential round trips through the agent
loop), noisier in the chat UI, and it makes a mixed snack — one pantry-backed
item plus one one-off custom food — physically impossible to express in a
single call, because the two item types live in two different tools.

After this work ships, the agent logs everything the user mentioned in one
message as **exactly one tool call**, regardless of how many foods or which
kinds, and the web chat shows a single chip that names the count.

## Goals and Non-Goals

### Goals

- Replace the two single-item MCP tools (`log_consumption`, `log_custom_meal`)
  with one unified `log_consumption_batch` tool that accepts a list of
  **heterogeneous** items — each independently pantry-backed, recipe-backed, or
  a custom one-off.
- Add a **best-effort** batch endpoint to the Go API
  (`POST /nutrition-log/batch`): each item validated and inserted
  independently, successful inserts never rolled back because a sibling failed,
  per-item results returned so the agent can tell the user what (if anything)
  did not log.
- Keep the existing single-item API endpoints (`POST /nutrition-log`,
  `POST /nutrition-log/custom`) untouched — the web and mobile apps call them
  directly for manual, non-agent logging.
- Surface the batch in the web chat as one chip that reads
  **"Log Consumption (N items)"**, preserving the at-a-glance signal that
  multiple foods were logged.
- Keep `meal` and `consumed_at` **per-item**, so one call can span meal buckets
  (breakfast + a snack) or backfill several entries at different times.

### Non-Goals

- **Removing or changing the single-item API endpoints.** They have non-agent
  consumers (web/mobile manual logging) and stay exactly as they are. The batch
  endpoint is purely additive.
- **A batch *update* or *delete* endpoint.** This SOW batches creation only;
  editing and deleting log entries remain single-item.
- **Client-side batching in web/mobile manual-logging UIs.** Those flows log one
  item at a time by direct user action and are out of scope; only the
  agent-driven path is consolidated.
- **Changing macro estimation, lookup, or denormalization behavior.** Each
  item's macros are computed exactly as the single endpoints compute them
  today (frozen at log time); this SOW changes *how many* items move per call,
  not *what* the numbers are.
- **A transaction that rolls the whole batch back on one failure.** Explicitly
  rejected in favor of best-effort: one bad macro estimate must not block
  logging the rest of a meal.

## Proposed Solution

A single MCP tool, `log_consumption_batch`, takes a list of items where each
item is a discriminated union on a `kind` field:

- `kind: "pantry"` → `pantry_item_id, quantity, meal, consumed_at?`
- `kind: "recipe"` → `recipe_id, quantity, meal, consumed_at?`
- `kind: "custom"` → `name, calories, protein_g, fat_g, carbs_g, meal, consumed_at?`

The tool forwards the list to a new Go API endpoint, `POST /nutrition-log/batch`,
which validates and inserts each item independently and returns a per-item
results array. The MCP tool returns that array to the agent so the agent's
reply can mention any items that failed. The web chat reads an item count
streamed on the existing `tool_use_start` SSE event and renders one chip.

The work spans four repos and is described API-first, since the endpoint is the
contract everything else is written against.

### `prog-strength-api`: best-effort batch endpoint

New route in `internal/nutrition/handler.go`'s `Mount`, registered alongside
the existing literal segments (before `/{id}`):

```go
r.Post("/batch", h.createLogEntriesBatch)
```

**Request** — a thin envelope around a list. Each item reuses the field names
the single endpoints already accept, discriminated by which fields are present
(the same exactly-one-of logic the single handlers enforce today). A `kind`
discriminator is set by the MCP tool and is the authoritative selector:

```json
{
  "items": [
    { "kind": "pantry", "pantry_item_id": "…", "quantity": 1, "meal": "snack" },
    { "kind": "custom", "name": "Rice krispy treat", "calories": 90,
      "protein_g": 1, "fat_g": 2, "carbs_g": 17, "meal": "snack" }
  ]
}
```

**Best-effort semantics.** The handler iterates `items`, and for each one runs
the *same* build-and-insert logic the single handlers run, collecting either
the created entry or a per-item error. A failure on item *i* (unknown pantry
id, item not owned by the user, out-of-range macro, bad `meal`) records an
error for *i* and **does not** abort the loop or undo prior inserts.

**Response** — `200` with a per-item results array and roll-up counts, in the
API's standard envelope:

```json
{ "data": {
    "results": [
      { "index": 0, "ok": true, "entry": { /* logEntryDTO */ } },
      { "index": 1, "ok": false, "error": "pantry item not found" }
    ],
    "logged": 1,
    "failed": 1
  } }
```

`results` is index-aligned to the request `items`. `200` is returned whenever
the **envelope** was well-formed, even if every item failed — per-item outcomes
live in the array, not the HTTP status, so the agent always gets a structured
report. Request-level `400` is reserved for a malformed envelope:

- `items` empty or missing.
- `items` length over the cap (**50**) — a guard against a runaway model
  request, well above any real meal.

**Refactor for shared logic.** `createLogEntry`
(`handler.go:321-399`) and `createCustomLogEntry` (`handler.go:405-467`) each
contain the core "validate this request and build a `NutritionLogEntry` with
denormalized macros" logic. Extract that core into two reusable helpers (e.g.
`buildLogEntry(ctx, userID, req) (NutritionLogEntry, error)` and
`buildCustomLogEntry(...)`) so the single handlers and the batch loop run one
code path. The batch handler dispatches on `kind` to the right helper. No
behavior change to the single endpoints — they keep their current request/
response shapes and just call the extracted helper.

**Repository.** Inserts can reuse the existing single-insert repository method
in a loop (simplest, and best-effort doesn't need a transaction). A
`CreateMany` batch insert is an optional optimization, not required for
correctness; default to looping the existing method unless a test shows it
matters.

### `prog-strength-mcp`: replace the two tools with one

In `nutrition.py`, **remove** the `log_consumption` and `log_custom_meal` tool
registrations and **add** `log_consumption_batch`. Each item is a Pydantic
discriminated union so the JSON schema reads clearly to the model:

```python
class PantryItem(BaseModel):
    kind: Literal["pantry"]
    pantry_item_id: str
    quantity: float = Field(gt=0)
    meal: Literal["breakfast", "lunch", "dinner", "snack"]
    consumed_at: str | None = None

class RecipeItem(BaseModel):
    kind: Literal["recipe"]
    recipe_id: str
    quantity: float = Field(gt=0)
    meal: Literal["breakfast", "lunch", "dinner", "snack"]
    consumed_at: str | None = None

class CustomItem(BaseModel):
    kind: Literal["custom"]
    name: str = Field(min_length=1, max_length=200)
    calories: float = Field(ge=0, le=100_000)
    protein_g: float = Field(ge=0, le=10_000)
    fat_g: float = Field(ge=0, le=10_000)
    carbs_g: float = Field(ge=0, le=10_000)
    meal: Literal["breakfast", "lunch", "dinner", "snack"]
    consumed_at: str | None = None

Item = Annotated[PantryItem | RecipeItem | CustomItem, Field(discriminator="kind")]
```

```python
@mcp.tool
async def log_consumption_batch(items: list[Item]) -> dict[str, Any]:
    """Log everything the user ate in one message as a single call."""
```

The tool's docstring carries the guidance the two old docstrings held, adapted
to the batch: **collect every food the user mentioned in their message into one
call**; per-item `meal` and `consumed_at`; check the pantry first and use
`kind:"pantry"`/`"recipe"` when a saved item matches, otherwise `kind:"custom"`
with best-estimate macros; the per-item field descriptions carry over verbatim
from the current tools (the dumbbell-style quantity guidance, the meal-bucket
cues, the RFC3339 `consumed_at` note).

A new `api_client.log_consumption_batch(auth, items)` posts the list to
`/nutrition-log/batch` and returns the `data` results payload. The tool returns
that payload so the agent can read `failed` / per-item `error` and tell the
user which foods didn't log.

### `prog-strength-agent`: stream the item count

The web chip needs to know how many items the batch call carried, and that
count is only knowable from the tool input. In
`model_harness.py`, the `tool_use_start` SSE event
(`model_harness.py:206-211`) currently forwards only `block.name`. Add an
`item_count` field derived from `block.input` — `len(block.input["items"])` for
`log_consumption_batch`, omitted/`None` for every other tool. One small,
backward-compatible addition; no schema model exists for these events (they are
inline dicts), so no type to extend.

### `prog-strength-web`: the count chip

Two small changes:

- `components/chat/types.ts` — extend `ToolCall` with an optional
  `itemCount?: number`, populated from the `item_count` field on the
  `tool_use_start` SSE event where the stream is parsed.
- `components/chat/tool-pill.tsx` — `humanizeToolName` would render the raw tool
  name as "Log Consumption Batch". Special-case `log_consumption_batch` to the
  label **"Log Consumption"**, and when `itemCount` is present append
  ` (N items)` (singular "1 item" when count is 1). All other tools keep the
  generic `humanizeToolName` path.

## Implementation Details

### Removal safety

Removing the two MCP tools is safe: the agent loop is their only consumer, and
the agent will call `log_consumption_batch` instead (prompt/tool guidance moves
with the tool). The Go API's single endpoints stay live for the web/mobile
manual-logging UIs, which call them directly via their `lib/api.ts` wrappers
(`createNutritionLogEntry`, `createCustomNutritionLogEntry`) and never went
through MCP.

### Agent prompt

Wherever the system prompt names `log_consumption` / `log_custom_meal` for the
nutrition-logging intent, update it to `log_consumption_batch` and state the
"one call for the whole message" expectation, including the mixed pantry +
custom case. (The exact prompt location is in `prog-strength-agent`'s prompt
module; the implementer should grep the prompt for the old tool names and the
`log_nutrition` intent rules.)

### Mobile

`prog-strength-mobile`'s chat (if it renders tool-call chips) is **out of scope**
for the count label in this SOW; the shared agent SSE change is harmless to it
(an unused extra field). A matching mobile chip tweak can follow if desired.

## Testing

- **`prog-strength-api`** (`internal/nutrition/handler_test.go`): all-success
  mixed batch (pantry + recipe + custom in one call); partial failure (a valid
  item logs, an unknown pantry id fails, `results`/`logged`/`failed` correct and
  index-aligned); all-fail still `200` with errors; empty `items` → `400`;
  over-cap (>50) → `400`; per-item ownership enforced (another user's pantry id
  fails just that item); macros denormalized identically to the single
  endpoint. Plus a regression check that the extracted helpers didn't change
  single-endpoint behavior.
- **`prog-strength-mcp`** (`tests/test_nutrition_tools.py`, respx): the tool
  forwards the correct batch body for a heterogeneous list; the discriminated
  union validates each `kind`'s required fields; per-item failure in the API
  response is surfaced to the agent; auth guard fires before the call; the two
  removed tools are gone from the registered tool set.
- **`prog-strength-agent`**: `tool_use_start` carries `item_count` for
  `log_consumption_batch` and omits it otherwise.
- **`prog-strength-web`**: `tool-pill` renders "Log Consumption (2 items)" for a
  batch call with `itemCount: 2`, "Log Consumption (1 item)" for 1, and a plain
  label when `itemCount` is absent.

## Rollout

1. **api PR** — `POST /nutrition-log/batch` + extracted build helpers + tests.
   Mergeable first; dormant until the MCP tool points at it. No change to single
   endpoints.
2. **agent PR** — `item_count` on `tool_use_start`; prompt updated to name
   `log_consumption_batch`. (Pair with the mcp PR so the prompt and tool ship
   together.)
3. **mcp PR** — remove the two tools, add `log_consumption_batch` +
   `api_client` method + tests. This is the cutover; once merged the agent logs
   via the batch path.
4. **web PR** — `ToolCall.itemCount` + the count chip. Safe any time; degrades
   to a plain chip if the count is absent.

## Open Questions

1. **Batch cap value.** 50 is a generous runaway guard; no realistic meal
   approaches it. Lower (e.g. 25) if we'd rather fail loud on an obviously
   broken model request, but 50 leaves comfortable headroom.
2. **`custom` item field name.** The single custom endpoint takes `name` on the
   request and stores it as `custom_meal_name`. The batch item uses `name` for
   symmetry with the existing tool; confirm the batch handler maps `name` →
   `custom_meal_name` consistently with `createCustomLogEntry`.
3. **Should the chip count failures?** v1 shows the *attempted* item count from
   `tool_use_start` (known before execution). Showing "2 logged, 1 failed" would
   need the count plumbed onto `tool_result` instead/additionally. Deferred
   unless the failure case proves common in practice.
