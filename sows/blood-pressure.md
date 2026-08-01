---
status: draft
repos:
  - prog-strength-api
  - prog-strength-mcp
  - prog-strength-agent
  - prog-strength-web
  - prog-strength-docs
---

# Blood Pressure

**Status**: Draft · **Last updated**: 2026-08-01

## Introduction

**Prog Strength** tracks what a user does — the lifts, the runs, the food, the steps — and one thing about what their body *is*: their bodyweight. Blood pressure is the other number in that second category, and arguably the more consequential one. It is the most widely monitored cardiovascular vital in the world, roughly half of adults have readings above the healthy range, and the condition is famously silent — there are no symptoms to notice, only numbers to watch. A lifter who cares enough to log every set is exactly the person who should be watching it, particularly since heavy resistance training, sodium intake, sleep, and bodyweight all move the number.

Today there is nowhere in the product to put a reading. A user with a home cuff — a cheap, common device — takes a measurement, sees `124/78` on a screen for ten seconds, and then loses it. There is no history, no trend, and no way to answer the only question that matters: *is my baseline drifting?* A single reading is close to meaningless; blood pressure is noisy across the day and across the week, and its value comes entirely from the average and the trend. The product is well positioned to supply exactly that, because it already knows how to store a timestamped measurement, chart it, and let an agent write to it.

This SOW adds a **Blood Pressure** page. A user logs a reading as often as they like — several times a day, retroactively, through a form or by telling the agent — and gets back a chart of systolic and diastolic over time against the healthy range, an average that gives them a real baseline, and a plain-language explanation of what the two numbers mean and why the trend is worth watching. They can add a Blood Pressure tile to their dashboard for a glance from the home screen. And this page is the first to carry the **agent chat bar as its top row**, establishing the pattern that every feature surface keeps the coach one keystroke away.

## Proposed Solution

Blood pressure is shaped almost exactly like bodyweight: a timestamped measurement, many per day, corrected by editing or deleting, read back as a range for a chart and as a list for a log. So it follows the path bodyweight already paved — a small dedicated Go domain with a per-entry table, a thin MCP forwarder, an agent intent, and a web page built from the existing chart-card, timeline-rail, and stat-tile patterns. The differences from bodyweight are that a reading carries **two required numbers plus an optional pulse** instead of one, and that it has **no user-set goal**: the healthy range is a published clinical classification, identical for everyone, so it is a constant in the code rather than a row in a table.

A new `internal/bloodpressure` package owns one table (`user_blood_pressure`) and four routes under `/blood-pressure`, mirroring `/bodyweight` route-for-route. The one piece of real logic in the domain is **classification** — turning a systolic/diastolic pair into `normal` / `elevated` / `stage_1` / `stage_2` / `crisis` — which lives server-side and ships on every entry DTO so the web page, the agent, and any future mobile client cannot drift from each other on what a number means.

The dashboard gains a **Blood Pressure tile**, added to the Go catalog and its TypeScript mirror. It is deliberately **not** in the default layout: a user who has never logged a reading should not inherit an empty card. They add it from the tray when they want it.

On the web, `/blood-pressure` becomes a first-class sidebar destination whose top row is the **agent chat bar** — the dashboard's `CommandBar`, promoted out of the dashboard's private components into the shared `components/` directory so every future page can adopt it without a copy. The rest of the page is a hero dual-line chart against the healthy band, summary tiles, a log affordance, a day-grouped reading rail, and an "About blood pressure" explainer that teaches systolic vs. diastolic and states plainly that none of this is medical advice.

## Goals and Non-Goals

### Goals

