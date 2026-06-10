---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Progress Page Modernization

**Status**: Shipped · **Last updated**: 2026-06-10

## Introduction

The Progress page was the first feature Prog Strength shipped. It groups exercises by muscle, normalizes each per-exercise estimated 1RM against that exercise's current baseline so disparate lifts share a common Y-axis, plots the result, and offers a per-workout detail table beneath the chart. The underlying model — "is the lifter getting stronger, plateaued, or regressing?" — is sound and worth preserving. The page's current rendering of the answer is not.

Three problems compound to make the chart misleading. **First**, a single least-squares regression is fit across the combined scatter of every normalized point regardless of which exercise it came from, so a quarter where the lifter shifts from heavy bench to assistance work for shoulder reasons reads as "−17.5%" in the headline — strength didn't drop, the exercise mix did. **Second**, the headline number is rendered in a large fixed-tone red whether the regression is real or noise, which trains the lifter to ignore it. **Third**, the page's central concept — the baseline that "current capability" is measured against — is computed by an exponentially-weighted average that's actually robust and stable across the date-range filter, but the page never says any of that, so users perceive the % values as ambiguous and shifting. Add to that an overcrowded filter row (twelve muscle-group chips) and a per-workout table whose "% baseline" column is mathematically identical to the chart's Y-value but isn't visibly tied to it, and the answer-at-a-glance loop the page is supposed to provide breaks down.

This SOW reworks the page around its actual job — *am I getting stronger?* — in three layers. First, **correctness fixes**: per-exercise trend lines instead of one cross-exercise regression, defensible aggregate stat cards built on top of per-exercise slopes, a minimum-sessions threshold so a "trend through two points" never ships, and explicit labeling of the recency-weighted baseline wherever a % appears. Second, **filter consolidation**: twelve muscle chips collapse to five movement-pattern chips (Push, Pull, Legs, Core, All), with the rollup done in the API so chart math (slopes, baselines, aggregates) stays on the server. Third, a **chart rework**: per-exercise connected lines with their own direction indicators replace the single cloud-of-dots scatter, and the per-workout table's existing column is rebranded so the chart and table visibly speak the same language. The existing dark aesthetic, typography, and layout band are preserved; running data and any cross-domain context are explicitly out of scope. One question, answered cleanly: strength, up or down?

## Proposed Solution

The page keeps its three vertical bands — filter header, stat cards + chart, per-workout table — and its core normalization model (each exercise indexed to its own recency-weighted baseline). Everything else around the math is reworked.

**API rework.** The existing `GET /workouts/progression` endpoint is extended along three axes. A new optional `movement_pattern` query parameter (`push` | `pull` | `legs` | `core` | `all`) is added beside the existing `muscle_group` param; the handler resolves the pattern to its constituent muscle groups server-side and runs the same per-exercise query path. Exactly one of `movement_pattern` or `muscle_group` is required — passing both, or neither, returns 400. The response gains a `per_exercise_trends` array carrying one entry per exercise that contributed to the chart: session count in range, slope as %/month relative to baseline, and the regression's start/end coordinates. The combined cross-exercise `trendline` field at the top level is dropped — fitting one line across exercises with different baselines was the load-bearing source of the misleading headline. A new `aggregate` block carries `lifts_tracked`, `lifts_progressing`, `median_slope_per_month`, and the `min_sessions_threshold` used (3) so the frontend doesn't hard-code the cutoff. Exercises with fewer than 3 sessions in range still appear in `points` and `exercise_baselines`, but their `per_exercise_trends` entry has `slope_per_month: null` and `trendline: null` — the renderer treats those as "not enough data" rather than fitting a line through two points. The recency-weighted baseline math is unchanged; what changes is that the response now carries a `baseline_model` discriminator (`recency_weighted_current`) the UI can label, so users see what "100%" means without having to read the code.

**Movement-pattern rollup.** Push = chest + shoulders + triceps; Pull = back + biceps + forearms; Legs = quads + hamstrings + glutes + calves; Core = core; All = every muscle in the catalog. The mapping is defined in one place in the backend (`internal/exercise/movement_pattern.go`) and exposed in the `aggregate` block as `muscle_groups_included` so the UI can render an unobtrusive caption ("Showing chest, shoulders, triceps"). The per-exercise data path is unchanged — the page still sees the same exercises with the same baselines as if the lifter had clicked each muscle in turn. The `muscle_group` parameter is preserved unchanged for any future drill-down (clicking a chip to focus a single muscle within a pattern) and for the existing tests / agent flows.

**Web — filter row.** The twelve muscle-group chips collapse to five movement-pattern chips. The date-range chips (`30d` / `60d` / `90d`) are unchanged. The active chip uses the existing pill aesthetic (rounded-full, border + tinted background) but in a neutral accent tone rather than a per-muscle hue, since "Push" doesn't have an established color. URL state (`?pattern=push&days=90`) gives the page linkable views and survives reload.

