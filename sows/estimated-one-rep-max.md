# Estimated One Rep Max Time Series Table

**Status**: Shipped · **Last updated**: 2026-05-18

## Introduction

The core promise of **Prog Strength** is to tell a lifter whether they are actually getting stronger. The app cannot answer that question well at the level lifters most often ask it: the muscle group.

A well-designed program rotates exercises within a muscle group, so any single exercise generates only a handful of data points per month — a meaningful per-exercise trend can take 90+ days to emerge. And the raw one rep max numbers across those exercises are not directly comparable: barbell, dumbbell, cable, and machine variants tax different stabilization systems and sit on different absolute strength scales. Plotted or averaged together they describe which exercise the lifter happened to run that week, not whether their chest actually got stronger.

Solving this gives lifters a single, honest signal per muscle group that draws on every relevant session they have logged, and shifts the "first useful insight" milestone from 90+ days down to a few weeks.

## Proposed Solution

**Prog Strength** will maintain a new time series table of estimated one rep maxes, keyed by exercise. Every time a workout is logged, the estimated one rep max for each exercise in that workout is computed and appended to this table as a new row, building a per-exercise history of strength over time.

For each exercise, the user's *current* estimated one rep max is derived as a recency-weighted running average across the lifter's recent entries for that exercise — not the latest entry alone. Weighting more recent entries more heavily smooths out short-term dips like deload weeks, illness, or off days, so the baseline reflects sustained capability rather than a single data point that may have been deliberately conservative.

When analyzing a muscle group, the system uses each exercise's recency-weighted current one rep max as that exercise's baseline, and every historical entry for the exercise is expressed as a fraction of that baseline. This converts an absolute weight into a relative measure of where each set sat relative to the lifter's current capability on that exercise. Once every exercise within a muscle group is on the same relative scale, they can be plotted together as a single trendline that is robust to the inherent strength differences between them.

## Goals and Non-Goals

### Goals

- Persist a per-(workout, exercise) time series of estimated one rep maxes that becomes the source of truth for all downstream "am I getting stronger" analyses.
- Provide a recency-weighted baseline per exercise that is robust to short-term dips like deload weeks.
- Enable a muscle-group progress view by normalizing each exercise's history against its current baseline.
- Stay fully derivable from `workouts`, so the table can be truncated and rebuilt at any time without data loss.

### Non-Goals

- Storing per-set rows. Aggregation to per-(workout, exercise) min/avg/max is intentional; per-set granularity is not needed for the muscle-group view and would inflate row counts significantly.
- Persisting the baseline itself. The baseline is a derived read-time computation.

Originally this section also listed "replacing the existing `/workouts/progression` endpoint" as a non-goal. That decision flipped during implementation: the per-exercise endpoint was retired and `/workouts/progression` now serves the muscle-group view exclusively. The two views never coexisted in production.

## Implementation Details

### Table Schema

A new table, `exercise_one_rep_max_history`, stores one row per (workout, exercise) pair. The table is fully derived from `workouts` — every workout write produces a corresponding write here — and is read as a denormalized view of strength over time.

