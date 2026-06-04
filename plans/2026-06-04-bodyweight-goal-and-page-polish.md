# Bodyweight Goal + Web Page Polish — Implementation Plan

> **For agentic workers:** Implement task-by-task. Each task is implemented by a
> subagent, then spec-reviewed and code-quality-reviewed before moving on.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a per-user bodyweight goal (new table + read/write API + chart line
+ stat tile + toolbar affordance), add `PUT /bodyweight/{id}` to power an inline
edit affordance, and land the adjacent web polish (chart-card title, tooltip
rounding fix, restyled accent stat tiles, icon-only row actions with edit/delete
modals).

**SOW:** `sows/bodyweight-goal-and-page-polish.md`

**Repos:** `prog-strength-api` (Go), `prog-strength-web` (Next.js 16 / React 19 /
TS / recharts 3), `prog-strength-docs` (this plan + SOW status flip).

**Merge order:** api → web → docs. The API PR is independently revertable until
web merges (new endpoints just go dark).

## Key decisions / deviations from the SOW prose

- **Migration number is `011`, not `010`.** The SOW body says
  `010_user_bodyweight_goal.sql`, but `010_chat_sessions_last_intent.sql` already
  exists on `main`. The next free number is `011`. Filename:
  `internal/db/migrations/011_user_bodyweight_goal.sql`. Table name stays
  `user_bodyweight_goal` per the SOW data-model table.
- **`--surface-2` already exists** in `app/globals.css` (`#27272a`), one shade
  lighter than `--surface` (`#18181b`) which the chart card uses. No new variable
  needed — the accent tiles use `bg-[var(--surface-2)]` directly as the SOW's
  example code shows.