- Log a blood-pressure reading — **systolic, diastolic, and optional pulse** — for any instant, current or retroactive, from the web form and from the agent.
- Store readings as **timestamped entries with no per-day limit**, matching bodyweight; corrections are edits or soft deletes, never silent overwrites.
- **Classify every reading server-side** into the five ACC/AHA categories and return the category on the entry.
- A new **Blood Pressure** sidebar page showing, top to bottom: the **agent chat bar**, range tabs, a **hero dual-line chart** of daily-average systolic and diastolic with the healthy band and the 130/80 reference lines, summary stat tiles, a log affordance, a day-grouped reading rail, and an **explainer card**.
- A **Blood Pressure dashboard tile** available from the add-tile tray, showing the latest reading, its category, and a dual sparkline.
- **Promote `CommandBar` to a shared component** (`components/command-bar.tsx`) so it is reusable by any page, with the dashboard switched over to the shared import.
- An MCP `blood_pressure` module and an agent `log_blood_pressure` intent, so `"log 122 over 78"` works in chat.

### Non-Goals

- **Mobile parity.** Web only in this SOW; mobile follows the established per-phase parity workflow as its own dispatch.
- **Device sync** from a connected cuff, Apple Health, Health Connect, Withings, or Omron. Manual and agent entry only.
- **Per-user or doctor-set targets.** The reference lines are the published classification, identical for every user. A goal table is explicitly not built (see Open Questions).
- **Notifications, alerts, or reminders** of any kind — no "time to measure" nudge, no push on a high reading.
- **Medical advice, diagnosis, or medication guidance**, from the UI or the agent. The product reports numbers and their published classification; that is the whole scope.
- **Retrofitting the chat bar onto the other existing pages.** This SOW makes the component shared and proves it on one page. Adoption elsewhere is a separate, mechanical change.
- **Derived cardiovascular analytics** — mean arterial pressure, pulse pressure trends, morning-vs-evening splits, correlation against bodyweight or training load.
- **Backfill.** This is a new data type; there is nothing to seed.

## Implementation Details

### Data Model

New migration `internal/db/migrations/050_blood_pressure.sql`. **Numbering note:** `050` is the next free index at the time of writing; if another SOW lands first, renumber to the next free one. One table, following the conventions of the surrounding migrations (no FK on `user_id`, matching the other per-user content tables; CHECK constraints that mirror handler-side validation; soft delete via `deleted_at`).

**`user_blood_pressure`** — one row per reading.

| Column | Type | Description |
| --- | --- | --- |
| `id` | TEXT PRIMARY KEY | From `internal/id`. |
| `user_id` | TEXT NOT NULL | Owner. No FK, matching `bodyweight_entries`. |
| `systolic` | INTEGER NOT NULL | `CHECK (systolic BETWEEN 50 AND 300)`. The bounds are physiological-plausibility guards against fat-fingered input, not clinical limits. |
| `diastolic` | INTEGER NOT NULL | `CHECK (diastolic BETWEEN 30 AND 200)`. |
| `pulse` | INTEGER NULL | Beats per minute. Optional — every home cuff prints it alongside the pair, so capturing it costs one nullable column and loses nothing if omitted. `CHECK (pulse IS NULL OR pulse BETWEEN 20 AND 250)`. |
| `measured_at` | TEXT NOT NULL | RFC3339 instant, stored UTC — same treatment as `bodyweight_entries.measured_at`. |
| `created_at` | TEXT NOT NULL | Insert time. |
| `deleted_at` | TEXT NULL | Soft delete. |

Additional table-level constraint: `CHECK (systolic > diastolic)`. This is a physical invariant, not a clinical one — systolic is the peak and diastolic the trough of the same pressure wave, so a pair where they are equal or inverted is a data-entry error every time.

Index:

```sql
CREATE INDEX idx_bp_user_measured
    ON user_blood_pressure(user_id, measured_at DESC) WHERE deleted_at IS NULL;
```

There is **no goal table**. The healthy range is a published classification identical for every user (see Algorithms), so it is a constant in Go, not per-user state. This is the one structural difference from bodyweight, which has `user_bodyweight_goal`.

### API — `internal/bloodpressure`

A new domain package following the established anatomy — `bloodpressure.go` (the `Entry` type and `Validate`), `category.go`, `errors.go`, `repository.go`, `sqlite_repository.go`, `handler.go`, and matching `*_test.go` — registered in `internal/server/server.go`:

```go
bloodpressure.NewHandler(bpRepo).Mount(r)
```

**No memory repository.** The in-memory implementations were retired (see [`deprecate-in-memory-repositories.md`](deprecate-in-memory-repositories.md)); `internal/bodyweight` ships only `sqlite_repository.go`, and this package matches. Handler tests use the SQLite repo against a temp database via `internal/testutil`, exactly as the bodyweight handler tests do.

Routes mount behind the existing auth middleware; the caller's id comes from `auth.UserIDFrom(ctx)` and responses go through the `httpresp` helpers — structurally identical to `internal/bodyweight/handler.go`.

- **`GET /blood-pressure?since=&until=`** — the user's non-deleted readings, newest `measured_at` first. `since` inclusive, `until` exclusive, either optional, parsed by the same `parseSinceUntil` shape bodyweight uses. **No keyset pagination**: bodyweight's list returns the whole range and the reading rail paginates client-side, and this page reuses that rail unchanged.
- **`POST /blood-pressure`** — create. Body `{ "systolic": int, "diastolic": int, "pulse": int|null, "measured_at": string|null }`. `measured_at` defaults to `time.Now().UTC()` when omitted. Returns the stored entry (`201`).
- **`PUT /blood-pressure/{id}`** — partial update. Every field is a pointer; only supplied fields overlay onto the existing row, matching `bodyweight`'s `updateRequest`. `id`, `user_id`, and `created_at` are always preserved. `404` on missing, soft-deleted, or cross-user ids.
- **`DELETE /blood-pressure/{id}`** — soft delete. `204`.

The entry DTO carries the stored fields plus the derived category:

```json
{
  "id": "bp_…",
  "systolic": 122,
  "diastolic": 78,
  "pulse": 61,
  "category": "elevated",
  "measured_at": "2026-08-01T13:04:00Z",
  "created_at": "2026-08-01T13:04:12Z"
}
```

Ownership is enforced at the storage layer (every query carries a `user_id` predicate, cross-user ids return `ErrNotFound`), mirroring the `bodyweight.Repository` contract so handlers never have to remember the clause.

### Algorithms

**Classification.** A reading maps to one of five categories from the 2017 ACC/AHA guideline. The rule is *higher category wins* — a reading is classified by whichever of its two numbers is worse — so the checks must be evaluated in descending severity and the order is load-bearing:

```
crisis    if systolic >= 180 or diastolic >= 120
stage_2   if systolic >= 140 or diastolic >=  90
stage_1   if systolic >= 130 or diastolic >=  80
elevated  if systolic >= 120                       // diastolic < 80 by fallthrough
normal    otherwise                                // systolic < 120 and diastolic < 80
```

Note the asymmetry at the `elevated` tier: the guideline defines Elevated as `120–129 systolic AND <80 diastolic`. A reading of `125/85` is **Stage 1**, not Elevated, because the diastolic already crossed 80 — which the ordering above produces for free, since `stage_1` is tested first. Getting this wrong in the other direction (testing `elevated` before `stage_1`) would silently under-report, so it gets an explicit table test with `125/85`, `135/75`, `115/95`, and each boundary value (`119/79`, `120/79`, `130/80`, `140/90`, `180/120`) as cases.

The classification lives in `internal/bloodpressure/category.go` and is the single source of truth. It ships on the entry DTO and on the dashboard section, so no client re-derives it for a stored reading.

**Why no age adjustment.** The user's original framing asked whether the healthy threshold varies by age. It does not, under the current guideline: the 2017 ACC/AHA revision deliberately applies the same `≥130/80` threshold to adults of every age, including older adults — the previous practice of relaxing the target with age was dropped. So a single fixed classification is both simpler *and* the clinically correct choice, and a per-user target would be a product invention rather than a personalization of something real.

