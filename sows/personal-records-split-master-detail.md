---
status: ready_for_implementation
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Personal Records — Split Master-Detail Redesign

**Status**: Ready for implementation · **Last updated**: 2026-06-20

## Introduction

Personal Records is the "trophy case" — the page a Prog Strength user opens to see how strong and how fast they are right now. Today it is two unrelated surfaces wearing one tab bar: a dense compact-tile **Lifts** grid and a roomier card **Running** grid, each with its own card design, its own expand-on-click chart, and its own visual language. The page's most motivating idea — the gap between what you've logged and what you're capable of *right now* — is compressed into a tiny "+22" delta and a fingernail-sized sparkline, and the running max-effort estimate (a forward projection with a confidence band) is hidden a click away on a separate detail page. A wide band of space below the grid does nothing.

A Design Exploration ([`dx/personal-records.md`](../dx/personal-records.md)) put five directions in front of the owner. The selected direction is **`split-master-detail`**: a two-pane ledger that unifies both tabs into one layout. A narrow left column lists every record **ranked by readiness** (most-due first, so "what should I attempt next" is the top row), and a wide right pane holds a **featured estimation chart pinned in view**, updating as you select a row. There is no expand-in-place — the detail is always present. The featured chart frames each PR as progress toward a new best effort: for a lift, the estimated-1RM trend climbing over a dashed logged-PR reference line; for a run, the max-effort estimate and its confidence band descending toward the logged best.

This SOW implements that variant production-quality against real data. It is **web-only**: every data source it needs already exists (`GET /personal-records`, `GET /personal-records/{exercise_id}/history`, the running best-efforts list, and the running max-effort estimate), and every design token it uses already lives in `globals.css`. No API, infrastructure, or migration work. A true forward-projection 1RM model for lifts — the lift analogue of running's Bayesian estimator — remains explicitly out of scope and is a separate follow-up SOW; this redesign uses the existing retrospective est-1RM trend, framed against the logged-PR line.

## Proposed Solution

The `/personal-records` page is restructured from "render one of two grids" into "render one shared ledger + detail." The page keeps its current shell — the `Personal Records` title, the URL-backed `ViewSwitcher` (`?view=lifts|running`), the Lifts-only `Customize` button, and the `N due · N/total tested` readiness summary. Below that shell, both views now render the same component: a `PRLedger` (master) beside a `PRDetail` (detail), in a fixed two-column split with a strong vertical divider.

The **master** is a ranked, scannable list. Lift rows are ordered by readiness (largest est-vs-PR gap first), running rows by "has a projected PR" then covered-distance then uncovered. Each row is a single compact line: a discipline-colored status dot (steel-blue lift / green-teal run by default, `--warning` sand when the record is "due" / a PR is in reach, `--faint` when untested/uncovered), the record name, a secondary stat (logged PR weight / pace), and a right-aligned figure (current est-1RM / best time) with a `+gap` or "PR near" cue. The selected row carries the periwinkle `--accent` left-border + `--accent-soft` background — the one place the accent appears, as selection chrome. Untested lifts and uncovered distances render as dimmed, non-selectable rows.

The **detail** pane is pinned and always shows the selected record's featured estimation chart. For a lift it plots the estimated-1RM history as a line with a soft area fill, a dashed horizontal reference at the logged-PR weight, and the latest point emphasized in `--warning` — with a heading that states the gap ("+22 over PR — go for a max"). For a run it plots the max-effort estimate-over-time with its confidence band, a dashed reference at the logged best, the recent attempts as chips, and the confidence/band summary. Both reuse the project's existing Recharts-based chart code (the `ProgressionChart` / `EstimateChart` already render these series) extended with the dashed reference line the variant introduced; we do not hand-roll SVG as the disposable mockup did.

Selection is local component state, defaulting to the most-due lift and to `5k` (or the first covered distance) on Running. Switching tabs preserves each tab's selection. On viewports below `md`, the split collapses to a single column — the ledger stacked above the detail — matching the mockup's responsive behavior.

The redesign **retires** the now-unused per-card components (`LiftsView`, the PR-page `RunningView`, `RunningPRCard`, `ExpandChevron`, `Sparkline`) whose only consumers are this page, and **reuses** the shared logic (`readiness.ts`, `format.ts`, `lib/format`, `useDistanceUnit`, `ViewSwitcher`, `HeadlineExercisesModal`) and the four existing API client functions. Data fetching keeps the current lazy-per-view pattern for the ledger lists and adds a lazy-per-selection query for the detail (the selected lift's 1RM history; the selected distance's max-effort estimate), cached for the page lifetime so re-selecting a row never refetches.