**Web — stat cards.** Three cards aligned with the question the page answers:

1. **Lifts progressing** — "4 / 6" — exercises with a positive `slope_per_month` above a small noise threshold, over the count of exercises with ≥ 3 sessions in range. Tone: positive when ≥ 50%, negative when ≤ 25%, neutral otherwise.
2. **Strength trend** — median of per-exercise `slope_per_month` values, formatted as `+1.4%/mo` with an up/down/flat arrow and matching tone. Median (not mean) so one rocketing or tanking exercise can't capture the headline.
3. **Best session** — kept from today: the highest normalized point in range, with date and exercise. The most motivating absolute number on the page; preserved because removing it loses signal users actually liked.

The static red `-17.5%` headline goes away in this rework.

**Web — chart.** Each exercise renders as its own connected `<Line>` (not a `<Scatter>`), in its existing color, with dots on each data point. The chart drops the single dashed regression line that fit across every dot. The reference line at `y = 1.0` stays, labeled explicitly ("Current baseline — your recency-weighted capability"). Per-exercise legend entries gain a direction indicator (↑ / → / ↓) and a per-month slope readout when the exercise has ≥ 3 sessions; entries with fewer sessions read "Not enough data yet." Overplotting near the baseline — the current page's biggest visual problem — drops out naturally once each exercise is its own line rather than a cloud.