**Daily averaging.** The hero chart plots one systolic and one diastolic point per day. When a day holds several readings, the point is the arithmetic mean of that day's values, rounded to the nearest integer for display. Days with no readings are gaps in the line, not zeros. This is the same daily-average treatment the bodyweight trend chart applies, chosen so the two pages read as one system and so a user who measures three times on Tuesday doesn't get a spiky chart that hides the trend. Bucketing is by the **user's local calendar day**, not UTC.

### Dashboard Tile

**Go catalog** (`internal/dashboard/tiles.go`): add `TileBloodPressure TileID = "blood_pressure"` and place it in `Catalog` after `TileBodyweight` and before `TileRecovery` — the health-metric cluster. `Catalog` order fixes the add-tile tray order, and `tiles_test.go` asserts the Go and TypeScript lists stay identical in **set and order**, so the mirror below is not optional.

**Section builder** (`internal/dashboard/bloodpressure.go`): a pure `buildBloodPressure(entries []bloodpressure.Entry, loc *time.Location) *BloodPressureSection`, returning `nil` when the user has no readings (the same "absent section" convention every other tile follows). The section:

| Field | JSON | Description |
| --- | --- | --- |
| `Latest` | `latest` | The newest reading: `systolic`, `diastolic`, `measured_at`. |
| `Category` | `category` | The latest reading's category. |
| `Avg30` | `avg_30d` | Mean systolic and diastolic over the trailing 30 days, rounded to integers. Nil when the window is empty. |
| `SystolicSpark` | `systolic_spark` | ~8 daily-average points, oldest→newest, via the existing `downsampleFloats`. |
| `DiastolicSpark` | `diastolic_spark` | The same days, same length, same order. |

The two sparks are computed from **one** day-bucketed series so their indices align — the card draws them as two lines on a shared x-axis, and a mismatch in length or day set would silently skew one against the other. A test asserts equal length and identical day alignment.

Day bucketing uses the `timezone` parameter `GET /dashboard/summary` already accepts (the web client passes `browserTz()`), per the project's timezone/date-window convention: the API converts a timezone plus local dates into UTC bounds, and no client builds UTC windows itself.

**Default layout** (`internal/dashboard/layout_resolve.go`): **unchanged**. `defaultLayout` continues to return running, lifting, steps, nutrition, bodyweight, [recovery,] streak. Blood pressure is opt-in from the tray — a user who has never taken a reading should not land on an empty card, and unlike Recovery there is no connection state to key a conditional default off. A test asserts the default layout does not contain `blood_pressure`.

**TypeScript mirror** (`lib/dashboard-tiles.ts`): add `"blood_pressure"` to the `TileId` union and the matching `TILE_CATALOG` entry in the same position:

```ts
{
  id: "blood_pressure",
  title: "Blood Pressure",
  href: "/blood-pressure",
  description: "Latest reading and trend against the healthy range.",
}
```

`lib/dashboard-tiles.test.ts` holds an exhaustive `Record<TileId, true>`; adding the id makes that test fail until the new key is listed, which is the intended tripwire.

**Card** (`app/(app)/dashboard/_components/tile-renderer.tsx`): a `BloodPressureCard` case in the exhaustive `switch`. The `never` default means omitting the case is a **compile** error, so the tile cannot ship without a card. The card renders a `MiniCard` with a `BigNum` of `"122/78"`, a `MetaRow` of category / 30-day average / time since the last reading, and a compact dual sparkline. Its `!section.present` branch shows a "Log your first reading" CTA like the other cards.

The dual sparkline is a small extension of the existing `Spark` component (which takes a single series). Prefer adding an optional second series to `spark.tsx` over forking it; if that muddies the component, a sibling `dual-spark.tsx` is acceptable — but not a copy-paste fork of `Spark` with a second `<path>` bolted on.

### Web — the shared chat bar

`CommandBar` currently lives at `app/(app)/dashboard/_components/command-bar.tsx`. This SOW **moves it to `components/command-bar.tsx`** and updates the dashboard's import. This follows the repo's stated rule — page-private components live in `_components/`, anything used by two or more pages moves up to `components/`, *promoted on second use, not preemptively*. This is that second use.

