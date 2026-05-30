# Daily Nutrition Log

**Status**: Shipped · **Last updated**: 2026-05-29

## Introduction

Strength is built outside the gym as much as inside it, and **Prog Strength** today knows nothing about either side of that equation: there's no place to log a meal, no place to log a bodyweight, and so the chat agent's claims about diet — protein adequacy, caloric balance, weight trajectory — can only be guesses. The lifter who tracks macros religiously sees the difference; the agent can't.

This feature gives the agent (and the user) ground truth on both axes. Users build a small digital pantry of foods they buy regularly with per-serving macros, optionally compose pantry items into recipes for the meals they repeat (a "Standard Breakfast" of five eggs, two strips of turkey bacon, and a bagel collapses to a single tap), and log consumption throughout the day either via a form on the new `/nutrition` page or by telling the agent in chat. Bodyweight gets the same treatment on a sibling `/bodyweight` page — a quick entry form, a trend chart, and an MCP tool so the agent can record a morning weight from natural language. Date navigation lets the user scroll backwards through the log; the agent gets read access to the same history so it can speak to protein intake against bodyweight, caloric intake against weight trend, and (eventually) carb intake against running volume.

After this ships, a typical interaction looks like: open the app post-breakfast, tap "Standard Breakfast" — or say it in chat — and the day's macros are on screen before coffee is finished. The agent stops guessing about diet and starts citing it.

## Proposed Solution

Four new persistent concepts on the API:

1. **Pantry items** — user-owned food entries with per-serving macros (calories, protein, fat, carbs), a serving size, and a serving unit (`"egg"`, `"slice"`, `"100g"`, etc.). Users build up a pantry over time so logging is fast.
2. **Recipes** — user-owned named collections of pantry items with per-component quantities. Recipe macros are derived by summing components; nothing is stored on the recipe row itself for macros. A recipe is essentially a saved "quick-add" bundle.
3. **Nutrition log entries** — one row per logged consumption event, tagged with either a `pantry_item_id` or a `recipe_id` (exactly one), a quantity (multiplier), and a `consumed_at` timestamp. Macros are denormalized onto the row at log time so editing a pantry item later does not retroactively change yesterday's totals.
4. **Bodyweight entries** — one row per measurement, weight + unit + `measured_at` timestamp. Unit is denormalized per row so a user changing their preferred unit doesn't reinterpret historical readings.

CRUD endpoints for each. Read endpoints support date-range filtering for daily summaries (frontend) and historical context (agent). Soft-deletes on every table to match the workout-domain pattern already established in the codebase.

The agent gets new MCP tools mirroring the API: `create_pantry_item`, `list_pantry_items`, `create_recipe`, `list_recipes`, `log_consumption`, `list_nutrition_log`, `log_bodyweight`, `list_bodyweight`, and a small `get_daily_macros` summary tool that returns one row per day with totals (saves the agent from re-aggregating on every turn). UPDATE and DELETE intentionally stay UI-only at MVP — the agent can create and read, but the user controls revisions. Friction-shifting: creating a wrong entry by mistake is annoying but recoverable; an agent unintentionally rewriting yesterday's log is harder to notice.

The frontend gets a new `/nutrition` page with three views: **today's log** (date-pickable, previous/next day arrows, daily macros widget at the top, list of logged consumptions below with quick-add affordances), **pantry** (table of saved items with edit/delete and a "new item" form), and **recipes** (list of recipes with their derived macros widget and a builder that picks pantry items + quantities). A separate `/bodyweight` page hosts the trend chart + the quick-entry form. Sidebar entries for both. The chat tab is unchanged; the agent just has more tools.

This is a large feature. We ship it as one SOW because the data model is internally cohesive and splitting "pantry" from "log" from "bodyweight" leaves intermediate states where the app has half a story. The implementation order across the codebase is API → MCP → frontend, with bodyweight able to ship independently of nutrition at any point along the way.

## Goals and Non-Goals

### Goals

- Persist user-owned pantry items, recipes (with components), nutrition log entries, and bodyweight entries across four new tables. All soft-delete capable.
- Expose CRUD HTTP endpoints under `/pantry-items`, `/recipes`, `/nutrition-log`, and `/bodyweight` (auth required).
- Denormalize macros onto nutrition log entries at log time so historical totals are immutable under future pantry-item edits.
- Add date-range queries to the nutrition-log and bodyweight list endpoints so the frontend can render a chosen day's totals and the agent can pull weekly/monthly windows.
- Add a derived "daily totals" read endpoint that aggregates a date range into per-day macro sums — saves the agent from re-aggregating on every request, and feeds the daily-summary widget on the frontend.
- Expose MCP tools for every CRUD action the agent should be able to perform on the user's behalf (full create/read on all four domains; no agent-side update or delete in v1).
- Add `/nutrition` and `/bodyweight` pages to the web app with the views described in the Proposed Solution.
- Add a recharts line chart of bodyweight over time, with a 7-day rolling average overlay so daily noise doesn't dominate the visual signal.

