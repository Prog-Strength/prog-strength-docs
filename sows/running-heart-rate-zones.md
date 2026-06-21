---
status: ready_for_implementation
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Running Heart Rate Zones

**Status**: Ready for implementation · **Last updated**: 2026-06-21

## Introduction

A run's average and max heart rate — the only HR numbers **Prog Strength** surfaces today — answer "how hard, on average" but not the question runners actually train against: *what kind of work did this run do?* Two runs can share an average HR of 150 while one was a steady aerobic hour and the other a stack of threshold intervals around long recoveries. The distinction is the whole point of structured endurance training. Time spent in each heart-rate zone is how a runner knows whether a session built their aerobic base, sharpened their threshold, or taxed their VO₂max — and whether an "easy" day was actually easy.

The data to answer this is already in hand. Every running activity stores per-trackpoint heart rate (see the existing `GET /activities/{id}` response). What's missing is the interpretation: classifying each moment of the run into a zone and reporting how long was spent in each. This SOW adds that interpretation **on the backend**, as a derived part of the activity response, so every client — the web detail page first, the agent and mobile app later — reads one consistent, authoritative zone breakdown rather than each reinventing the math.

Heart-rate zones are anchored to a reference: an athlete's maximum heart rate. **Prog Strength** stores no such reference today (no max HR, resting HR, LTHR, or age on the user), and we deliberately keep it that way for v1 — asking a runner to know and enter their true max HR is real friction, and most don't. Instead the system **estimates** the reference and improves it as it observes more of the athlete's running. The estimation logic is the interesting, evolving part of this feature, so it is built as a dedicated, documented engine with a single home, designed to grow in sophistication over time.

## Proposed Solution

**Prog Strength** will compute a five-zone heart-rate breakdown for every running activity and include it as an additive `heart_rate_zones` block on the activity response. Zones use the widely-recognized **percent-of-max-HR** model. The reference max HR is estimated by a new, self-contained **heart-rate-zone estimation engine** that degrades gracefully from a cold start (no data → a sane population default) through a calibrating phase up to a calibrated estimate derived from a spike-resistant statistic over the athlete's recent runs. The response carries a confidence signal so clients can show an honest "calibrating" state early and a real breakdown once the estimate is trustworthy.

The web running detail page gains a **heart-rate zones widget**: a horizontal stacked bar of time-in-zone with a labeled legend, plus a "calibrating" message while the engine is still learning the athlete's heart rate.

All knobs the engine reasons with — the population default, the calibration threshold, the recency window, the zone boundaries — live in `config.toml` (version-controlled, non-secret), consistent with the centralized-config convention. The engine itself is a pure Go package with no database or HTTP coupling, so it is the single source of truth for zone logic and is unit-testable in isolation.

## Goals and Non-Goals

### Goals

- Add a `heart_rate_zones` block to the running activity response, computed entirely on the backend.
- Classify every HR-bearing moment of a run into one of five percent-of-max-HR zones and report **time** (not sample count) in each.
- Estimate the athlete's reference max HR with a cold-start default that improves as more runs are observed, resisting sensor spikes, and report the estimate's confidence.
- House all zone and estimation logic in one pure, documented engine package that can grow in accuracy without changing its callers.
- Surface a zones widget on the web running detail page, including a "system calibrating" state.
- Keep all engine tunables in `config.toml`.

### Non-Goals

- **A user-facing max-HR / resting-HR / LTHR / age setting.** v1 estimates the reference; no new profile field, no settings UI, no migration to `users`. (The engine is structured so a future "I know my max HR" override slots in cleanly.)
- **Heart-rate-reserve (Karvonen) or threshold (LTHR) zone models.** Percent-of-max only for v1. The engine's `model` field names the active model so additional models can be added later without a breaking response change.
- **Zones for non-running activities.** The block is computed for running activities; the engine is sport-agnostic but only wired into the running path here.
- **Ingest-time precomputation / caching of zones.** Computed at read time. A documented growth path to cache a per-run robust HR statistic at ingest exists but is not built now.
- **User-editable zone boundaries.** Standard boundaries from config; no per-user customization.

## Implementation Details

### The estimation engine: `internal/hrzones`

