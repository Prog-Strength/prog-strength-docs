---
status: draft
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Running Detail Metric Alignment

**Status**: Draft · **Last updated**: 2026-07-03

> Cross-repo SOW. Moves the running-detail derivation (splits, pace-strip
> summary, intervals) server-side so every number on `/running/[id]` is
> computed once, from one calibrated trackpoint stream, under one policy —
> and adds an invariant gate that verifies the assembled response reconciles
> with itself before it is returned. Follow-up to
> [`treadmill-run-calibration.md`](treadmill-run-calibration.md).

## Introduction

The running detail page shows the same run four ways — stat tiles, a splits
table, a pace chart, and heart-rate-zone bars — and today those four surfaces
disagree. On a real treadmill run the Avg Pace tile reads **11:02 /mi** while
the splits average roughly **10:30 /mi**; Mi 1 shows **10:26** of time over
**1.0 mi** yet a pace of **10:11**; the Best tile (9:07), the FASTEST split
tag (9:39), and the chart's "Fastest 8:33" are three different numbers all
labeled fastest. Users doing the arithmetic conclude the data is wrong.

The numbers diverge because they are computed in two places under different
policies. The tiles are server values frozen at TCX ingest (avg pace = total
duration ÷ total distance). The splits table and pace chart are derived in
the browser (`lib/running-splits.ts`) from the downsampled trackpoint stream,
and that derivation **discards data**: segments whose pace sample is missing
or slower than a dropout cutoff (410 s/km) are excluded from split-pace math,
and non-advancing trackpoint pairs are dropped entirely. So a split's Pace
column is not its Time ÷ Dist, split times don't sum to the Time tile, and no
combination of splits reproduces the Avg Pace tile. Distance calibration made
the stakes higher: the API now rescales trackpoints and summary in one
transaction precisely so surfaces agree, but the Best tile is rescaled by an
approximation (old value ÷ factor) instead of recomputed, and nothing anywhere
verifies that the response a user sees is self-consistent.

Once this ships, every number on the detail page reconciles: each split's
pace is exactly its time over its distance, split times/distances sum to the
Time/Distance tiles, the distance-weighted split paces reproduce the Avg Pace
tile, Best is directly comparable to the splits, and the API refuses to ship
a response that fails those checks silently — indoor, outdoor, calibrated, or
not.

## Proposed Solution

Make the API the single derivation site. A new derivation stage in
`internal/activity` runs on `GET /activities/{id}`: from the stored
(calibration-rescaled) trackpoints and summary it computes the splits table,
a pace-strip summary (fastest/slowest clean pace, dropout count), per-point
clean-pace flags, interval segments, and a display-unit Best pace — then an
**invariant gate** checks the assembled response against itself (sums,
identities, orderings; see Algorithms) and logs at ERROR if anything fails
to reconcile, before returning it.

The derivation policy changes in one deliberate way: **a split's pace is its
total time over its total distance** — all segments count, including slow and
non-advancing ones. "Dropout" samples remain flagged so the chart can bridge
them visually, but they no longer distort split or summary arithmetic. This
is what makes the page reconcilable by construction rather than by luck.

The web page becomes render-only: it deletes its splits/strip/interval
derivation and draws the server blocks verbatim, with all pace formatting
funneled through one formatter. The Best tile becomes the fastest rolling
window of one display unit (mile or km), so `Best ≤ fastest split` always
holds and the tile finally relates to the table under it.

## Goals and Non-Goals

### Goals

- Every split row satisfies `pace = time ÷ distance` exactly (one rounding step).
- Split times sum to the Time tile; split distances sum to the Distance tile.
- Distance-weighted average of split paces equals the Avg Pace tile.
- Best tile = fastest rolling display-unit window, recomputed (not scaled) after calibration; `chart fastest ≤ Best ≤ fastest split` holds.
- Splits, strip summary, intervals, and clean-pace flags are computed server-side from the calibrated trackpoint stream; web renders them verbatim.
- An invariant gate validates every detail response (including calibrated indoor runs) and fails loudly in tests, ERROR-logs in production.
- One pace formatter on the web (`lib/pace-format.ts`); the three ad-hoc copies are removed.