**Web — per-workout table.** The existing table and the "1RM estimates" / "Sets × Reps × Weight" toggle are unchanged in structure. The "% Baseline" column header is renamed "% of current capability" (matching the chart's reference-line label), and a one-line caption above the table reads "Same metric as the chart — each row's % is how that workout's heaviest set compared to your current `<Exercise>` baseline." The two surfaces already produce identical numbers (the column is `normalized_max`, which is exactly what the chart plots); the change is making the connection visible.

**Out of scope, called out so future SOWs can pick them up.** Running progression overlays, cross-domain context, training-load metrics, plateau auto-detection, custom movement-pattern definitions, expansion of a movement-pattern chip into its underlying muscles, and any new agent tooling. This SOW is strength-only and read-only.

## Goals and Non-Goals

### Goals

- Extend `GET /workouts/progression` with an optional `movement_pattern` query parameter (`push` | `pull` | `legs` | `core` | `all`). Exactly one of `movement_pattern` or `muscle_group` must be supplied; the existing `muscle_group` flow is preserved unchanged. Returns 400 with code `missing_filter` if neither, 400 with code `conflicting_filters` if both.
- Define the movement-pattern → muscle-group mapping in one Go location (`internal/exercise/movement_pattern.go`) as a typed enum. The mapping is referenced by the handler when resolving exercises and is echoed in the response's `aggregate.muscle_groups_included` field so the UI doesn't duplicate it.
- Add a `per_exercise_trends` array to the response: one entry per exercise that contributed at least one point, carrying `exercise_id`, `session_count`, `slope_per_month` (relative to that exercise's baseline, in % units; `null` when `session_count < 3`), and `trendline` (start/end coordinates evaluated at `since`/`until`; `null` when `session_count < 3` or the regression is degenerate).
- Drop the top-level `trendline` field. The single cross-exercise regression is removed entirely — it was the load-bearing source of the misleading "−17.5%" headline.
- Add an `aggregate` block to the response:
  - `lifts_tracked` — count of exercises with `session_count >= min_sessions_threshold`.
  - `lifts_progressing` — count of exercises with `slope_per_month > +0.25` (the noise floor; see § Algorithms).
  - `median_slope_per_month` — median of `slope_per_month` across exercises with `session_count >= min_sessions_threshold`. `null` when fewer than one exercise qualifies.
  - `min_sessions_threshold` — the constant 3, exposed so the UI's "Not enough data yet" copy isn't hard-coded.
  - `muscle_groups_included` — the resolved set of muscle groups behind the requested filter.
- Add a `baseline_model` field at the response root with the value `"recency_weighted_current"`, a discriminator the UI uses to label what the % values mean. The actual baseline math (`RecencyWeightedBaseline` evaluated at `now`) is unchanged.
- Per-exercise slope is computed by least-squares regression on the normalized points for that exercise, with the X-axis scaled to months elapsed (not raw milliseconds) so `slope_per_month` is the regression slope itself rather than a post-hoc unit conversion. Pure function; same `(since, until, now)` test-determinism contract as the existing endpoint.
- The minimum-sessions threshold is 3. Exercises below threshold still appear in `points` and `exercise_baselines` (so dots render on the chart) but have `null` slope and trendline and are excluded from the `aggregate` numerators and denominators.
- Web: replace the twelve muscle-group chips in `app/(app)/progress/page.tsx` with five movement-pattern chips: Push / Pull / Legs / Core / All. Active chip uses a neutral accent style (no per-muscle hue map). Date-range chips unchanged.
- Web: URL-back the active filter state via `?pattern=push&days=90`. Default on first load: `pattern=all&days=90`. Updating either chip uses `router.replace` (not `push`) so the back button doesn't fill with filter-toggle history.
- Web: replace the three existing stat tiles ("Trend over period", "Best session", "Exercises tracked") with:
  - **Lifts progressing** — `${lifts_progressing} / ${lifts_tracked}`. Tone positive when ratio ≥ 0.5, negative when ≤ 0.25, neutral otherwise. Label: "lifts progressing".
  - **Strength trend** — `+${median_slope_per_month}%/mo` with sign and directional arrow. Tone positive when > +0.5, negative when < −0.5, neutral otherwise. Label: "median per-exercise slope".
  - **Best session** — preserved from today: highest `normalized_max` in window with date and exercise.
- Web: rework the chart from one `<Scatter>` per exercise to one `<Line>` per exercise. Points stay visible (one dot per data point), connecting line is the same hue as the dots. Remove the top-level dashed trendline overlay. Keep the `<ReferenceLine y={1.0}>` with an updated label "Current baseline — recency-weighted capability."
- Web: per-exercise legend entries gain a direction indicator and a slope readout, both fed by `per_exercise_trends`:
  - `↑ +2.1%/mo` when slope > +0.5
  - `→ +0.2%/mo` when |slope| ≤ 0.5
  - `↓ −1.4%/mo` when slope < −0.5
  - `· not enough data` when `session_count < 3`
- Web: rename the "% Baseline" column in the estimates table to "% of current capability." Add a one-line caption above the table: "Same metric as the chart — each row's % is how that workout's heaviest set compared to your current `<Exercise>` baseline." The "1RM estimates" / "Sets × Reps × Weight" toggle and the per-exercise filter pills are unchanged.
- Web: preserve the dark aesthetic, the existing card border/radius/spacing, typography scale, and the existing `WorkoutLink`, `FilterPill`, `ViewToggle`, and `CustomTooltip` components. The rework reshapes the page, not the design system.
- Web: lazy-fetch on filter change via TanStack Query keyed `["progression", pattern, days]` and `["workouts", since, until]`. Cached across filter changes; staleTime 60 s so quick toggles don't refetch.
- Tests per § Tests: Go unit tests for per-exercise slope math, threshold behavior, aggregate computation, and movement-pattern resolution; web unit tests for filter URL state, stat-card tone selection, and legend direction-indicator rendering; web integration tests via `msw` for the page's data flow under both `pattern=push` and `pattern=all`.

### Non-Goals

- **Running data on the Progress page.** Explicitly excluded by the brief. Running PRs live on `/personal-records` (see `sows/running-best-efforts.md`). The Progress page stays strength-only this iteration.
- **Cross-domain context.** No overlay of lift trend vs. running mileage, no body-weight trend, no nutrition surface, no chat-driven annotations. The page answers one question and only one.
- **Training-load or volume metrics.** Sets × reps × weight in the table view stays as-is — a raw lookup, not a derived training-load number. RPE, tonnage, intensity histograms, etc. are out of scope.
- **Plateau auto-detection beyond the trend slope.** No "you've plateaued for 6 weeks" callouts beyond what the direction indicator (↑/→/↓) already conveys. Detection logic is one open follow-up.
- **Custom movement-pattern definitions.** Movement patterns are backend-fixed: Push, Pull, Legs, Core, All. A lifter who trains shoulders separately and wants a "Shoulders + Core" preset is out of scope.
- **Expanding a movement-pattern chip into its underlying muscles.** The brief flagged this as a power-user "v2" idea. The data is there server-side; the UI surface is a follow-up.
- **New agent tooling.** No MCP changes. The agent already reads progression via the existing endpoint and gets the new fields automatically once the API ships; that's enough.
- **Reframing the baseline as anything other than recency-weighted current capability.** The brief considered "first session in the selected range" and "all-time peak"; both were ruled out because the current recency-weighted model is already robust to outlier first sessions and stable across filter changes — the bug was that nothing on the page said so. Switching to a fragile-to-outliers or biased-low baseline would regress the math. See Open Question #1 if revisiting.
- **Adjustable minimum-sessions threshold.** The threshold is a backend constant (3). A "show all trends even with 2 sessions" toggle is not in v1.
- **Charting slope confidence intervals or R².** The slope readout is a point estimate; no uncertainty bars in v1. A follow-up if the agent wants confidence-aware reasoning.
- **Restyle of the existing design system.** Colors, spacing, typography, and component library are unchanged.

## Implementation Details

### API Surface

`GET /workouts/progression` evolves in place. Routing, auth middleware, and the surrounding handler structure are unchanged.

**Query parameters:**

- `movement_pattern` (optional) — one of `push`, `pull`, `legs`, `core`, `all`. New.
- `muscle_group` (optional) — unchanged enum (`chest`, `back`, etc.). Preserved for drill-down and existing callers.
- `since`, `until` — unchanged (RFC3339, defaults `until - 90d` … `now`).

Exactly one of `movement_pattern` or `muscle_group` must be supplied:

- Neither → 400, `{error: "movement_pattern or muscle_group is required", code: "missing_filter"}`.
- Both → 400, `{error: "specify either movement_pattern or muscle_group, not both", code: "conflicting_filters"}`.

**Response shape:**

```json
{
  "filter": {
    "movement_pattern": "push",
    "muscle_groups_included": ["chest", "shoulders", "triceps"]
  },
  "since": "2026-03-11T00:00:00Z",
  "until": "2026-06-09T00:00:00Z",
  "baseline_model": "recency_weighted_current",
  "exercise_baselines": [
    { "exercise_id": "barbell-bench-press", "exercise_name": "Barbell Bench Press", "baseline": 245.0, "unit": "lb" }
  ],
  "points": [
    { "workout_id": "wk_2a...", "exercise_id": "barbell-bench-press", "exercise_name": "Barbell Bench Press",
      "performed_at": "2026-04-12T17:30:00Z",
      "normalized_max": 0.927,
      "avg_estimated_1rm": 220.4, "max_estimated_1rm": 227.0, "min_estimated_1rm": 215.0,
      "set_count": 5, "unit": "lb" }
  ],
  "per_exercise_trends": [
    {
      "exercise_id": "barbell-bench-press",
      "session_count": 8,
      "slope_per_month": 2.1,
      "trendline": {
        "start_at": "2026-03-11T00:00:00Z",
        "start_value": 0.91,
        "end_at": "2026-06-09T00:00:00Z",
        "end_value": 0.99
      }
    },
    {
      "exercise_id": "overhead-press",
      "session_count": 2,
      "slope_per_month": null,
      "trendline": null
    }
  ],
  "aggregate": {
    "lifts_tracked": 5,
    "lifts_progressing": 3,
    "median_slope_per_month": 1.4,
    "min_sessions_threshold": 3
  }
}
```

When `muscle_group` is supplied instead of `movement_pattern`, the `filter` block reads:

```json
"filter": { "muscle_group": "chest", "muscle_groups_included": ["chest"] }
```

The top-level `trendline` field present in today's response is removed. The web app is the only consumer; the rework lands it on the new fields.

`baseline_model` is the discriminator the UI uses to label what "100%" means. Today's value: `"recency_weighted_current"`. If we ever switch the model (Open Question #1), this value changes and the UI's label changes with it — no extra coordination required.

### Algorithms

**Per-exercise slope (`slope_per_month`).** Least-squares regression on the normalized points for one exercise, with the X-axis scaled to months elapsed since `since`:

```
For each point p of the exercise's contributions:
  x_i = (p.performed_at - since).Hours() / (24 * 30.4375)   // months elapsed; 30.4375 = avg days/month
  y_i = p.normalized_max                                    // already a unit-normalized fraction

slope = (n * Σ x_i*y_i  -  Σ x_i * Σ y_i)  /  (n * Σ x_i^2  -  (Σ x_i)^2)
```

The slope is reported in **percentage points per month** (multiply the unitless regression slope by 100). A slope of `2.1` means the lifter's normalized capability on this exercise is increasing by 2.1 percentage points per month over the window — a barbell bench point that read 0.91 at the start of a 90-day window would read 0.97 at the end if the trend held. Pure function, no IO. Denominator-zero (all points share the same X) returns nil.

**Noise threshold for "progressing."** An exercise counts as progressing when `slope_per_month > +0.25`. The threshold is intentionally low — week-to-week scatter in estimated 1RM is several percent, but a per-month slope of 0.25 in a window of 8+ sessions reliably reflects real direction. The value is a backend constant (`progressingSlopeThreshold = 0.25`); same constant gates the stat-card tone logic so backend and frontend agree.

**Aggregate `median_slope_per_month`.** Median of `slope_per_month` across exercises with `session_count >= 3`. Median (not mean) so one exercise rocketing or tanking can't capture the headline. Returns `null` when zero exercises qualify. Computed inline alongside the per-exercise loop — no second pass.

**Minimum-sessions threshold.** `min_sessions_threshold = 3`. Exercises with `session_count < 3` are emitted in `points` and `exercise_baselines` so the chart shows the dots, but their `per_exercise_trends` entry carries `slope_per_month: null` and `trendline: null`, and they are excluded from both `lifts_tracked` and `lifts_progressing`. The threshold is exposed in the response so the UI's "Not enough data yet" copy isn't hard-coded.

**Baseline (unchanged).** `RecencyWeightedBaseline(entries, now, DefaultBaselineWindow, DefaultBaselineTau)`, as today. This produces a stable per-exercise number that captures "current capability" and is robust to outlier sessions (it's an exponentially-weighted average, not a single point). The SOW does not modify this math; what it adds is the `baseline_model` discriminator so the UI labels it.

**Movement-pattern resolution.** A new file `internal/exercise/movement_pattern.go` defines:

```go
type MovementPattern string

const (
    MovementPush  MovementPattern = "push"
    MovementPull  MovementPattern = "pull"
    MovementLegs  MovementPattern = "legs"
    MovementCore  MovementPattern = "core"
    MovementAll   MovementPattern = "all"
)

var MovementPatternMuscleGroups = map[MovementPattern][]MuscleGroup{
    MovementPush: {MuscleChest, MuscleShoulders, MuscleTriceps},
    MovementPull: {MuscleBack, MuscleBiceps, MuscleForearms},
    MovementLegs: {MuscleQuads, MuscleHamstrings, MuscleGlutes, MuscleCalves},
    MovementCore: {MuscleCore},
    MovementAll:  AllMuscleGroups(),
}
```

The handler resolves the parameter once and queries `exerciseRepo.List(ListOptions{MuscleGroups: groups})` — a small repository signature change from the current single-group `MuscleGroup` filter to a slice; the SQLite implementation uses `IN (...)` with a parameterized list.

### Data Model

No schema migrations. The endpoint is read-side only, and all the new fields are derived at request time from `exercise_one_rep_max_history` (the same source the existing endpoint uses). No new tables, no column additions, no index changes.

### Web — Page Redesign

Touches one route: `app/(app)/progress/page.tsx`. The redesign keeps the page's three-band layout (filter header → stat cards → chart → table) and the existing components that work well (`StatTile`, `FilterPill`, `ViewToggle`, `WorkoutLink`, `CustomTooltip`, `EstimatesTableBody`, `SetsTableBody`, `buildSetsRows`). The chart guts change; the surrounding scaffolding does not.

**File structure.**

```
app/(app)/progress/
  page.tsx                        # shell: URL state, filter header, query orchestration
  _components/
    MovementPatternFilter.tsx     # 5-chip Push/Pull/Legs/Core/All control
    StatCards.tsx                 # the three stat tiles + tone math
    ProgressChart.tsx             # per-exercise <Line>s + legend with direction indicators
    EstimatesTable.tsx            # the existing EstimatesTableBody, lifted out (and its column rename)
    SetsTable.tsx                 # the existing SetsTableBody, lifted out
    TablesSection.tsx             # the existing tab/filter wrapper around the two tables
```

The current `page.tsx` mixes shell, fetch, header, stat math, chart, and tables in ~1,000 lines. Extracting one component per render concern keeps each file at a size where the chart's per-exercise math, the stat-card tone logic, and the URL-state handling can be read in isolation. Matches the `_components/` pattern the running route already uses.

**URL state.** The page reads `?pattern=<p>&days=<d>` from `useSearchParams()`. Valid `pattern` values: `push`, `pull`, `legs`, `core`, `all`. Valid `days` values: `30`, `60`, `90`. Unknown / missing → `pattern=all`, `days=90`. Updates via `router.replace` so back-button history isn't cluttered.

**Data fetching.** TanStack Query, two queries:

- `["progression", pattern, days]` — `listProgression(token, { movement_pattern: pattern, since, until })`. `staleTime: 60_000`.
- `["workouts", since, until]` — unchanged from today, used by the Sets view. `staleTime: 60_000`.

Both queries trigger off the same `(pattern, days)` selection. The cache survives filter toggles, so flipping `60d` → `90d` → `60d` paints from cache the second time.

**`<MovementPatternFilter />`.** Five pills in one row, replacing today's two-row twelve-chip block. Each pill uses the existing `FilterPill`-style chrome but with a neutral accent color rather than the per-muscle hue map (there is no established color for "Push"). Active pill: accent border + accent-tinted background. Inactive: surface background + muted text. The date-range row beneath is unchanged.

A small caption under the chart band reads "Showing: Chest, Shoulders, Triceps" sourced from `aggregate.muscle_groups_included` so the user can see which underlying muscles a pattern resolves to without a second click.

**`<StatCards />`.** Three tiles in `grid-cols-1 sm:grid-cols-3`, mirroring today's layout. Tile internals use the existing `StatTile` component (no design changes there); only the inputs change:

```tsx
<StatTile
  value={`${aggregate.lifts_progressing} / ${aggregate.lifts_tracked}`}
  label="Lifts progressing"
  tone={progressTone(aggregate.lifts_progressing, aggregate.lifts_tracked)}
/>
<StatTile
  value={formatSlope(aggregate.median_slope_per_month)}
  label="Median per-exercise slope"
  tone={slopeTone(aggregate.median_slope_per_month)}
/>
<StatTile
  value={bestPoint ? `${formatPercent(bestPoint.normalized_max)} of baseline` : "—"}
  label={bestPoint ? `Best session • ${formatDate(bestPoint.performed_at)}` : "Best session"}
/>
```

Tone selectors:

- `progressTone(p, t)` → `positive` if `t > 0 && p / t >= 0.5`; `negative` if `t > 0 && p / t <= 0.25`; `neutral` otherwise (also neutral when `t == 0`).
- `slopeTone(s)` → `positive` if `s > 0.5`; `negative` if `s < -0.5`; `neutral` for `null` or `[-0.5, 0.5]`.
- The thresholds match the backend's `progressingSlopeThreshold` so a slope that the API says counts as "progressing" never reads as a neutral tone on the card.

`formatSlope(null)` → `"—"`. `formatSlope(s)` → `"+1.4%/mo"` (signed, one decimal).

**`<ProgressChart />`.** The chart guts change from `ComposedChart` of `<Scatter>` to `LineChart` of `<Line>`s, one per exercise. Approximate shape:

```tsx
<LineChart data={chartData} margin={...}>
  <CartesianGrid stroke="#27272a" strokeDasharray="3 3" />
  <XAxis dataKey="t" type="number" domain={["dataMin", "dataMax"]} tickFormatter={dateTick} ... />
  <YAxis type="number" domain={yDomain} tickFormatter={pctTick} ... />
  <Tooltip content={<CustomTooltip exerciseColors={exerciseColors} />} ... />

  <ReferenceLine
    y={1.0}
    stroke={COLOR_REFERENCE}
    strokeDasharray="2 4"
    label={{ value: "Current baseline — recency-weighted capability", ... }}
  />

  {exercises.map(ex => (
    <Line
      key={ex.exercise_id}
      data={pointsByExercise.get(ex.exercise_id)}
      dataKey="normalized_max"
      stroke={exerciseColors.get(ex.exercise_id)}
      strokeWidth={2}
      connectNulls={false}
      isAnimationActive={false}
      dot={{ r: 3 }}
    />
  ))}
</LineChart>
```

`chartData` is the union of per-exercise series keyed by `t` (milliseconds since epoch) for shared-axis scaling; Recharts handles the per-`<Line>` `data` overrides for the actual points. The single dashed `<Line>` overlay through every point (today's "trend") is dropped. The per-exercise direction lives in the legend, not the chart.

Below the chart, the legend gains a per-exercise direction indicator + slope readout:

```tsx
{exerciseBaselines.map(b => {
  const trend = trendsByExerciseId.get(b.exercise_id);
  return (
    <LegendSwatch
      key={b.exercise_id}
      color={exerciseColors.get(b.exercise_id)}
      label={`${b.exercise_name} · baseline ${formatNumber(b.baseline)} ${b.unit}`}
      direction={directionLabel(trend)}
    />
  );
})}
```

`directionLabel(trend)` returns:

- `{ arrow: "↑", text: "+2.1%/mo", tone: "positive" }` for `slope_per_month > 0.5`
- `{ arrow: "→", text: "+0.2%/mo", tone: "neutral" }` for `|slope| ≤ 0.5`
- `{ arrow: "↓", text: "−1.4%/mo", tone: "negative" }` for `slope_per_month < −0.5`
- `{ arrow: null, text: "not enough data", tone: "neutral" }` for `null` slope (session_count < 3 or degenerate regression)

Direction tone uses the existing `text-emerald-300` / `text-[var(--danger)]` palette; the muted "not enough data" case uses `text-[var(--muted)]`.

The single "Dashed line = trend" line at the bottom of today's legend is removed (there is no dashed trendline anymore).

**`<EstimatesTable />` and `<TablesSection />`.** Lifted out of today's `page.tsx` with one column-header change and one caption addition. The `"% Baseline"` header reads `"% of current capability"`. A new caption sits between the section header and the filter pills:

> Same metric as the chart — each row's % is how that workout's heaviest set compared to your current `<Exercise>` baseline.

`<Exercise>` is rendered as the current filter's first applicable exercise name when one is selected, or the literal word "exercise" when "All" is the filter. The row body, `WorkoutLink`, and `FilterPill` chips are unchanged.

`<SetsTable />` and `buildSetsRows` are lifted out unchanged — they have no exposure to baseline labeling and don't need to know about per-exercise trends.

**Empty / loading / error states.**

- **Loading** (no cached data, query in flight): existing "Loading progression…" placeholder.
- **Empty** (no points in range): existing `EmptyHint`, copy updated to reference the movement pattern instead of the muscle group: `"No push sessions in this window"`.
- **All exercises below threshold** (every `per_exercise_trends.slope_per_month` is null): chart still renders the dots, all legend entries read "not enough data," stat cards render `0 / 0` and `—` with neutral tone, and a banner above the chart reads "You have at least 3 sessions needed per exercise to fit a trend. Keep logging — direction shows up around session 3." Banner copy is one paragraph in `_components/ProgressChart.tsx`.
- **Error** (network / 401): existing error banner; 401 still bounces to `/login` via the current `clearToken` path.

**Accessibility.**

- Movement-pattern chips use `aria-pressed` matching today's muscle-group chips. The five pills sit in a `<div role="group" aria-label="Movement pattern">`.
- The stat cards use the existing `StatTile` markup; tone is conveyed by color, but the value text itself ("4 / 6", "+1.4%/mo ↑") carries the direction so a screen reader hears the signal independent of color.
- The chart legend's direction arrow is wrapped in an `aria-label` ("trending up", "trending down", "flat") so the visual glyph isn't the sole signal.

### Tests

**Go API (`internal/workout/`):**

- `muscle_group_progression_test.go` — extend existing tests with:
  - `TestComputeMuscleGroupProgression_PerExerciseSlope` — synthetic histories where two exercises trend in opposite directions; assert each exercise's `slope_per_month` matches the analytical regression to ±0.05; assert top-level `trendline` is absent.
  - `TestComputeMuscleGroupProgression_MinSessionsThreshold` — an exercise with 2 sessions in range emits points + baseline but `per_exercise_trends` entry has `null` slope + trendline and the exercise is excluded from `lifts_tracked`.
  - `TestComputeMuscleGroupProgression_Aggregate_MedianSlope` — five exercises with slopes [-1.0, 0.5, 1.0, 2.0, 3.0]; assert `median_slope_per_month == 1.0`.
  - `TestComputeMuscleGroupProgression_Aggregate_LiftsProgressing` — same five exercises; assert `lifts_progressing == 4` (every slope > +0.25 except the −1.0 one).
  - `TestComputeMuscleGroupProgression_TopLevelTrendlineRemoved` — assert no `trendline` field appears in the JSON-marshaled response.
- `movement_pattern_test.go` (new) — table-driven assertions that each pattern resolves to the expected muscle groups, that `MovementAll` includes every catalog muscle, and that `Valid()` rejects unknown strings.
- `handler_test.go` — extend with:
  - `TestProgressionHandler_MovementPattern_Push` — request `?movement_pattern=push` returns a response with `filter.movement_pattern == "push"` and `filter.muscle_groups_included == ["chest", "shoulders", "triceps"]`.
  - `TestProgressionHandler_MovementPattern_All` — `?movement_pattern=all` resolves to every catalog muscle.
  - `TestProgressionHandler_MissingFilter` — 400 with code `missing_filter`.
  - `TestProgressionHandler_ConflictingFilters` — both params supplied → 400 with code `conflicting_filters`.
  - `TestProgressionHandler_MuscleGroupLegacy` — existing `?muscle_group=chest` path returns `filter.muscle_group == "chest"` and `filter.muscle_groups_included == ["chest"]`. The original test cases for the single-muscle path keep passing without modification.

**Web (`prog-strength-web`):**

- `lib/api.test.ts` — extend `listProgression` tests for the new `movement_pattern` option and the new response shape (`per_exercise_trends`, `aggregate`, `baseline_model`, top-level `trendline` absent).
- `app/(app)/progress/_components/StatCards.test.tsx` — table-driven cases for `progressTone` and `slopeTone` boundary values; render the three cards under (a) a positive aggregate, (b) a negative aggregate, (c) an all-zeros aggregate, and assert tone classes.
- `app/(app)/progress/_components/ProgressChart.test.tsx` — render the chart with three exercises: one trending up (`+2.1%/mo`), one flat (`+0.1%/mo`), one with `null` slope (2 sessions). Assert each legend entry's arrow + slope readout matches `directionLabel`. Assert no dashed top-level trendline node is rendered.
- `app/(app)/progress/_components/MovementPatternFilter.test.tsx` — click Push; assert `router.replace("/progress?pattern=push&days=90")`. Use the keyboard arrow keys; assert focus traversal across the five chips.
- `app/(app)/progress/page.test.tsx` — integration via `msw`:
  - Default load lands at `?pattern=all&days=90`.
  - Flipping `60d` triggers a new `/workouts/progression?movement_pattern=all&since=...&until=...` request; the workouts query refires with the new window.
  - Stat tiles render with the response's aggregate values.
  - Empty-window response renders the empty-state hint with the movement-pattern copy.
- PR description checklist (manual smoke):
  - Toggle each of the five chips; verify the chart re-renders with the appropriate exercise mix.
  - Toggle between `30d` / `60d` / `90d`; verify URL updates and the chart re-renders.
  - Verify the "Best session" tile preserves the existing date + exercise pattern.
  - Verify the table's renamed column header and the chart-vs-table caption match.
  - Reload mid-filter; verify the URL state restores the chart.

**Migration test:** none required — no DB changes.

### Rollout

Coordinated cross-repo dispatch. Merge order:

1. `prog-strength-api` — `internal/exercise/movement_pattern.go`, handler param routing, per-exercise slope + aggregate computation, `baseline_model` field, top-level `trendline` removal. The change is read-side only; no migration, no backfill. Deploys via the existing release workflow.
2. `prog-strength-web` — `progress/page.tsx` reshape, `_components/` extraction, TanStack Query wiring, URL state, the new stat cards, the chart rework, the table column rename. Deploys via the existing release workflow.
3. `prog-strength-docs` — this SOW, flipped from `draft` to `shipped`.

The web release depends on the API release because the new endpoint fields are load-bearing for the redesigned page. The web change is a single contiguous PR (one route, the chart rework can't be staged half-rendered behind the old data shape).

**Revert behavior.** Reverting the web release puts the existing page back; it consumes only fields the new API still emits (`points`, `exercise_baselines`, and the absent top-level `trendline` reads as "no trendline"). The chart degrades silently — the existing combined regression line just doesn't draw — but the page still loads. Reverting the API release after the web is live degrades the new page: `per_exercise_trends` and `aggregate` come back undefined; the stat cards render `—` / `—` and the legend entries read "not enough data" universally. The page still functions; users see no slopes until either side is re-deployed.

## Open Questions

1. **Baseline model.** The SOW keeps the existing recency-weighted-current baseline and adds a label. The brief raised "first session in selected range" and "all-time peak" as alternatives; both were ruled out because the recency-weighted model is already robust to outlier first sessions (a 1RM attempt on day 1 of the window would pin a first-in-range baseline at peak and make every subsequent training session read as a regression) and stable across filter changes. Options: (a) ship with `recency_weighted_current` as specified; (b) switch to `first_in_range`, accept the outlier fragility, lean on the label to explain the shift; (c) switch to `all_time_peak`, accept the bias toward decline that makes most training sessions read < 100%. Tentative lean: (a). Open if there's a fourth model worth considering (`recency_weighted_at_since`, for example, which would be a smoothed "where you were when this window started" anchor — robust to outliers, shifts meaningfully with the filter).

2. **Noise threshold for "progressing."** The SOW commits to `slope_per_month > +0.25` as the threshold a lift must clear to count as progressing. Options: (a) 0.25 as specified; (b) 0.5, stricter — fewer false positives, more "flat" calls in the headline; (c) 0.0, anything-positive-counts — captures more "barely progressing" lifts in the numerator. Tentative lean: (a). The threshold sits well below the per-month noise floor for a well-logged lifter while still excluding obvious flat lifts.

3. **Whether to surface session count on the page.** The API returns `session_count` per exercise; the SOW currently uses it only to gate slope display ("not enough data yet"). A small "8 sessions" annotation next to each legend entry would let users see the data density behind the slope. Options: (a) hide session count (current SOW); (b) show it inline in the legend; (c) show it only in the tooltip on legend hover. Tentative lean: (a) — the legend is already information-dense with the direction arrow and slope, and the same info is implicit in the dot count on the chart line.

4. **Movement-pattern hue.** The SOW specifies a neutral accent for the active chip — no per-pattern color because "Push" doesn't have an established hue. Options: (a) neutral accent (current SOW); (b) assign each pattern a representative color (e.g. Push = red, Pull = blue, Legs = indigo, Core = emerald), salvaging the muscle-color memory the page used to have; (c) use the existing accent for the active state but tint the surrounding band when a non-`All` pattern is selected. Tentative lean: (a). Inventing pattern colors is a design exercise that doesn't pay rent on the page's central question.

5. **Best Session tile retention.** The third stat card is preserved from today. Options: (a) keep it as specified; (b) replace with "Exercises tracked" (count of lifts contributing in the window, more aligned with the page's headline question); (c) drop it and let the page run on two tiles. Tentative lean: (a). Removing it loses a motivating signal users react to, and the question it answers ("what was my best moment in this window?") doesn't overlap with the two new tiles.

6. **Direction-indicator thresholds.** The SOW uses ±0.5 %/mo for the ↑/→/↓ split. Options: (a) ±0.5 as specified; (b) ±0.25 matching the `lifts_progressing` threshold so "progressing" and "↑" mean the same thing; (c) ±1.0 stricter — only meaningfully-positive slopes get the arrow. Tentative lean: (a) — the indicator and the headline answer different questions ("which direction is this lift heading" vs. "is it gaining ground"), so different thresholds is honest. Open if collapsing the two thresholds reads simpler.
