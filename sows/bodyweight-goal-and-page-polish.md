---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Bodyweight Goal + Web Page Polish

**Status**: Shipped · **Last updated**: 2026-06-04

## Introduction

The bodyweight page on the web app gives the user a clean view of their scale readings — a daily-average trend line over a 30/60/90/all time range, a four-tile statistics summary, and a paginated entries table — but it stops short of answering the question the user actually carries onto the scale. "Am I getting closer?" needs a reference point, and the page doesn't have one. The trend line goes up or down on the chart but there is nothing telling the user where they're trying to go, and the statistics tiles report descriptive numbers (Average, Min, Max, Delta) without any goal-relative interpretation. A user cutting toward 175 lb has to do the comparison in their head every time they open the page.

While that gap is being closed, several adjacent polish items have accumulated on the same surface. The four statistics tiles use the page background color rather than a treatment that lifts them off the chart card, so the gray surrounding panel reads as if it's been punctured in those four spots. The entries table's only action is a red "Delete" text button — useful but visually heavy, and there is no edit affordance at all, so a typo'd reading has to be deleted and re-logged. The chart card has no title, so the page header is the only thing telling a fresh viewer what the section is. And the recharts tooltip is rendering the daily-average value with the full float precision instead of the `formatNumber` rounding the rest of the page applies to weights, which produces strings like `184.20000000000002 lb` on hover and looks unfinished.

This SOW bundles a single coordinated round of work covering the goal weight (data model + read/write API + chart + tile + affordance), the table polish (icon-only actions + edit modal), and the smaller chart-card improvements (title, tooltip rounding). One dispatch produces one API PR and one web PR; nothing else moves.

## Proposed Solution

A new tiny domain on the API — `bodyweight`-adjacent, modeled on the macro-goals row from `daily-macro-goals.md` — owns one row per user with two fields: `weight` (real) and `unit` (`lb` or `kg`). Two endpoints, both shaped against `/me/...`:

- `GET /me/bodyweight-goal` — returns the user's goal, or a 200 with `{weight: 0, unit: "lb", created_at: null, updated_at: null}` when the user has never set one, so the web client never has to special-case 404.
- `PUT /me/bodyweight-goal` — replaces the goal atomically. Set-replacement semantics (PUT, not PATCH) because the two fields are conceptually one goal.

The unit is stored on the row, not assumed at read time, mirroring the anti-drift property the bodyweight entries already have. A user who saves a goal of 175 lb sees 175 lb forever, regardless of which unit toggle they used to log readings the following month.

The bodyweight CRUD also gains a `PUT /bodyweight/{id}` so the new pencil-icon edit affordance has somewhere to call. The body is a partial update of `{weight, unit, measured_at}` — same shape pattern the nutrition log-entry edit uses — and the repository soft-delete-aware. This is a new write surface; the bodyweight handler currently only ships `POST`, `GET /`, and `DELETE /{id}`.

On the web client, the bodyweight page absorbs the rest of the work in one PR:

The toolbar separator above the entries table gains a right-justified goal affordance — a small target icon, the label `Goal:`, and the value. The affordance is not a button; it is highlightable, clickable text with a subtle hover background. Clicking it opens a goal modal — same modal shell the bodyweight log uses — with one weight input and a lb/kg toggle that persists across visits via the same `ps_bodyweight_unit` key. When no goal exists, the affordance reads `Goal: Set goal weight` (italic, muted) and clicking it opens the modal in create mode; when a goal exists, the affordance reads `Goal: 175 lb` (value bold) and clicking opens the modal in edit mode.

