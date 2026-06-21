# The heart-rate-zone estimation engine (`internal/hrzones`)

**Status**: Living note · **Last updated**: 2026-06-21 · **Provenance**: [`sows/running-heart-rate-zones.md`](../sows/running-heart-rate-zones.md)

`internal/hrzones` in `prog-strength-api` is the **single home for everything
about heart-rate zones**: the zone model, the reference-max-HR estimation, the
classification of trackpoints, and the time-in-zone accumulation. This note
records *why* it is structured the way it is and where it is expected to grow,
so the next change lands in the right place.

## Why a dedicated, pure package

The interesting, evolving part of this feature is not the arithmetic of binning
a heart rate into a zone — it is the **estimation of the reference the zones are
anchored to**. We deliberately store no athlete max-HR / resting-HR / LTHR / age
(asking a runner for their true max HR is real friction, and most don't know
it), so the system has to *estimate* the reference and sharpen it as it observes
more running. That logic will get more sophisticated over time.

To let it grow without rippling into callers, the engine is a **pure Go package
with no dependency on the database, HTTP, or the `activity` package**. Callers
gather the inputs (`hrzones.Stats`) and hand them in; the engine returns a plain
`hrzones.Result`. The activity handler only ever calls `EstimateReference` then
`Compute`. Everything else is internal. This boundary is what makes the
estimation exhaustively unit-testable and what keeps the optimisation in
["Growth path"](#growth-path-ingest-time-caching) below a repository-only change.

## The zone model (v1)

Five zones on **percent of max HR** (the widely-recognised model). Boundaries
are ascending fractions of the reference in `config.toml`'s `[hr_zones]`
section (`zone_upper_bounds = [0.60, 0.70, 0.80, 0.90]`), so four interior
bounds yield five zones. Zone 1 is open-ended low (anything under 60%) and
Zone 5 is open-ended high (≥ 90%), so every sample maps to exactly one zone.
Per-zone bpm ranges are derived once in the engine from the reference and the
fractions (lower inclusive, upper exclusive) so clients render exact ranges
without repeating the maths.

Time-in-zone is accumulated from **real elapsed time**, not sample counts
(trackpoint spacing varies): each consecutive trackpoint pair contributes its
`dt` to the zone of the pair's mean HR; pairs missing HR at either end are
skipped.

## The estimation ladder and confidence

`EstimateReference` degrades gracefully:

1. **Estimated** (cold start) — no usable history → the population default
   (`population_default_max_hr`, ≈190), or this run's own p99 if it already
   exceeds the default. Anchoring a *first* run to its own max would pin Zone 5
   by construction, so the population default is the cold anchor.
2. **Calibrating** — some history but below `calibrated_run_threshold` →
   `max(p99 of recent runs, this run's p99)`.
3. **Calibrated** — enough HR-bearing runs in the recency window → the **p99**
   across the athlete's recent runs.

The reference is always clamped to `[min_reference_bpm, max_reference_bpm]`.
We use the **99th percentile, not the single highest sample**, because chest
straps and optical sensors routinely emit a lone phantom spike (often 210–220
bpm) before contact stabilises; p99 rejects the isolated sample while still
self-calibrating, and the clamp is a second backstop. The `confidence` signal
(`estimated` / `calibrating` / `calibrated`) is surfaced on the response so
clients can show an honest "calibrating" banner early and a real breakdown once
the estimate is trustworthy. Nothing about the banner is sticky — it clears the
moment `HistoryRunCount` crosses the threshold.

## Intended evolution

The package is structured so each of these slots in **without changing the
engine's callers or the response shape**:

- **Additional zone models.** `Result.Model` names the active model
  (`percent_max_hr` today). Heart-rate-reserve (Karvonen) or threshold (LTHR)
  models can be added behind the same interface; the named `model` field means
  adding one is not a breaking response change.
- **A user-supplied max-HR override.** v1 estimates the reference and adds no
  profile field. When "I know my max HR" arrives, it slots into
  `EstimateReference` as the highest-confidence source ahead of the p99 ladder —
  the `Reference.Source` / `Confidence` plumbing already exists.
- <a name="growth-path-ingest-time-caching"></a>**Ingest-time caching.** Zones
  are computed at read time today; pre-launch volumes make that comfortably
  cheap. If volume grows, persist a per-run robust HR statistic (the run's p99)
  to a new nullable `activities` column at ingest, reducing the repository's
  `RecentHRStats` to a cheap aggregate over that column. Because of the engine
  boundary this touches **only the repository** — never the engine or the
  response shape.