### Non-Goals

- **A curated public food database.** Pantry items are user-private; no shared catalog like the exercise catalog. Building a real food database is a different product (USDA-style import + verification + admin workflow). Out of scope here.
- **Barcode scanning or photo-based logging.** Native-only features that don't fit a managed Expo / Next.js web stack and don't move the needle on the "I eat the same thing every day" core use case.
- **Calorie / macro goals + adherence scoring.** The agent can comment on intake vs. bodyweight trends from raw data; we don't introduce a "set my daily protein goal to 200g" UI yet. Goal-setting is a follow-up once the log itself has signal.
- **Multi-unit conversion (g ↔ oz, ml ↔ cup).** Serving unit is a free-text descriptor stored alongside the item. If a user wants gram-precision, they enter their item that way; if they want per-egg, they enter it that way. No on-the-fly conversion at v1.
- **Ad-hoc nutrition entries with macros typed directly into the log.** Every log entry references a pantry item or recipe. Logging a restaurant meal means first creating a pantry item for it (or asking the agent to do so) — friction we accept to keep the schema clean and the pantry an asset that grows over time. Revisit if users complain.
- **Agent-side UPDATE / DELETE of pantry items, recipes, or log entries.** Read + create only for the agent. UI owns revisions. Adding write paths is a follow-up once the create path is well-understood.
- **Body composition beyond weight** (body-fat %, muscle mass, measurements). One number per measurement keeps the trend chart honest. Composition is a different sensor pipeline.
- **Time-zone-aware "today" math on the API.** Timestamps are UTC, queries accept UTC ranges, the frontend computes the user's local-day boundaries before querying. A future per-user timezone preference can centralize this.
- **Backfill of any historical data.** New tables; no prior data to migrate. New users (and existing users) see empty states until they log.
- **Bodyweight averaging / per-day deduplication on write.** A user can log bodyweight multiple times per day; the trend chart picks the earliest entry per day for display. Storage layer doesn't enforce one-per-day.

## Implementation Details

### Data Model

Four new tables in one migration (`006_nutrition_and_bodyweight.sql`). Keeping them in one migration because they're a coherent unit and writing four sequential migrations for the same SOW would just be ceremony.

#### `pantry_items`

User's saved food items. Read-mostly: created once, edited occasionally, referenced by every log entry.

| Column | Type | Description |
| --- | --- | --- |
| `id` | text | Primary key. |
| `user_id` | text | Owner. |
| `name` | text | Free-form display name (e.g. `"Eggland's Best Large Egg"`). |
| `calories` | real | Per serving. CHECK >= 0. |
| `protein_g` | real | Per serving. CHECK >= 0. |
| `fat_g` | real | Per serving. CHECK >= 0. |
| `carbs_g` | real | Per serving. CHECK >= 0. |
| `serving_size` | real | Numeric magnitude of one serving. CHECK > 0. |
| `serving_unit` | text | Free-text unit label (`"egg"`, `"slice"`, `"100g"`, `"cup"`). |
| `created_at` | datetime | |
| `updated_at` | datetime | |
| `deleted_at` | datetime | NULL when active; non-NULL for soft-deleted. |

Index: `(user_id, name)` filtered on `deleted_at IS NULL` for the pantry list view's search-as-you-type.

#### `recipes` and `recipe_items`

A recipe is a named bag of (pantry_item, quantity) pairs. Macros are computed at read time and at log time — never stored on the recipe row.

`recipes`:

| Column | Type | Description |
| --- | --- | --- |
| `id` | text | Primary key. |
| `user_id` | text | Owner. |
| `name` | text | Display name (e.g. `"Standard Breakfast"`). |
| `created_at` | datetime | |
| `updated_at` | datetime | |
| `deleted_at` | datetime | Soft delete. |

`recipe_items`:

| Column | Type | Description |
| --- | --- | --- |
| `id` | text | Primary key. |
| `recipe_id` | text | FK → `recipes(id)` ON DELETE CASCADE. |
| `pantry_item_id` | text | FK → `pantry_items(id)`. No cascade — a deleted pantry item should not silently mutate recipes. |
| `quantity` | real | Number of servings of the pantry item this recipe uses. CHECK > 0. |
| `position` | integer | 0-indexed display order within the recipe. |
| `created_at` | datetime | |