### Non-Goals

- Changing calibration mechanics (factor bounds, compounding, reset semantics, best-effort regeneration) — shipped in [`treadmill-run-calibration.md`](treadmill-run-calibration.md).
- Rescaling calories on calibration. Calories are device-measured (HR-based), not distance-derived; leaving them fixed is correct and documented here as deliberate.
- Changing HR-zone semantics. Zones remain time-based (calibration-invariant); their total is HR-covered time, not run duration — the gate checks `≤ duration`, not equality.
- Mobile UI work. Mobile parity later consumes the new response blocks instead of porting `running-splits.ts`; nothing here blocks on mobile.
- MCP/agent changes. The response additions are additive; forwarders pass them through unchanged.
- Route geometry, list-view or dashboard metric changes.

## Implementation Details

### Data Model

No schema changes. `activities` and `activity_trackpoints` are untouched;
`raw_distance_meters` / `environment` (migration 036) already carry
calibration state. Stored `best_pace_sec_per_km` keeps its ingest-time
meaning (fastest rolling 1 km, metric) for any non-detail consumer; the
detail response's Best is derived at read time.

### Read Path (derivation stage)

`GET /activities/{id}` gains a `unit` query param (`mi` | `km`, default
`mi`), following the client-supplies-locale convention used by the
timezone/local-date list endpoints. For running activities the handler runs
the derivation stage after loading trackpoints:

1. **Segments**: consecutive downsampled trackpoint pairs → `{dDist, dTime,
   clean}`. Nothing is discarded: `dDist ≤ 0` pairs contribute their `dTime`
   (and zero distance) to the bucket of their start point.
2. **Splits**: bucket segments by `floor(startDistance / bucketMeters)`
   (bucket = 1609.344 m or 1000 m per `unit`). Per bucket: distance = Σ dDist,
   time = Σ dTime, `pace = time / distance` (all segments — the policy
   change), avg HR over segment endpoints carrying HR, elevation Δ =
   last − first. Fastest/slowest tags over full (≥ 95 %) buckets when ≥ 2
   exist.
3. **Clean-pace flags**: each trackpoint DTO gains `clean_pace: bool` —
   present, positive, and ≤ the dropout threshold (410 s/km, now a server
   constant). The chart draws gaps where false; the client no longer owns the
   threshold.
4. **Strip summary**: `fastest` / `slowest` over clean samples only (the
   chart header describes the drawn line, which visibly has gaps),
   `dropout_count` = trackpoints with a pace sample that is not clean.
5. **Best (display unit)**: fastest rolling `bucketMeters` window over the
   trackpoint stream — the existing rolling-window algorithm generalized to
   the unit. Serialized as `best_pace_sec_per_unit`; the stored metric field
   stays in the DTO for compatibility.
6. **Intervals**: the warmup/work/recovery/cooldown detection moves from
   `lib/running-splits.ts` to the derivation stage unchanged in behavior,
   consuming the same splits/segments it does today.

The pace strip itself is **not** duplicated as a parallel array: chart points
are a pure presentation mapping of the already-served trackpoints (distance ÷
bucketMeters on x, unit-converted pace on y). Every *number* rendered as text
(header fastest/slowest, dropout count) comes from `strip_summary`. This
keeps the payload flat — the detail response already carries ~300 trackpoints
and is forwarded through MCP to the agent, where doubling point data has real
token cost.

### Write Path (calibration fix)

`Calibrate` (sqlite_repository.go) stops scaling `best_pace_sec_per_km` by
`÷ f` and instead recomputes the fastest rolling 1 km from the rescaled
trackpoints inside the same transaction. Uniform scaling moves which window
is fastest, so the scale shortcut is an approximation; recomputing makes the
stored value exact and lets the gate assert it. `ChangeEnvironment` is
unchanged.