## Goals and Non-Goals

### Goals

- Restructure `app/(app)/personal-records/page.tsx` to render a single shared **ledger + detail** layout for both tabs, replacing the `view === "lifts" ? <LiftsView/> : <RunningView/>` branch. Preserve the existing header chrome: title, `ViewSwitcher` (`?view=` URL state), Lifts-only `Customize` button + `HeadlineExercisesModal`, and the `summarizeReadiness` "N due · N/total tested" line.
- New `_components/PRLedger.tsx` — the master list. Renders a ranked `<ul>` of rows for the active tab, a `Most due first` / `Best efforts` section label, and the empty/loading/error states. Rows are `<button>`s with `aria-current`/`aria-selected`, keyboard-navigable; untested-lift / uncovered-distance rows are dimmed and non-selectable.
- New `_components/LiftLedgerRow.tsx` and `RunLedgerRow.tsx` (or one row component parameterized by tab) per the variant: status dot (discipline color / `--warning` when due / `--faint` when untested), name, secondary stat (PR weight / pace), right-aligned figure (current est-1RM / best time) + `+gap` / "PR near" cue.
- New `_components/PRDetail.tsx` dispatching to `LiftDetail` / `RunDetail`:
  - **LiftDetail** — heading "Estimated 1RM · progress toward a new max", the exercise name, logged-PR subline (weight · reps with the `ea` dumbbell modifier · date), the large current est-1RM figure, the `+N over PR — go for a max` cue when due, and the featured chart. Empty state when the lift has no PR ("Log a heavy set on this lift to start its estimate.").
  - **RunDetail** — heading "Max-effort estimate · progress toward a new best", the distance label, the confidence + band + logged-best subline, the large estimate figure, the "under best — PR in reach" cue, the featured chart with confidence band, and the recent attempts as chips. Insufficient-data empty state for distances without a projection (reuse the running detail page's `insufficient` logic: `estimate === null || basis === "insufficient_data"`).
- Add a shared **featured estimation chart** that both detail panes use, built on the existing Recharts chart code (extend `ProgressionChart` or extract a `FeaturedEstimateChart` from it + `EstimateChart`). It must support: (a) a line + soft area fill of the metric series; (b) a **dashed horizontal reference line** labeled "logged PR {weight}" (lifts) or "logged best {time}" (running); (c) an optional confidence band (running); (d) the latest point emphasized in `--warning`; (e) "up is better" (lifts, est-1RM) vs "down is faster" (running, time) y-axis semantics with the existing `m:ss` tick formatting and the "lower is faster" note on running.
- Readiness ranking helpers in `readiness.ts`: `liftsByReadiness(records)` (descending by gap; untested last) and a running rank (`5k`/projected-PR first, then covered by nothing-special order, then uncovered last). Unit-tested; the ledger consumes these so "most due first" is defined once.
- Data wiring (all existing endpoints — no API changes):
  - Ledger lists: keep the lazy-per-view `useQuery`s — `listPersonalRecords` (lifts) and `listRunningBestEfforts` (running).
  - Lift detail: lazy `getExerciseOneRMHistory(exerciseId)` keyed by the selected exercise, for the dated est-1RM trend; the dashed reference uses the row's logged `weight`.
  - Running detail: lazy `getRunningMaxEffort(distanceKey)` keyed by the selected distance, for the estimate, `estimate_history`, `attempts`, and actual-best.
  - Default selection: most-due lift; `5k` or first covered distance. Selection is local state, preserved per tab across tab switches.
- Responsive: a `md:grid-cols-[260px_1fr]` split that stacks to a single column (ledger above detail) below `md`, matching the variant.
- Use only existing design-system tokens (verified present in `globals.css`): `--discipline-lift-dot`, `--discipline-run-dot`, `--discipline-run-bg`, `--warning`, `--success`, `--accent` / `--accent-soft` / `--accent-fg`, `--surface` / `--surface-2`, `--border`, `--muted` / `--faint`. The periwinkle `--accent` appears **only** as selection chrome (active tab, selected row) — never as an activity or "due" color.
- Retire the now-unused PR-page components — `LiftsView`, the personal-records `RunningView`, `RunningPRCard`, `ExpandChevron`, `Sparkline` — and their tests, after confirming (done in this SOW's research) their only consumers are this page. The activities/dashboard `running-view` is a *different* component and is untouched.
- Tests — see § Tests.

### Non-Goals

- **A forward-projection 1RM model for lifts.** The lift detail uses the existing retrospective est-1RM trend (`exercise_one_rep_max_history`) against the logged-PR line. A Bayesian/parametric *projected* 1RM with a confidence band — the lift analogue of the running estimator — is a separate follow-up SOW. The featured chart is built so a forward band can later overlay without rework.
- **Any API, database, or infrastructure change.** Every endpoint and token already exists. If implementation surfaces a genuinely missing field, raise it rather than silently adding API scope.
- **Mobile app.** `prog-strength-mobile` is out of scope; this is the `prog-strength-web` surface only.
- **The other four DX idioms.** `compact-dashboard-rail`, `editorial-spotlight`, `progression-timeline`, and `gauge-grid` are not built; the DX selection chose `split-master-detail`.
- **Changing the running max-effort detail page** (`/progress/running/[distanceKey]`). The running detail pane links to it for the full estimate/attempts view; that page is unchanged.
- **Deep-linking the selected row.** v1 keeps `?view=` only; the selected record is local state (see Open Questions).
- **New readiness semantics.** The 5% "due" threshold and the gap math in `readiness.ts` are reused as-is; only the ranking helpers are added.

## Implementation Details

### Page structure

`page.tsx` keeps its header and its two lazy `useQuery`s (one per view), and replaces the body branch with:

```tsx
<div className="grid grid-cols-1 md:grid-cols-[260px_1fr] ...">
  <PRLedger
    view={view}
    records={view === "lifts" ? liftsQuery.data : runningQuery.data}
    state={/* pending | error | ready */}
    selectedId={view === "lifts" ? selLift : selRun}
    onSelect={/* set per-view selection */}
  />
  <PRDetail view={view} selectedId={...} />
</div>
```

Selection is two pieces of local state (`selLift`, `selRun`) initialized from the ranked order once data arrives (most-due lift; `5k`/first covered distance). Keeping them separate preserves each tab's selection across `ViewSwitcher` toggles. The `Customize` flow and readiness summary are unchanged.

### Master — `PRLedger` + rows

The ledger maps the ranked records to rows. Ranking comes from `readiness.ts`:

- `liftsByReadiness(records)` — `deriveReadiness` each, sort by `gap` descending (most over PR first); untested (`!hasPR`) sink to the bottom in stable name order.
- `runningByReadiness(records)` — rank `5k`/has-projection = 0, covered (`duration_seconds !== null`) = 1, uncovered = 2; stable within a rank.

A row shows: status dot (`--faint` untested → `--warning` due → discipline dot otherwise), name, secondary (PR weight / pace), right figure (est-1RM / best time), and the `+gap` (lifts) or "PR near" (running 5k) cue. Selected → `border-[var(--accent)] bg-[var(--accent-soft)]`. Non-selectable rows (`!hasPR` / uncovered) are `opacity-60` and ignore clicks. Rows are real buttons with `aria-current="true"` when selected and an accessible label naming the record and its figure.

### Detail — `PRDetail`, `LiftDetail`, `RunDetail`

`PRDetail` switches on `view` and the selected id, fetching the detail series lazily:

- **Lift**: `useQuery(["pr-history","lifts",exerciseId], () => getExerciseOneRMHistory(token, exerciseId))`. The featured chart plots `points[].estimated_1rm` over `performed_at`, with a dashed reference at the selected row's logged `weight`. Heading + figure from the `personalRecordDTO` row already in hand (no second fetch for the headline numbers).
- **Run**: `useQuery(["running-max-effort", distanceKey], () => getRunningMaxEffort(token, distanceKey))`. Render the `insufficient` empty state when `estimate === null || basis === "insufficient_data"`; otherwise the featured chart plots `estimate_history[].seconds` with the `lower/upper` band, a dashed reference at the actual-best seconds, and `attempts` as chips. A "View full estimate →" link to `/progress/running/{distanceKey}` carries the user to the existing detail page for the complete attempts table.

Both queries are lazy (fire on first selection) and cached for the page lifetime, so clicking between rows re-reads cache. This mirrors the existing `ProgressionChart` fetch pattern and keeps `page.test.tsx`'s "one query per surface, detail lazy" shape.

### Featured chart

Extract the shared chart from today's `ProgressionChart` (lifts est-1RM over time) and `EstimateChart` (running band) into one `FeaturedEstimateChart` taking: `series` (dated points), `referenceValue` + `referenceLabel` (the dashed PR/best line — **new**, the one capability the variant adds over the shipped charts), optional `band` (`lower[]`/`upper[]`), `direction` (`up` lifts / `down` running), and a `formatY`. It keeps the existing Recharts axes, `m:ss` tick formatting, the "lower is faster" note for running, the single-point and loading/error states, and the design-system stroke/grid colors. The latest point is emphasized in `--warning` to tie the chart to the "you're due" cue. Hand-rolled SVG from the disposable mockup is **not** carried over.

### Tests

**Web (Vitest + Testing Library):**

- `readiness.test.ts` — `liftsByReadiness` orders by descending gap and sinks untested lifts last; `runningByReadiness` orders 5k/projection → covered → uncovered. Pure functions, deterministic `now`.
- `PRLedger.test.tsx` — renders both tabs ranked; the top row is the most-due record; untested/uncovered rows are present, dimmed, and non-selectable; selecting a row fires `onSelect` and marks `aria-current`.
- `PRDetail.test.tsx` — lift detail shows the est-1RM figure, the "+N over PR" cue when due, and the empty state for an untested lift; run detail shows the estimate + band + attempts for a projected distance and the insufficient-data state otherwise.
- `FeaturedEstimateChart.test.tsx` — renders the dashed reference line + label; running variant renders the band; single-point and loading states hold.
- `page.test.tsx` — update for the new layout: still one ledger query per view (lazy), detail query fires only on selection; default selection is the most-due lift / `5k`; tab switch preserves each tab's selection.
- Remove the tests for retired components (`LiftsView.test.tsx`, `RunningPRCard.test.tsx`, and any `Sparkline`/`ExpandChevron`/PR-page-`RunningView` tests) as those components are deleted.
- PR description includes a manual smoke checklist: dense account (8 lifts incl. an untested Deadlift; 4 covered + 2 uncovered distances), sparse account (1–2 PRs, 1 distance), tab switching preserves selection, mobile single-column stack, and keyboard navigation of the ledger.

### Rollout

Single `prog-strength-web` PR (plus this docs SOW). No API, migration, or infra; no cross-repo coordination.

1. Build the ledger + detail components and the shared `FeaturedEstimateChart`; restructure `page.tsx`; add the ranking helpers + tests; delete the retired components and their tests.
2. Deploy via the existing release workflow. The change is contained to `/personal-records`; the Vercel preview renders the full redesign for a final visual pass against the chosen DX variant before merge.
3. Revertable as a single PR. No data migration means revert is purely a UI rollback.

## Open Questions

1. **Deep-linking the selected row.** v1 keeps selection in local state (default most-due). Options: (a) leave it local; (b) add `?lift=`/`?dist=` so a selection is shareable/reloadable. Tentative lean: (a) — the default-to-most-due behavior is the point of the page, and URL-encoding selection adds history-stack noise for little gain. Cheap to add later if users want to link a specific record.
2. **Mobile interaction.** Below `md` the split stacks (ledger above detail), so selecting a row updates a detail pane further down the page. Options: (a) stacked as the mockup shows; (b) a drill-in (tap row → full-screen detail with a back affordance). Tentative lean: (a) for v1; revisit if the scroll-to-detail feels disconnected on a phone.
3. **Lift detail series source.** The detail chart uses `getExerciseOneRMHistory` (dated, full) rather than the row's `recent_estimated_1rm_points` (10-point downsample). Confirm the history endpoint's points are the right granularity for the featured chart, or whether it wants a date window. Tentative lean: use the full history as-is — it's already one point per workout and bounded in practice; add a window only if a long-tenured lift's chart gets noisy.
4. **Attempts shown in the running detail.** The pane shows the recent attempt chips; the full attempts table stays on `/progress/running/{distanceKey}`. Confirm "recent few + link" is enough here, or whether the pinned detail should show more. Tentative lean: recent few + link — the ledger detail is a glanceable summary, not the full analysis surface.