Indexes:

- `recipe_items(recipe_id, position)` — the dominant read query, "give me this recipe's components in order."
- `UNIQUE(recipe_id, position)` — slot uniqueness within a recipe; the write path writes positions densely from 0 in one transaction.

Note: `recipe_items` does not soft-delete. The CASCADE on `recipe_id` covers recipe deletion; pantry-item deletion does not propagate (the recipe row stays as-is, and the macro derivation will simply omit a missing component with a warning on the read path).

#### `nutrition_log_entries`

One row per consumed item. Exactly one of `pantry_item_id` or `recipe_id` is set per row, enforced by CHECK. Macros are denormalized so historical totals are immutable.

| Column | Type | Description |
| --- | --- | --- |
| `id` | text | Primary key. |
| `user_id` | text | Owner. |
| `consumed_at` | datetime | When the user actually ate it. UTC. |
| `pantry_item_id` | text | FK → `pantry_items(id)`, nullable. |
| `recipe_id` | text | FK → `recipes(id)`, nullable. |
| `quantity` | real | Multiplier: number of servings (pantry item) or number of batches (recipe). CHECK > 0. |
| `calories` | real | Computed at log time. Frozen against future pantry edits. |
| `protein_g` | real | Computed at log time. |
| `fat_g` | real | Computed at log time. |
| `carbs_g` | real | Computed at log time. |
| `created_at` | datetime | |
| `deleted_at` | datetime | Soft delete. |

CHECK constraint: `(pantry_item_id IS NOT NULL) <> (recipe_id IS NOT NULL)` — exactly one of the two foreign keys is set per row.

Index: `(user_id, consumed_at DESC)` filtered on `deleted_at IS NULL` — drives both "today's entries" and the daily-aggregate read path.

#### `bodyweight_entries`

| Column | Type | Description |
| --- | --- | --- |
| `id` | text | Primary key. |
| `user_id` | text | Owner. |
| `weight` | real | Magnitude. CHECK > 0. |
| `unit` | text | `"lb"` or `"kg"`. CHECK IN ('lb','kg'). Denormalized so a user changing their preference doesn't reinterpret history. |
| `measured_at` | datetime | When the reading was taken. UTC. |
| `created_at` | datetime | |
| `deleted_at` | datetime | Soft delete. |

Index: `(user_id, measured_at DESC)` filtered on `deleted_at IS NULL` — drives the trend chart and the agent's bodyweight tool.

### Write Path

- **Create pantry item** — validate macros ≥ 0 and serving_size > 0; insert.
- **Update pantry item** — full replacement of editable fields. Does NOT cascade to nutrition_log entries (deliberate; see denormalization rationale).
- **Delete pantry item** — soft-delete the row. Recipes referencing it stay intact but flag the component as "missing" on read; the read path warns the user via a small UI badge. The pantry item can be undeleted by writing `deleted_at = NULL`.
- **Create recipe** — insert recipe row, then insert recipe_items in one transaction with `position` taken from request order. Validate that every `pantry_item_id` belongs to the same user.
- **Update recipe** — replace the recipe name and its components atomically: DELETE all `recipe_items` for the recipe, INSERT the new set in order. Same set-replacement pattern used by `user_headline_exercises`.
- **Delete recipe** — soft-delete the recipe; CASCADE deletes `recipe_items` (hard cascade on the row, not the soft-delete column).
- **Create nutrition log entry** — validate that exactly one of pantry/recipe ID is present and that it belongs to the user; compute denormalized macros from the referenced pantry item or recipe components; insert.
- **Update nutrition log entry** — fields editable: `consumed_at`, `quantity`. Re-derive macros from the original pantry/recipe and the new quantity.
- **Delete nutrition log entry** — soft-delete.
- **Create bodyweight entry** — insert with the user's current preferred unit if the client doesn't specify, otherwise honor the request body. CHECK constraints enforce shape.
- **Delete bodyweight entry** — soft-delete.

Consistency invariants:

- Every log entry's macros are non-negative.
- Every recipe's pantry components belong to the recipe's owner.
- A recipe's `position` slots are dense (0..N-1) inside any successfully-committed transaction.
- A pantry item's `deleted_at` does not change the macros of any historical log entry that referenced it.

### Read Path

Three shapes the frontend and agent need:

- **Per-domain list** — `GET /pantry-items`, `GET /recipes`, `GET /nutrition-log`, `GET /bodyweight`, all supporting `since` / `until` filters where dates apply and excluding soft-deleted rows by default. Recipes' list response inlines their derived macros widget (sum of components × quantities) so a caller doesn't N+1 fetch components.
- **Single resource** — `GET /pantry-items/{id}`, etc., for edit-form pre-population.
- **Daily aggregate** — `GET /nutrition-log/daily?since=…&until=…` returns one row per day in the range with totals (calories, protein_g, fat_g, carbs_g, entry_count). Computed server-side over `nutrition_log_entries` so the frontend daily widget and the agent's `get_daily_macros` tool share one source of truth.

### Macro Math

Pantry-item macros are stored per serving. The total contribution of one log entry is:

```
entry_calories = quantity × pantry_item.calories          (when pantry_item_id is set)
entry_calories = quantity × sum_over_components(
                              recipe_item.quantity × pantry_item.calories
                            )                              (when recipe_id is set)
```

Same shape for protein / fat / carbs. The recipe form is exactly the pantry-item form repeated over the recipe's component list. We compute this once at log time and store the result on the row.

For the bodyweight trend chart's 7-day rolling average:

```
avg_at_day_d = mean(weights observed in [d-6, d])
```

Computed client-side from the per-day reading (the earliest measurement of that day). No server-side smoothing — the raw entries are the source of truth, and recharts handles the windowing in JS.

### API Surface

All endpoints under `auth.RequireUser`; no public reads.

**Pantry items**

- `POST /pantry-items` — create. Body: `{name, calories, protein_g, fat_g, carbs_g, serving_size, serving_unit}`. Response: created item.
- `GET /pantry-items?since=…&until=…&q=…` — list. `q` filters by name substring (case-insensitive). Date filters are on `created_at`. Soft-deleted excluded.
- `GET /pantry-items/{id}` — one item.
- `PUT /pantry-items/{id}` — full replacement of editable fields.
- `DELETE /pantry-items/{id}` — soft-delete.

**Recipes**

- `POST /recipes` — create. Body: `{name, components: [{pantry_item_id, quantity}]}`. Response: created recipe with derived macros.
- `GET /recipes` — list with components + derived macros inlined.
- `GET /recipes/{id}` — one recipe.
- `PUT /recipes/{id}` — full replacement of name and component set.
- `DELETE /recipes/{id}` — soft-delete.

**Nutrition log**

- `POST /nutrition-log` — create. Body: `{consumed_at, pantry_item_id?, recipe_id?, quantity}`. Server validates exactly-one-of and derives macros.
- `GET /nutrition-log?since=…&until=…` — list entries with denormalized macros.
- `GET /nutrition-log/daily?since=…&until=…` — daily aggregates (one row per UTC date in range with totals).
- `PUT /nutrition-log/{id}` — edit `consumed_at` and/or `quantity`. Re-derives macros.
- `DELETE /nutrition-log/{id}` — soft-delete.

**Bodyweight**