- **New web components stay as in-file siblings in `page.tsx`**, matching the
  existing `BodyweightLogModal` / `ChartCard` / `StatTile` which are already
  in-file siblings. (The nutrition modals live under `components/nutrition/`, but
  the bodyweight page's own modal is inline; we stay consistent with that file.)
- **Goal lives in the existing `internal/bodyweight` package** as a sibling of
  `Entry`, per the SOW ("`internal/bodyweight/goal` would over-split").

---

## prog-strength-api

### Task A1 — Migration + `Goal` domain type + validation + errors

Files: `internal/db/migrations/011_user_bodyweight_goal.sql`,
`internal/bodyweight/goal.go`, `internal/bodyweight/goal_test.go`. Extend
`internal/bodyweight/errors.go`.

- [ ] `011_user_bodyweight_goal.sql` — create table `user_bodyweight_goal`:
  - `user_id TEXT PRIMARY KEY`
  - `weight REAL NOT NULL CHECK(weight > 0 AND weight <= 2000)`
  - `unit TEXT NOT NULL CHECK(unit IN ('lb', 'kg'))`
  - `created_at DATETIME NOT NULL`
  - `updated_at DATETIME NOT NULL`
  - No FK on `user_id` (matches `009_user_macro_goals.sql`). Mirror its comment style.
- [ ] `goal.go` — `Goal` struct: `UserID string`, `Weight float64`,
  `Unit user.WeightUnit`, `CreatedAt *time.Time`, `UpdatedAt *time.Time` (pointer
  timestamps so the "never set" state is zero-value + nil, exactly like
  `nutrition.MacroGoals`). Add `const MaxGoalWeight = 2000`.
  - `func (g Goal) Validate() error` — fail-at-first-error in SOW order:
    1. `Weight <= 0` → `ErrWeightNonPositive`
    2. `Weight > MaxGoalWeight` → `ErrWeightTooLarge`
    3. `Unit` not `lb`/`kg` → `ErrInvalidUnit` (reuse existing sentinel)
- [ ] `errors.go` — add `ErrWeightTooLarge = errors.New("weight must be <= 2000")`.
  Reuse the existing `ErrWeightNonPositive` and `ErrInvalidUnit`.
- [ ] `goal_test.go` — `Goal.Validate()` rejects weight ≤ 0, weight > 2000, and
  unit ∉ {lb, kg}; accepts the happy cases (175 lb, 80 kg). Use `errors.Is`.

### Task A2 — Repository: goal upsert/get + entry update (interface, sqlite, memory, tests)

Files: `internal/bodyweight/repository.go`, `sqlite_repository.go`,
`memory_repository.go`, and repo tests in `bodyweight_test.go` (or a new
`sqlite_repository_test.go` following `internal/exercise/sqlite_repository_test.go`).

- [ ] `repository.go` — add to the `Repository` interface:
  - `GetBodyweightGoal(ctx context.Context, userID string) (Goal, error)` —
    returns zero-valued `Goal{UserID: userID}` with nil timestamps when never set;
    NEVER returns `ErrNotFound`. (Mirror `nutrition.GetMacroGoals`.)
  - `UpsertBodyweightGoal(ctx context.Context, g Goal, now time.Time) (Goal, error)` —
    atomic insert-or-replace; sets `created_at` on insert, `updated_at` every call;
    re-reads to return real timestamps.
  - `UpdateEntry(ctx context.Context, e *Entry) error` — validates via
    `e.Validate()`, `UPDATE ... WHERE id = ? AND user_id = ? AND deleted_at IS NULL`,
    returns `ErrNotFound` on 0 rows affected. Does NOT touch `created_at`.
    (Mirror `nutrition.UpdateNutritionLogEntry`, `sqlite_repository.go:283`.)
- [ ] `sqlite_repository.go`:
  - `GetBodyweightGoal` — `SELECT user_id, weight, unit, created_at, updated_at
    FROM user_bodyweight_goal WHERE user_id = ?`; `sql.ErrNoRows` →
    `Goal{UserID: userID}, nil`.
  - `UpsertBodyweightGoal` — `INSERT ... ON CONFLICT(user_id) DO UPDATE SET
    weight = excluded.weight, unit = excluded.unit, updated_at = excluded.updated_at`
    (created_at untouched in the UPDATE clause); then re-read via `GetBodyweightGoal`.
    Mirror `macro_goals_sqlite.go`.
  - `UpdateEntry` — `UPDATE bodyweight_entries SET weight = ?, unit = ?,
    measured_at = ? WHERE id = ? AND user_id = ? AND deleted_at IS NULL`; check
    `RowsAffected() == 1` else `ErrNotFound`.
- [ ] `memory_repository.go` — add a `goals map[string]*Goal` (guarded by the
  existing mutex). Implement all three methods with defensive copies on return,
  matching the existing memory-repo style. `UpdateEntry` enforces the same
  ownership / soft-delete `ErrNotFound` semantics as `Delete`.
- [ ] Repo tests:
  - `UpsertBodyweightGoal`: insert then update — second call replaces value,
    preserves `CreatedAt`, bumps `UpdatedAt`.
  - `GetBodyweightGoal` empty state: nil timestamps, no error.
  - `UpdateEntry`: happy path updates only supplied fields and preserves
    `CreatedAt`; soft-deleted row → `ErrNotFound`; cross-user row → `ErrNotFound`.
  - Run against both memory and sqlite where the existing tests do.

### Task A3 — Handlers + routing: `/me/bodyweight-goal` (GET/PUT) and `PUT /bodyweight/{id}`

Files: `internal/bodyweight/handler_goal.go`,
`internal/bodyweight/handler.go` (add `update` + mount), and tests
`handler_goal_test.go` + entry-update cases in `bodyweight_test.go`/`handler_test.go`.

- [ ] `handler_goal.go`:
  - `goalDTO` — `Weight float64`, `Unit string`, `CreatedAt *time.Time`,
    `UpdatedAt *time.Time` (json `weight`, `unit`, `created_at`, `updated_at`).
    `toGoalDTO(g Goal) goalDTO`.
  - `putBodyweightGoalRequest` — `Weight *float64`, `Unit *string` (pointers so
    missing fields are detectable).
  - `getMyBodyweightGoal` — `userID := auth.UserIDFrom`; `repo.GetBodyweightGoal`;
    `httpresp.OK(toGoalDTO(g))`. Empty state returns the zero DTO → JSON
    `{"weight":0,"unit":"lb","created_at":null,"updated_at":null}`. NOTE: ensure the
    empty `Goal{}` serializes `unit` as `"lb"` — set `Unit: "lb"` default in
    `GetBodyweightGoal`'s not-found branch OR in `toGoalDTO` when unit is empty.
  - `putMyBodyweightGoal` — decode body; validate (weight required & > 0; weight
    ≤ 2000; unit required & ∈ {lb,kg}) with the exact SOW 400 messages
    (`"weight must be positive"`, `"weight must be <= 2000"`,
    `"unit must be 'lb' or 'kg'"`); `repo.UpsertBodyweightGoal(ctx, Goal{...},
    time.Now().UTC())`; `httpresp.OK(toGoalDTO(saved))`.
- [ ] `handler.go`:
  - Inside `Mount`, add `r.Put("/{id}", h.update)` to the existing
    `r.Route("/bodyweight", ...)` block, and add a sibling
    `r.Route("/me/bodyweight-goal", func(r chi.Router) { r.Get("/", h.getMyBodyweightGoal); r.Put("/", h.putMyBodyweightGoal) })`
    inside `Mount` (same pattern as `nutrition` mounting `/me/macro-goals`).
  - `update` handler — partial body `{weight?, unit?, measured_at?}`: `repo.Get`
    the existing row first (404 → `ErrNotFound`), overlay supplied fields, validate
    (`weight > 0` if supplied, `unit ∈ {lb,kg}` if supplied), `repo.UpdateEntry`,
    return 200 with `toDTO`. Use `*float64` / `*string` / `*time.Time` request
    fields so omitted fields preserve existing values.
- [ ] Tests:
  - `handler_goal_test.go` — GET empty-state shape (weight 0, unit "lb", nil
    timestamps); PUT happy path returns saved row with non-nil timestamps; PUT
    validation cases (zero, negative, >2000, invalid unit) each 400 with expected
    message; PUT-after-PUT returns the second value (idempotent set-replacement).
  - Entry-update cases — `PUT /bodyweight/{id}` happy path returns the updated row;
    soft-deleted row → 404; different user's row → 404; weight ≤ 0 → 400; partial
    body preserves omitted fields. Use the `authctx.WithUserID` + `httptest` pattern
    from the nutrition handler tests.
- [ ] Verify: `go build ./...`, `go test ./internal/bodyweight/...`,
  `go test ./...`, `go vet ./...` all green.

---

## prog-strength-web

> All web tasks edit `app/(app)/bodyweight/page.tsx` (and W1 edits `lib/api.ts`),
> so they run **sequentially**. After each: `npm run typecheck`, `npm run lint`,
> `npm run build` must pass.

### Task W1 — `lib/api.ts`: goal type + 3 client functions

- [ ] Add `BodyweightGoal` type: `{ weight: number; unit: "lb" | "kg";
  created_at: string | null; updated_at: string | null }`.
- [ ] `getBodyweightGoal(token)` → `GET /me/bodyweight-goal`, `unwrap` with a
  sensible default (`{ weight: 0, unit: "lb", created_at: null, updated_at: null }`).
- [ ] `putBodyweightGoal(token, { weight, unit })` → `PUT /me/bodyweight-goal`,
  JSON body, throws if no row returned (mirror `createBodyweightEntry`).
- [ ] `updateBodyweightEntry(token, id, payload)` where payload is
  `{ weight?; unit?; measured_at? }` → `PUT /bodyweight/{id}`, returns
  `BodyweightEntry`, throws if none returned.

### Task W2 — Goal state + fetch wiring + `GoalAffordance` + `BodyweightGoalModal` + toolbar

- [ ] Page state: `goal`, `showGoalModal`, `editingEntry`, `deletingEntry` (the
  last two are consumed in W5 but declare them here to avoid churn).
- [ ] Fetch the goal in the same load path as entries; add a `refetchGoal` and
  call it after a successful goal PUT. Treat `weight === 0 && created_at === null`
  as "no goal".
- [ ] `GoalAffordance` — transparent `<button>` (no outline), `hover:bg-white/5`,
  green target icon + muted `Goal:` + either `Set goal weight` (italic muted) when
  unset or the value (bold `tabular-nums`, e.g. `175 lb`) when set.
- [ ] Wrap the existing Log toolbar row so it becomes
  `<div className="flex items-center justify-between border-b ... pb-3">` with the
  Log `ToolbarButton` on the left and `GoalAffordance` right-justified.
- [ ] `BodyweightGoalModal` — clone `BodyweightLogModal`'s shell (backdrop, escape,
  scroll lock, busy). One weight input + lb/kg `<select>` persisting via
  `ps_bodyweight_unit` (`UNIT_PREFERENCE_KEY`). Pre-fill value+unit in edit mode;
  empty + persisted unit in create mode. Submit → `putBodyweightGoal`; close on
  success; refetch goal; inline server errors.

