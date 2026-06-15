---
status: proposed
repos:
  - prog-strength-api
  - prog-strength-mcp
  - prog-strength-agent
  - prog-strength-web
  - prog-strength-mobile
  - prog-strength-docs
---

# Daily Steps Logging

**Status**: Proposed · **Last updated**: 2026-06-14

## Introduction

**Prog Strength** tracks the deliberate training a user does — the workouts they lift, the runs they record — but it is blind to the other half of physical activity: how much a person simply *moves* on an ordinary day. Daily step count is the single most universal proxy for that movement. Nearly every user already carries a phone or wears a watch that reports a daily step total, and "how many steps did I get today?" is one of the most common things people check about their own activity. Today there is nowhere in the app to capture or look at that number, so a meaningful dimension of a user's activity is invisible.

This SOW adds **daily steps logging**. A user records their step count for a day — today's, or any past day — either through a form in the app or by telling the agent ("log 8,400 steps yesterday"). They can set a **daily steps goal** the way they already set a bodyweight goal: a single target they aim to hit on an average day. A new **Steps** sub-view in the Activities page shows their daily step history as a bar chart with a goal line, a row of summary statistics, and a paginated log of their entries — all scoped to the same timeframe selector that already governs the Overview, Workouts, and Running sub-views, so switching between tabs keeps one consistent window. The Activities **Overview** also gains a few headline steps statistics once a user has any history, so the "how active was I?" digest includes daily movement, not just structured training.

The model is intentionally simple and true to how the data actually arrives. A step count is a **daily total** — that is what a watch or phone reports — so steps are stored as **one canonical total per day**, keyed by date. Logging a day that already has a count replaces it; there is no notion of multiple partial step entries to sum. This keeps the bar chart (one bar per day), the log table (one row per day), and editing (re-log the day) all trivially consistent.

Logging is **manual or agent-driven** in this first version. Automatic synchronization from Apple Health, Google Fit, or Fitbit is explicitly out of scope here — it is a larger integration with its own auth and background-sync concerns, and the manual/agent path is the foundation it would later write into. After this work ships, a user can log steps from the web app, the mobile app, or by asking the agent; set and adjust a daily goal; and see their movement trend at a glance alongside the rest of their activity.

## Proposed Solution

Steps follow the well-worn path that **bodyweight** already paved in this codebase: a small dedicated domain in the Go API with a per-entry table and a separate single-row-per-user goal table, an MCP tool plus an agent prompt update for conversational logging, and a UI surface that reuses existing chart, stat-tile, and goal-setting patterns. The one structural difference from bodyweight is that steps are **date-keyed and upserted** (one total per day) rather than a stream of timestamped readings.

A new `internal/steps` domain owns two tables created in a single migration: `user_steps` (one row per user per date) and `user_steps_goal` (one row per user — a near-exact clone of `user_bodyweight_goal`, minus the unit, since steps are a unitless count). The HTTP surface mirrors bodyweight's split between content routes (`/steps`) and the goal route (`/me/steps-goal`), with the create/update collapsed into a single idempotent **upsert by date** (`PUT /steps/{date}`).

An MCP `steps` module exposes `log_steps` (and goal get/set), modeled on the existing `log_bodyweight` tool, and the agent's prompt is extended so it knows it can log steps for any date and read the user's goal. On the front end, the Activities page gains a fourth sub-view — **Steps** — built as a new view component that takes the page-level timeframe (`days`) exactly as the existing views do, so it stays in lockstep with the shared timeframe pills. The Steps view renders summary stat tiles, a daily bar chart with a goal reference line, and a paginated log table. The Activities Overview view picks up a small set of steps tiles when history exists. The mobile app mirrors the same surfaces under its Activities tab.

## Goals and Non-Goals

### Goals

- Log a **daily step total** for any date (today or retroactive) via a **web/mobile form** and via the **agent**.
- Store steps as **one canonical total per day**, keyed by `(user_id, date)`; re-logging a day **upserts** (replaces) it.
- Set and update a single **daily steps goal** per user, mirroring the bodyweight goal.
- A new **Steps sub-view** in the Activities page showing, top to bottom: summary **stat tiles**, a **daily bar chart** with a **goal line** when a goal is set, and a **paginated log table** of entries.
- The Steps sub-view (chart + table) **defaults to the Activities page-level timeframe** so it stays in sync with the other sub-views.
- The Activities **Overview** gains headline **steps statistics** when the user has any step history.
- Ship to **web and mobile**.

### Non-Goals