The chart card gains a bold `Bodyweight` title above the recharts area. When the goal exists, the chart renders a horizontal dashed green line across the x-axis at the goal weight (converted to the page's display unit), and the legend gains a third entry — line swatch + `Goal` — so the user can read the chart without context. When the goal does not exist, neither the line nor the legend entry is rendered.

The statistics tiles are restyled and reshuffled. Each tile gains a 3px colored accent strip on top, keyed to the tile's identity: Average is blue (matching the trend line), Goal is green (matching the goal line), Min is slate, Max is amber. The tile background is a lighter shade than the chart card so it visually lifts off the surface rather than blending into the page background. The four tiles are: Average | Goal | Min | Max. The standalone Delta tile is removed; the delta and delta-percent are folded into the Average tile as a third sub-line under the value and label ("−2.4 lb (−1.3%) over range"). The new Goal tile shows the goal value with a "X lb to go" sub-line when set, or "— — —" + "Not set" when not set.

The entries-table action column swaps the red "Delete" text button for two always-tinted ghost icons: a muted-gray pencil (opens the edit modal) and an always-red trash icon (opens a confirm-delete modal). Both icons sit on a transparent background with a faint hover-fill, matching the page's existing ghost-icon vocabulary. The edit modal is a new `BodyweightEditModal` that parallels the nutrition `LogEntryEditModal` — it lets the user change weight, unit, or measured_at, calls `PUT /bodyweight/{id}`, and re-renders the entries list and stat tiles on save. The delete confirmation modal replaces the inline `confirm()` dialog the page currently uses.

The recharts tooltip already passes a `formatNumber`-based formatter, but the daily-average series is slipping through and printing the raw IEEE-754 string. The fix is to track down why the formatter's `name === "weight"` branch covers the scatter but the `avg` series is hitting an unformatted code path, and apply `formatNumber` to both series consistently. The investigation likely terminates in either the series `name` prop or the recharts version's payload shape; the fix is a one-line conditional change in the chart card's `Tooltip` formatter.

Pagination is **not** new — `PAGE_SIZE = 20` already lives on the page at `app/(app)/bodyweight/page.tsx:41` and the existing `<Pagination>` component already handles long lists. The SOW notes this so reviewers know it's not being re-shipped; the only pagination work is to verify it still renders correctly after the row-icon and edit-modal changes land.

No schema migration on `bodyweight_entries`. No agent changes. No mobile changes. The goal is web-first read signal; if the agent eventually needs `get_bodyweight_goal` to comment on cuts in chat, that's a separate SOW that adds the MCP tool layer.

## Goals and Non-Goals

### Goals

- Persist one row per user in a new `user_bodyweight_goal` table with `weight REAL` and `unit TEXT` columns. UPSERT-on-PUT so first save creates the row and subsequent saves replace it.
- Expose the goal via `GET /me/bodyweight-goal` and `PUT /me/bodyweight-goal` on the Go API. JWT-gated under the existing auth middleware. Validate `weight > 0`, `weight ≤ 2000` (catches typo zeros while admitting any plausible value in either unit), and `unit ∈ {lb, kg}`. Empty-state read returns 200 with `{weight: 0, unit: "lb", created_at: null, updated_at: null}` so the web client never has to dance around 404.
- Add `PUT /bodyweight/{id}` to the existing bodyweight handler. Partial body `{weight?, unit?, measured_at?}`; the repository update path is soft-delete-aware and returns `ErrNotFound` on a deleted or different-user row. The endpoint exists to power the new pencil-icon edit affordance.
- Render a right-justified goal affordance on the same separator as the Log button on the web bodyweight page. The affordance is highlightable, clickable text (target icon + `Goal:` label + value or `Set goal weight` italic muted) with no button outline. Clicking opens a goal modal in create or edit mode depending on whether a goal exists.
- Render a horizontal dashed green goal line across the chart when the goal exists, converted to the page's display unit, with a third legend entry. The legend entry is added only when the goal exists.
- Render a bold `Bodyweight` title above the chart inside the chart card.
- Restyle the four statistics tiles with a 3px colored accent strip on top (Average=blue, Goal=green, Min=slate, Max=amber), a lighter tile background than the chart card, and reshuffle the layout to Average | Goal | Min | Max. The standalone Delta tile is removed; the delta value and percent fold into the Average tile as a sub-line ("−2.4 lb (−1.3%) over range"). The new Goal tile shows the value + "X lb to go" sub-line, or "— — —" + "Not set" when no goal exists.
- Replace the entries-table red "Delete" text button with always-tinted ghost icons in the actions column: a muted-gray pencil (opens edit modal) and an always-red trash icon (opens delete-confirm modal). Both icons use the page's existing ghost-button hover treatment.
- Add a `BodyweightEditModal` component paralleling the nutrition `LogEntryEditModal`. Lets the user change weight, unit, and measured_at; calls `PUT /bodyweight/{id}`; refetches both the entries list and the stat tiles on save.
- Replace the inline `confirm()` delete dialog with a proper confirmation modal that matches the nutrition delete-modal pattern.
- Fix the daily-avg recharts tooltip so the value is rounded through `formatNumber` (one decimal place) instead of printing the raw float. Investigate why the existing formatter's `name === "weight"` branch covers the scatter but the avg series prints unformatted; apply `formatNumber` consistently to both.
- Verify the existing client-side pagination (`PAGE_SIZE = 20`) still renders correctly after the row-icon and edit-modal changes land. Do not re-implement.
- Go API tests covering: `GET /me/bodyweight-goal` empty-state shape, PUT happy path, PUT validation (zero, negative, >2000, invalid unit), the new `PUT /bodyweight/{id}` happy path + soft-deleted row + cross-user-id returns `ErrNotFound`.

### Non-Goals

- **MCP tools / agent surface for the goal.** The macro-goals SOW added `get_macro_goals` and `set_macro_goals` because the user wanted the agent to read and update targets in chat. The bodyweight goal here is web read signal only — there's no current scenario where Claude needs to know it. When that changes (e.g. "Claude, how am I trending against my goal?" becomes a desired chat answer), add the two MCP tools as a follow-up SOW. Same pattern; ~50 lines.
- **Mobile.** The mobile bodyweight surface isn't in scope. Once the API endpoints ship, mobile can mirror the same UX in its own dispatch. The contract is stable from day one so mobile catches up without re-negotiating.
- **Goal trajectory / forecasting.** The chart shows the goal line and the trend line; we let the user's eyes do the comparison. A projected "you'll hit your goal on Nov 14" annotation would be useful but requires a forecasting model (handles flat trend, wrong-direction trend, very little data) and crosses out of UI polish into product math. File when there's user feedback asking for it.
- **Goal tolerance band.** No `±2 lb shaded zone around the goal`. Useful for maintenance phases but adds a tolerance concept the API doesn't currently model; users on cuts and bulks usually want a single line, not a zone. Revisit if the maintenance use case grows.
- **Time-bounded goal phases.** A user can't have a "cut goal" that auto-swaps to a "maintenance goal" on a future date. One current goal at a time. Same reasoning as the macro-goals SOW: a goal-history table and active-phase selector is a real feature, not v1.
- **Server-side pagination on `GET /bodyweight`.** The endpoint still returns the full per-user history. As you noted, the data is small enough that this is acceptable; client-side pagination handles the table fine. Revisit when a user has >1000 entries and the round-trip starts to bite.
- **Schema migration on `bodyweight_entries`.** Storage is fine; only the handler gains a PUT and the new goal table is added.
- **Re-implementing client-side pagination.** Already exists at `PAGE_SIZE = 20`. The SOW notes this so it's not accidentally re-shipped.
- **Public sharing of the goal.** Strictly per-user, like macro goals.
- **Pre-shipping a default goal.** New users see the empty-state affordance until they explicitly save. We're not picking 180 lb for everyone.

## Implementation Details

### Data Model

One new migration: `010_user_bodyweight_goal.sql`.

| Column | Type | Description |
| --- | --- | --- |
| `user_id` | text | Owning user. Primary key. One row per user. |
| `weight` | real | Goal weight. `CHECK (weight > 0 AND weight <= 2000)`. |
| `unit` | text | `lb` or `kg`. `CHECK (unit IN ('lb', 'kg'))`. |
| `created_at` | timestamp | First save. Set on insert; immutable thereafter. |
| `updated_at` | timestamp | Every save. |

The unit is stored per row, not assumed at read time. Matches the bodyweight-entry anti-drift property: a saved 175 lb goal renders as 175 lb in the future regardless of what unit the page chooses to display in.

### Go API: handler + repository

The goal lives in its own package — `internal/bodyweight/goal` would over-split; `internal/bodyweight` already exists and is the natural home, so add the goal as a sibling of the existing `Entry` types within that package. New files:

- `internal/bodyweight/goal.go` — `Goal` struct, validation, error sentinels.
- `internal/bodyweight/goal_repository.go` (or extend `repository.go` directly) — `GetBodyweightGoal(ctx, userID) (Goal, error)` and `UpsertBodyweightGoal(ctx, g Goal, now time.Time) (Goal, error)` interface methods.
- `internal/bodyweight/handler_goal.go` — `getMyBodyweightGoal` and `putMyBodyweightGoal` handlers. Mount inside the existing `bodyweight.Handler.Mount` exactly the way `nutrition.Handler.Mount` already mounts `/me/macro-goals` at `internal/nutrition/handler.go:57` — each domain adds its own `r.Route("/me/<thing>", ...)` to the top-level router from inside its own `Mount`; no shared `/me` subrouter needs to be threaded through `internal/server/`.

Empty-state read shape (mirrors `macroGoalsDTO`):

```json
{
  "weight": 0,
  "unit": "lb",
  "created_at": null,
  "updated_at": null
}
```

PUT request body:

```json
{ "weight": 175, "unit": "lb" }
```

Validation, fail-at-first-error:

1. `weight > 0` → 400 `"weight must be positive"`
2. `weight <= 2000` → 400 `"weight must be <= 2000"`
3. `unit in {"lb","kg"}` → 400 `"unit must be 'lb' or 'kg'"`

PUT response is the saved goal with non-nil `created_at` + `updated_at`.

### Go API: `PUT /bodyweight/{id}`

Add a new method on `bodyweight.Repository`:

```go
UpdateEntry(ctx context.Context, e *Entry) error
```

Implementation pattern matches the nutrition `UpdateNutritionLogEntry` in `sqlite_repository.go:282`: re-derive validity, UPDATE where `id = ? AND user_id = ? AND deleted_at IS NULL`, return `ErrNotFound` when 0 rows affected. Memory repo mirrors the same shape.

Handler — `internal/bodyweight/handler.go`:

```go
r.Put("/{id}", h.update)
```

Body shape:

```json
{ "weight": 184.6, "unit": "lb", "measured_at": "2026-06-04T07:12:00Z" }
```

All three fields optional; omitted fields preserve the existing value (look up the row first, the same way nutrition's update does). Validate `weight > 0` if supplied, `unit ∈ {lb, kg}` if supplied. 404 on not-found / soft-deleted / wrong-user, 400 on validation, 200 with the updated row on success.

### Web client: page layout

`app/(app)/bodyweight/page.tsx` is the single file that grows; new components are extracted as siblings.

State additions:

```ts
const [goal, setGoal] = useState<BodyweightGoal | null>(null);
const [showGoalModal, setShowGoalModal] = useState(false);
const [editingEntry, setEditingEntry] = useState<BodyweightEntry | null>(null);
const [deletingEntry, setDeletingEntry] = useState<BodyweightEntry | null>(null);
```

The `goal` shape:

```ts
type BodyweightGoal = {
  weight: number;
  unit: "lb" | "kg";
  created_at: string | null;
  updated_at: string | null;
};
```

`weight === 0 && created_at === null` is the empty state; the page treats this as "no goal set" and renders the affordance / tile accordingly.

New API client functions in `lib/api.ts`:

```ts
export async function getBodyweightGoal(token: string): Promise<BodyweightGoal>;
export async function putBodyweightGoal(token: string, goal: { weight: number; unit: "lb" | "kg" }): Promise<BodyweightGoal>;
export async function updateBodyweightEntry(token: string, id: string, payload: { weight?: number; unit?: "lb" | "kg"; measured_at?: string }): Promise<BodyweightEntry>;
```

Mount the goal fetch in the same `useEffect` that fetches entries on page load; refetch the goal after a successful PUT. After a successful entry edit or delete, refetch entries (the stat tiles and chart are derived from the entries list, so a single refetch covers all dependent renders).

### Web client: toolbar + goal modal

Toolbar separator row inside `<main>`:

```tsx
<div className="flex items-center justify-between border-b border-[var(--border)] pb-3">
  <ToolbarButton onClick={() => setShowLog(true)} icon={<PencilIcon />} label="Log" />
  <GoalAffordance goal={goal} onClick={() => setShowGoalModal(true)} />
</div>
```

`GoalAffordance` is a plain `<button>` with `background: transparent; border: 0; padding: 4px 6px; border-radius: 4px;` and a `hover:bg-white/5` rule so it lights up faintly on hover but doesn't read as a buttoned control. Layout: target icon (green `text-emerald-500`), `Goal:` (muted), then either `Set goal weight` (italic muted) when `goal === null || goal.weight === 0`, or the value (bold `tabular-nums`) when set.

`BodyweightGoalModal` is a new component matching `BodyweightLogModal` (`bodyweight/page.tsx:628`) almost line-for-line:

- Same modal shell: backdrop, centered card, escape-to-close, body scroll lock, busy state.
- Single `<form>` with a weight input and a lb/kg `<select>` toggle. Pre-fills with the current goal value/unit when editing, with empty + the localStorage-persisted unit when creating.
- Submit calls `putBodyweightGoal`, closes on success, surfaces server errors inline like the log modal does.

### Web client: chart card

`ChartCard` (`bodyweight/page.tsx:256`) gains:

1. A `<h3 className="text-base font-semibold tracking-tight">Bodyweight</h3>` at the top of the card. The page header's `<h1>` says "Bodyweight" too, but the chart-card title acts as a section label so the chart reads as a self-contained unit (the page might one day gain other sections — projection, weekly digest — and the title gives the chart its own identity).
2. A `<ReferenceLine>` from recharts at `y = convertWeight(goal.weight, goal.unit, displayUnit)` when `goal !== null && goal.weight > 0`:

   ```tsx
   {goal && goal.weight > 0 && (
     <ReferenceLine
       y={convertWeight(goal.weight, goal.unit, displayUnit)}
       stroke="#10b981"
       strokeDasharray="6 4"
       strokeWidth={1.5}
       label={{
         value: `Goal ${formatNumber(convertWeight(goal.weight, goal.unit, displayUnit))} ${displayUnit}`,
         position: "right",
         fill: "#10b981",
         fontSize: 10,
       }}
     />
   )}
   ```

3. A third `Legend` entry rendered when the goal is set:

   ```tsx
   {goal && goal.weight > 0 && <Legend color="#10b981" label="Goal" dashed />}
   ```

   The `<Legend>` helper component gains a `dashed` boolean that renders the swatch as a dashed line (matches the goal-line stroke style).

### Web client: tooltip rounding fix

`Tooltip` formatter currently (`bodyweight/page.tsx:343`):

```ts
formatter={(value, name) => {
  const v = typeof value === "number" ? value : Number(value);
  return [
    `${formatNumber(v)} ${displayUnit}`,
    name === "weight" ? "Reading" : "Daily avg",
  ];
}}
```

This applies `formatNumber` to both series in principle. The bug is somewhere between the recharts payload shape and the formatter args — likely `name` is arriving as the dataKey (`"avg"`) for the line series and matching the `else` branch correctly, but the value being shown isn't the formatter's return value because of a stale `<Line>` config. Investigation steps for the implementer:

1. Add a `console.log(value, name, typeof value)` inside the formatter, hover the line, read the args.
2. Likely fix: ensure the `<Line>` does not include `t` (the x-axis timestamp) in the tooltip payload — set `<Line dataKey="avg" name="Daily avg">` (move display label off the series `name` prop, off the dataKey) and adjust the formatter to use the cleaner branch.
3. If recharts is rendering an extra payload row from a non-numeric field, hide it with `<Tooltip itemSorter={...}>` or by trimming the payload shape upstream.

Acceptance: hovering any avg data point shows `184.2 lb` (one decimal), not `184.20000000000002 lb`.

### Web client: stat tiles

Replace the existing `StatTile` with an `AccentStatTile` that takes a `tone` prop:

```tsx
type Tone = "avg" | "goal" | "min" | "max";

function AccentStatTile({
  tone,
  label,
  value,
  sublabel,
  empty,
}: {
  tone: Tone;
  label: string;
  value: string;
  sublabel?: string;
  empty?: boolean;
}) {
  const stripColor: Record<Tone, string> = {
    avg: "#3b82f6",
    goal: "#10b981",
    min: "#94a3b8",
    max: "#f59e0b",
  };
  return (
    <div className="relative overflow-hidden rounded-md border border-[var(--border)] bg-[var(--surface-2)] px-3 py-3 pt-3.5">
      <span aria-hidden style={{ background: stripColor[tone] }} className="absolute inset-x-0 top-0 h-[3px]" />
      <p className={`text-xl font-semibold tracking-tight tabular-nums ${empty ? "text-[var(--muted)] tracking-widest" : ""}`}>
        {value}
      </p>
      <p className="mt-0.5 text-[10px] font-semibold uppercase tracking-wider text-[var(--muted)]">{label}</p>
      {sublabel && <p className="mt-0.5 text-[10px] text-[var(--muted)]">{sublabel}</p>}
    </div>
  );
}
```

`--surface-2` is a new CSS variable: one shade lighter than `--surface` so the tile lifts off the chart card. If the theme system doesn't easily support a new variable, use an inline `style={{ background: "#25282c" }}` (Tailwind via arbitrary value) — the design system question is solvable in either direction; the implementer picks.

Tile assignments:

- **Average** — `tone="avg"`, value = formatted avg, label = `"Average"`, sublabel = `"−2.4 lb (−1.3%) over range"` when delta is computable, else `"<count> readings"`.
- **Goal** — `tone="goal"`, value = formatted goal weight, label = `"Goal"`, sublabel = `"X.X lb to go"` (latest daily-avg minus goal, signed by direction), or `empty={true}` + `value="— — —"` + sublabel `"Not set"` when no goal.
- **Min** — `tone="min"`, value = formatted min, label = `"Min"`, sublabel = the date.
- **Max** — `tone="max"`, value = formatted max, label = `"Max"`, sublabel = the date.

The standalone Delta tile is removed; `computeStats` keeps computing `delta` and `deltaPercent` because Average's sublabel reads them.

### Web client: row actions + edit/delete modals

`BodyweightTable` action cell (`bodyweight/page.tsx:535`) becomes:

```tsx
<td className="px-4 py-2 text-right">
  <div className="inline-flex items-center gap-1">
    <IconButton aria-label="Edit" onClick={() => onEdit(e)} tone="muted">
      <PencilIcon />
    </IconButton>
    <IconButton aria-label="Delete" onClick={() => onDelete(e)} tone="danger">
      <TrashIcon />
    </IconButton>
  </div>
</td>
```

`IconButton` is a new ghost-icon button: transparent background, `tone="muted"` paints the icon `text-[var(--muted)]` at rest and lifts to `text-[var(--foreground)]` on hover; `tone="danger"` paints the icon `text-[var(--danger)]` at rest and uses the existing red on hover. Both apply the `hover:bg-white/5` faint background hint.

`TrashIcon` is a new SVG matching the page's stroked iconography (1.8 stroke, rounded join, 16×16 viewBox).

The handlers stop deleting inline; instead they set state:

```ts
onEdit: (entry) => setEditingEntry(entry);
onDelete: (entry) => setDeletingEntry(entry);
```

`BodyweightEditModal` parallels `LogEntryEditModal` from nutrition:

- Modal shell same as `BodyweightLogModal`.
- Form fields: weight (number input), unit (select), measured_at (datetime-local, pre-filled from the entry).
- Submit calls `updateBodyweightEntry(token, entry.id, payload)`, closes on success, refetches the entries list.
- Surfaces server errors inline.

`BodyweightDeleteModal` is a thin confirmation modal: title `"Delete this reading?"`, body explaining the trend chart will update, buttons Cancel / Delete (red). On confirm calls `deleteBodyweightEntry(token, entry.id)`, closes on success, refetches. Replaces the existing `confirm()` call.

### Stats computation update

`computeStats` (`bodyweight/page.tsx:830`) keeps its current shape; only the consumer changes. The `Delta` stat is removed from the tile grid but `delta` / `deltaPercent` continue to be returned so the Average tile's sublabel can read them.

The Goal tile sublabel needs a "X lb to go" computation:

```ts
const toGo = goal && stats.avg !== null
  ? convertWeight(goal.weight, goal.unit, displayUnit) - stats.avg
  : null;
// Format: if toGo > 0 the user needs to lose (goal is below avg → cut); show as "X.X lb to lose"?
// Simpler: show absolute distance, signed.
```

The implementer chooses the phrasing — `"4.2 lb to go"` is the easiest read; `"4.2 lb above goal"` / `"3.1 lb below goal"` is more precise. Either is fine; pick one and stay consistent.

### Empty state

When `entries.length === 0`, the existing chart card already renders `"No readings in this range."` — keep that path. The stat tile grid is hidden in this state (existing behavior).

When `entries.length > 0 && !goal`, the chart renders without the goal line and without the third legend entry, and the Goal tile shows the empty state. No CTA in the tile itself; the affordance in the toolbar is the only call-to-action so it stays the single discoverable surface.

### Tests

**Go API** (`prog-strength-api`):

- `internal/bodyweight/goal_test.go` — `Goal.Validate()` rejects weight ≤ 0, weight > 2000, and unit ∉ {lb, kg}; accepts the happy cases.
- `internal/bodyweight/handler_goal_test.go` — `GET /me/bodyweight-goal` for a user with no goal returns the empty-state shape (`weight: 0`, nil timestamps); `PUT` happy path returns the saved row; PUT validation cases each return 400 with the expected message; PUT after PUT returns the second value (idempotent set-replacement).
- Extend `internal/bodyweight/bodyweight_test.go` (or add `handler_test.go`) — `PUT /bodyweight/{id}` happy path returns the updated row; PUT on a soft-deleted row returns 404; PUT against a different user's row returns 404; PUT with weight ≤ 0 returns 400; partial body preserves omitted fields.
- SQLite repo tests for `UpsertBodyweightGoal` (insert then update) and `UpdateEntry` (preserves CreatedAt; updates only the supplied fields).

**Web** (`prog-strength-web`):

No test runner today. Verification is `tsc --noEmit`, `eslint`, `next build`, and the hand-test checklist below.

### Rollout

1. Merge `prog-strength-api` first (goal endpoints + `PUT /bodyweight/{id}` + migration). Until the matching web PR merges, the page still works exactly as before; the new endpoints are dark.
2. Deploy the API.
3. Merge `prog-strength-web` after the API is live. The page picks up the goal affordance, chart line, restyled tiles, icon-only row actions, and edit modal in one PR.
4. Hand-test in the deployed environment with a real account:
   - Open the bodyweight page. With no goal set, the affordance reads `Goal: Set goal weight` (italic muted), the Goal tile reads `— — —` / `Not set`, the chart shows no goal line, the legend has two entries.
   - Click the affordance. Modal opens in create mode. Enter 175 / lb. Save. Modal closes, the affordance flips to `Goal: 175 lb`, the Goal tile fills with the value + "X lb to go", the chart renders the dashed green line at 175 with `Goal 175 lb` label, the legend has three entries.
   - Click the affordance again. Modal opens in edit mode with 175 / lb pre-filled. Change to 173. Save. The chart line, affordance, and tile all update to 173.
   - Click the pencil icon on an entry. Edit modal opens with the entry's values pre-filled. Change the weight. Save. The entries list, chart, and stat tiles all reflect the change.
   - Click the trash icon on an entry. Confirmation modal opens. Confirm. The entry disappears, the chart and tiles update.
   - Hover any daily-avg point on the chart. Tooltip shows `184.2 lb` (one decimal), not `184.20000000000002 lb`.
   - Switch the time-range tabs (30/60/90/all). All four tiles re-derive correctly. The goal line and goal tile stay constant.

Independent revertability: the API PR is safe to revert until the web PR merges (the new endpoints just go dark). The web PR depends on the API; reverting the web PR after both are deployed restores the old page surface but the new endpoints remain live and unused. Schema migration `010_user_bodyweight_goal.sql` is additive and safe to leave in place after a revert.