### Task W3 — Chart card: title + goal `ReferenceLine` + dashed legend entry + tooltip rounding fix

- [ ] Add `<h3 className="text-base font-semibold tracking-tight">Bodyweight</h3>`
  at the top of `ChartCard`. Thread `goal` + `displayUnit` into `ChartCard`.
- [ ] Import `ReferenceLine` from recharts. When `goal && goal.weight > 0`, render
  it at `y={convertWeight(goal.weight, goal.unit, displayUnit)}`, `stroke="#10b981"`,
  `strokeDasharray="6 4"`, `strokeWidth={1.5}`, with the right-positioned
  `Goal <n> <unit>` label (per SOW snippet).
- [ ] Extend the `Legend` helper with a `dashed?: boolean` that renders the swatch
  as a dashed line; render a third legend entry `<Legend color="#10b981"
  label="Goal" dashed />` only when `goal && goal.weight > 0`.
- [ ] Tooltip rounding fix: investigate why the avg series prints the raw float;
  ensure the formatter applies `formatNumber` to BOTH the scatter (`weight`) and
  the line (`avg`) series so hovering an avg point shows `184.2 lb`, not
  `184.20000000000002 lb`. Keep the `Reading` / `Daily avg` labels. Confirm the
  formatter returns `[formatNumber(v) + " " + displayUnit, label]` for both.

