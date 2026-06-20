---
type: dx
status: draft
surface: personal-records
idioms:
  - compact-dashboard-rail
  - editorial-spotlight
  - progression-timeline
  - gauge-grid
  - split-master-detail
references:
  - Linear
  - Whoop
  - Strava
  - Garmin Connect
  - Robinhood
  - Apple Fitness
  - Oura
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Personal Records

**Status**: Draft · **Last updated**: 2026-06-19

## Context

Personal Records is the "trophy case" — the page a Prog Strength user opens to see, in one glance, how strong and how fast they are right now. It's a two-view page today: a **Lifts** tab (heaviest set on each headline lift, plus the current recency-weighted estimated 1RM) and a **Running** tab (fastest window over each standard distance). The two tabs have drifted into two different surfaces. Lifts is a dense compact-tile grid with a readiness dot, an estimate delta, and an inline trend spark; Running is a roomier card grid with two text links and an expand chevron. Same page, same purpose, two unrelated visual languages — switching tabs feels like switching apps.

The deeper miss is that the page under-sells its single most motivating idea: **the gap between what you've logged and what you're capable of right now.** On Lifts, the estimated 1RM already sits above the logged PR for most exercises (bench 305 logged, 327 estimated — you're likely due for a new max), but that story is compressed into a tiny "+22" and a sparkline the size of a fingernail. On Running, the richer max-effort estimate (a forward projection with a confidence band, an estimate-over-time series, and the attempts behind it) lives a click away on a separate detail page, never previewed on the PR surface itself. And the page has a large band of dead space beneath the grid that does nothing.

This DX explores the **whole Personal Records page as one surface**: a single shared card-and-grid system both tabs inherit, and a deliberate use of the empty space below the grid as a featured "estimate & progress" chart that frames each PR as *progress toward a new best effort*. The win for the user is a trophy case that doesn't just record the past — it points at the next PR and tells them when to go get it.

## The surface

The page lives at `app/(app)/personal-records/page.tsx`. A segmented **Lifts / Running** control sits beside the title (URL-backed via `?view=`), and a **Customize** button (headline-lift selection) shows only on Lifts. Below the header is a single scroll region that today holds either `LiftsView` or `RunningView`.

Both views render a grid of one card per record, and both must handle: **dense** (8+ records), **sparse / empty-state** (a headline lift never trained, or a distance never covered — render a placeholder card, not a gap), **loading** (skeleton tiles), and **error** (inline message). Every variant must render *both* tabs through the same shared card/grid system — parity is the point of this exploration, not a nice-to-have.

The motivating new element every variant must surface is the **featured estimation chart** in the space beneath the grid, framed as progress toward a new best effort:
- **Lifts** — the estimated-1RM trend over time plotted against a reference line at the user's logged PR weight; the est-1RM sitting *above* the PR line is the "ready for a new max" signal. Data already exists: `GET /personal-records/{exercise_id}/history` returns the trend; `GET /personal-records` returns the current estimate + a downsampled spark series. No new backend model is in scope.
- **Running** — the existing max-effort estimate (forward projection + confidence band + the actual best, with the attempts behind it). Data already exists behind the running max-effort endpoint and the `/progress/running/[distanceKey]` detail page.

**Realistic fixtures** (mirror these in the mockups; values from a real account):

Lifts — `GET /personal-records` rows:

```json
[
  { "exercise_name": "Barbell Bench Press",        "weight": 305, "reps": 2,  "unit": "lb", "achieved_at": "2026-05-11", "current_estimated_1rm": 327, "recent_estimated_1rm_points": [305,308,311,316,322,327] },
  { "exercise_name": "Barbell High Bar Back Squat", "weight": 335, "reps": 4,  "unit": "lb", "achieved_at": "2026-06-19", "current_estimated_1rm": 369, "recent_estimated_1rm_points": [330,341,350,358,366,369] },
  { "exercise_name": "Barbell Bent Over Row",       "weight": 205, "reps": 3,  "unit": "lb", "achieved_at": "2026-05-11", "current_estimated_1rm": 234, "recent_estimated_1rm_points": [210,218,225,231,236,234] },
  { "exercise_name": "Dumbbell Bench Press",        "weight": 105, "reps": 8,  "unit": "lb", "achieved_at": "2026-04-28", "current_estimated_1rm": 137, "recent_estimated_1rm_points": [118,123,128,132,135,137] },
  { "exercise_name": "Incline Dumbbell Bench Press","weight": 95,  "reps": 8,  "unit": "lb", "achieved_at": "2026-06-16", "current_estimated_1rm": 116, "recent_estimated_1rm_points": [100,104,109,112,115,116] },
  { "exercise_name": "Barbell Deadlift",            "weight": null,"reps": null,"unit": null, "achieved_at": null,         "current_estimated_1rm": null, "recent_estimated_1rm_points": null },
  { "exercise_name": "Leg Press",                   "weight": 365, "reps": 10, "unit": "lb", "achieved_at": "2026-04-22", "current_estimated_1rm": 487, "recent_estimated_1rm_points": [430,448,462,475,483,487] },
  { "exercise_name": "Barbell Military Press",      "weight": 155, "reps": 5,  "unit": "lb", "achieved_at": "2026-06-13", "current_estimated_1rm": 175, "recent_estimated_1rm_points": [158,163,167,170,173,175] }
]
```

(Dumbbell weight is **per dumbbell** — render "105 ea × 8", not a combined pair. Note the `ea` modifier for any exercise whose name matches `/dumbbell/i`.)

Running — `GET` best-efforts rows + a max-effort estimate for one distance:

```json
{
  "best_efforts": [
    { "distance_key": "1mi",           "distance_label": "1 Mile",        "duration_seconds": 370,  "pace_sec_per_km": 230, "activity_start_time": "2026-04-06" },
    { "distance_key": "2mi",           "distance_label": "2 Mile",        "duration_seconds": 752,  "pace_sec_per_km": 233, "activity_start_time": "2026-04-06" },
    { "distance_key": "5k",            "distance_label": "5K",            "duration_seconds": 1181, "pace_sec_per_km": 236, "activity_start_time": "2026-04-06" },
    { "distance_key": "10k",           "distance_label": "10K",           "duration_seconds": 2495, "pace_sec_per_km": 249, "activity_start_time": "2026-04-11" },
    { "distance_key": "half_marathon", "distance_label": "Half Marathon", "duration_seconds": null, "pace_sec_per_km": null, "activity_start_time": null },
    { "distance_key": "marathon",      "distance_label": "Marathon",      "duration_seconds": null, "pace_sec_per_km": null, "activity_start_time": null }
  ],
  "max_effort_5k": {
    "seconds": 1175, "lower_seconds": 1150, "upper_seconds": 1206, "basis": "fitted_curve", "confidence": "medium",
    "actual_best_seconds": 1181,
    "estimate_history": [
      { "as_of": "2026-02-01", "seconds": 1240, "lower_seconds": 1180, "upper_seconds": 1320 },
      { "as_of": "2026-03-01", "seconds": 1212, "lower_seconds": 1170, "upper_seconds": 1270 },
      { "as_of": "2026-04-06", "seconds": 1188, "lower_seconds": 1160, "upper_seconds": 1222 },
      { "as_of": "2026-05-10", "seconds": 1175, "lower_seconds": 1150, "upper_seconds": 1206 }
    ],
    "attempts": [
      { "achieved_at": "2026-04-06", "duration_seconds": 1181, "source": "race_like" },
      { "achieved_at": "2026-03-22", "duration_seconds": 1205, "source": "long_run_window" }
    ]
  }
}
```

Design-system tokens (v0.4, in `../design-system.md`) every variant uses: near-black surfaces, **periwinkle `#9aa6d6` is app chrome only — never an activity or "due" color**, lift discipline = steel-blue (`#5598d8`), run discipline = green-teal (`#46b893`), `--warning` sand (`#d6b87f`) for the "due for a max" cue, `--success` for a positive delta, Manrope with `-0.03em` numeric tracking, 14px card radius, hairline borders.

## Idioms

Every variant honors the invariants above (shared card system across both tabs, estimate-as-progress, a featured below-grid chart, existing data only). They diverge on **layout, structure, density, and composition** — not palette, accent, or type family (`scope: in-system`).

- **compact-dashboard-rail** — Maximum density, figure-first. A 4-column tile grid where each card is a tight stat: name + a readiness dot whose intensity scales with the est-vs-PR gap, the PR figure, a one-line est delta, and a micro progress-line drawn against a faint dashed PR/best reference. Beneath the grid, a **docked featured estimation rail** that fills the dead space; clicking any tile swaps that record into the rail, which opens on the user's most-"due" record. Tight type scale, hairline grid lines, minimal padding. *Leans on Linear's compact tabular density and Whoop's single-accent-on-charcoal stat composition.* (This idiom captures the direction sketched in brainstorming — included as one option in the spread, not the foregone winner.)

- **editorial-spotlight** — Inverts the hierarchy: the **featured estimation chart is a large hero at the top** — "Next up: Bench Press, +22 lb over your logged PR" — with the chart as the centerpiece, and the full PR grid demoted to a quieter, roomier supporting list below. Big display figures, generous vertical rhythm, lots of negative space, one record's progress story told at a time. *Leans on Apple Fitness's summary-hero composition and Strava's "you're trending toward a PR" editorial framing.*

- **progression-timeline** — Trend-first rather than figure-first. Each record is a full-width horizontal **row dominated by its over-time progression line** (est-1RM climbing toward/over the PR line; best-effort time descending) with the current figure as a right-aligned endpoint label — a rail of trends you scan top to bottom, like a sparkline table. The featured area is a full-width expanded timeline of the selected row. Moderate density, horizontal rhythm, the chart *is* the card. *Leans on Garmin Connect's progression charts and GitHub's contribution-rail scanning pattern.*

- **gauge-grid** — Readiness expressed as **geometry, not a dot**. Each card centers a radial (or vertical-bar) gauge showing the current estimate as a fill toward — and past — the logged PR mark, so "how ready am I" is read at a glance from how far the arc has swept. The featured panel below is a large arc gauge plus the estimate trend and (running) confidence band. Centered composition, circular rhythm, figures sized inside the gauge. *Leans on Whoop's recovery-ring composition and Oura's gauge-as-progress logic — rendered strictly in the DS palette.*

- **split-master-detail** — A two-pane **ledger**: a narrow left column lists every record as a compact row **ranked by readiness** (most-due first, so "what should I attempt next" is the top row), and a wide right pane holds the featured estimation chart pinned in view, updating as you select a row. No expand-in-place; the detail is always present. Dense list on the left, calm focused chart on the right, strong vertical divider. *Leans on Linear's and Robinhood's master-detail list-plus-chart layouts.*

## References

- **Linear** — its compact tabular density and restrained hairline borders (for `compact-dashboard-rail`), and its master-detail list-plus-content layout (for `split-master-detail`). Take the information density and the calm, not its grayscale palette.
- **Whoop** — recovery-ring composition and single-accent-on-charcoal stat cards (for `gauge-grid` and the density of `compact-dashboard-rail`). Take the ring-as-progress idea and the "one number that matters" focus.
- **Strava** — the "best efforts" and trend framing, and the editorial "you're trending toward a PR" summary voice (for `editorial-spotlight`). Take the motivational framing of progress, not its orange.
- **Garmin Connect** — over-time progression and best-effort trend charts (for `progression-timeline`). Take the trend-rail-per-metric layout.
- **Robinhood** — charting a value against a reference/goal line, and the master-detail price-plus-list view (for `split-master-detail` and the PR-reference-line treatment across all variants). Take the "value vs. a goal line" reading.
- **Apple Fitness** — the large summary-hero composition that leads with one headline number and a supporting chart (for `editorial-spotlight`). Take the hero-then-supporting-grid hierarchy.
- **Oura** — gauge-as-progress, where a single arc encodes "how far toward the target" at a glance (for `gauge-grid`). Take the gauge-reading, rendered in the DS palette.

## Selection criteria

A note to myself at the gate, not a rubric for the worker.

- **Parity has to feel effortless.** When I toggle Lifts ↔ Running, it should read as one page showing two disciplines — not two designs sharing a tab bar. Whichever variant makes the running max-effort estimate and the lifts est-1RM feel like the *same idea* wins points.
- **The "go attempt a new max" moment has to land.** The whole reason for the estimate is to nudge a PR attempt. I'm looking for the variant where the est-vs-PR gap is the thing my eye goes to first — not buried in a delta. Does it make me want to go lift?
- **The dead space below the grid should earn its keep.** The featured estimation chart should feel like the payoff of the page, not a footer. But it shouldn't bury the at-a-glance trophy-case scan either — I still want to see all my numbers fast.
- **It has to survive a sparse account.** A new user with two PRs and one running distance shouldn't see a broken or empty-feeling page. Empty-state cards and the "not enough data yet" estimate state need to look intentional in whichever variant I pick.
- **It must read native to the app.** `in-system` — if a variant only works by reaching for a color or type treatment outside design-system v0.4, that's disqualifying, however nice it looks.