Behavior is unchanged: a controlled text field behind the periwinkle `›` chevron, `--accent` border plus `--accent-line` ring on focus, and on submit it calls `onSubmit(trimmed)` and clears. The consumer supplies the handler; both consumers route to `/chat?prompt=${encodeURIComponent(value)}`. The component already accepts a `placeholder` prop, so no signature change is needed — the Blood Pressure page passes its own copy:

> `Ask your coach, or log a reading — "122 over 78 this morning"`

The dashboard's existing placeholder and behavior stay exactly as they are; this is a move plus one import change, and the existing dashboard tests must keep passing untouched.

**Deliberately out of scope:** an inline answer strip that keeps the user on the page. Submitting navigates to `/chat`, exactly as the dashboard does today. Making the bar answer in place is a real upgrade — it would let a user log a reading and watch the chart refresh without leaving — but it needs a reusable SSE-backed component and a refetch contract, and doing it as a follow-up means it swaps the internals of one shared component and every adopting page gets it at once. That is the right shape for that work; it is not this SOW.

### Web — the Blood Pressure page

> **Critical implementer note:** the web repo is the **"This is NOT the Next.js you know"** fork. Consult `node_modules/next/dist/docs/` and the validated patterns already in the repo before writing code. `app/(app)/bodyweight/` is the direct structural reference for the chart card, the log affordance, and the reading rail; `app/(app)/dashboard/page.tsx` is the reference for the chat-bar row.

**Sidebar** (`components/sidebar.tsx`): a new `NAV` entry `{ href: "/blood-pressure", label: "Blood Pressure", icon: <HeartPulseIcon /> }`, placed **after Bodyweight and before Recovery** so the body-metric surfaces cluster. `HeartPulseIcon` is a new inline line-icon in the same file, matching the stroke weight and 20px box of its neighbors. Default prefix-matching highlight behavior applies; no `exact` flag is needed (no sibling entries below it).

**Route** `app/(app)/blood-pressure/page.tsx`. Page flow, top to bottom:

1. **Header** — `h1` "Blood Pressure" plus a one-line subtitle, matching the dashboard and bodyweight header block (`border-b`, `px-3 py-4 sm:px-6`).
2. **Chat bar** — `<CommandBar onSubmit={…} placeholder={…} />` as the first element of the scroll body, above everything else. This is the pattern the SOW is establishing: on a feature page the coach sits at the top, before the feature's own content.
3. **Range tabs** — 30 / 60 / 90 / All, the same `RangeKey` shape the bodyweight page uses, with the `border-b` doubling as the separator.
4. **Hero card** — chart at the top, stat tiles tucked inside the same bordered box below it so the two read as one unit (the bodyweight chart-card pattern).
5. **Log toolbar** — a pencil-icon "Log" button, matching bodyweight's, opening the log modal.
6. **Reading rail** — day-grouped, one node per day, paginated whole-day.
7. **About card** — the explainer.

**The hero chart.** A recharts composition with, back to front: a shaded band across the healthy region, dashed `ReferenceLine`s at **130** and **80**, then the systolic and diastolic lines over daily averages. Requirements:

- The y-axis domain must always include **both reference lines and the band**, so they can never clip off-screen — the same class of fix as `bodyweight-goal-in-yaxis-domain`. A user whose readings sit entirely at 110/70 must still see the 130 line above them.
- Lines are `type="monotone"` with gaps for days that have no reading — a missing day is a break, never a zero and never a straight interpolation across a two-week hole.
- The tooltip shows both numbers as `122/78`, the day, the reading count contributing to that day's average, and the day's category.
- A legend distinguishes the two series; systolic is visually the primary (heavier stroke), since it is the number people quote and the one that drives most classifications.
- Below `sm:` the y-axis width and label placement follow the bodyweight chart's `matchMedia`-driven responsive handling.

