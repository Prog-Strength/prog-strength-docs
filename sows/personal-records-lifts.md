---
type: sow
status: ready_for_implementation
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Personal Records — Lifts View — Compact-Dashboard

**Status**: Ready for implementation · **Last updated**: 2026-06-18

> Frontend-led SOW with a small supporting API change. It implements a chosen DX
> variant and therefore inherits that DX's `scope` — here **`in-system`**. The
> visual foundation (design-system **v0.4**, oura-calm) is already decided, so this
> SOW **conforms** to it; it does **not** re-tone the system or touch shared tokens.
> The web rebuild lands in `prog-strength-web`; a backing field is added in
> `prog-strength-api`; `prog-strength-docs` flips the DX to `selected` and marks
> this SOW shipped.

## Introduction

`/personal-records` is the app's **trophy case** — where a lifter goes to see their
bests. Its **Lifts view** (`app/(app)/personal-records/page.tsx` → `LiftsView` →
one `LiftPRCard` per backend-curated headline lift) shipped as a uniform
three-column grid of identical cards: per lift, the heaviest tested set (weight ×
reps + date), the recency-weighted **estimated 1RM** with a `+X vs PR` delta, an
amber **"Time for a max?"** badge, a `View workout →` link, and an expand chevron
revealing an inline `ProgressionChart`. It works, but the Design Exploration
`dx/personal-records-lifts.md` (PR Prog-Strength/prog-strength-web#83) identified
four things that undercut it: **the real signal is buried** (the PR-vs-estimate
*gap* reads as two disconnected numbers plus a redundant badge that fires on 7 of 8
lifts — a binary that's true everywhere is noise); **it's flat** (every lift an
equal-weight box, no hierarchy, no ordering); **boxy label chrome** (`CURRENT
ESTIMATED 1RM`, `SET ON …` furniture repeated per card); and **the never-PR'd lift
is dead weight** (a full-height card showing only `—`).

The DX explored the **whole Lifts view** across **five `in-system` variants** —
all sharing design-system v0.4 (near-black ramp, single periwinkle accent as
app-chrome, Manrope, 14px hairline panels), diverging only on layout, density, and
composition, and all treating readiness (`gapPct = (est − PR) / PR`) as a
**continuous magnitude** rather than the shipped binary badge. The owner selected
**`compact-dashboard`** (reference: **Whoop**'s overview grid + a trading
dashboard): **the whole case at a glance** — every lift visible at once in a tight
tile grid, each tile a mini-stat (PR figure, a small est-1RM delta, a **readiness
dot that scales with the gap**, recency, and a **tiny inline trend spark**), small
functional type, very tight vertical rhythm. It's the power-user "show me
everything" answer and the variant that scales best to 12 lifts. A lightweight
chrome strip summarizes the case (`N due · N/total tested`) beside the Lifts/Running
switcher and Customize button.

This SOW reimplements that variant **production-quality** against the real route
and data. Because `scope: in-system`, **no palette, accent, type, or
design-system change** — it conforms to v0.4. The one substantive backend addition
is a **compact estimated-1RM trend series per lift on the list response**, so the
per-tile sparks render from the single `GET /personal-records` the page already
calls — no per-tile fetches.

## Proposed Solution

Rebuild the **Lifts view** (`LiftsView` and the per-lift card) into the
compact-dashboard tile grid, keeping the page's orchestration: the `?view=`-routed
shell, the **Lifts/Running `ViewSwitcher`**, the **Customize** modal
(`HeadlineExercisesModal`, Lifts-only), the lazy per-view TanStack queries, and the
**Running view untouched**. The mockup on the DX branch
(`app/design-explore/personal-records-lifts/_variants/CompactDashboard.tsx` +
`_fixtures.ts`) is the **visual spec, not code to promote**.

Three pieces of real work:

1. **API — a compact 1RM trend on the list response.** The mockup's spark is
   **synthetic** (`sparkSeries` is seeded fake data — there's no series on
   `PersonalRecord`). The real per-lift series already exists at `GET
   /personal-records/{exercise_id}/history` (it powers the expand chart), but
   fetching it per tile is N round-trips on load. The elegant fix: the
   `personalRecords` handler (`internal/workout/handler.go`) **already fetches each
   exercise's 1RM history in its list loop** to compute `current_estimated_1rm` — so
   we emit a **downsampled, bounded trend array** from those same in-hand entries on
   each row. **Zero extra DB queries, zero extra round-trips**; the page gets sparks
   from the request it already makes.

2. **Web — a pure, tested readiness derivation.** The mockup's `deriveReadiness`
   (gap, `gapPct`, `ready`, `daysSince`, `isDumbbell`) and the duplicate inline
   computation in today's `LiftPRCard` are consolidated into **one tested helper**
   over the real `PersonalRecord`, using a real `new Date()` for staleness (the
   mockup pins `TODAY`) and the existing `format.ts` rather than the mockup's `fmt*`
   copies.

3. **Web — the dashboard rebuild.** `LiftsView` becomes the dense tile grid; each
   tile renders the PR figure, the scaling readiness dot, the compact delta, the
   recency line, and the **real spark** from the new field. The never-PR'd lift gets
   a deliberate dashed "untested" tile. The existing per-lift `View workout →` and
   the lazy `ProgressionChart` move into a **tile expand** (Open Question #2).

The chrome `N due · N/total tested` summary is derived client-side for free.

## Goals and Non-Goals

### Goals

**API (`prog-strength-api`)**

- **Add a compact estimated-1RM trend to each `/personal-records` row.** Extend
  `personalRecordDTO` (`internal/workout/handler.go`) with a nullable, downsampled
  series — e.g. `recent_estimated_1rm_points: []float64` (oldest→newest values,
  rounded like the existing fields), **capped at a small N** (~8–12 points) and
  `null`/empty when the lift has no history. Built from the `entries` the
  `personalRecords` loop **already fetches** per slug for the baseline — **no new
  query, no new endpoint, no extra DB load.** Decide the window + cap + downsample
  rule (Open Question #1).
- **Keep `current_estimated_1rm` and the existing per-exercise history endpoint
  unchanged** — `GET /personal-records/{exercise_id}/history` still backs the
  expand chart; the new field is purely additive to the list row.
- **Test** the handler emits the trend for a lift with history (right length/cap,
  ascending), `null`/empty for a never-trained lift, and that the existing fields
  are unaffected. Go tests + the api CI pipeline green; ships via the api repo's
  semantic-release.

**Web (`prog-strength-web`)**

- **Extend the `PersonalRecord` type** in `lib/api.ts` with the new
  `recent_estimated_1rm_points` field (nullable), so the web degrades gracefully if
  the field is absent (older API) — the spark simply doesn't render.
- **Add a pure, tested readiness module** (e.g. `_components/readiness.ts`) over
  `PersonalRecord`, returning `{ hasPR, isDumbbell, gap, gapPct, ready, daysSince }`:
  `hasPR = weight !== null && workout_id !== null`; `gap = current_estimated_1rm −
  weight`; `gapPct = gap / weight × 100`; `ready = gapPct ≥ 5` (the shipped
  threshold, **kept** as the "due" definition for the summary count, no longer the
  page's only signal — magnitude drives the dot); `daysSince` from `achieved_at`
  against a real `new Date()`; `isDumbbell` from the exercise name (the only
  available signal; see Open Question #3). Replaces the duplicated inline gap math.
- **Rebuild `LiftsView` into the compact-dashboard tile grid**, conforming to v0.4:
  - **Tile grid** — `grid-cols-2 sm:grid-cols-3 lg:grid-cols-4`, tight `gap-2`,
    hairline `--surface` tiles, small functional Manrope with tabular figures;
    renders in **API order** (the user controls the set + order via Customize —
    compact-dashboard does not re-rank). Composes at **3 and 12** lifts.
  - **Tile content** — exercise name (truncates, `title` tooltip); the **PR figure**
    (`weight` big, `unit × reps` small); a compact **est-1RM delta** (`+X` in
    `--success`, `est <n>` faint; a calm `·` when fresh, `gapPct < 5`); the
    **readiness dot** whose `--warning` intensity **scales with `gapPct`** (capped
    ~30%), going **steel `--discipline-lift-dot`** when fresh; the **recency** line
    (`fmtAgo`); and the **trend spark** rendered from `recent_estimated_1rm_points`
    (warm when not-fresh, calm steel when fresh; omitted when the series is
    empty/null).
  - **Color discipline (in-system)** — neutral by default; `--warning` only on the
    scaling readiness signal; `--success` on the positive delta; steel-blue
    `--discipline-lift-*` available but **never** as selection/active (the periwinkle
    accent's job); periwinkle stays app-chrome.
- **Give the never-PR'd lift a deliberate home** — a **dashed, transparent
  "untested" tile** (`—`, `untested`), lighter than a real PR tile, no dot, delta,
  spark, chart, or link.
- **Preserve the progression chart and workout link** — a tile click expands the
  existing lazy `ProgressionChart` (`kind="lifts"`, `exerciseId`) with **`View
  workout →`** inside the expanded panel; suppressed for untested lifts (Open
  Question #2).
- **Add the chrome summary strip** — `N due · N/total tested`, derived via
  `readiness.ts`; keep the real **`ViewSwitcher`**, the **intro sentence**, and the
  **Customize** button + `HeadlineExercisesModal` (Lifts-only) with its `onSaved`
  refetch.
- **Define loading / error states that fit the grid** — a **skeleton tile grid**
  for `isPending`; the existing inline danger panel for `error`; keep the empty
  `records.length === 0` → "No headline lifts configured." message.
- **Handle every state the DX enumerated**: ready-by-degree (a +7% and a +33% lift
  read as different dot/spark intensities, not the same badge); **at/near max**
  (fresh reads as *fine*); **never-PR'd** (dashed tile); **high-rep vs low-rep** PR
  (reps legible as estimate-trust context); **long names** (designed truncation);
  **3 vs 12** lifts; **lb and kg**; **loading/error**.
- **Surface dumbbell PRs honestly** — `weight` is **per dumbbell**, not the pair;
  the tile must not imply a combined load (Open Question #3).
- **Tests** — the readiness module (gap/gapPct/ready/daysSince, the never-PR'd null
  path, dumbbell detection, kg); plus component tests that the grid renders tiles
  incl. the dashed untested tile, the dot/spark reflect `gapPct`/the series, the
  spark is omitted when the series is empty, the summary count is right, **switching
  to Running still works**, Customize opens and refetches, and a tile expand mounts
  exactly one `ProgressionChart` query (the `page.test.tsx` invariant). **CI green.**

### Non-Goals

- **Any design-system or shared-token change.** `scope: in-system` against v0.4 —
  conform only; no token/accent/type edit, no `design-system.md` change. (Contrast
  `sows/activities-page-redesign.md`, a greenfield pick adopted app-wide.)
- **A new history endpoint or per-tile history fetch.** The trend is **embedded in
  the existing list response** from already-fetched entries; the per-exercise
  `GET …/history` endpoint is reused unchanged for the expand chart. **No N+1.**
- **Changing the estimated-1RM math, the recency-weighted baseline, the
  headline-lift curation, or the `readyForAttempt` 5% rule** — the trend is a
  downsample of the existing history series; the derivation is unchanged.
- **MCP or SDK changes.** The web calls the Go API directly (`lib/api.ts` `fetch`);
  the MCP does not surface PR sparks. Neither repo is in `repos:`.
- **Redesigning the Running view.** Only the **page-level shell** Lifts shares with
  Running (header, switcher, intro, Customize) is in scope; `RunningView` /
  `RunningPRCard` internals are untouched.
- **Re-ranking the lifts.** compact-dashboard keeps **API order**; ordering-by-
  readiness is the *leaderboard*/*momentum* idiom, not this one.
- **Promoting the DX mockup code.** `CompactDashboard.tsx` / `_fixtures.ts` are the
  visual spec, not code to copy; the `design-explore/personal-records-lifts` route
  stays gated and never ships. The other four variants are not built; the DX PR is
  **closed, never merged**.

## Implementation Details

### API (`prog-strength-api`)

In `internal/workout/handler.go`, the `personalRecords` loop already calls
`h.repo.ListOneRepMaxHistory(ctx, userID, slug, &since, &until)` per slug and walks
`entries` to compute `RecencyWeightedBaseline`. Reuse those `entries`: downsample
their `MaxEstimated1RM` values into a compact ascending series (the per-exercise
history endpoint already emits ascending), cap at N, round via the existing
`round1`, and attach to a new nullable `personalRecordDTO` field
(`recent_estimated_1rm_points`). Emit `null`/empty when there are no in-window
entries. No repository, query, or migration change. Mirror the existing handler
tests (`internal/workout`) for the new field across the has-history /
never-trained / cap cases. Note: if the spark wants a **longer** window than the
baseline window (Open Question #1), widen only the series' downsample source, not
the baseline computation.

### Web — readiness derivation

A new pure module (`_components/readiness.ts`) — production's replacement for the
mockup's `deriveReadiness` and the single home for the gap math currently inlined in
`LiftPRCard`. Pure, React-free, `new Date()`-based; reuses `format.ts`. Tile
rendering and the chrome summary both consume it, so "due" is defined once.

### Web — Lifts view rebuild

Replace the three-column `LiftPRCard` grid in `LiftsView.tsx` with the dense tile
grid. Build a `Tile` (real PR) and the dashed untested tile mirroring the mockup's
structure but wired to the derivation and the real spark series; add the
skeleton-grid loading state. Keep `LiftsView`'s `records / isPending / error`
contract from `page.tsx`. Reuse `ExpandChevron` + `ProgressionChart` for the
expanded panel; `format.ts` stays the shared formatter. `LiftPRCard.tsx`'s inline
gap logic is removed in favor of `readiness.ts`. Extend the `PersonalRecord` type in
`lib/api.ts` with the new nullable field; the spark renders only when present.

### Web — page chrome (`page.tsx`)

Keep `?view=` routing, the lazy queries, `ViewSwitcher`, intro copy, and the
Customize modal wiring. Add the `N due · N/total tested` summary (from
`liftsQuery.data` via `readiness.ts`) into the Lifts header; hide on Running. No
change to the Running branch.

## Rollout

The new field is **additive and nullable**, so the two code repos can ship
independently and in either order — the web degrades to no-spark until the API field
is live.

1. **`prog-strength-api`** — add `recent_estimated_1rm_points` to the list response
   + handler tests; ship via the api semantic-release pipeline.
2. **`prog-strength-web`** — the readiness module + Lifts-view rebuild + chrome
   summary + tests, in one PR; Vercel preview across 3 / 8 / 12 lifts and
   loading/error. Sparks light up once the API field is deployed.
3. **`prog-strength-docs`** — flip `dx/personal-records-lifts.md` to `status:
   selected` (winning idiom `compact-dashboard`); mark this SOW `shipped`.

### Verification after rollout

- `GET /personal-records` returns `recent_estimated_1rm_points` per trained lift
  (capped, ascending, ending near `current_estimated_1rm`), and `null`/empty for the
  never-trained lift, with all existing fields intact.
- `/personal-records` (Lifts) reads as a **dense overview**: every headline lift on
  one screen, the **readiness dot + spark scaling by degree** so a +7% and a +33%
  lift look *different* — not the same badge everywhere.
- The **never-PR'd lift** is a quiet dashed "untested" tile (no spark); **freshly-
  tested** lifts read as fine (calm `·`).
- A tile **expands its progression chart** and exposes **`View workout →`**;
  expanding fires exactly one history query per lift; the **sparks cost no extra
  requests** (Network shows one `/personal-records` call).
- The **`N due · N/total tested`** summary, the **Lifts/Running switcher**, the
  **intro**, and **Customize** (opens, save refetches) all work; **Running** shows
  the untouched view.
- Holds at **3 and 12** lifts, **long truncating names**, **lb and kg**, and
  **loading (skeleton) / error (panel)** states; **dumbbell** PRs don't imply a
  combined load.
- `design-system.md` unchanged; the `design-explore/personal-records-lifts` route
  stays gated / 404 in production; **no DX mockup code shipped**; the DX PR is closed
  (never merged).

## Open Questions

1. **Spark window, cap, and shape.** The list loop currently fetches entries within
   `DefaultBaselineWindow` (for the baseline). The spark wants enough points to read
   as a trend. **Lean:** emit up to **~10 points**, **downsampled** (even stride) so
   long-history lifts don't bloat the row, as a **bare `[]float64`** of values
   (oldest→newest — a spark needs no timestamps; the expand chart has them). Decide
   whether the spark window reuses the baseline window or widens to "all history,
   downsampled" — **lean widen** so a lift with a long progression shows its arc, not
   just the recent slice. Confirm the cap + window at review.
2. **Where the progression chart + `View workout` live in a dense tile.** The mockup
   dropped both. **Lean:** tile is a click target that **expands the existing lazy
   `ProgressionChart` inline**, with `View workout →` inside the expanded panel —
   reuses today's lazy-fetch and the one-query invariant. Alternative: tile links
   straight to the workout and the chart is dropped here. Confirm expand vs.
   link-only.
3. **Per-dumbbell labeling.** `weight` is per dumbbell; the mockup shows a bare `105
   lb × 8`, and `isDumbbell` is a **name regex** (no equipment field). **Lean:** when
   `isDumbbell`, append a quiet `ea`/`/ea` marker so the tile never implies a combined
   load, accepting the name-regex heuristic as the only available signal. Confirm the
   marker and whether the heuristic is acceptable.
4. **Untested tiles: in place vs. grouped last.** compact-dashboard renders in API
   order. **Lean:** keep API order (honors Customize intent, matches the chosen
   variant); don't sink untested tiles to the end (that's the leaderboard idiom).
   Flag in case grouping reads better at 12 lifts.