A new pure Go package, `internal/hrzones`, is the single source of truth for everything about heart-rate zones: the zone model, reference-max-HR estimation, classification, and time-in-zone accumulation. It has **no dependency on the database, HTTP, or the `activity` package** — callers gather the inputs it needs and hand them in, and it returns a plain result. This boundary is what lets the estimation logic grow in complexity over time without rippling into callers, and what makes it exhaustively unit-testable.

The package exposes a small surface:

```go
package hrzones

// Config mirrors the [hr_zones] section of config.toml. Passed in once at
// construction so the engine has no global state.
type Config struct {
    PopulationDefaultMaxHR int     // e.g. 190 — cold-start reference when no data
    CalibratedRunThreshold int     // e.g. 5  — HR-bearing runs needed to trust the p99 estimate
    RecencyWindowDays      int     // e.g. 90 — how far back "recent" runs reach
    MinReferenceBpm        int     // e.g. 100 — clamp floor for any estimate
    MaxReferenceBpm        int     // e.g. 230 — clamp ceiling; rejects gross sensor spikes
    // Zone boundaries as ascending fractions of max HR, e.g. [0.60, 0.70, 0.80, 0.90].
    // Five zones ⇒ four interior boundaries. Names parallel the boundaries.
    ZoneUpperBounds []float64
    ZoneNames       []string
}

// Stats is the historical context the engine needs, computed by the caller
// (the activity repository) from stored runs. Engine stays pure.
type Stats struct {
    RecentHRSamplesP99 *int // 99th-percentile HR across the athlete's recent runs; nil if none
    HistoryRunCount    int  // # of HR-bearing runs in the recency window (excludes the current run)
    CurrentRunP99      *int // 99th-percentile HR of THIS run; nil if the run has no HR
}

type Confidence string

const (
    ConfidenceEstimated   Confidence = "estimated"   // cold start, population default
    ConfidenceCalibrating Confidence = "calibrating" // some history, below threshold
    ConfidenceCalibrated  Confidence = "calibrated"  // enough history to trust the estimate
)

type Reference struct {
    MaxHRBpm   int
    Source     string     // "population_default" | "current_run" | "p99_recent_runs"
    Confidence Confidence
}

type Zone struct {
    Number      int
    Name        string
    LowerPct    float64
    UpperPct    float64
    MinBpm      int
    MaxBpm      int
    TimeSeconds int
    TimePct     float64
}

type Result struct {
    Model          string // "percent_max_hr"
    Reference      Reference
    TotalHRSeconds int
    Zones          []Zone
    Calibrating    bool // true unless Confidence == ConfidenceCalibrated; drives the widget banner
}

// EstimateReference applies the cold-start → calibrating → calibrated ladder.
func (e *Engine) EstimateReference(s Stats) Reference

// Compute classifies trackpoints against a reference and accumulates time-in-zone.
// Returns ok=false when the run carries no usable HR (caller omits the block).
func (e *Engine) Compute(ref Reference, tps []Trackpoint) (Result, bool)
```

The activity handler only ever calls `EstimateReference` then `Compute`. Everything else is internal and documented.

### Zone model

Five zones on percent of max HR. Boundaries come from `Config.ZoneUpperBounds`; the table below shows the defaults. Every HR value maps to exactly one zone — Zone 1 is an open-ended low catch-all (anything under 60%) and Zone 5 is open-ended high, so no sample is left unclassified.

| Zone | Name | Range (% max) | Example bpm at max = 191 |
| --- | --- | --- | --- |
| 1 | Recovery | < 60% | 0–114 |
| 2 | Aerobic | 60–70% | 115–133 |
| 3 | Tempo | 70–80% | 134–152 |
| 4 | Threshold | 80–90% | 153–171 |
| 5 | VO₂max | ≥ 90% | 172–191 |

`MinBpm`/`MaxBpm` per zone are computed from the reference and the boundary fractions (lower bound inclusive, upper bound exclusive; rounding handled once in the engine) so clients can render exact ranges without repeating the math.

### Reference max HR: the cold-start → learning ladder

`EstimateReference` picks the reference and its confidence from `Stats`:

1. **Calibrated** — `HistoryRunCount >= CalibratedRunThreshold` and `RecentHRSamplesP99 != nil`:
   reference = `RecentHRSamplesP99`, source `p99_recent_runs`, confidence `calibrated`.
2. **Calibrating** — some history (`HistoryRunCount > 0`, `RecentHRSamplesP99 != nil`) but below threshold:
   reference = `max(RecentHRSamplesP99, CurrentRunP99)`, source `p99_recent_runs`, confidence `calibrating`.
3. **Estimated (cold start)** — no usable history:
   reference = `PopulationDefaultMaxHR`, or `max(default, CurrentRunP99)` if this run already exceeds the default; source `population_default` (or `current_run` if the run drove it), confidence `estimated`.

The final reference is always clamped to `[MinReferenceBpm, MaxReferenceBpm]`.

Two deliberate choices:

- **p99, not the single highest sample.** Chest straps and optical sensors routinely emit a phantom spike (often a lone 210–220 bpm) before contact stabilizes. Using the 99th percentile across recent runs rejects such isolated samples while still self-calibrating; the `MaxReferenceBpm` clamp is a second backstop against gross errors.
- **Population default, not this run's own max, for the cold case.** Anchoring a first run to its own max would pin Zone 5 by construction — every easy first run would falsely read as maximal. Against a population default (≈190), an easy first run (max ~150) correctly reads as low-aerobic, and the estimate sharpens as real runs accumulate.

The calibration banner clears automatically the moment `HistoryRunCount` crosses `CalibratedRunThreshold`; nothing about it is sticky.

### Time-in-zone computation

Time-in-zone is accumulated from real elapsed time, **not** by counting samples — trackpoint spacing varies (the production data has both 9s and 10s gaps). For each consecutive trackpoint pair `(i-1, i)`:

- `dt = elapsed_seconds[i] - elapsed_seconds[i-1]`.
- Classify the interval by the **mean** of the two endpoints' `heart_rate_bpm`, mapped through the zone boundaries.
- Add `dt` to that zone's `TimeSeconds`.
- Skip the interval entirely if either endpoint lacks HR (the stationary start/end carry null HR).

`TotalHRSeconds` is the sum of all classified intervals, and each zone's `TimePct = TimeSeconds / TotalHRSeconds`, so percentages always sum to 1.0 over the HR-covered portion of the run.

### Config

A new `[hr_zones]` section in `prog-strength-api/config.toml` carries every tunable, loaded into `hrzones.Config` via the existing config package:

```toml
[hr_zones]
population_default_max_hr = 190
calibrated_run_threshold  = 5
recency_window_days       = 90
min_reference_bpm         = 100
max_reference_bpm         = 230
zone_upper_bounds         = [0.60, 0.70, 0.80, 0.90]
zone_names                = ["Recovery", "Aerobic", "Tempo", "Threshold", "VO2max"]
```

No values here are secret; all are reviewable in version control, per the config-vs-secrets convention.

### Repository support

The `activity` repository gains one read method that produces the engine's `Stats` for a user, e.g.:

```go
RecentHRStats(ctx, userID string, window time.Duration, excludeActivityID string) (hrzones.Stats, error)
```

It computes, over the user's non-deleted **running** activities whose `start_time` falls within the recency window (excluding the activity being viewed): the count of HR-bearing runs (`HistoryRunCount`) and the 99th-percentile HR across their trackpoints (`RecentHRSamplesP99`). `CurrentRunP99` is computed from the viewed run's own trackpoints. Because SQLite has no native percentile aggregate, the percentile is computed in Go over the fetched HR samples.