**Stat tiles** (inside the hero card): **Average** (`sys/dia` over the selected range), **Latest** (the reading plus its category chip), **In normal range** (percentage of readings in the range classified `normal`), **Readings** (count, and days covered). Tiles are computed over the selected timeframe, using the existing `components/stat-tile.tsx`.

**Log modal** — systolic, diastolic, optional pulse, and a date/time defaulting to now. Same modal chrome, focus trap, and Escape handling as the bodyweight action sheet. Client-side validation mirrors the API bounds and the `systolic > diastolic` invariant so the user gets an inline message rather than a 400.

**Reading rail** — the bodyweight readings timeline generalized or paralleled: days as nodes with a connector, `READINGS_PER_PAGE = 20` with days never split across a page boundary, edit and delete emitted as callbacks so the page owns the mutations. Each reading shows `122/78`, its category chip, the pulse when present, and the time.

**About card** (`_components/bp-about.tsx`) — brief, factual, and static:

- What blood pressure is: the force of blood against artery walls, reported as two numbers.
- **Systolic** (the top, larger number): the pressure during a heartbeat, when the heart contracts and pushes blood out.
- **Diastolic** (the bottom number): the pressure between beats, when the heart is refilling.
- Why monitoring matters: high blood pressure has no symptoms, a single reading is noisy, and the useful signal is the average and the trend over weeks — which is exactly what a log of readings produces.
- The classification table (`<120/<80` Normal, `120–129/<80` Elevated, `130–139 or 80–89` Stage 1, `≥140 or ≥90` Stage 2, `>180 or >120` Crisis), with the source named.
- A plain closing line: this is general information, not medical advice, and readings are not a diagnosis — talk to a clinician about your numbers.

The card sits at the **bottom** of the page in the normal state, where a returning user scrolls past their data to reach it. In the **empty state** — no readings yet — it is **promoted above** the log CTA, because at that moment it is the only useful content on the page and it is what turns a blank screen into a reason to take a reading.

**File boundaries.** `page.tsx` owns data fetching, mutations, and layout only. The pieces live in `_components/`: `bp-trend-card.tsx` (chart + band + reference lines + tiles), `bp-log-modal.tsx`, `bp-readings-timeline.tsx`, `bp-about.tsx`. Pure helpers — classification mirror, daily averaging, category display metadata — live in `lib/blood-pressure.ts` and are unit-tested without a DOM. This is called out because `app/(app)/bodyweight/page.tsx` has grown to ~1,250 lines by absorbing its modals and helpers; this page should not repeat that, and a reviewer should push back if `page.tsx` starts collecting modal bodies.

**Client classification mirror.** The API is authoritative for a *stored* reading's category, and the page must use the DTO's `category` field for stored readings rather than recomputing it. `lib/blood-pressure.ts` also exports a `classify()` mirror plus the threshold constants, needed for values the server never classified — a **daily average**, and the live preview in the log modal before the reading is saved. Its test asserts the mirror agrees with the documented table at every boundary, so drift from the Go implementation fails CI rather than showing the user two different categories for the same pair.

**API client** (`lib/api.ts`): add `listBloodPressure`, `createBloodPressureEntry`, `updateBloodPressureEntry`, `deleteBloodPressureEntry` following the existing wrapper shape. No ad-hoc `fetch` in components.

### Web — chart colors

Recharts writes `stroke`/`fill` as SVG attributes, where CSS `var(--token)` does **not** resolve, so the chart needs a literal mirror module — `lib/blood-pressure-chart-colors.ts`, the same device as `lib/bodyweight-chart-colors.ts`.

**Do not copy `lib/bodyweight-chart-colors.ts`.** That file still holds the **retired violet `#8b7cf6`** from before the v0.4 re-tone that replaced violet with periwinkle. It is stale, and cloning it would propagate a dead palette into a new surface. Fixing that file is out of scope here — it is someone else's PR — but the new module must mirror the *current* tokens:

| Role | Value | Token mirrored |
| --- | --- | --- |
| Systolic line | `#9aa6d6` | `--accent` — the primary series |
| Diastolic line | `#7d818c` | `--muted` — clearly subordinate, neutral, reads cleanly against the periwinkle |
| Healthy band fill | `rgba(134, 179, 159, 0.10)` | `--success` at low alpha |
| Reference lines (130 / 80) | `#7d818c`, dashed | `--muted` — the same quiet treatment bodyweight gives its goal marker |
| Grid / axis / tooltip | as in `bodyweight-chart-colors.ts` | `--border`, `--muted`, `--surface` |

**Design-system conformance.** Everything above is an existing token; no new palette is introduced. Two usages are deliberate extensions worth naming so the design system can absorb or reject them:

1. **`--success` as a chart region fill.** The design system warns that the HR-zone scale must not encode value judgment. Here the judgment *is* the data — the band is a published clinical classification of the plotted values, not a stylistic opinion — so a status tone is the correct family. It is held at 10% alpha so it reads as a region, not a highlight.
2. **`--accent` as a data series.** The accent is documented as app chrome and explicitly not an activity hue. A blood-pressure reading is not an activity and has no discipline hue to claim, and systolic is the page's subject — the same role the accent plays as the bodyweight trend line. Diastolic takes `--muted` rather than a second hue, so the pair reads as one measurement with a primary and a secondary, not as two competing series.

Category chips use the status tones directly: `normal` → `--success`, `elevated` and `stage_1` → `--warning`, `stage_2` and `crisis` → `--danger`.

### MCP — `blood_pressure` module

A new `src/prog_strength_mcp/blood_pressure.py`, registered in `server.py`, modeled on `bodyweight.py`. The MCP stays a thin forwarder over the Go API — no classification, no averaging, no validation beyond the pydantic `Field` bounds that produce a good error message.

- **`log_blood_pressure(systolic, diastolic, pulse=None, measured_at=None)`** — `POST /blood-pressure`. `measured_at` accepts an explicit RFC3339 instant or is omitted for now; the agent resolves relative phrases ("this morning") before calling. Field descriptions must spell out that `systolic` is the higher number, since a model transcribing `"122 over 78"` has to assign them correctly.
- **`list_blood_pressure(since=None, until=None)`** — `GET /blood-pressure`, for coaching context and for the intent prefetch below.

Auth forwarding uses the existing `_auth_header_or_raise()` + `APIClient` pattern unchanged.

### Agent — `log_blood_pressure` intent

`prog-strength-agent` uses an intent registry (`src/prog_strength_agent/intents.py`) where each intent declares a prefetch, a rules string, and a formatter; `model_router.py` builds the classifier enum from `KNOWN_INTENTS`, so adding the intent to that tuple is what makes it routable.

- Add `"log_blood_pressure"` to `KNOWN_INTENTS`.
- **Prefetch:** `list_blood_pressure` for the trailing 14 days, mirroring `_log_bodyweight_prefetch`.
- **Format:** `RECENT BLOOD PRESSURE (last 14 days, most recent first)` with one line per reading — `measured_at · 122/78 (elevated) · pulse 61`.
- **Rules:** the user is logging a reading; map the spoken form correctly (the first/larger number is systolic); confirm the resolved timestamp back when the phrasing was relative; state the reading's category as a plain fact when it is outside `normal`; and reference the recent trend only when the new reading is a meaningful departure from it.

**Safety rules, stated in the intent's prompt and non-negotiable:** the agent does not diagnose, does not interpret a reading as a condition, and never discusses medication — dosage, timing, starting, or stopping. On a `crisis`-range reading it states the classification factually and says to contact a clinician; it does not escalate further, alarm, or speculate about cause. This is the same boundary the About card draws in the UI, and the two should not disagree.

### Tests

