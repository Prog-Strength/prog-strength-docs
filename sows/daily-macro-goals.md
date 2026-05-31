# Daily Macro Goals

**Status**: Shipped · **Last updated**: 2026-05-31

## Introduction

The daily-nutrition-log work that shipped earlier this month gave **Prog Strength** the ability to know what the user ate — pantry items, recipes, daily macro aggregates. What it can't do yet is tell the user (or the agent) what they were *aiming* for. Every macros chart today is an absolute number floating in space: "you ate 142g of protein" is informative, "you ate 142g of protein against your 180g target" is actionable.

The lifter who's actively cutting wants to know whether today is under-budget on calories before they decide on a snack. The lifter who's bulking wants to see whether they're hitting protein every day. The hybrid athlete needs different carb targets on running days than on rest days (out of scope for v1 — see Non-Goals). All of them need a place to *set* a daily target and a glance-able answer for "am I close?"

This work adds a persistent per-user macro goal — protein, carbs, fat, and calories per day — that the user sets themselves on the nutrition page, that the agent can read and update via new MCP tools, and that drives a new visual on both clients: four ring-shaped progress indicators showing today's intake as a fraction of each goal. After this ships, "how am I doing today?" gets a single-screen answer on both web and mobile, and the agent can answer the same question in conversation without making up numbers.

## Proposed Solution

A new tiny domain on the API — `nutrition`-adjacent but distinct enough to be its own table — owns one row per user with four integer fields: `protein_g`, `carbs_g`, `fat_g`, `calories`. Two endpoints, both shaped against `/me/...` like the headline-exercises surface from `custom-headline-lifts.md`:

- `GET /me/macro-goals` — returns the user's goals (or 200 with all-zero / sentinel fields when the user has never set them, so the client never has to special-case 404).
- `PUT /me/macro-goals` — replaces the user's goals atomically with the four numbers in the body. Set-replacement semantics (not PATCH per-field) because the four fields are conceptually one goal, not four independent ones.

The MCP server gets two new tools that wrap those endpoints: `get_macro_goals` and `set_macro_goals`. The agent can read the goals to comment on progress and can also set them when the user asks ("set my protein target to 180g") — same JWT pass-through pattern the existing tools use. The agent doesn't get its own computed "compare today to goals" tool; it composes `get_macro_goals` with the existing `get_daily_macros` and writes the answer in prose. Less surface area, more flexibility.

On the clients, the Nutrition page's Today view gains a row of four ring charts above the existing macro totals. Each ring shows today's intake for one macro (protein, carbs, fat, calories) as an arc filling 0–100% of the corresponding goal. Tapping a ring opens a Set Goals modal (one form for all four numbers). The four-rings layout was deliberately chosen over a single combined chart so the user can read each macro independently — protein adequacy is its own question and shouldn't be averaged with calories.

Calorie consistency is intentionally NOT enforced. The user can set `protein=180, carbs=300, fat=70, calories=2400` even though those macros sum to 2,550 kcal — some lifters set independent macro targets and a slightly looser calorie ceiling, some lifters set calories then derive macros. The Set Goals form surfaces the computed-from-macros total as a hint underneath the calories input, but doesn't block the save. If the user wants strict consistency they can read the hint and edit accordingly.

## Goals and Non-Goals

### Goals

- Persist one row per user in a new `user_macro_goals` table with `protein_g`, `carbs_g`, `fat_g`, and `calories` integer columns. UPSERT-on-PUT semantics so the user's first save creates the row and subsequent saves replace it.
- Expose the goals via `GET /me/macro-goals` + `PUT /me/macro-goals` on the Go API. JWT-gated under the existing auth middleware. Validate each field is a non-negative integer ≤ a sane cap (proposed 10,000 to catch typos; nobody legitimately needs to log 100,000g of protein/day).
- Add two MCP tools — `get_macro_goals` and `set_macro_goals` — to `prog-strength-mcp`. Same JWT-pass-through shape as the existing tools; no agent-side state.
- On the Nutrition page's Today view on both web and mobile, render four ring charts (one per macro) showing today's intake as a fill of that macro's goal. Each ring also shows the absolute "intake / goal" numbers underneath so the user doesn't have to do the arithmetic.
- A Set Goals affordance (button on web, tap-a-ring on mobile) opens a modal/sheet with one form for all four numbers. Save calls `PUT /me/macro-goals` and refetches.
- When the user has not set goals yet, the rings render as empty grey outlines with a "Set goals" call to action. No misleading "0% of 0" division-by-zero math.
- The agent can answer "how am I doing on protein today?" by composing `get_macro_goals` + `get_daily_macros` and citing the comparison. No prompt changes required; the agent's existing reasoning handles the join when both tools return structured data.