| Column | Type | Description |
| --- | --- | --- |
| `id` | text | Primary key. |
| `user_id` | text | Owning user. |
| `workout_id` | text | Foreign key to `workouts`. Cascades on update and delete so this table never drifts from its source. |
| `exercise_id` | text | Catalog slug, e.g. `barbell-bench-press`. |
| `performed_at` | text (RFC3339) | Mirrored from the parent workout so time-range filters do not need a join. |
| `min_estimated_1rm` | real | Smallest per-set estimated 1RM among this exercise's sets in the workout. |
| `avg_estimated_1rm` | real | Mean per-set estimated 1RM across those sets. Carried for display in the per-workout estimates table on the Progress page. |
| `max_estimated_1rm` | real | Largest per-set estimated 1RM among those sets. Used by the recency-weighted baseline (max chosen over avg so warmup sets don't deflate the signal). |
| `set_count` | integer | Number of sets aggregated. |
| `unit` | text | `lb` or `kg`, mirrored from the underlying sets. |
| `created_at` | text (RFC3339) | Row creation timestamp. |
| `updated_at` | text (RFC3339) | Row last-modified timestamp; touched when the parent workout is edited. |

A composite index on (`user_id`, `exercise_id`, `performed_at` DESC) supports the dominant query pattern: *"this user's most recent entries for this exercise."*

### Write Path

This table is always kept in lockstep with `workouts`:

- **Workout created** — insert one row for each distinct exercise in the workout, with min/avg/max computed across that exercise's sets.
- **Workout updated** — recompute and update the rows belonging to that workout. New exercises insert new rows; removed exercises delete the matching rows.
- **Workout deleted** — delete all rows for that workout.

Because every row is fully derivable from `workouts`, the entire table can be rebuilt from scratch at any time — useful for backfills and for correcting any past drift.

### Estimated One Rep Max Formula

Estimated one rep max is computed per set from reps and weight. Two formulas are commonly used in the strength training community:

**Epley**

```
1RM = weight × (1 + reps / 30)
```

**Brzycki**

```
1RM = weight × 36 / (37 − reps)
```

For the rep ranges that dominate strength training (1–8 reps), the two formulas agree within roughly 2%. They diverge at higher rep counts — Epley estimates higher, Brzycki estimates lower beyond about 12 reps — but the rep ranges that drive the *"am I getting stronger"* question almost always sit in the low end.

**Decision**: use Epley alone. It is the simpler formula, it is already in use elsewhere in the codebase, and the practical difference from Brzycki inside the relevant rep range is smaller than the noise from session-to-session variation. If a high-rep hypertrophy track later needs better fidelity, a second formula column can be added without breaking existing analyses.

A single-rep set is a true rep max; Epley collapses to `weight × (1 + 1/30) ≈ 1.033 × weight`, close enough to the lifted weight to ignore.

### Recency-Weighted Baseline

The user's *current* estimated one rep max for an exercise — the baseline used to normalize the muscle-group progress view — is computed at read time from this table as an exponentially time-weighted moving average over recent entries:

```
baseline = Σ (wᵢ × vᵢ) / Σ wᵢ

where  vᵢ = max_estimated_1rm of entry i
       wᵢ = exp( −(t_now − tᵢ) / τ )
       τ  = 45 days
```

`max_estimated_1rm` was chosen over `avg_estimated_1rm` because the per-workout average is pulled down by warmup sets, which has nothing to do with the lifter's actual capability. The per-workout max is almost always one of the working sets, so it tracks the load that answers "what could this person do today?" without polluting the signal with warmup protocol drift. A future warmup-flag-on-sets feature can replace this with a "working-set average" without changing the formula's shape.

Only entries with `performed_at` within the last **90 days** are included. Older entries are not representative of the lifter's current capability through the noise of program changes, deloads, and accessory rotations.

The 45-day time constant gives a half-life of about 31 days — each successive month of training contributes roughly half the weight of the prior month. Worked example: a lifter who benches once a week for twelve weeks and then deloads has thirteen entries in the window. The deload entry's weight is `1.0` and the sum of all weights is about `6.0`, so the deload contributes roughly 17% of the baseline. If the deload session's max is 90% of the lifter's normal value, the baseline drops by about 2% — visible but not crashing. The baseline tracks sustained capability rather than the lifter's most recent training day.

If fewer than two entries exist within the 90-day window for an exercise, the baseline is simply that single entry's `max_estimated_1rm` — there is not enough data for a meaningful weighted average until the second session.

The window length and time constant are tunable parameters. The values above are a defensible starting point; they should be revisited once enough beta users have logged a meaningful training history to validate the smoothing behavior against real deload and progression patterns.

### Backfill Strategy

Because every row is fully derivable from `workouts`, backfilling existing data is straightforward and idempotent. The mechanism splits across two stages:

1. **SQL migration** — creates the empty table and its composite index. No data population in the migration itself; this keeps the migration step bounded and cheap.
2. **In-process backfill function** (`BackfillOneRepMaxHistory`) — runs on every startup, gated by an `existing > 0` row-count check on the table. On the first boot after the migration ships, the gate passes and the function iterates over every non-deleted workout, calls the same aggregation function as the live write path, and inserts the rows in a single transaction. Subsequent boots find rows present and skip.

The shared aggregation function is the single most important invariant of the backfill. The live write path and the backfill both call `AggregateOneRepMax`, so backfilled rows are guaranteed to match what a freshly written row would have contained for the same workout. Without that, the two write paths could subtly diverge over time and downstream analyses would see inconsistent shapes.

Failures during the backfill are recoverable: truncate the table and restart the API. The next boot finds it empty and rebuilds from `workouts`. There is no destructive state in the new table, and `workouts` is untouched.

At current beta scale (low thousands of workouts across all users), the backfill completes in well under a minute on the API instance. If the table later grows past a few hundred thousand rows, the backfill should be promoted to a separate CLI command so the API container starts faster on deploys; until then, in-process at startup is the simpler story.

## Open Questions

1. **Warmup sets** — *partially resolved*. The original concern was that warmup sets pollute the per-workout `avg_estimated_1rm` and therefore the baseline. The recency-weighted baseline now reads `max_estimated_1rm` instead of `avg_estimated_1rm`, which addresses the spirit of the question: max almost always picks a working set, so warmups don't get into the signal. A per-set warmup flag on the `workouts` schema remains a future option — it would let us also clean up the displayed `avg` and `min` values — but it is no longer load-bearing for the baseline math.

2. **Mixed-unit workouts**. It is unusual but possible for a workout to contain both `lb` and `kg` sets for the same exercise. Two options: (a) the row's `unit` is the most-common unit (matching the existing `/workouts/progression` handler) and the minority sets are dropped from aggregation, or (b) all sets are converted to the user's preferred unit before aggregation. **Shipped with (a)**; revisit if mixed-unit users surface as a real cohort.

3. **External API surface**. Should `exercise_one_rep_max_history` be exposed via its own endpoint, or used purely internally to compute the muscle-group view? **Shipped internal-only**; raw history is not exposed. Exposing it publicly would invite premature consumer dependencies on a schema we expect to tune.