Pre-launch data volumes make read-time computation comfortably cheap. **Growth path (not built now, documented in the engine):** if volume grows, persist a per-run robust HR statistic (the run's p99) to a new nullable `activities` column at ingest, reducing `RecentHRStats` to a cheap aggregate over that column. The engine boundary means this optimization changes only the repository, never callers or response shape.

### Response shape

The single-activity response (`GET /activities/{id}`, used by the web running detail page) gains one additive, non-breaking field inside `data`. Nothing existing changes.

```json
"heart_rate_zones": {
  "model": "percent_max_hr",
  "max_hr_reference_bpm": 191,
  "reference_source": "p99_recent_runs",
  "reference_confidence": "calibrated",
  "calibrating": false,
  "total_hr_seconds": 3170,
  "zones": [
    { "zone": 1, "name": "Recovery",  "lower_pct": 0.0, "upper_pct": 0.60, "min_bpm": 0,   "max_bpm": 114, "time_seconds": 240,  "time_pct": 0.076 },
    { "zone": 2, "name": "Aerobic",   "lower_pct": 0.60, "upper_pct": 0.70, "min_bpm": 115, "max_bpm": 133, "time_seconds": 980,  "time_pct": 0.309 },
    { "zone": 3, "name": "Tempo",     "lower_pct": 0.70, "upper_pct": 0.80, "min_bpm": 134, "max_bpm": 152, "time_seconds": 760,  "time_pct": 0.240 },
    { "zone": 4, "name": "Threshold", "lower_pct": 0.80, "upper_pct": 0.90, "min_bpm": 153, "max_bpm": 171, "time_seconds": 690,  "time_pct": 0.218 },
    { "zone": 5, "name": "VO2max",    "lower_pct": 0.90, "upper_pct": 1.0,  "min_bpm": 172, "max_bpm": 191, "time_seconds": 500,  "time_pct": 0.158 }
  ]
}
```

Assembly lives in the existing presenter (`toActivityDTO` in `internal/activity/handler.go`), following the established pattern for read-time derived fields (`avg_pace_sec_per_km`, `best_pace_sec_per_km`). The handler builds `Stats` from the repository, calls `EstimateReference` then `Compute`, and attaches the result as a DTO.

**Edge case — a run with no HR at all:** when the viewed run has zero HR-bearing trackpoints, `Compute` returns `ok = false` and the handler **omits** `heart_rate_zones` entirely (the field is `omitempty`). Zones are undefinable without per-point HR, and the widget renders nothing. This is distinct from the cold-start case (a run *with* HR but no history), which still produces zones at `estimated` confidence.

### Web widget (`prog-strength-web`)

The running detail page (`app/(app)/running/[id]/page.tsx`) gains a heart-rate zones component under the existing stats band, built in the established hand-rolled-SVG style of `PaceStrip.tsx` — no new charting dependency:

- A **horizontal stacked bar**, one segment per zone, width proportional to `time_pct`.
- A **legend**: per zone, the name, bpm range (`min_bpm`–`max_bpm`), time, and percentage.
- Zone colors derive from the design system — a periwinkle-anchored single-hue ramp (light→saturated across Z1→Z5) rather than the cliché green→red, to stay in-system with the v0.4 palette (`design-system.md`).
- When `reference_confidence` is `estimated` or `calibrating`, show a subtle banner: *"Calibrating — zones will sharpen as Prog Strength learns your heart rate."* When `calibrated`, no banner.
- When `heart_rate_zones` is absent from the response, the component does not render.

### Documentation

The engine ships with a package `doc.go` explaining the zone model, the estimation ladder, the confidence states, and the documented ingest-cache growth path — so the "single source of truth" is also self-describing. A companion note is added under `prog-strength-docs` describing the engine as the home for the heart-rate-zone estimation logic and its intended evolution (additional models, a future user-supplied max-HR override, ingest-time caching).

### Testing

- **Engine (pure, table-driven):**
  - Classification at and across every boundary HR (off-by-one around inclusive-lower / exclusive-upper).
  - Time-in-zone with variable `dt` (9s/10s gaps) and null-HR intervals skipped; `time_pct` sums to 1.0.
  - The estimation ladder: cold start with no data → population default; cold start where this run exceeds the default → `current_run`; calibrating → `max(recent, current)`; calibrated → `p99_recent_runs`; clamp floor and ceiling.
  - Spike rejection: an injected phantom 220 sample does not move a p99-based reference.
  - Single-run / circular case produces `estimated` confidence and `calibrating: true`.
- **Repository:** `RecentHRStats` window filtering (in/out of `recency_window_days`), exclusion of the current and deleted activities, run-count and p99 correctness.
- **Handler (`handler_test.go`):** the import-happy-path fixture asserts the `heart_rate_zones` block, its zone count, summed `time_pct`, and confidence; a no-HR fixture asserts the block is omitted.