### Task W4 — Accent stat tiles: Average | Goal | Min | Max

- [ ] Replace `StatTile` with `AccentStatTile({ tone, label, value, sublabel?,
  empty? })` per the SOW snippet: 3px top accent strip keyed by tone
  (avg `#3b82f6`, goal `#10b981`, min `#94a3b8`, max `#f59e0b`),
  `bg-[var(--surface-2)]`, optional sublabel, `empty` dims the value with
  `tracking-widest`.
- [ ] Reshuffle the grid to **Average | Goal | Min | Max** (Delta tile removed):
  - Average: `tone="avg"`, sublabel = `"−2.4 lb (−1.3%) over range"` when delta is
    computable, else `"<count> readings"`.
  - Goal: `tone="goal"`, value = formatted goal in `displayUnit`, sublabel =
    `"X.X lb to go"` (signed distance from latest daily-avg to goal); when no goal,
    `empty` + value `"— — —"` + sublabel `"Not set"`.
  - Min: `tone="min"`, sublabel = date. Max: `tone="max"`, sublabel = date.
- [ ] `computeStats` keeps returning `delta` / `deltaPercent` (Average reads them);
  the standalone Delta tile is gone. Add the `toGo` computation
  (`convertWeight(goal.weight, goal.unit, displayUnit) - stats.avg`, phrased
  consistently). Tile grid stays hidden when `entries.length === 0`.

### Task W5 — Row actions: `IconButton` + `TrashIcon` + edit & delete modals

- [ ] Add `IconButton({ tone: "muted" | "danger", ... })` ghost button:
  transparent bg, `hover:bg-white/5`; `muted` paints `text-[var(--muted)]` → hover
  `text-[var(--foreground)]`; `danger` paints `text-[var(--danger)]`. Add
  `TrashIcon` (stroked, 1.8 width, rounded join, matching `PencilIcon`).
- [ ] `BodyweightTable` action cell → two icon buttons (pencil `onEdit(e)`, trash
  `onDelete(e)`). Replace the `onDelete(id)` prop with `onEdit(entry)` /
  `onDelete(entry)` that set `editingEntry` / `deletingEntry`. Remove the old
  red "Delete" text button and the page-level `confirm()` flow.
- [ ] `BodyweightEditModal` (parallels nutrition `LogEntryEditModal`) — shell like
  `BodyweightLogModal`; fields weight (number) / unit (select) / measured_at
  (datetime-local, pre-filled from the entry); submit →
  `updateBodyweightEntry(token, entry.id, payload)`; close on success; refetch
  entries; inline errors.
- [ ] `BodyweightDeleteModal` — thin confirm modal, title `"Delete this reading?"`,
  body noting the trend chart updates, Cancel / Delete (red); confirm →
  `deleteBodyweightEntry(token, entry.id)`; close on success; refetch.
- [ ] Verify pagination (`PAGE_SIZE = 20`, `<Pagination>`) still renders after the
  row-icon + modal changes (do NOT re-implement).

---

## prog-strength-docs

### Task D1 — Mark the SOW shipped

- [ ] On `feat/bodyweight-goal-and-page-polish`: this plan is committed; then edit
  `sows/bodyweight-goal-and-page-polish.md` frontmatter `status: shipped`, body
  `**Status**: Shipped`, `**Last updated**: 2026-06-04`. Commit
  `docs: mark bodyweight-goal-and-page-polish as shipped`.

## Verification checklist (post-merge hand-test — from the SOW Rollout section)

- No goal: affordance `Goal: Set goal weight` (italic muted), Goal tile `— — —` /
  `Not set`, no chart goal line, two legend entries.
- Set 175 lb → affordance `Goal: 175 lb`, tile fills + "X lb to go", dashed green
  line at 175 with label, three legend entries.
- Edit goal 175 → 173: line, affordance, tile all update.
- Pencil on a row → edit modal pre-filled → change weight → list/chart/tiles update.
- Trash on a row → confirm modal → confirm → row gone, chart/tiles update.
- Hover any avg point → `184.2 lb`, not `184.20000000000002 lb`.
- Switch 30/60/90/all → tiles re-derive; goal line + goal tile stay constant.