- **API repository:** create / get / list-by-range / soft delete / partial update, ownership scoping (a second user cannot read or mutate another's readings), and every CHECK boundary including `systolic > diastolic` rejection.
- **API classification:** the table test described in Algorithms — every tier boundary plus the `125/85` and `135/75` asymmetry cases.
- **API handler:** `POST` defaults `measured_at`, `PUT` overlays only supplied fields, `400` on out-of-range and inverted pairs, `404` on cross-user ids, and `category` present on every returned entry.
- **API dashboard:** section is `nil` with no readings; sparks are equal-length and day-aligned; the 30-day average excludes older readings; the default layout does **not** include `blood_pressure`; the Go↔TS catalog contract test passes with the new id in both lists.
- **MCP:** `log_blood_pressure` forwards the right body and surfaces `APIError` cleanly.
- **Agent:** the intent is registered and routable; the formatter renders an empty history and a populated one.
- **Web:** `lib/blood-pressure.ts` classification mirror against the boundary table; daily averaging (multi-reading days, gap days, timezone bucketing); `BpTrendCard` renders both reference lines and keeps them inside the y-axis domain when readings sit far below them; the About card is promoted in the empty state and demoted otherwise; `CommandBar` still renders and submits from its new shared location and the dashboard's existing tests pass unchanged; the `BloodPressureCard` empty and populated branches.

### Rollout

Entirely additive — no existing endpoint, table, or surface changes behavior. The one file that moves is `CommandBar`, and that is a move plus an import update with identical behavior. Recommended order:

1. **API** — migration `050`, `internal/bloodpressure`, the four routes. Ship.
2. **API dashboard** — tile id, section builder, catalog. Ship (the tile is invisible until a user adds it).
3. **MCP + Agent** — the `blood_pressure` module and the `log_blood_pressure` intent, against the live API.
4. **Web** — `CommandBar` promotion, the sidebar entry, the page, the tile card and TS mirror.

No feature flag. Every surface is empty-safe: the page renders its explainer and a log CTA before the first reading, the tile is absent from the default layout, and the agent intent is inert until a user mentions blood pressure.

## Open Questions

1. **An optional `note` on a reading.** Context matters for blood pressure — "after coffee", "post-squats", "left arm" — and a free-text column is one nullable field. Options: add `note TEXT NULL` now; add a structured tag enum (position / arm / time-of-day); omit entirely. **Tentative lean: omit for v1.** The rail and the chart have nowhere meaningful to put it yet, and adding a nullable column later is a trivial migration. Revisit if the first weeks of real use produce readings the user can't tell apart.

2. **A crisis-range banner in the UI.** A reading at or above `180/120` is the one case where "just show the number" feels thin. Options: a static factual banner on the reading and in the About card ("readings this high warrant contacting a clinician"); nothing beyond the category chip; a dismissible page-level alert. **Tentative lean: the static factual banner**, no notification and no interruption — it matches the agent's rule and stays inside the not-medical-advice boundary.

3. **Extracting a shared `vitals` shape.** Bodyweight and blood pressure are now two near-identical measurement domains, and resting heart rate or SpO2 would be a third. Options: leave them independent; extract shared repository/handler scaffolding; unify onto one table with a metric discriminator. **Tentative lean: leave them independent until a third manual vital actually lands.** Unifying today means migrating a shipped table and rewriting four working surfaces for a payoff that arrives later. Record the trigger, not the refactor.

4. **Sidebar length.** The nav goes from eleven entries to twelve with this addition. Options: leave it flat; introduce a "Health" group holding Bodyweight, Blood Pressure, and Recovery; move Blood Pressure under Bodyweight as a sub-view. **Tentative lean: leave it flat for now** — the user explicitly asked for a top-level page, and grouping is a whole-sidebar design decision that deserves its own DX ticket rather than being decided by whichever feature happens to push the count over the line.

5. **Mobile parity timing.** Web-only here. Options: fold mobile into this dispatch; follow the established research → plan → subagent parity phase afterward. **Tentative lean: a separate parity phase**, consistent with how the mobile app has taken every other feature.