### Non-Goals

- **Time-bounded goal phases.** A user can't have a "cut" goal that automatically swaps to a "bulk" goal next month — one current goal at a time. Phase-switching is genuinely useful but adds a goal history table, an active-phase selector, and a date-range query on every read. File separately if it ever comes up; the v1 user can update their single goal manually when their training block changes.
- **Per-day-of-week goals** (e.g., higher carbs on lifting days). Same reason — interesting, much more design surface, defer.
- **Computed-from-formulas goals.** No "tell me your bodyweight and goal and the app suggests targets" calculator. Users set their own numbers; the agent is the one with the heuristics to recommend, and a dedicated calculator tool would step on that conversational flow. (The agent's heuristics — 0.8–1.0g protein/lb, etc. — would land in a separate SOW if/when we formalize them.)
- **Enforcing macro/calorie consistency.** The form surfaces the math as a hint but doesn't reject. A future "lock to derived" toggle could be useful but adds a UI mode for an edge case.
- **Goal trends or "this week vs goal" rollups.** The rings are today-only. Historical goal-adherence (a streak counter, a weekly graph) is a follow-up.
- **Public sharing or comparison.** Goals are strictly per-user, never visible to anyone else.
- **Pre-shipping a default goal set.** A new user sees an empty state until they explicitly save goals. We're not picking numbers for them.

## Implementation Details

### Data Model

One new migration: `009_user_macro_goals.sql`.

| Column | Type | Description |
| --- | --- | --- |
| `user_id` | text | Owning user. Primary key. One row per user. |
| `protein_g` | integer | Daily protein target in grams. Non-negative; ≤ 10,000. |
| `carbs_g` | integer | Daily carbohydrate target in grams. Same bounds. |
| `fat_g` | integer | Daily fat target in grams. Same bounds. |
| `calories` | integer | Daily calorie target. Non-negative; ≤ 100,000. |
| `created_at` | datetime | First write. |
| `updated_at` | datetime | Bumped on every PUT. |

No `deleted_at` — the row is a per-user singleton, not user content. If the user wants to clear their goals they can set all four to zero (the UI treats zeros as "unset" for the empty-state ring rendering).

Index: the primary key on `user_id` is the only access path. No additional indexes.

No foreign key on `user_id` — same convention as `workouts` and `chat_sessions`, since OAuth-only users aren't always in the DB before their first write.

### Write Path

**`PUT /me/macro-goals`**

Body: `{ "protein_g": int, "carbs_g": int, "fat_g": int, "calories": int }`. All four fields required; missing-field requests return 400. Validation:

- Each macro field: non-negative integer ≤ 10,000
- Calories: non-negative integer ≤ 100,000
- Reject negative or non-integer with a specific 400 message per field

Transaction:

- `INSERT INTO user_macro_goals ... ON CONFLICT (user_id) DO UPDATE SET ...` — single SQLite UPSERT, no separate get-then-write race.
- `created_at` set on insert only; `updated_at` set on both insert and update.

Returns the persisted row.

### Read Path

**`GET /me/macro-goals`**

Returns 200 with the user's row. Critically, returns 200 with a zero-valued row (and `created_at`/`updated_at` set to null) when the user has never written goals — the client uses the zero state to render the "Set goals" empty UI without a 404 dance.

Single statement: `SELECT ... FROM user_macro_goals WHERE user_id = ?`. No N+1, no joins.

### API Surface

| Path | Method | Auth | Purpose |
| --- | --- | --- | --- |
| `/me/macro-goals` | GET | required | Read the authed user's goals. Returns zeros when never set. |
| `/me/macro-goals` | PUT | required | Replace the authed user's goals atomically. |

No DELETE — clearing goals is "set all to zero", which is conceptually a different intent (you've abandoned your goals, but the row stays so the audit trail of `updated_at` is preserved).

### MCP tools (`prog-strength-mcp`)

Two new tools wrapping the API endpoints. Same JWT-pass-through pattern the existing `list_workouts` / `create_workout` tools use; no MCP-side state.

- **`get_macro_goals()`** — no arguments. Calls `GET /me/macro-goals` and returns the four numbers plus the timestamps. The agent uses this to answer "what are my goals?" or to compose with `get_daily_macros` for "how am I doing today?"
- **`set_macro_goals(protein_g, carbs_g, fat_g, calories)`** — calls `PUT /me/macro-goals`. The agent uses this when the user says "set my protein target to 200g" — the agent reads current goals first, applies the diff in its own reasoning, then writes the whole set back (matching the API's set-replacement contract).

No new `compare_today_to_goals` tool. The agent already has `get_daily_macros` (from the daily-nutrition-log SOW); composing the two reads is cheap and the prose answer is what the user actually wants. Adding a precomputed comparison tool would freeze the format the agent uses, which we'd regret the first time we want to nudge it.

### Web client (`prog-strength-web`)

- `lib/api.ts` gains two new fetchers: `getMacroGoals(token)` and `putMacroGoals(token, payload)`. Same `unwrap` envelope as the rest of the lib.
- `app/(app)/nutrition/page.tsx` — the Today view (today's macro aggregate) gains a new MacroGoalRings component at the top, fetching `getMacroGoals` + the existing daily-macros endpoint and rendering four rings. A "Set goals" button in the top right of the section opens the existing modal pattern (see `workout-modal.tsx` for the shape).
- New `components/macro-goals-modal.tsx` — single form for the four numbers + a computed-calories hint that updates live as the user types macros. Save calls `putMacroGoals` and bubbles the result up.
- Ring chart implementation: hand-rolled SVG donut. The Progress page already uses recharts for line charts but rings are simple enough that pulling in `RadialBarChart` is overkill. ~30 lines of SVG with `strokeDasharray` driving the fill arc.

### Mobile client (`prog-strength-mobile`)

- `lib/api.ts` gets the same two fetchers, ported verbatim per the "edit twice" convention.
- `components/nutrition/today-view.tsx` (already exists) gains the same MacroGoalRings row at the top. Tapping any ring opens a sheet (RN `Modal` with `presentationStyle="pageSheet"`, same pattern the headline-exercises sheet uses) with the four-number form.
- Ring chart: SVG via `react-native-svg` (already a dep from the Progress tab work). Same shape as the web's hand-rolled donut.

### Algorithms

Two pieces of math, both trivial:

**Ring fill percentage**: `min(intake / goal, 1) * 100` when `goal > 0`, else `0` (renders the empty grey state). Over-goal display (intake > goal) is the open question below; the simple "clamp to 100%" reading is the lean.

**Live calories-from-macros hint inside the Set Goals form**:

```
hint_calories = protein_g * 4 + carbs_g * 4 + fat_g * 9
delta = calories - hint_calories
```

Show the user `"Macros total {hint_calories} kcal ({delta} {over|under} your calorie target)"` so they can decide whether the inconsistency is intentional. No save-time validation either way.

### Backfill or Migration

No backfill. New table, fresh rows added on first PUT per user. Existing users see the empty-state ring UI until they write goals.

## Open Questions

1. **How to render "intake exceeded goal"?** Clamping the visual ring at 100% is the easy answer but hides important information — going 50% over calories on a cut matters. Options: (a) ring fills past 100% by stacking a second-color arc, (b) ring caps at 100% but the text label below reads "112% of goal" in a warning color, (c) ring caps at 100% silently and the per-macro tile turns amber. Tentative lean: (b). The arc reaches 100%, the text below tells the truth, color signals "over" without screaming.
2. **What's a sensible per-macro upper bound?** 10,000g of protein/day is comically wrong, but capping too aggressively rejects legitimate edge cases (a 350lb strongman on a force-feeding block might be at 700g+ protein). 10,000 catches typos (10000 instead of 100) without rejecting any realistic input. Lean: ship with 10,000g per macro, 100,000 calories. Revisit if anyone complains.
3. **Should the agent's `set_macro_goals` do a read-modify-write transparently, or require the agent to pass all four fields verbatim?** Set-replacement is the API contract, but conversational UX often wants "just bump my protein by 20g" without touching the other three. Tentative lean: agent reads current with `get_macro_goals`, applies its own diff, writes the full set back. Keeps the API simple; pushes the merge cost into the agent's prompt where it belongs.
4. **Single Set Goals form, or a row of four inline tap-to-edit numbers?** Inline editing is faster for "bump protein by 10" but lets the user partially-save (e.g., update protein then navigate away — the four fields aren't really independent semantically). Tentative lean: a single modal form for v1; inline editing is a follow-up once the muscle memory is established.
5. **Should rings be color-coded per macro** (red/blue/green/amber) or all one accent color? Color makes scanning faster but introduces semantic load (does red mean "over goal" or "this is the calories ring"?). Tentative lean: all one accent color, with the warning color reserved exclusively for "intake exceeded goal" per Q1.