- **Automatic device sync** from Apple Health, Google Fit, Fitbit, or any HealthKit/Health Connect source. v1 is manual + agent only.
- **Multiple step entries per day** / intraday or hourly granularity. A day has exactly one total.
- **Distance / calorie / floors** or any non-step movement metric.
- **Steps in the Overview combined chart.** Overview gets stat tiles only; the combined area chart is unchanged (can be revisited later).
- **Goal history / phases.** One current goal per user, like bodyweight.
- **Backfill.** This is a brand-new data type; there is no existing step data to seed.

## Implementation Details

### Data Model

New migration `internal/db/migrations/019_steps.sql`. **Numbering note:** the proposed *User Timeline* SOW also reserves `019`; whichever feature implements second renumbers to the next free index. Two tables, following the conventions of the surrounding migrations (no FK on `user_id`, matching the other per-user-content tables; CHECKs that mirror handler-side validation).

**`user_steps`** — one row per user per day.

- `id TEXT PRIMARY KEY` (from `internal/id`)
- `user_id TEXT NOT NULL`
- `date TEXT NOT NULL` — calendar date `YYYY-MM-DD` (the day the steps were taken, in the user's local frame as supplied by the client/agent)
- `steps INTEGER NOT NULL` — `CHECK (steps >= 0 AND steps <= 200000)` (upper bound guards against fat-fingered input; ~200k is comfortably above any real day)
- `created_at`, `updated_at`
- `UNIQUE (user_id, date)` — the key that makes logging an **upsert**

**`user_steps_goal`** — one row per user (clone of `user_bodyweight_goal`, no unit).

- `user_id TEXT PRIMARY KEY`
- `goal INTEGER NOT NULL` — `CHECK (goal > 0 AND goal <= 200000)`
- `created_at`, `updated_at`

### API — `internal/steps`

A new domain package following the established anatomy (`model.go`, `goal.go`, `errors.go`, `metrics.go`, `repository.go`, `sqlite_repository.go`, `memory_repository.go`, `handler.go`, `handler_goal.go`, and matching `*_test.go`), registered in `internal/server/server.go`:

```go
steps.NewHandler(stepsRepo).Mount(r)
```

Routes mount behind the existing auth middleware; the caller's id comes from `authctx.UserIDFrom(ctx)`, and responses use the `httpresp` helpers — identical to `internal/bodyweight/handler.go`.

- **`GET /steps?since=&until=&limit=&before=`** — list a user's days. Supports the same `since`/`until` range mode the activities/bodyweight lists use (for the chart) and keyset pagination via `limit`/`before` (for the log table). Newest first. Returns `{ steps: [...], next_before }`.
- **`PUT /steps/{date}`** — upsert the total for `{date}` (`YYYY-MM-DD`). Body `{ "steps": int }`. Validates the date is a real calendar date and not absurdly in the future, and `0 <= steps <= 200000`. Returns the stored entry (`200`). This single idempotent route covers both "log today" and "correct a past day."
- **`DELETE /steps/{date}`** — remove a day's entry. `204`.
- **`GET /me/steps-goal`** — the current goal, or an empty/unset response when none. Mirrors `getMyBodyweightGoal`.
- **`PUT /me/steps-goal`** — upsert the goal. Body `{ "goal": int }`, `0 < goal <= 200000`. Mirrors `putMyBodyweightGoal`.

DTOs, JSON tags, and validation-to-`400` mapping follow the bodyweight handlers exactly.

### MCP — `steps` module

A new `src/prog_strength_mcp/steps.py`, registered in `server.py`, modeled on `bodyweight.py`'s `log_bodyweight`:

- **`log_steps(date, steps)`** — upsert the day's total via `PUT /steps/{date}`. `date` accepts an explicit `YYYY-MM-DD` or a relative phrase the agent resolves ("today", "yesterday") before calling. This is the single tool that satisfies both daily and retroactive logging.
- **`get_steps_goal()` / `set_steps_goal(goal)`** — read/upsert the goal via `/me/steps-goal`.
- **`get_steps(since, until)`** *(optional, for coaching context)* — read recent days so the agent can answer "how have my steps been this week?"

These call through the existing `api_client.py` with the same auth-forwarding the other tools use; the MCP remains a thin forwarder over the Go API.

### Agent — prompt update

Extend the agent's system/tooling prompt so it knows it can:

- Log a step count for **today or any past date** (`log_steps`), confirming the date it resolved when the user is relative/ambiguous.
- Read and set the user's **daily steps goal**.
- Optionally reference recent step history when giving activity feedback.

This is a prompt/registration change only; no new routing or intent classification is required. Keep the addition minimal and consistent with how `log_bodyweight` is described.

### Web — Steps sub-view

> **Critical implementer note:** the web repo is the **"This is NOT the Next.js you know"** fork. Before writing code, consult `node_modules/next/dist/docs/` and the validated patterns already in the repo. `app/(app)/activities/page.tsx` and `components/activities/` are the direct structural references for this work; the bodyweight page (`app/(app)/bodyweight/page.tsx`, `components/bodyweight/`) is the reference for the goal-setting affordance and the goal-in-chart-domain handling.

**Activities shell (`app/(app)/activities/page.tsx`):**

- Extend the `View` union to `"overview" | "workouts" | "running" | "steps"` and accept `view === "steps"` from the `?view=` param.
- Add a **Steps** `ToolbarButton` (with a new footprints/steps line icon) to the view-switcher group.
- Render `<StepsView days={days} />`, passing the page-level timeframe so the Steps chart and log table share the same window as every other sub-view.

**`components/activities/steps-view.tsx` (new):** top to bottom —

1. **Stat tiles** — Avg daily steps, Total steps, Best day, and Goal (showing the target and how average daily steps compare to it; this is the "achieve daily on average" framing). Tiles are computed over the selected timeframe.
2. **Daily bar chart** — a recharts `BarChart`, one bar per day across the timeframe, with a horizontal **`ReferenceLine` at the goal** when a goal is set. Replicate the `bodyweight-goal-in-yaxis-domain` fix so the goal value is always included in the y-axis domain and the line never clips off-screen.
3. **Paginated log table** — one row per day (date, steps, and edit/delete affordances), timeframe-filtered, paginating via the API's `before` cursor.

A goal-setting control (action sheet / inline editor) mirrors the bodyweight page so a user sets or changes their daily goal from the Steps view.

**Overview (`components/activities/activities-overview-view.tsx`):** add a small set of **steps tiles** (e.g. Avg daily steps and goal attainment) that render only when the user has step history, so the digest reflects daily movement. The combined chart is unchanged.

**API client (`lib/api.ts` / client layer):** add `listSteps`, `upsertStepsForDate`, `deleteStepsForDate`, `getStepsGoal`, `putStepsGoal`, following the existing client function shape.

No new sidebar entry — Steps lives under Activities.

### Mobile — Steps sub-view

Mirror the web surfaces under the mobile Activities tab (`app/(tabs)/activities`) and `components/activities/`:

- Add **Steps** to the Activities sub-view switcher.
- A `steps-view` + `steps-chart` (one bar per day, goal reference line) and a paginated log list, scoped to the shared timeframe, with a goal-setting sheet — reusing the bodyweight mobile components (`components/nutrition/bodyweight-view.tsx`, `bodyweight-chart.tsx`) as the structural model and the shared mobile API client in `lib/`.
- Overview gains the same steps tiles when history exists.

Mobile follows the established per-phase parity workflow (research → plan → subagent); if the dispatcher prefers, the mobile build may be split into its own implementation phase after API + web land (see Rollout).

### Tests

- **API repository** (`sqlite` + `memory`): upsert-by-date semantics (`UNIQUE (user_id, date)` replaces rather than duplicates), range and keyset reads, goal upsert, and validation bounds.
- **API handler:** `PUT /steps/{date}` upsert + validation `400`s (bad date, out-of-range steps), `DELETE`, list pagination, and goal get/put — mirroring the bodyweight handler tests. Include an authz check that a second user cannot read or modify another user's steps.
- **MCP:** `log_steps` date resolution and forwarding; goal get/set.
- **Web:** vitest coverage for `StepsView` (timeframe sync, goal line presence/absence, log pagination) and the Overview steps tiles' conditional render.
- **Mobile:** component/render tests per repo conventions.

### Rollout

Entirely additive — no changes to existing endpoints, data, or surfaces. Recommended order:

1. **API** — migration `019` (renumber if Timeline lands first), `internal/steps` domain, endpoints. Ship.
2. **MCP + Agent** — `steps` tool module and prompt update against the live API.
3. **Web** — the Steps sub-view + Overview tiles.
4. **Mobile** — the Steps sub-view, in this dispatch or as a follow-up parity phase.

No feature flag required; the Steps view is empty-safe (renders an empty/CTA state before any entry) and all endpoints are new.

## Open Questions

- **Stat tile set** — the four proposed tiles (Avg daily steps, Total, Best day, Goal/attainment) are a starting point; the exact set and the precise goal-attainment phrasing ("avg vs goal" vs "% of days hit") can be tuned during implementation.
- **Date frame** — `date` is stored as the client-supplied calendar day. If steps later pair with timezone-sensitive surfaces, confirm this matches the app's existing local-day handling (as used by the nutrition log). Assumed consistent for v1.
- **Future-date logging** — assume logging a date slightly in the future (e.g. crossing midnight in another timezone) is tolerated within a small window; reject dates far in the future.
- **Agent read access** — `get_steps` for coaching context is marked optional; include it if the agent prompt benefits, omit if it adds noise.
- **Migration index collision** with the Timeline SOW's `019` — resolved by renumbering whichever ships second.