- `POST /bodyweight` — create. Body: `{weight, unit, measured_at}`. `unit` optional (defaults to user's preferred unit).
- `GET /bodyweight?since=…&until=…` — list entries, most recent first.
- `DELETE /bodyweight/{id}` — soft-delete.

No `PUT /bodyweight` — corrections are delete + recreate, which keeps the trend chart's audit trail clean.

Response envelopes follow the existing `{service, message, data}` shape.

### MCP Tools

The agent gets read access everywhere and create access on log/measurement events. No update / delete for v1 — the user owns revisions.

**Pantry**

- `list_pantry_items(q?)` → list of items with macros and serving info.
- `create_pantry_item(name, calories, protein_g, fat_g, carbs_g, serving_size, serving_unit)` → created item.

**Recipes**

- `list_recipes()` → recipes with components + derived macros.
- `create_recipe(name, components)` → created recipe.

**Nutrition log**

- `log_consumption(consumed_at, pantry_item_id?, recipe_id?, quantity)` → created entry. Same exactly-one-of validation as the HTTP endpoint.
- `list_nutrition_log(since, until)` → entries.
- `get_daily_macros(since, until)` → per-day totals. Optimized for the common agent prompt "how did my macros look this week?"

**Bodyweight**

- `log_bodyweight(measured_at, weight, unit?)` → created entry.
- `list_bodyweight(since, until)` → entries.

Tools live in three new modules under `prog-strength-mcp/src/prog_strength_mcp/`: `pantry.py`, `nutrition.py`, `bodyweight.py`. Each exports a `register(mcp, api)` function following the existing `workouts.py` / `exercises.py` pattern.

### Frontend Pages

Two new top-level routes, both auth-gated under the `(app)` route group:

- `/nutrition` — date-aware default landing on today. Header: date selector (previous / next day buttons, "today" reset, calendar popover). Below the header, a macro summary widget for the selected day (calories total + protein/fat/carbs with progress bars showing each macro's share of total calories). Below that, the day's log entries in chronological order, each row showing the pantry-item or recipe name, quantity, time, and contribution to the day's totals. A "quick-add" row at the top of the list opens a small picker — recent items, then recipes, then full pantry search.
- `/pantry` — two-tab page (Pantry / Recipes). The Pantry tab is a table of saved items with edit-in-place and a "New item" form at the top. The Recipes tab is a list of recipes with their derived macro widget; clicking a recipe opens an editor with the component picker.
- `/bodyweight` — recharts line chart of bodyweight over time (raw points + dashed 7-day average overlay), an entry form at the top, and a paginated list of recent entries below.

Sidebar links: "Nutrition" and "Bodyweight" — separate links so the user can land in either context with one click.

Visual language matches the rest of the app: dark surface, accent on actionable elements, muted on metadata. Macro pills follow the same hue-per-domain pattern as the muscle-group pills on `/progress`.

### Backfill or Migration

**Mechanism**: none required. All four tables are new. New users and existing users see empty pantry, empty recipe list, empty nutrition log, empty bodyweight history until they log.

**Recoverability**: tables are independent of `workouts` and `personal_records`. If any of these is corrupted, dropping and re-creating the table is safe — the user loses logged history (a real loss, no derivation possible), but the rest of the app is unaffected. No cross-table cascade reaches the workout side of the schema.

**Scale boundary**: at single-user scale with one nutrition entry per meal and one bodyweight per morning, we're looking at ~1500 nutrition rows and ~365 bodyweight rows per year. SQLite handles this comfortably indefinitely. If the multi-user beta opens later, the per-user indexes on `(user_id, consumed_at DESC)` and `(user_id, measured_at DESC)` keep query cost proportional to per-user data, not global data. A real scale boundary lives at the millions-of-rows-per-user level, which is decades away at human eating frequency.

## Resolutions

1. **Ad-hoc nutrition log entries.** Shipped the lean: pantry-only, every log entry references either a pantry item or a recipe. The schema CHECK enforces exactly-one. Mid-conversation "log this restaurant meal" turns work via the agent calling `create_pantry_item` first and then `log_consumption` against the new ID. Worth revisiting if the friction shows up in real usage, but the pantry-as-asset framing has held so far.

2. **Recipe component limit.** Shipped at 20, enforced server-side in `Recipe.Validate` and mirrored in the web recipe-builder UI. Matches the headline-exercises cap pattern. Easy to bump with a one-line constant change if it ever bites.

3. **Bodyweight per-day duplication policy.** Shipped option (a): the API accepts multiple readings per day without enforcement. The trend chart on `/bodyweight` plots every reading and overlays a 7-day rolling average, which absorbs daily noise without losing intra-day data points. No per-day uniqueness constraint exists in `bodyweight_entries`.

4. **Serving unit canonicalization.** Shipped option (a): `serving_unit` is free text. The schema only enforces non-empty. Math is purely magnitude-based; the unit string is descriptive metadata. No suggestion list in the UI yet — revisit if normalized units become useful for cross-item aggregation.

5. **Goal tracking.** Shipped option (c): deferred entirely. Daily macro tiles show each macro's share of total macro-calories (using 4/4/9 per-gram math) as a no-config visual cue, but there is no stored goal anywhere. The agent does soft coaching from raw bodyweight + macros in conversation. A future "set my protein target" feature would warrant its own SOW.

6. **Agent write surface.** Shipped the lean: agent can `create_pantry_item`, `create_recipe`, `log_consumption`, and `log_bodyweight`, but cannot update or delete any of them. Revisions go through the UI. The risk balance has held — the agent's create paths are reliable; rewrite paths would expand the blast radius of a bad multi-step turn without proportional UX gain.

7. **Time-zone handling for "today".** Shipped option (a): the frontend computes local-day boundaries (`startOfLocalDay` / `endOfLocalDay` in `/nutrition/page.tsx`) and queries the API with UTC ranges. The agent passes UTC RFC3339 to its read tools. No timezone field on the user record. A future scheduled-task feature (e.g. reminders at a user-local time) is what would drive (b); not warranted yet.