### Algorithms — the invariant gate

After assembling the detail DTO for a running activity, verify (tolerances
absorb float/rounding; paces compared in sec-per-unit):

```
I1  |Σ split.distance − distance_meters|            ≤ 1 m
I2  |Σ split.time − duration_seconds|               ≤ 1 s
I3  ∀ split: |pace − time/distance|                 ≤ 0.5 s   (identity)
I4  |Σ(pace·dist)/Σdist − avg_pace|                 ≤ 1 s     (weighted mean)
I5  |avg_pace_sec_per_km − duration/(dist/1000)|    ≤ 0.5 s   (stored vs recomputed)
I6  strip.fastest ≤ best_pace_sec_per_unit ≤ fastest-split pace  (+1 s slack;
    window-size ordering: sample ⊆ rolling unit ⊆ aligned full split)
I7  Σ zone.time_seconds ≤ duration + 1 s; |Σ zone.time_pct − 1| ≤ 0.01
```

On violation: log at ERROR with activity ID, invariant name, got/want — and
**still serve the response**. A read must never 500 over an accounting
mismatch; the gate's teeth are in CI, where table-driven tests (outdoor GPS
fixture, `treadmill_5k.tcx`, calibrated + reset, dropout-heavy, missing-HR)
assert zero violations, plus a fuzz/property test that random valid tracks
always pass the gate.

### API Surface

`GET /activities/{id}?unit=mi|km` (default `mi`). Additive response fields
for running activities:

```jsonc
{
  "splits": [{ "index": 1, "distance_meters": 1609.3, "duration_seconds": 626,
               "pace_sec_per_unit": 626.0, "avg_hr_bpm": 132,
               "elevation_delta_meters": null, "partial": false,
               "fastest": false, "slowest": false }],
  "strip_summary": { "fastest_sec_per_unit": 513, "slowest_sec_per_unit": 656,
                      "dropout_count": 8 },
  "best_pace_sec_per_unit": 547.0,
  "intervals": null,                      // or IntervalSegment[]
  "unit": "mi",
  "trackpoints": [{ /* existing fields */, "clean_pace": true }]
}
```

`PATCH` (environment) and `POST /calibrate` return the same enriched detail
shape they return today, now including the derived blocks, so the client can
keep replacing the whole session object.

### Web

- `app/(app)/running/[id]/page.tsx` passes the active unit to
  `getRunningSession` and re-fetches when the unit toggles (the toggle
  already round-trips a `PATCH /me`).
- `SplitsSpine`, `PaceRecap`, and the tiles render server blocks verbatim.
  `deriveRunningActivity`, `buildSplits`, `buildPaceStrip`, and interval
  detection are deleted from `lib/running-splits.ts`; `parseTargetPace`
  (plan-text parsing, no trackpoint math) stays.
- Pace formatting consolidates on `lib/pace-format.ts`; the local copies in
  `SplitsSpine` and `distance-unit-context` delegate to it so a value cannot
  render off-by-one between two widgets.

### Backfill or Migration

None required — no schema change, and derivation happens at read time.
Previously calibrated indoor runs may carry a slightly-off stored
`best_pace_sec_per_km` from the `÷ f` shortcut; data is pre-launch dev data,
and any subsequent calibrate/reset recomputes it exactly. No backfill job.

## Open Questions

1. **Default `unit` when the param is absent.** Options: hardcode `mi`
   (matches current UI default) or read the user's stored distance-unit
   preference. Lean: hardcode `mi`; the client always sends the param, and a
   profile lookup on every detail read buys nothing today.
2. **Should the gate also run in `POST /calibrate` responses?** It assembles
   the same DTO, so yes at zero extra cost. Lean: run it on every assembled
   running detail DTO regardless of entry path.
