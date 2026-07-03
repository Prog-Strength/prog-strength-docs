# Running Detail Metric Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make every number on the running detail page derive from one server-side computation over the calibrated trackpoint stream — splits, strip summary, intervals, unit-aware best pace — verified by an invariant gate before the response returns; the web page becomes render-only.

**Architecture:** A new pure derivation module in `internal/activity` (`derivation.go`) consumes the stored (already calibration-rescaled) `[]Trackpoint` plus a display unit and produces splits (pace = total time ÷ total distance over ALL segments — the policy change), a strip summary (fastest/slowest clean pace + dropout count), a fastest-rolling-display-unit best pace, and ported interval detection. `GET /activities/{id}` (and `POST /calibrate`) gain a `?unit=mi|km` param; `buildDetailDTO` attaches the derived blocks and runs `checkDetailInvariants` (`invariants.go`), which ERROR-logs violations but never fails the response. `Calibrate` recomputes stored best pace from the rescaled trackpoints instead of scaling by ÷f. The web deletes `deriveRunningActivity`/`buildSplits`/`detectIntervals`, renders server blocks verbatim, refetches on unit toggle, and funnels all pace formatting through `lib/pace-format.ts`.

**Tech Stack:** Go 1.25 (chi, SQLite/go-sqlite3), Next.js 16 / React 19 / TypeScript / Vitest.

**Cross-repo dependency order:** `prog-strength-api` is the contract and must land first. `prog-strength-web` consumes it. `prog-strength-docs` flips the SOW status last. MCP/agent are untouched (additive response fields pass through the forwarder).

**Spec:** `prog-strength-docs/sows/running-detail-metric-alignment.md` (PR #186).

**Conventions for the implementer (API):** comments explain *why*; one concept per file in domain packages; `httpresp` envelope; TDD. Lint with the CI-pinned golangci-lint: `go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run ./...`. Tests: `go test ./internal/activity/...`.
**Conventions (web):** design-system v0.4 tokens only; `npm test` (vitest run), `npm run typecheck`, `npm run lint`.

---

## Repo: prog-strength-api

### Task A1: Derivation module — unit, segments, splits

**Files:**
- Create: `internal/activity/derivation.go`
- Test: `internal/activity/derivation_test.go`

The heart of the SOW: splits whose pace is total time ÷ total distance over ALL consecutive-pair segments (nothing discarded), bucketed by the segment's start point. Distance deltas are taken as-is (can be ≤ 0 on garbage cumulative streams) so split distances telescope exactly to `last − first`; time deltas are clamped at 0.

- [ ] **Step 1: Write the failing tests**

```go
package activity

import (
	"math"
	"testing"
)

// tp builds a Trackpoint for derivation tests. Pace is per-km; nil = no sample.
func tp(seq, elapsed int, dist float64, pace *float64, hr *int, elev *float64) Trackpoint {
	return Trackpoint{Sequence: seq, ElapsedSeconds: elapsed, DistanceMeters: dist,
		PaceSecPerKm: pace, HeartRateBpm: hr, ElevationMeters: elev}
}

func fp(v float64) *float64 { return &v }
func ip(v int) *int         { return &v }

// steadyTrack builds an evenly-paced track: n+1 points, stepMeters apart,
// stepSeconds apart, all with a clean pace sample.
func steadyTrack(n int, stepMeters float64, stepSeconds int) []Trackpoint {
	tps := make([]Trackpoint, 0, n+1)
	for i := 0; i <= n; i++ {
		var pace *float64
		if i > 0 {
			pace = fp(float64(stepSeconds) / (stepMeters / 1000))
		}
		tps = append(tps, tp(i, i*stepSeconds, float64(i)*stepMeters, pace, nil, nil))
	}
	return tps
}

// TestDeriveRunning_SplitsReconcileByConstruction is the SOW's core promise:
// split distances sum to the stream span, split times sum to the duration,
// each split's pace IS its time over its distance, and the distance-weighted
// mean of split paces reproduces total time / total distance.
func TestDeriveRunning_SplitsReconcileByConstruction(t *testing.T) {
	// 5.3 "km" at 100 m / 30 s steps => 53 segments, 5300 m in 1590 s.
	tps := steadyTrack(53, 100, 30)
	d := deriveRunning(tps, UnitKm)

	if len(d.Splits) != 6 { // 5 full km + 300 m partial
		t.Fatalf("splits = %d, want 6", len(d.Splits))
	}
	var sumDist float64
	var sumTime int
	for _, s := range d.Splits {
		sumDist += s.DistanceMeters
		sumTime += s.DurationSeconds
		if s.PaceSecPerUnit == nil {
			t.Fatalf("split %d has nil pace", s.Index)
		}
		// I3: pace ≡ (time/dist) * bucket, exactly (one float op apart).
		want := float64(s.DurationSeconds) / s.DistanceMeters * 1000
		if math.Abs(*s.PaceSecPerUnit-want) > 0.5 {
			t.Errorf("split %d pace = %.2f, want %.2f", s.Index, *s.PaceSecPerUnit, want)
		}
	}
	if math.Abs(sumDist-5300) > 0.01 {
		t.Errorf("sum split dist = %.2f, want 5300", sumDist)
	}
	if sumTime != 1590 {
		t.Errorf("sum split time = %d, want 1590", sumTime)
	}
	last := d.Splits[len(d.Splits)-1]
	if !last.Partial {
		t.Error("trailing 300 m split should be partial")
	}
}

// TestDeriveRunning_DropoutSegmentsStayInSplitMath is the policy change: a
// mid-run stationary stretch (pace sample nil) still contributes its time, so
// the containing split's pace slows accordingly instead of pretending the
// stop never happened.
func TestDeriveRunning_DropoutSegmentsStayInSplitMath(t *testing.T) {
	// 1 km in 300 s, then 60 s stationary (no distance, nil pace), then 1 km in 300 s.
	tps := []Trackpoint{
		tp(0, 0, 0, nil, nil, nil),
		tp(1, 300, 1000, fp(300), nil, nil),
		tp(2, 360, 1000, nil, nil, nil),   // stationary: dDist=0, dTime=60
		tp(3, 660, 2000, fp(300), nil, nil),
	}
	d := deriveRunning(tps, UnitKm)
	if len(d.Splits) != 2 {
		t.Fatalf("splits = %d, want 2", len(d.Splits))
	}
	// The stationary segment starts at exactly 1000 m => bucket 1. Split 2
	// carries 1000 m in 360 s => pace 360 s/km, NOT 300.
	s2 := d.Splits[1]
	if s2.DurationSeconds != 360 {
		t.Errorf("split 2 time = %d, want 360", s2.DurationSeconds)
	}
	if s2.PaceSecPerUnit == nil || math.Abs(*s2.PaceSecPerUnit-360) > 0.5 {
		t.Errorf("split 2 pace = %v, want 360", s2.PaceSecPerUnit)
	}
	// Sum of split times still equals the full elapsed duration.
	if got := d.Splits[0].DurationSeconds + s2.DurationSeconds; got != 660 {
		t.Errorf("sum split time = %d, want 660", got)
	}
}

// TestDeriveRunning_MileBuckets checks the unit switch: same track, mile
// buckets, pace expressed per mile.
func TestDeriveRunning_MileBuckets(t *testing.T) {
	// 2 miles at 3218.688 m, steps of 160.9344 m / 60 s => 600 s per mile.
	tps := steadyTrack(20, 160.9344, 60)
	d := deriveRunning(tps, UnitMiles)
	if len(d.Splits) != 2 {
		t.Fatalf("splits = %d, want 2", len(d.Splits))
	}
	for _, s := range d.Splits {
		if s.PaceSecPerUnit == nil || math.Abs(*s.PaceSecPerUnit-600) > 0.5 {
			t.Errorf("split %d pace = %v, want 600 s/mi", s.Index, s.PaceSecPerUnit)
		}
	}
}

// TestDeriveRunning_FastestSlowestTags: tags only among full splits, only
// when at least two full splits exist.
func TestDeriveRunning_FastestSlowestTags(t *testing.T) {
	// km 1 in 360 s, km 2 in 300 s, 200 m tail in 80 s.
	tps := []Trackpoint{
		tp(0, 0, 0, nil, nil, nil),
		tp(1, 360, 1000, fp(360), nil, nil),
		tp(2, 660, 2000, fp(300), nil, nil),
		tp(3, 740, 2200, fp(400), nil, nil),
	}
	d := deriveRunning(tps, UnitKm)
	if len(d.Splits) != 3 {
		t.Fatalf("splits = %d, want 3", len(d.Splits))
	}
	if !d.Splits[1].Fastest || d.Splits[0].Fastest {
		t.Error("km 2 should be tagged fastest")
	}
	if !d.Splits[0].Slowest || d.Splits[2].Slowest {
		t.Error("km 1 should be tagged slowest; partial never tagged")
	}

	// A single full split gets no tags.
	one := deriveRunning(steadyTrack(10, 100, 30), UnitKm) // 1 km exactly
	for _, s := range one.Splits {
		if s.Fastest || s.Slowest {
			t.Error("tags require >= 2 full splits")
		}
	}
}

// TestDeriveRunning_HRAndElevationPerSplit: split HR is the mean of
// segment-endpoint HRs; elevation delta is last-minus-first in the bucket.
func TestDeriveRunning_HRAndElevationPerSplit(t *testing.T) {
	tps := []Trackpoint{
		tp(0, 0, 0, nil, nil, fp(100)),
		tp(1, 300, 500, fp(600), ip(130), fp(104)),
		tp(2, 600, 1000, fp(600), ip(150), fp(102)),
	}
	d := deriveRunning(tps, UnitKm)
	if len(d.Splits) != 1 {
		t.Fatalf("splits = %d, want 1", len(d.Splits))
	}
	s := d.Splits[0]
	if s.AvgHRBpm == nil || math.Abs(*s.AvgHRBpm-140) > 0.01 {
		t.Errorf("avg hr = %v, want 140", s.AvgHRBpm)
	}
	if s.ElevDeltaMeters == nil || math.Abs(*s.ElevDeltaMeters-(-2)) > 0.01 {
		t.Errorf("elev delta = %v, want -2 (104 -> 102 across segment endpoints)", s.ElevDeltaMeters)
	}
}

// TestDeriveRunning_Degenerate: empty and single-point tracks derive to
// nothing rather than panicking.
func TestDeriveRunning_Degenerate(t *testing.T) {
	for _, tps := range [][]Trackpoint{nil, {tp(0, 0, 0, nil, nil, nil)}} {
		d := deriveRunning(tps, UnitMiles)
		if len(d.Splits) != 0 {
			t.Errorf("degenerate track produced %d splits", len(d.Splits))
		}
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `go test ./internal/activity/ -run TestDeriveRunning -v`
Expected: FAIL — `undefined: deriveRunning`, `undefined: UnitKm`, etc.

- [ ] **Step 3: Implement `derivation.go` (segments + splits only; strip/best/intervals stubs come in A2/A3)**

```go
package activity

import "math"

// This file is the read-time derivation for the running detail response: the
// single computation every rendered number on /running/[id] traces back to.
// Policy (SOW running-detail-metric-alignment): a split's pace is its TOTAL
// time over its TOTAL distance — dropout/stationary segments are not excluded
// from split math (they remain a chart-rendering concern via clean-pace
// flags), so split rows, the duration/distance tiles, and the avg-pace tile
// reconcile by construction. checkDetailInvariants (invariants.go) verifies
// exactly that on every assembled response.

// DistanceUnit selects the split-bucket length (and pace denominator) for the
// read-time derivation. The client passes ?unit=; splits are inherently
// unit-shaped (mile rows vs km rows), so this cannot be a render-side concern.
type DistanceUnit string

const (
	UnitMiles DistanceUnit = "mi"
	UnitKm    DistanceUnit = "km"
)

const metersPerMile = 1609.344

func (u DistanceUnit) Valid() bool { return u == UnitMiles || u == UnitKm }

// BucketMeters is one display unit expressed in meters.
func (u DistanceUnit) BucketMeters() float64 {
	if u == UnitMiles {
		return metersPerMile
	}
	return 1000
}

// paceDropoutSecPerKm mirrors the web's former PACE_DROPOUT_SEC_PER_KM: a
// per-point pace sample slower than this is treated as a device dropout —
// flagged (clean_pace=false) so the chart renders a gap, and excluded from
// the strip summary's fastest/slowest. It no longer excludes anything from
// split or summary arithmetic.
const paceDropoutSecPerKm = 410.0

// isCleanTrackpointPace reports whether a per-point pace sample is plottable:
// present, positive, and not slower than the dropout threshold.
func isCleanTrackpointPace(paceSecPerKm *float64) bool {
	return paceSecPerKm != nil && *paceSecPerKm > 0 && *paceSecPerKm <= paceDropoutSecPerKm
}

// Split is one distance bucket of the run. PaceSecPerUnit is nil only when
// the bucket covered no distance (pure stationary tail).
type Split struct {
	Index           int
	Partial         bool
	DistanceMeters  float64
	DurationSeconds int
	PaceSecPerUnit  *float64
	AvgHRBpm        *float64
	ElevDeltaMeters *float64
	Fastest         bool
	Slowest         bool
}

// StripSummary carries the pace-chart header numbers: min/max over CLEAN
// samples (the header describes the drawn line, which has gaps) and how many
// pace-carrying samples were flagged as dropouts.
type StripSummary struct {
	FastestSecPerUnit *float64
	SlowestSecPerUnit *float64
	DropoutCount      int
}

// IntervalSegment is one labeled bout of a detected interval workout.
type IntervalSegment struct {
	Kind            string // "warmup" | "work" | "recovery" | "cooldown"
	Rep             *int
	Label           string
	DistanceMeters  float64
	DurationSeconds int
	PaceSecPerUnit  *float64
	AvgHRBpm        *float64
}

// Derivation is everything the detail page renders beyond the raw summary
// fields, computed in one pass from the stored trackpoints.
type Derivation struct {
	Splits             []Split
	StripSummary       StripSummary
	BestPaceSecPerUnit *float64
	// Intervals is nil unless the workout confidently looks like intervals;
	// the client additionally gates display on the linked plan's run type.
	Intervals []IntervalSegment
}

// segment is one consecutive-pair slice of the track. dDist is taken as-is
// (a non-monotonic cumulative stream yields <= 0) so bucket distances
// telescope exactly to last-minus-first; dTime is clamped at 0.
type segment struct {
	dDist  float64
	dTime  int
	clean  bool
	bucket int
	hr     *int
	elev   *float64
}

func buildSegments(tps []Trackpoint, bucketMeters float64) []segment {
	if len(tps) < 2 {
		return nil
	}
	segs := make([]segment, 0, len(tps)-1)
	for i := 1; i < len(tps); i++ {
		a, b := tps[i-1], tps[i]
		dTime := b.ElapsedSeconds - a.ElapsedSeconds
		if dTime < 0 {
			dTime = 0
		}
		segs = append(segs, segment{
			dDist:  b.DistanceMeters - a.DistanceMeters,
			dTime:  dTime,
			clean:  isCleanTrackpointPace(b.PaceSecPerKm),
			bucket: int(math.Floor(a.DistanceMeters / bucketMeters)),
			hr:     b.HeartRateBpm,
			elev:   b.ElevationMeters,
		})
	}
	return segs
}

// deriveRunning computes the full detail derivation for a running activity.
func deriveRunning(tps []Trackpoint, unit DistanceUnit) Derivation {
	bucketMeters := unit.BucketMeters()
	segs := buildSegments(tps, bucketMeters)
	return Derivation{
		Splits:             buildDerivedSplits(segs, bucketMeters),
		StripSummary:       buildStripSummary(tps, bucketMeters), // Task A2
		BestPaceSecPerUnit: bestRollingPace(tps, bucketMeters),   // Task A2
		Intervals:          detectIntervals(segs, bucketMeters),  // Task A3
	}
}

// splitAcc folds segments into one bucket.
type splitAcc struct {
	dist      float64
	timeSec   int
	hrSum     float64
	hrCount   int
	firstElev *float64
	lastElev  *float64
}

func buildDerivedSplits(segs []segment, bucketMeters float64) []Split {
	if len(segs) == 0 {
		return nil
	}
	byBucket := map[int]*splitAcc{}
	var order []int
	for _, s := range segs {
		acc, ok := byBucket[s.bucket]
		if !ok {
			acc = &splitAcc{}
			byBucket[s.bucket] = acc
			order = append(order, s.bucket)
		}
		acc.dist += s.dDist
		acc.timeSec += s.dTime
		if s.hr != nil {
			acc.hrSum += float64(*s.hr)
			acc.hrCount++
		}
		if s.elev != nil {
			if acc.firstElev == nil {
				acc.firstElev = s.elev
			}
			acc.lastElev = s.elev
		}
	}
	// Buckets appear in stream order, which is ascending for any monotonic
	// track; sort anyway so a jittery stream can't reorder rows.
	for i := 1; i < len(order); i++ {
		for j := i; j > 0 && order[j] < order[j-1]; j-- {
			order[j], order[j-1] = order[j-1], order[j]
		}
	}
	splits := make([]Split, 0, len(order))
	for i, b := range order {
		acc := byBucket[b]
		sp := Split{
			Index:           i,
			DistanceMeters:  acc.dist,
			DurationSeconds: acc.timeSec,
		}
		if acc.dist > 0 {
			// THE policy line: pace is total time over total distance.
			pace := float64(acc.timeSec) / acc.dist * bucketMeters
			sp.PaceSecPerUnit = &pace
		}
		if acc.hrCount > 0 {
			avg := acc.hrSum / float64(acc.hrCount)
			sp.AvgHRBpm = &avg
		}
		if acc.firstElev != nil && acc.lastElev != nil {
			d := *acc.lastElev - *acc.firstElev
			sp.ElevDeltaMeters = &d
		}
		splits = append(splits, sp)
	}
	if n := len(splits); n > 0 && splits[n-1].DistanceMeters < bucketMeters*0.95 {
		splits[n-1].Partial = true
	}
	// Fastest/slowest among FULL splits with a pace — only when >= 2 exist.
	var fastest, slowest *Split
	full := 0
	for i := range splits {
		s := &splits[i]
		if s.Partial || s.PaceSecPerUnit == nil {
			continue
		}
		full++
		if fastest == nil || *s.PaceSecPerUnit < *fastest.PaceSecPerUnit {
			fastest = s
		}
		if slowest == nil || *s.PaceSecPerUnit > *slowest.PaceSecPerUnit {
			slowest = s
		}
	}
	if full >= 2 {
		fastest.Fastest = true
		slowest.Slowest = true
	}
	return splits
}
```

For this step only, add temporary stubs at the bottom of `derivation.go` so it compiles (replaced in A2/A3):

```go
func buildStripSummary(tps []Trackpoint, bucketMeters float64) StripSummary { return StripSummary{} }
func bestRollingPace(tps []Trackpoint, windowMeters float64) *float64       { return nil }
func detectIntervals(segs []segment, bucketMeters float64) []IntervalSegment { return nil }
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `go test ./internal/activity/ -run TestDeriveRunning -v`
Expected: PASS (all six tests)

- [ ] **Step 5: Commit**

```bash
git add internal/activity/derivation.go internal/activity/derivation_test.go
git commit -m "feat(activity): read-time split derivation with time-over-distance pace policy"
```

### Task A2: Strip summary + rolling best pace

**Files:**
- Modify: `internal/activity/derivation.go` (replace the A1 stubs)
- Test: `internal/activity/derivation_test.go`

`bestRollingPace` is the stored-trackpoint generalization of the summarizer's `bestPace` (`tcx_summarizer.go:157-179`): same distance-anchored sliding window, parameterized by window length, returning seconds per window. It serves double duty: display-unit BEST here, and the exact 1 km recompute inside `Calibrate` (Task A6).

- [ ] **Step 1: Write the failing tests (append to `derivation_test.go`)**

```go
// TestBestRollingPace_MatchesWindowAndOrdering: the fastest rolling
// display-unit window is at most the fastest full split's pace (any aligned
// full bucket is itself a candidate window), and at least the fastest single
// clean sample.
func TestBestRollingPace_MatchesWindowAndOrdering(t *testing.T) {
	// km 1 at 360 s, km 2 at 300 s: fastest rolling km = the second km.
	tps := []Trackpoint{
		tp(0, 0, 0, nil, nil, nil),
		tp(1, 180, 500, fp(360), nil, nil),
		tp(2, 360, 1000, fp(360), nil, nil),
		tp(3, 510, 1500, fp(300), nil, nil),
		tp(4, 660, 2000, fp(300), nil, nil),
	}
	best := bestRollingPace(tps, 1000)
	if best == nil || math.Abs(*best-300) > 0.5 {
		t.Fatalf("best rolling km = %v, want 300", best)
	}
	// Too short for the window => nil.
	if got := bestRollingPace(tps[:2], 1000); got != nil {
		t.Errorf("sub-window track best = %v, want nil", got)
	}
	if got := bestRollingPace(nil, 1000); got != nil {
		t.Errorf("nil track best = %v, want nil", got)
	}
}

// TestBuildStripSummary_CleanMinMaxAndDropoutCount: fastest/slowest are over
// clean samples only, expressed per display unit; dropout_count counts
// pace-carrying samples flagged non-clean; nil-pace points count for neither.
func TestBuildStripSummary_CleanMinMaxAndDropoutCount(t *testing.T) {
	tps := []Trackpoint{
		tp(0, 0, 0, nil, nil, nil),          // no sample: not a dropout
		tp(1, 300, 1000, fp(300), nil, nil), // clean
		tp(2, 900, 1500, fp(1200), nil, nil), // dropout (>410)
		tp(3, 1260, 2500, fp(360), nil, nil), // clean
	}
	s := buildStripSummary(tps, 1000)
	if s.FastestSecPerUnit == nil || math.Abs(*s.FastestSecPerUnit-300) > 0.01 {
		t.Errorf("fastest = %v, want 300", s.FastestSecPerUnit)
	}
	if s.SlowestSecPerUnit == nil || math.Abs(*s.SlowestSecPerUnit-360) > 0.01 {
		t.Errorf("slowest = %v, want 360", s.SlowestSecPerUnit)
	}
	if s.DropoutCount != 1 {
		t.Errorf("dropout_count = %d, want 1", s.DropoutCount)
	}

	// Mile unit converts the same sec/km values to sec/mi.
	mi := buildStripSummary(tps, metersPerMile)
	if mi.FastestSecPerUnit == nil || math.Abs(*mi.FastestSecPerUnit-300*1.609344) > 0.01 {
		t.Errorf("mi fastest = %v, want %.2f", mi.FastestSecPerUnit, 300*1.609344)
	}

	// No clean samples => nil min/max.
	none := buildStripSummary([]Trackpoint{tp(0, 0, 0, nil, nil, nil)}, 1000)
	if none.FastestSecPerUnit != nil || none.SlowestSecPerUnit != nil || none.DropoutCount != 0 {
		t.Errorf("empty strip summary = %+v", none)
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `go test ./internal/activity/ -run 'TestBestRollingPace|TestBuildStripSummary' -v`
Expected: FAIL (stubs return zero values)

- [ ] **Step 3: Replace the two stubs in `derivation.go`**

```go
// buildStripSummary computes the pace-chart header numbers over CLEAN samples
// (the header describes the drawn line, which breaks at dropouts) plus the
// dropout count. Values are per display unit: a stored sec/km sample scaled
// by bucketMeters/1000.
func buildStripSummary(tps []Trackpoint, bucketMeters float64) StripSummary {
	var s StripSummary
	toUnit := bucketMeters / 1000
	for _, t := range tps {
		if t.PaceSecPerKm == nil {
			continue
		}
		if !isCleanTrackpointPace(t.PaceSecPerKm) {
			s.DropoutCount++
			continue
		}
		v := *t.PaceSecPerKm * toUnit
		if s.FastestSecPerUnit == nil || v < *s.FastestSecPerUnit {
			f := v
			s.FastestSecPerUnit = &f
		}
		if s.SlowestSecPerUnit == nil || v > *s.SlowestSecPerUnit {
			sl := v
			s.SlowestSecPerUnit = &sl
		}
	}
	return s
}

// bestRollingPace finds the fastest rolling windowMeters window over the
// stored trackpoints — the stored-stream generalization of the summarizer's
// ingest-time bestPace, returning seconds per windowMeters. Distance-anchored
// so a single noisy sample is diluted across the window. Nil when the track
// spans less than one window.
func bestRollingPace(tps []Trackpoint, windowMeters float64) *float64 {
	if len(tps) == 0 || tps[len(tps)-1].DistanceMeters-tps[0].DistanceMeters < windowMeters {
		return nil
	}
	best := math.Inf(1)
	left := 0
	for right := 0; right < len(tps); right++ {
		for left+1 < right && tps[right].DistanceMeters-tps[left+1].DistanceMeters >= windowMeters {
			left++
		}
		span := tps[right].DistanceMeters - tps[left].DistanceMeters
		if span >= windowMeters {
			elapsed := float64(tps[right].ElapsedSeconds - tps[left].ElapsedSeconds)
			if pace := elapsed / (span / windowMeters); pace < best {
				best = pace
			}
		}
	}
	if math.IsInf(best, 1) {
		return nil
	}
	return &best
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `go test ./internal/activity/ -run 'TestBestRollingPace|TestBuildStripSummary|TestDeriveRunning' -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add internal/activity/derivation.go internal/activity/derivation_test.go
git commit -m "feat(activity): strip summary and rolling display-unit best pace"
```

### Task A3: Port interval detection to Go

**Files:**
- Modify: `internal/activity/derivation.go` (replace the `detectIntervals` stub)
- Test: `internal/activity/derivation_test.go`

Behavior-preserving port of `detectIntervals`/`coalesce`/`denoise`/`labelBouts` from `prog-strength-web/lib/running-splits.ts:266-420`. Detection stays conservative (nil unless ≥3 alternating work bouts over ≥8 clean segments); the run-type gate moves to the CLIENT (it knows the linked plan), which only shows the intervals table when the plan says intervals AND the response carries them.

- [ ] **Step 1: Write the failing tests (append to `derivation_test.go`)**

```go
// intervalsTrack builds warmup + n×(work,recovery) + cooldown with clean
// samples throughout. Work pace 240 s/km, recovery/warmup/cooldown 420 —
// wait: 420 > 410 would flag dropout; use 400 s/km so everything stays clean.
func intervalsTrack(nReps int) []Trackpoint {
	tps := []Trackpoint{tp(0, 0, 0, nil, nil, nil)}
	dist, elapsed, seq := 0.0, 0, 1
	add := (func(meters float64, paceSecPerKm float64) {
		// 100 m steps so each bout is several segments long (survives denoise).
		steps := int(meters / 100)
		for i := 0; i < steps; i++ {
			dist += 100
			elapsed += int(paceSecPerKm * 0.1)
			p := paceSecPerKm
			tps = append(tps, tp(seq, elapsed, dist, &p, nil, nil))
			seq++
		}
	})
	add(800, 400) // warm-up
	for i := 0; i < nReps; i++ {
		add(400, 240) // work
		add(200, 400) // recovery
	}
	add(800, 400) // cool-down (the final recovery merges into it)
	return tps
}

// TestDetectIntervals_HappyPath: 4 reps detect with warmup/work/recovery/
// cooldown labels and per-bout pace = time/dist.
func TestDetectIntervals_HappyPath(t *testing.T) {
	segs := buildSegments(intervalsTrack(4), 1000)
	got := detectIntervals(segs, 1000)
	if got == nil {
		t.Fatal("expected detected intervals")
	}
	if got[0].Kind != "warmup" || got[0].Label != "Warm-up" {
		t.Errorf("first segment = %+v, want warm-up", got[0])
	}
	var reps int
	for _, s := range got {
		if s.Kind == "work" {
			reps++
			if s.Rep == nil || *s.Rep != reps {
				t.Errorf("work rep = %v, want %d", s.Rep, reps)
			}
			if s.PaceSecPerUnit == nil ||
				math.Abs(*s.PaceSecPerUnit-float64(s.DurationSeconds)/s.DistanceMeters*1000) > 0.5 {
				t.Errorf("work pace %v != time/dist", s.PaceSecPerUnit)
			}
		}
	}
	if reps != 4 {
		t.Errorf("work reps = %d, want 4", reps)
	}
	if last := got[len(got)-1]; last.Kind != "cooldown" {
		t.Errorf("last segment kind = %s, want cooldown", last.Kind)
	}
}

// TestDetectIntervals_ConservativeNil: steady runs and too-few-rep runs
// return nil rather than fabricating structure.
func TestDetectIntervals_ConservativeNil(t *testing.T) {
	if got := detectIntervals(buildSegments(steadyTrack(50, 100, 30), 1000), 1000); got != nil {
		t.Errorf("steady run detected intervals: %+v", got)
	}
	if got := detectIntervals(buildSegments(intervalsTrack(2), 1000), 1000); got != nil {
		t.Errorf("2-rep run detected intervals (needs >= 3): %+v", got)
	}
	if got := detectIntervals(nil, 1000); got != nil {
		t.Errorf("nil segments detected intervals: %+v", got)
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `go test ./internal/activity/ -run TestDetectIntervals -v`
Expected: FAIL (stub returns nil for the happy path)

- [ ] **Step 3: Replace the `detectIntervals` stub**

Port faithfully from `lib/running-splits.ts`; thresholds unchanged (rel ≤ 0.9 fast, ≥ 1.05 slow, neutral forward-fills, <60 m bouts denoise into the previous, ≥3 works, strict alternation):

```go
// bout is a coalesced run of same-class clean segments.
type bout struct {
	fast    bool
	dDist   float64
	dTime   int
	hrSum   float64
	hrCount int
}

// detectIntervals conservatively detects interval structure from CLEAN
// segments (detection heuristics predate the split-pace policy change and
// deliberately keep ignoring dropout segments — structure detection on
// device noise fabricates reps). Ported behavior-for-behavior from the web's
// lib/running-splits.ts. Returns nil unless >= 3 strictly-alternating work
// bouts emerge from >= 8 clean segments.
func detectIntervals(segs []segment, bucketMeters float64) []IntervalSegment {
	var clean []segment
	for _, s := range segs {
		if s.clean && s.dDist > 0 {
			clean = append(clean, s)
		}
	}
	if len(clean) < 8 {
		return nil
	}
	var totalDist float64
	var totalTime int
	for _, s := range clean {
		totalDist += s.dDist
		totalTime += s.dTime
	}
	if totalDist <= 0 {
		return nil
	}
	avgSecPerMeter := float64(totalTime) / totalDist

	// Classify each clean segment fast/slow, forward-filling the neutral band.
	classes := make([]bool, len(clean)) // true = fast
	prevFast := false
	for i, s := range clean {
		rel := (float64(s.dTime) / s.dDist) / avgSecPerMeter
		switch {
		case rel <= 0.9:
			prevFast = true
		case rel >= 1.05:
			prevFast = false
		}
		classes[i] = prevFast
	}

	bouts := coalesceBouts(clean, classes)
	bouts = denoiseBouts(bouts)

	// Plausibility: >= 3 work bouts, no two adjacent (strict alternation).
	var workIdx []int
	for i, b := range bouts {
		if b.fast {
			workIdx = append(workIdx, i)
		}
	}
	if len(workIdx) < 3 {
		return nil
	}
	for i := 1; i < len(workIdx); i++ {
		if workIdx[i] == workIdx[i-1]+1 {
			return nil
		}
	}
	return labelBouts(bouts, bucketMeters)
}

func coalesceBouts(clean []segment, classes []bool) []bout {
	var bouts []bout
	for i, s := range clean {
		if len(bouts) == 0 || bouts[len(bouts)-1].fast != classes[i] {
			bouts = append(bouts, bout{fast: classes[i]})
		}
		b := &bouts[len(bouts)-1]
		b.dDist += s.dDist
		b.dTime += s.dTime
		if s.hr != nil {
			b.hrSum += float64(*s.hr)
			b.hrCount++
		}
	}
	return bouts
}

// denoiseBouts folds any sub-60 m bout into its predecessor (keeping the
// predecessor's class), then re-merges neighbours left same-classed.
func denoiseBouts(bouts []bout) []bout {
	var merged []bout
	for _, b := range bouts {
		if n := len(merged); n > 0 && (b.dDist < 60 || merged[n-1].fast == b.fast) {
			p := &merged[n-1]
			p.dDist += b.dDist
			p.dTime += b.dTime
			p.hrSum += b.hrSum
			p.hrCount += b.hrCount
			continue
		}
		merged = append(merged, b)
	}
	return merged
}

func labelBouts(bouts []bout, bucketMeters float64) []IntervalSegment {
	firstWork, lastWork := -1, -1
	for i, b := range bouts {
		if b.fast {
			if firstWork < 0 {
				firstWork = i
			}
			lastWork = i
		}
	}
	var out []IntervalSegment
	rep := 0
	appendBout := func(kind, label string, repNum *int, b bout) {
		seg := IntervalSegment{
			Kind: kind, Rep: repNum, Label: label,
			DistanceMeters: b.dDist, DurationSeconds: b.dTime,
		}
		if b.dDist > 0 {
			pace := float64(b.dTime) / b.dDist * bucketMeters
			seg.PaceSecPerUnit = &pace
		}
		if b.hrCount > 0 {
			hr := b.hrSum / float64(b.hrCount)
			seg.AvgHRBpm = &hr
		}
		out = append(out, seg)
	}
	mergeSlow := func(bs []bout) (bout, bool) {
		var m bout
		found := false
		for _, b := range bs {
			if b.fast {
				continue
			}
			found = true
			m.dDist += b.dDist
			m.dTime += b.dTime
			m.hrSum += b.hrSum
			m.hrCount += b.hrCount
		}
		return m, found
	}
	if lead, ok := mergeSlow(bouts[:firstWork]); ok {
		appendBout("warmup", "Warm-up", nil, lead)
	}
	for i := firstWork; i <= lastWork; i++ {
		b := bouts[i]
		if b.fast {
			rep++
			r := rep
			appendBout("work", fmt.Sprintf("Rep %d", rep), &r, b)
		} else {
			r := rep
			appendBout("recovery", fmt.Sprintf("Recovery %d", rep), &r, b)
		}
	}
	if trail, ok := mergeSlow(bouts[lastWork+1:]); ok {
		appendBout("cooldown", "Cool-down", nil, trail)
	}
	return out
}
```

Add `"fmt"` to the imports of `derivation.go`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `go test ./internal/activity/ -run 'TestDetectIntervals|TestDeriveRunning' -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add internal/activity/derivation.go internal/activity/derivation_test.go
git commit -m "feat(activity): port conservative interval detection server-side"
```

### Task A4: Invariant gate

**Files:**
- Create: `internal/activity/invariants.go`
- Test: `internal/activity/invariants_test.go`

The SOW's gate: seven reconciliation checks over the assembled detail data. Returns a `[]string` of violations (empty = aligned); the handler ERROR-logs each but always serves the response — the gate's teeth are CI tests asserting emptiness.

- [ ] **Step 1: Write the failing tests**

```go
package activity

import (
	"strings"
	"testing"
)

// alignedActivity builds an Activity + Derivation pair that satisfies every
// invariant: steady 5.3 km, summary fields recomputed from the same stream.
func alignedActivity(t *testing.T) (Activity, Derivation) {
	t.Helper()
	tps := steadyTrack(53, 100, 30) // 5300 m in 1590 s
	avg := 1590.0 / 5.3
	best := bestRollingPace(tps, 1000)
	a := Activity{
		ActivityType:    ActivityRunning,
		DistanceMeters:  5300,
		DurationSeconds: 1590,
		AvgPaceSecPerKm: &avg,
		Trackpoints:     tps,
	}
	_ = best
	return a, deriveRunning(tps, UnitKm)
}

func TestCheckDetailInvariants_AlignedIsClean(t *testing.T) {
	a, d := alignedActivity(t)
	if v := checkDetailInvariants(a, d, UnitKm, nil); len(v) != 0 {
		t.Fatalf("aligned activity reported violations: %v", v)
	}
}

// Each mutation below breaks exactly the invariant named in the violation.
func TestCheckDetailInvariants_CatchesDrift(t *testing.T) {
	cases := []struct {
		name    string
		mutate  func(a *Activity, d *Derivation)
		wantSub string
	}{
		{"stored distance drifts from stream", func(a *Activity, d *Derivation) {
			a.DistanceMeters += 50
		}, "I1"},
		{"split time drops a segment", func(a *Activity, d *Derivation) {
			d.Splits[0].DurationSeconds -= 30
		}, "I2"},
		{"split pace not time-over-dist", func(a *Activity, d *Derivation) {
			p := *d.Splits[0].PaceSecPerUnit + 20
			d.Splits[0].PaceSecPerUnit = &p
		}, "I3"},
		{"stored avg pace stale", func(a *Activity, d *Derivation) {
			p := *a.AvgPaceSecPerKm + 10
			a.AvgPaceSecPerKm = &p
		}, "I5"},
		{"best slower than fastest split", func(a *Activity, d *Derivation) {
			p := 1e9
			d.BestPaceSecPerUnit = &p
		}, "I6"},
	}
	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			a, d := alignedActivity(t)
			tc.mutate(&a, &d)
			v := checkDetailInvariants(a, d, UnitKm, nil)
			if len(v) == 0 {
				t.Fatal("expected a violation")
			}
			found := false
			for _, s := range v {
				if strings.Contains(s, tc.wantSub) {
					found = true
				}
			}
			if !found {
				t.Errorf("violations %v missing %q", v, tc.wantSub)
			}
		})
	}
}

// TestCheckDetailInvariants_HRZones: zone seconds must fit inside the
// duration and percentages must sum to ~1.
func TestCheckDetailInvariants_HRZones(t *testing.T) {
	a, d := alignedActivity(t)
	good := &heartRateZonesDTO{TotalHRSeconds: 1500, Zones: []heartRateZoneDTO{
		{TimeSeconds: 900, TimePct: 0.6}, {TimeSeconds: 600, TimePct: 0.4},
	}}
	if v := checkDetailInvariants(a, d, UnitKm, good); len(v) != 0 {
		t.Fatalf("good zones reported violations: %v", v)
	}
	overflow := &heartRateZonesDTO{TotalHRSeconds: 2000, Zones: []heartRateZoneDTO{
		{TimeSeconds: 2000, TimePct: 1.0},
	}}
	if v := checkDetailInvariants(a, d, UnitKm, overflow); len(v) == 0 {
		t.Fatal("zone seconds exceeding duration should violate I7")
	}
	badPct := &heartRateZonesDTO{TotalHRSeconds: 1500, Zones: []heartRateZoneDTO{
		{TimeSeconds: 900, TimePct: 0.6}, {TimeSeconds: 600, TimePct: 0.3},
	}}
	if v := checkDetailInvariants(a, d, UnitKm, badPct); len(v) == 0 {
		t.Fatal("zone pct summing to 0.9 should violate I7")
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `go test ./internal/activity/ -run TestCheckDetailInvariants -v`
Expected: FAIL — `undefined: checkDetailInvariants`

- [ ] **Step 3: Implement `invariants.go`**

```go
package activity

import (
	"fmt"
	"math"
)

// Invariant tolerances. Everything the detail response renders is derived
// from one trackpoint stream, so disagreements beyond float/rounding noise
// mean a computation regressed (or a write path drifted, e.g. a calibrate
// that rescaled the summary but not the stream). Paces compare in
// sec-per-display-unit; distances in meters; times in seconds.
const (
	invariantDistTolMeters   = 0.5 // stored summary vs stream endpoint
	invariantSplitDistTol    = 0.01
	invariantTimeTolSeconds  = 1
	invariantPaceIdentityTol = 0.5
	invariantPaceMeanTol     = 1.0
	// I6 compares three window definitions (sample <= rolling unit <= aligned
	// full bucket). The bucket comparison is approximate at the edges (a
	// "full" split may span slightly less than one unit), so it gets slack.
	invariantOrderingTol = 2.0
	invariantPctTol      = 0.01
)

// checkDetailInvariants verifies the assembled running-detail response
// reconciles with itself (SOW running-detail-metric-alignment, "Algorithms").
// It returns human-readable violation strings tagged I1..I7; empty means
// aligned. Callers log violations at ERROR and STILL serve the response — a
// read must never 500 over an accounting mismatch; CI fixtures assert
// emptiness so violations are caught as regressions.
func checkDetailInvariants(a Activity, d Derivation, unit DistanceUnit, zones *heartRateZonesDTO) []string {
	var v []string
	bucket := unit.BucketMeters()

	if len(a.Trackpoints) >= 2 {
		first, last := a.Trackpoints[0], a.Trackpoints[len(a.Trackpoints)-1]

		// I1: stored distance == stream endpoint, and split distances
		// telescope to the stream span.
		if math.Abs(last.DistanceMeters-a.DistanceMeters) > invariantDistTolMeters {
			v = append(v, fmt.Sprintf("I1: stored distance %.2f != last trackpoint %.2f", a.DistanceMeters, last.DistanceMeters))
		}
		var sumDist float64
		var sumTime int
		for _, s := range d.Splits {
			sumDist += s.DistanceMeters
			sumTime += s.DurationSeconds
		}
		if span := last.DistanceMeters - first.DistanceMeters; math.Abs(sumDist-span) > invariantSplitDistTol {
			v = append(v, fmt.Sprintf("I1: split distances sum %.2f != stream span %.2f", sumDist, span))
		}

		// I2: split times sum to the duration tile.
		if int(math.Abs(float64(sumTime-a.DurationSeconds))) > invariantTimeTolSeconds {
			v = append(v, fmt.Sprintf("I2: split times sum %d != duration %d", sumTime, a.DurationSeconds))
		}
	}

	// I3: every split pace ≡ time/dist (identity by construction; asserted so a
	// future "optimization" can't quietly reintroduce filtered pace math).
	for _, s := range d.Splits {
		if s.PaceSecPerUnit == nil || s.DistanceMeters <= 0 {
			continue
		}
		want := float64(s.DurationSeconds) / s.DistanceMeters * bucket
		if math.Abs(*s.PaceSecPerUnit-want) > invariantPaceIdentityTol {
			v = append(v, fmt.Sprintf("I3: split %d pace %.2f != time/dist %.2f", s.Index, *s.PaceSecPerUnit, want))
		}
	}

	// I4 + I5: the avg-pace tile is reproducible both from the splits
	// (distance-weighted mean == total time / total distance) and from the
	// stored duration/distance pair.
	if a.AvgPaceSecPerKm != nil && a.DistanceMeters > 0 {
		avgPerUnit := *a.AvgPaceSecPerKm * bucket / 1000
		var sumDist float64
		var sumTime int
		for _, s := range d.Splits {
			sumDist += s.DistanceMeters
			sumTime += s.DurationSeconds
		}
		if sumDist > 0 {
			weighted := float64(sumTime) / sumDist * bucket
			if math.Abs(weighted-avgPerUnit) > invariantPaceMeanTol {
				v = append(v, fmt.Sprintf("I4: weighted split pace %.2f != avg pace %.2f", weighted, avgPerUnit))
			}
		}
		recomputed := float64(a.DurationSeconds) / (a.DistanceMeters / 1000)
		if math.Abs(recomputed-*a.AvgPaceSecPerKm) > invariantPaceIdentityTol {
			v = append(v, fmt.Sprintf("I5: stored avg pace %.2f != duration/distance %.2f", *a.AvgPaceSecPerKm, recomputed))
		}
	}

	// I6: window-size ordering — fastest sample <= fastest rolling unit
	// window <= fastest full split (each contains the next as a candidate).
	if d.BestPaceSecPerUnit != nil {
		if d.StripSummary.FastestSecPerUnit != nil &&
			*d.StripSummary.FastestSecPerUnit > *d.BestPaceSecPerUnit+invariantOrderingTol {
			v = append(v, fmt.Sprintf("I6: fastest sample %.2f slower than best window %.2f", *d.StripSummary.FastestSecPerUnit, *d.BestPaceSecPerUnit))
		}
		for _, s := range d.Splits {
			if s.Partial || s.PaceSecPerUnit == nil {
				continue
			}
			if *d.BestPaceSecPerUnit > *s.PaceSecPerUnit+invariantOrderingTol {
				v = append(v, fmt.Sprintf("I6: best window %.2f slower than full split %d pace %.2f", *d.BestPaceSecPerUnit, s.Index, *s.PaceSecPerUnit))
				break
			}
		}
	}

	// I7: HR-zone accounting — zone seconds fit inside the run's duration
	// (zones cover HR-carrying time only) and percentages sum to ~1.
	if zones != nil && zones.TotalHRSeconds > 0 {
		var zoneSec int
		var zonePct float64
		for _, z := range zones.Zones {
			zoneSec += z.TimeSeconds
			zonePct += z.TimePct
		}
		if zoneSec > a.DurationSeconds+invariantTimeTolSeconds {
			v = append(v, fmt.Sprintf("I7: zone seconds %d exceed duration %d", zoneSec, a.DurationSeconds))
		}
		if math.Abs(zonePct-1) > invariantPctTol {
			v = append(v, fmt.Sprintf("I7: zone percentages sum to %.3f", zonePct))
		}
	}

	return v
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `go test ./internal/activity/ -run TestCheckDetailInvariants -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add internal/activity/invariants.go internal/activity/invariants_test.go
git commit -m "feat(activity): invariant gate over the assembled running detail"
```

### Task A5: Handler wiring — unit param, derived DTO blocks, clean-pace flags, gate logging

**Files:**
- Modify: `internal/activity/handler.go`
- Test: `internal/activity/handler_test.go`

- [ ] **Step 1: Write the failing tests (append to `handler_test.go`)**

```go
// doGet drives the detail handler with an optional query string.
func doGet(t *testing.T, h *Handler, activityID, query string) *httptest.ResponseRecorder {
	t.Helper()
	req := httptest.NewRequest("GET", "/activities/"+activityID+query, nil)
	req = withParam(req.WithContext(authctx.WithUserID(req.Context(), testUserID)), "id", activityID)
	w := httptest.NewRecorder()
	h.get(w, req)
	return w
}

// TestGetDetail_DerivedBlocks: the detail response carries splits,
// strip_summary, best_pace_sec_per_unit, unit, and per-point clean_pace; the
// splits reconcile with the summary tiles (the SOW's user-visible promise).
func TestGetDetail_DerivedBlocks(t *testing.T) {
	h, _, _ := newTestHandler(t)
	id := importedID(t, h, "typical_5k.tcx")

	w := doGet(t, h, id, "?unit=km")
	if w.Code != http.StatusOK {
		t.Fatalf("status = %d; body=%s", w.Code, w.Body.String())
	}
	var env activityEnvelope
	if err := json.Unmarshal(w.Body.Bytes(), &env); err != nil {
		t.Fatalf("decode: %v", err)
	}
	if env.Data.Unit != "km" {
		t.Errorf("unit = %q, want km", env.Data.Unit)
	}
	if len(env.Data.Splits) == 0 {
		t.Fatal("expected splits on running detail")
	}
	var sumDist float64
	var sumTime int
	for _, s := range env.Data.Splits {
		sumDist += s.DistanceMeters
		sumTime += s.DurationSeconds
	}
	if math.Abs(sumDist-env.Data.DistanceMeters) > 1 {
		t.Errorf("split dist sum %.1f != distance tile %.1f", sumDist, env.Data.DistanceMeters)
	}
	if d := sumTime - env.Data.DurationSeconds; d < -1 || d > 1 {
		t.Errorf("split time sum %d != duration tile %d", sumTime, env.Data.DurationSeconds)
	}
	if env.Data.StripSummary == nil {
		t.Fatal("expected strip_summary")
	}
	if env.Data.BestPaceSecPerUnit == nil {
		t.Error("expected best_pace_sec_per_unit on a 5k")
	}
	// Default unit is miles.
	wMi := doGet(t, h, id, "")
	var envMi activityEnvelope
	if err := json.Unmarshal(wMi.Body.Bytes(), &envMi); err != nil {
		t.Fatalf("decode: %v", err)
	}
	if envMi.Data.Unit != "mi" {
		t.Errorf("default unit = %q, want mi", envMi.Data.Unit)
	}
	// Invalid unit is a 400.
	if w := doGet(t, h, id, "?unit=furlongs"); w.Code != http.StatusBadRequest {
		t.Errorf("invalid unit status = %d, want 400", w.Code)
	}
}

// TestGetDetail_CalibratedStaysAligned: after a calibration the derived
// blocks are recomputed from the rescaled stream and still reconcile — the
// regression the SOW exists to prevent.
func TestGetDetail_CalibratedStaysAligned(t *testing.T) {
	h, _, _ := newTestHandler(t)
	id := importedID(t, h, "treadmill_5k.tcx")

	var before activityEnvelope
	if err := json.Unmarshal(doGet(t, h, id, "?unit=km").Body.Bytes(), &before); err != nil {
		t.Fatalf("decode: %v", err)
	}
	target := before.Data.DistanceMeters * 0.93
	if w := doCalibrate(t, h, id, fmt.Sprintf(`{"distance_meters":%f}`, target)); w.Code != http.StatusOK {
		t.Fatalf("calibrate status = %d; body=%s", w.Code, w.Body.String())
	}

	var after activityEnvelope
	if err := json.Unmarshal(doGet(t, h, id, "?unit=km").Body.Bytes(), &after); err != nil {
		t.Fatalf("decode: %v", err)
	}
	var sumDist float64
	for _, s := range after.Data.Splits {
		sumDist += s.DistanceMeters
	}
	if math.Abs(sumDist-target) > 1 {
		t.Errorf("calibrated split dist sum %.1f != calibrated distance %.1f", sumDist, target)
	}
	if after.Data.AvgPaceSecPerKm == nil {
		t.Fatal("nil avg pace after calibrate")
	}
	wantAvg := float64(after.Data.DurationSeconds) / (target / 1000)
	if math.Abs(*after.Data.AvgPaceSecPerKm-wantAvg) > 0.5 {
		t.Errorf("avg pace %.2f != duration/distance %.2f", *after.Data.AvgPaceSecPerKm, wantAvg)
	}
}

// TestGetDetail_NonRunningHasNoDerivedBlocks: a walk gets no splits/strip.
func TestGetDetail_NonRunningHasNoDerivedBlocks(t *testing.T) {
	h, _, _ := newTestHandler(t)
	id := importedID(t, h, "walk.tcx") // any existing non-running fixture; see note below
	var env activityEnvelope
	if err := json.Unmarshal(doGet(t, h, id, "").Body.Bytes(), &env); err != nil {
		t.Fatalf("decode: %v", err)
	}
	if len(env.Data.Splits) != 0 || env.Data.StripSummary != nil || env.Data.BestPaceSecPerUnit != nil {
		t.Error("non-running detail should carry no derived blocks")
	}
}
```

Fixture note: use whatever non-running fixture `internal/activity/testdata/` already has (check `ls internal/activity/testdata/` — the walking fixture used by `TestSummarize_WalkHasNoBestEfforts`); adjust the filename in the test to match. The `activityEnvelope` test struct decodes into `activityDTO`, so the new fields are picked up automatically once Step 3 adds them.

- [ ] **Step 2: Run tests to verify they fail**

Run: `go test ./internal/activity/ -run TestGetDetail -v`
Expected: FAIL — `env.Data.Unit`, `env.Data.Splits` undefined (DTO fields don't exist yet)

- [ ] **Step 3: Implement the handler changes**

3a. Add the DTO types after `heartRateZonesDTO` (~`handler.go:237`):

```go
// splitDTO is one distance bucket of the run's derived splits table. Pace is
// per DISPLAY UNIT (the response's `unit`), and by construction equals
// duration_seconds / distance_meters normalized to one unit — the invariant
// gate asserts it.
type splitDTO struct {
	Index                 int      `json:"index"`
	Partial               bool     `json:"partial"`
	DistanceMeters        float64  `json:"distance_meters"`
	DurationSeconds       int      `json:"duration_seconds"`
	PaceSecPerUnit        *float64 `json:"pace_sec_per_unit"`
	AvgHRBpm              *float64 `json:"avg_hr_bpm"`
	ElevationDeltaMeters  *float64 `json:"elevation_delta_meters"`
	Fastest               bool     `json:"fastest"`
	Slowest               bool     `json:"slowest"`
}

// stripSummaryDTO carries the pace-chart header numbers so the client renders
// text straight from the server (the chart line itself is a presentation
// mapping of the trackpoints, which are already in the payload — the strip is
// deliberately NOT duplicated as a parallel array; the detail response flows
// through MCP to the agent where doubling point data has real token cost).
type stripSummaryDTO struct {
	FastestSecPerUnit *float64 `json:"fastest_sec_per_unit"`
	SlowestSecPerUnit *float64 `json:"slowest_sec_per_unit"`
	DropoutCount      int      `json:"dropout_count"`
}

// intervalSegmentDTO is one labeled bout of a detected interval workout.
type intervalSegmentDTO struct {
	Kind            string   `json:"kind"`
	Rep             *int     `json:"rep"`
	Label           string   `json:"label"`
	DistanceMeters  float64  `json:"distance_meters"`
	DurationSeconds int      `json:"duration_seconds"`
	PaceSecPerUnit  *float64 `json:"pace_sec_per_unit"`
	AvgHRBpm        *float64 `json:"avg_hr_bpm"`
}
```

3b. Add `CleanPace` to `trackpointDTO` (`handler.go:174-181`):

```go
type trackpointDTO struct {
	Sequence        int      `json:"sequence"`
	ElapsedSeconds  int      `json:"elapsed_seconds"`
	DistanceMeters  float64  `json:"distance_meters"`
	HeartRateBpm    *int     `json:"heart_rate_bpm"`
	PaceSecPerKm    *float64 `json:"pace_sec_per_km"`
	ElevationMeters *float64 `json:"elevation_meters"`
	// CleanPace marks a plottable pace sample (present, positive, and not
	// slower than the dropout threshold). The chart draws a gap where false;
	// the server owns the threshold so client and strip summary can't drift.
	CleanPace bool `json:"clean_pace"`
}
```

3c. Add the detail-only fields to `activityDTO` (after `HeartRateZones`, ~`handler.go:211`):

```go
	// Detail-only derived blocks (omitted on list responses and non-running
	// activities): the read-time derivation the detail page renders verbatim.
	// See sows/running-detail-metric-alignment.md.
	Unit               string               `json:"unit,omitempty"`
	Splits             []splitDTO           `json:"splits,omitempty"`
	StripSummary       *stripSummaryDTO     `json:"strip_summary,omitempty"`
	BestPaceSecPerUnit *float64             `json:"best_pace_sec_per_unit,omitempty"`
	Intervals          []intervalSegmentDTO `json:"intervals,omitempty"`
```

3d. In `toActivityDTO` (`handler.go:259-264`), the `trackpointDTO(tp)` struct conversion no longer compiles (field sets differ). Replace the loop body:

```go
	if withTrackpoints {
		dto.Trackpoints = make([]trackpointDTO, 0, len(a.Trackpoints))
		for _, tp := range a.Trackpoints {
			dto.Trackpoints = append(dto.Trackpoints, trackpointDTO{
				Sequence:        tp.Sequence,
				ElapsedSeconds:  tp.ElapsedSeconds,
				DistanceMeters:  tp.DistanceMeters,
				HeartRateBpm:    tp.HeartRateBpm,
				PaceSecPerKm:    tp.PaceSecPerKm,
				ElevationMeters: tp.ElevationMeters,
				CleanPace:       isCleanTrackpointPace(tp.PaceSecPerKm),
			})
		}
	}
```

3e. Add a unit-param parser next to `parseOptionalTimeParam`:

```go
// parseUnitParam reads ?unit= for the detail derivation. Absent defaults to
// miles (the UI default); anything but mi/km is a client error.
func parseUnitParam(r *http.Request) (DistanceUnit, bool) {
	raw := r.URL.Query().Get("unit")
	if raw == "" {
		return UnitMiles, true
	}
	u := DistanceUnit(raw)
	return u, u.Valid()
}
```

3f. Thread the unit through `get` (`handler.go:485-512`) — after the `activityID` guard:

```go
	unit, ok := parseUnitParam(r)
	if !ok {
		httpresp.Error(w, http.StatusBadRequest, "unit must be 'mi' or 'km'")
		return
	}
```

and change the build call to `h.buildDetailDTO(r.Context(), userID, *a, unit)`. Do the same in `calibrate` (parse the param right after the id guard; pass to `buildDetailDTO` at `handler.go:679`).

3g. Extend `buildDetailDTO` — new signature `func (h *Handler) buildDetailDTO(ctx context.Context, userID string, a Activity, unit DistanceUnit) (activityDTO, error)`. After the HR-zones block (before the final `return dto, nil`), replace the early `return` for non-running so derivation is running-gated but zones logic is untouched, and append:

```go
	// Read-time derivation + invariant gate (running only). Violations are
	// ERROR-logged but the response is still served: a read never 500s over
	// an accounting mismatch — CI fixtures assert the gate stays quiet.
	if a.ActivityType == ActivityRunning && len(a.Trackpoints) >= 2 {
		der := deriveRunning(a.Trackpoints, unit)
		dto.Unit = string(unit)
		dto.Splits = make([]splitDTO, 0, len(der.Splits))
		for _, s := range der.Splits {
			dto.Splits = append(dto.Splits, splitDTO{
				Index: s.Index, Partial: s.Partial,
				DistanceMeters: s.DistanceMeters, DurationSeconds: s.DurationSeconds,
				PaceSecPerUnit: s.PaceSecPerUnit, AvgHRBpm: s.AvgHRBpm,
				ElevationDeltaMeters: s.ElevDeltaMeters,
				Fastest:              s.Fastest, Slowest: s.Slowest,
			})
		}
		dto.StripSummary = &stripSummaryDTO{
			FastestSecPerUnit: der.StripSummary.FastestSecPerUnit,
			SlowestSecPerUnit: der.StripSummary.SlowestSecPerUnit,
			DropoutCount:      der.StripSummary.DropoutCount,
		}
		dto.BestPaceSecPerUnit = der.BestPaceSecPerUnit
		if len(der.Intervals) > 0 {
			dto.Intervals = make([]intervalSegmentDTO, 0, len(der.Intervals))
			for _, seg := range der.Intervals {
				dto.Intervals = append(dto.Intervals, intervalSegmentDTO{
					Kind: seg.Kind, Rep: seg.Rep, Label: seg.Label,
					DistanceMeters: seg.DistanceMeters, DurationSeconds: seg.DurationSeconds,
					PaceSecPerUnit: seg.PaceSecPerUnit, AvgHRBpm: seg.AvgHRBpm,
				})
			}
		}
		for _, violation := range checkDetailInvariants(a, der, unit, dto.HeartRateZones) {
			log.Printf("ERROR activity detail invariant violation: activity_id=%s %s", a.ID, violation)
		}
	}
	return dto, nil
```

Restructuring note for 3g: the current function returns early when `h.hrEngine == nil || a.ActivityType != ActivityRunning` (`handler.go:520-522`). Change that to only skip the ZONES section (wrap the zones code in `if h.hrEngine != nil && a.ActivityType == ActivityRunning { ... }`) so the derivation block above always runs for running activities even when the engine is unwired (tests construct handlers without it).

- [ ] **Step 4: Run the full activity test suite**

Run: `go test ./internal/activity/... -v -count=1 2>&1 | tail -30`
Expected: PASS, including the pre-existing calibrate/environment/HR tests. `TestGetDetail_CalibratedStaysAligned` may FAIL on the avg/best relationship until Task A6 — if it fails ONLY on best-pace recompute grounds, proceed to A6 then re-run.

- [ ] **Step 5: Commit**

```bash
git add internal/activity/handler.go internal/activity/handler_test.go
git commit -m "feat(activity): unit-aware derived blocks + invariant gate on the detail response"
```

### Task A6: Calibrate recomputes best pace from the rescaled stream

**Files:**
- Modify: `internal/activity/sqlite_repository.go` (the `Calibrate` method)
- Modify: `internal/activity/sqlite_repository_test.go` (`TestCalibrate_UniformScale`)

Uniform scaling moves which window is fastest, so the current `best_pace_sec_per_km / f` is an approximation. Recompute the fastest rolling 1 km from the rescaled trackpoints inside the same transaction, using `bestRollingPace` from Task A2.

- [ ] **Step 1: Update the existing test to demand a recompute**

In `TestCalibrate_UniformScale` (`sqlite_repository_test.go:363-418`), find the assertion that best pace equals `old / f` and replace it with:

```go
	// Best pace is RECOMPUTED from the rescaled trackpoints (not scaled ÷f):
	// scaling moves which window is fastest, so ÷f is only an approximation.
	var scaledTps []Trackpoint
	for _, tp := range updated.Trackpoints {
		scaledTps = append(scaledTps, tp)
	}
	wantBest := bestRollingPace(scaledTps, 1000)
	switch {
	case wantBest == nil:
		if updated.BestPaceSecPerKm != nil {
			t.Errorf("best pace = %v, want nil (track shorter than 1 km after scaling)", *updated.BestPaceSecPerKm)
		}
	case updated.BestPaceSecPerKm == nil:
		t.Error("best pace nil, want recomputed value")
	default:
		if math.Abs(*updated.BestPaceSecPerKm-*wantBest) > 0.5 {
			t.Errorf("best pace = %.2f, want recomputed %.2f", *updated.BestPaceSecPerKm, *wantBest)
		}
	}
```

(If the loop building `scaledTps` is redundant because `updated.Trackpoints` is already `[]Trackpoint`, pass it directly: `wantBest := bestRollingPace(updated.Trackpoints, 1000)`.)

- [ ] **Step 2: Run to verify it fails**

Run: `go test ./internal/activity/ -run TestCalibrate_UniformScale -v`
Expected: FAIL (stored value is still old ÷ f, which differs from the recompute whenever scaling shifts the fastest window; if the fixture happens to make them equal, tighten the fixture — e.g. calibrate by 0.9 — until they differ, or accept equality and rely on the handler test from A5)

- [ ] **Step 3: Implement the recompute in `Calibrate`**

In `sqlite_repository.go`, remove `best_pace_sec_per_km = best_pace_sec_per_km / ?` from the first UPDATE (and drop `f` from its args):

```go
	if _, err := tx.ExecContext(ctx, `
		UPDATE activities
		SET distance_meters = ?,
		    avg_pace_sec_per_km = ?
		WHERE id = ? AND user_id = ?
	`, newDistanceMeters, avgPace, activityID, userID); err != nil {
		return nil, err
	}
```

Then, after the trackpoint-rescale UPDATE and before `tx.Commit()`:

```go
	// Recompute best pace from the rescaled trackpoints rather than scaling
	// the old value by ÷f: uniform scaling moves which window is fastest, so
	// ÷f is only an approximation. The rolling window runs over the stored
	// (downsampled) points — the same stream every read-time surface uses,
	// which is what lets the detail invariant gate compare them.
	rows, err := tx.QueryContext(ctx, `
		SELECT elapsed_seconds, distance_meters
		FROM activity_trackpoints
		WHERE activity_id = ?
		ORDER BY sequence
	`, activityID)
	if err != nil {
		return nil, err
	}
	var tps []Trackpoint
	for rows.Next() {
		var tp Trackpoint
		if err := rows.Scan(&tp.ElapsedSeconds, &tp.DistanceMeters); err != nil {
			rows.Close()
			return nil, err
		}
		tps = append(tps, tp)
	}
	if err := rows.Err(); err != nil {
		rows.Close()
		return nil, err
	}
	rows.Close()
	if _, err := tx.ExecContext(ctx, `
		UPDATE activities SET best_pace_sec_per_km = ? WHERE id = ? AND user_id = ?
	`, bestRollingPace(tps, 1000), activityID, userID); err != nil {
		return nil, err
	}
```

(`*float64` binds as NULL when nil, matching the pre-existing NULL-stays-NULL behavior.)

- [ ] **Step 4: Run the suite**

Run: `go test ./internal/activity/... -count=1`
Expected: PASS including `TestCalibrate_UniformScale`, `TestCalibrateHandler_IndoorHappyPath`, `TestPatchEnvironment_RegenReflectsCalibration`, and A5's `TestGetDetail_CalibratedStaysAligned`

- [ ] **Step 5: Commit**

```bash
git add internal/activity/sqlite_repository.go internal/activity/sqlite_repository_test.go
git commit -m "fix(activity): recompute best pace from rescaled trackpoints on calibrate"
```

### Task A7: Property test + full verification

**Files:**
- Create: `internal/activity/invariants_fuzz_test.go`

- [ ] **Step 1: Write the property test** (mirrors the style of `best_efforts_fuzz_test.go`)

```go
package activity

import (
	"math/rand"
	"testing"
)

// TestInvariants_RandomTracksAlwaysAlign is the SOW's property test: for any
// plausible monotonic track, the summary recomputed the way ingest computes
// it plus the read-time derivation must pass the invariant gate. Seeded PRNG
// keeps failures reproducible.
func TestInvariants_RandomTracksAlwaysAlign(t *testing.T) {
	rng := rand.New(rand.NewSource(42))
	for trial := 0; trial < 200; trial++ {
		n := 2 + rng.Intn(400)
		tps := make([]Trackpoint, 0, n)
		dist, elapsed := 0.0, 0
		for i := 0; i < n; i++ {
			var pace *float64
			if i > 0 {
				dStep := rng.Float64() * 60 // 0–60 m per step (incl. stationary)
				tStep := 1 + rng.Intn(30)
				dist += dStep
				elapsed += tStep
				if dStep > 0.5 {
					p := float64(tStep) / (dStep / 1000)
					// Leave some samples nil (stationary filter) at random.
					if rng.Float64() > 0.1 {
						pace = &p
					}
				}
			}
			tps = append(tps, Trackpoint{Sequence: i, ElapsedSeconds: elapsed, DistanceMeters: dist, PaceSecPerKm: pace})
		}
		if dist <= 0 {
			continue
		}
		avg := float64(elapsed) / (dist / 1000)
		a := Activity{
			ActivityType:    ActivityRunning,
			DistanceMeters:  dist,
			DurationSeconds: elapsed,
			AvgPaceSecPerKm: &avg,
			Trackpoints:     tps,
		}
		for _, unit := range []DistanceUnit{UnitMiles, UnitKm} {
			d := deriveRunning(tps, unit)
			if v := checkDetailInvariants(a, d, unit, nil); len(v) != 0 {
				t.Fatalf("trial %d unit %s: violations %v", trial, unit, v)
			}
		}
	}
}
```

- [ ] **Step 2: Run it**

Run: `go test ./internal/activity/ -run TestInvariants_RandomTracksAlwaysAlign -v`
Expected: PASS. If a trial fails, the derivation or a tolerance is wrong — fix the code, not the test, unless the generated track is genuinely non-physical.

- [ ] **Step 3: Full test + lint**

Run: `go test ./... -count=1` then `go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run ./...`
Expected: both clean.

- [ ] **Step 4: Commit, push, PR**

```bash
git add internal/activity/invariants_fuzz_test.go
git commit -m "test(activity): property test — random tracks always pass the invariant gate"
git push -u origin feat/running-detail-metric-alignment
gh pr create --title "feat(activity): server-side running detail derivation + invariant gate" --body "Implements sows/running-detail-metric-alignment.md (API half). Adds ?unit=mi|km derived splits/strip-summary/intervals/best to the detail response, per-point clean_pace flags, an ERROR-logging invariant gate, and an exact best-pace recompute on calibrate."
```

(Branch off `main` at the start of A1: `git checkout -b feat/running-detail-metric-alignment`.)

---

## Repo: prog-strength-web

Prereq: the API PR above is merged/deployed (or run against a local API build). Branch: `git checkout -b feat/running-detail-render-only`.

### Task W1: API types + unit param

**Files:**
- Modify: `lib/api.ts` (RunningSession/RunningTrackpoint types ~1676-1710; `getRunningSession` ~1792; `calibrateRunningSession` ~1857)
- Test: `lib/api.test.ts`

- [ ] **Step 1: Write the failing tests** (follow the file's existing fetch-mock pattern — see the `calibrateRunningSession` test at `lib/api.test.ts:642` for the idiom):

```ts
it("getRunningSession passes the unit param through", async () => {
  const fetchMock = vi.fn().mockResolvedValue(
    jsonResponse({ message: "ok", data: { id: "a1", trackpoints: [] } }),
  );
  vi.stubGlobal("fetch", fetchMock);
  await getRunningSession("tok", "a1", "km");
  expect(fetchMock).toHaveBeenCalledWith(
    expect.stringContaining("/activities/a1?unit=km"),
    expect.anything(),
  );
});

it("calibrateRunningSession passes the unit param through", async () => {
  const fetchMock = vi.fn().mockResolvedValue(
    jsonResponse({ message: "ok", data: { id: "a1" } }),
  );
  vi.stubGlobal("fetch", fetchMock);
  await calibrateRunningSession("tok", "a1", 5000, "mi");
  expect(fetchMock).toHaveBeenCalledWith(
    expect.stringContaining("/activities/a1/calibrate?unit=mi"),
    expect.anything(),
  );
});
```

(Adapt helper names — `jsonResponse` or whatever the file's local response builder is called — to the existing test file's conventions.)

- [ ] **Step 2: Run to verify failure**

Run: `npx vitest run lib/api.test.ts`
Expected: FAIL — signatures don't accept unit

- [ ] **Step 3: Implement**

3a. New wire types next to `RunningSession`:

```ts
/** One distance bucket of the server-derived splits table. Pace is per the
 *  response's `unit` and equals duration_seconds/distance_meters normalized
 *  to one unit — the backend's invariant gate asserts it. */
export type RunningSplit = {
  index: number;
  partial: boolean;
  distance_meters: number;
  duration_seconds: number;
  pace_sec_per_unit: number | null;
  avg_hr_bpm: number | null;
  elevation_delta_meters: number | null;
  fastest: boolean;
  slowest: boolean;
};

/** Pace-chart header numbers, server-computed over clean samples only. */
export type RunningStripSummary = {
  fastest_sec_per_unit: number | null;
  slowest_sec_per_unit: number | null;
  dropout_count: number;
};

/** One labeled bout of a server-detected interval workout. */
export type RunningIntervalSegment = {
  kind: "warmup" | "work" | "recovery" | "cooldown";
  rep: number | null;
  label: string;
  distance_meters: number;
  duration_seconds: number;
  pace_sec_per_unit: number | null;
  avg_hr_bpm: number | null;
};
```

3b. Extend `RunningSession` (after `heart_rate_zones`):

```ts
  // Server-derived detail blocks (running only; absent on list responses).
  // The page renders these verbatim — no client-side re-derivation.
  unit?: "mi" | "km";
  splits?: RunningSplit[];
  strip_summary?: RunningStripSummary;
  best_pace_sec_per_unit?: number;
  intervals?: RunningIntervalSegment[];
```

3c. Extend `RunningTrackpoint` with `clean_pace: boolean;` (after `pace_sec_per_km`), documented as: server-owned plottability flag; the chart draws a gap where false.

3d. Signatures:

```ts
export async function getRunningSession(
  token: string,
  id: string,
  unit: "mi" | "km" = "mi",
): Promise<RunningSession> {
  const resp = await fetch(
    `${config.apiUrl}/activities/${encodeURIComponent(id)}?unit=${unit}`,
    { headers: { Authorization: `Bearer ${token}` } },
  );
  // ... unchanged unwrap ...
}
```

and `calibrateRunningSession(token, id, distanceMeters, unit: "mi" | "km" = "mi")` fetching `/activities/${id}/calibrate?unit=${unit}`.

- [ ] **Step 4: Run to verify pass**

Run: `npx vitest run lib/api.test.ts` — Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add lib/api.ts lib/api.test.ts
git commit -m "feat(api-client): unit param + server-derived running detail types"
```

### Task W2: One pace formatter

**Files:**
- Modify: `lib/pace-format.ts`
- Modify: `lib/distance-unit-context.tsx` (`formatPaceValue`, lines 68-75)
- Test: `lib/pace-format.test.ts`

- [ ] **Step 1: Failing tests** (append to `lib/pace-format.test.ts`):

```ts
import { formatPaceClock, formatPaceClockOrDash, formatPaceDelta } from "./pace-format";

it("formatPaceClockOrDash dashes null and delegates otherwise", () => {
  expect(formatPaceClockOrDash(null)).toBe("—");
  expect(formatPaceClockOrDash(626)).toBe("10:26");
});

it("formatPaceDelta signs and formats", () => {
  expect(formatPaceDelta(-65)).toBe("−1:05");
  expect(formatPaceDelta(5)).toBe("+0:05");
});
```

- [ ] **Step 2: Run to verify failure** — `npx vitest run lib/pace-format.test.ts` → FAIL (functions missing)

- [ ] **Step 3: Implement** (append to `lib/pace-format.ts`):

```ts
/** Null-tolerant wrapper: "—" for null, else formatPaceClock. */
export function formatPaceClockOrDash(sec: number | null): string {
  return sec == null ? "—" : formatPaceClock(sec);
}

/** Signed pace delta in sec/unit: "+m:ss" slower, "−m:ss" faster. */
export function formatPaceDelta(deltaSec: number): string {
  const sign = deltaSec < 0 ? "−" : "+";
  const abs = Math.abs(Math.round(deltaSec));
  return `${sign}${Math.floor(abs / 60)}:${String(abs % 60).padStart(2, "0")}`;
}
```

Then in `lib/distance-unit-context.tsx`, make `formatPaceValue` delegate (delete its manual m:ss lines):

```ts
import { formatPaceClock } from "./pace-format";

export function formatPaceValue(secPerKm: number | null, unit: DistanceUnit): string {
  if (secPerKm == null || !Number.isFinite(secPerKm) || secPerKm <= 0) return "—";
  return formatPaceClock(unit === "mi" ? secPerKm * KM_PER_MILE : secPerKm);
}
```

- [ ] **Step 4: Run** — `npx vitest run lib/pace-format.test.ts lib/distance-unit-context.test.ts` → PASS

- [ ] **Step 5: Commit** — `git add lib/pace-format.ts lib/pace-format.test.ts lib/distance-unit-context.tsx && git commit -m "refactor(web): consolidate pace formatting on lib/pace-format"`

### Task W3: Gut `lib/running-splits.ts` to a strip mapper + target-pace parser

**Files:**
- Modify: `lib/running-splits.ts`
- Modify: `lib/running-splits.test.ts`

Delete `deriveRunningActivity`, `buildSplits`, `buildSegments`, `detectIntervals` + helpers, and the `Split`/`IntervalSegment`/`RunningDerivation`/`Segment` types. Keep: `DistanceUnit`, `PACE_DROPOUT_SEC_PER_KM` note (now server-owned — remove the constant, the flag arrives on the wire), `PaceStripPoint`, `parseTargetPace`, and a new strip mapper driven by the server's `clean_pace` flag:

- [ ] **Step 1: Rewrite tests first.** In `lib/running-splits.test.ts`, delete every test of `deriveRunningActivity`/`buildSplits`/`detectIntervals` (that behavior now lives — and is tested — in the API). Keep `parseTargetPace` tests. Add:

```ts
import { buildPaceStrip } from "./running-splits";
import type { RunningTrackpoint } from "./api";

const pt = (
  dist: number,
  pace: number | null,
  clean: boolean,
): RunningTrackpoint => ({
  sequence: 0,
  elapsed_seconds: 0,
  distance_meters: dist,
  heart_rate_bpm: null,
  pace_sec_per_km: pace,
  elevation_meters: null,
  clean_pace: clean,
});

it("buildPaceStrip trusts the server clean_pace flag and converts units", () => {
  const points = buildPaceStrip(
    [pt(0, null, false), pt(1609.344, 300, true), pt(3218.688, 1200, false)],
    "mi",
  );
  expect(points[0].paceSecPerUnit).toBeNull();
  expect(points[1].distanceUnit).toBeCloseTo(1);
  expect(points[1].paceSecPerUnit).toBeCloseTo(300 * 1.609344);
  expect(points[2].paceSecPerUnit).toBeNull(); // dropout: gap, not a value
});
```

- [ ] **Step 2: Run to verify failure** — `npx vitest run lib/running-splits.test.ts` → FAIL

- [ ] **Step 3: Rewrite `lib/running-splits.ts`** to exactly:

```ts
/**
 * Thin presentation mapping for the running detail page. The heavy derivation
 * (splits, strip summary, interval detection, best pace) moved SERVER-SIDE
 * (prog-strength-api internal/activity/derivation.go) so every rendered
 * number comes from one computation behind an invariant gate — see
 * sows/running-detail-metric-alignment.md. What remains here is pure
 * unit-mapping of the already-served trackpoints into chart coordinates
 * (the strip is deliberately not duplicated on the wire) and the tolerant
 * target-pace parser for plan prescriptions.
 */

import type { RunningTrackpoint } from "./api";

const METERS_PER_MILE = 1609.344;
const METERS_PER_KM = 1000;
const KM_PER_MILE = METERS_PER_MILE / METERS_PER_KM; // 1.609344

export type DistanceUnit = "mi" | "km";

export type PaceStripPoint = {
  /** Cumulative distance in the active unit. */
  distanceUnit: number;
  /** Pace in sec per active unit; null = dropout/gap → break the line. */
  paceSecPerUnit: number | null;
};

/** Map served trackpoints to chart points. Plottability is the server's
 *  clean_pace flag — the client no longer owns a dropout threshold. */
export function buildPaceStrip(
  trackpoints: RunningTrackpoint[],
  unit: DistanceUnit,
): PaceStripPoint[] {
  const bucketMeters = unit === "mi" ? METERS_PER_MILE : METERS_PER_KM;
  return trackpoints.map((t) => ({
    distanceUnit: t.distance_meters / bucketMeters,
    paceSecPerUnit:
      t.clean_pace && t.pace_sec_per_km != null
        ? unit === "mi"
          ? t.pace_sec_per_km * KM_PER_MILE
          : t.pace_sec_per_km
        : null,
  }));
}

/** Tolerant parser: extract a target work pace from free-text run_details.
 *  Returns sec per ACTIVE UNIT, or null when nothing is confidently found. */
export function parseTargetPace(runDetails: string | null, unit: DistanceUnit): number | null {
  // ... keep the existing implementation verbatim (lines 424-449 of the old file) ...
}
```

(Copy the existing `parseTargetPace` body unchanged.)

- [ ] **Step 4: Typecheck will now fail where the deleted types were imported** (`SplitsSpine.tsx`, `page.tsx`). That's Task W4 — do NOT commit yet if the repo's husky hooks run typecheck; otherwise commit here and fix forward in W4:

Run: `npx vitest run lib/running-splits.test.ts` → PASS (tests only)

- [ ] **Step 5: Proceed to W4 before committing** (single commit covers W3+W4 so the tree never has a broken typecheck).

### Task W4: Render-only detail page

**Files:**
- Modify: `app/(app)/running/[id]/page.tsx`
- Modify: `app/(app)/running/[id]/_components/SplitsSpine.tsx`
- Modify: `app/(app)/running/[id]/_components/PaceRecap.tsx`
- Modify: `app/(app)/running/[id]/_components/CalibrateDistanceModal.tsx` (pass unit to the calibrate call — find its `calibrateRunningSession(` call and add the active unit arg; it can take `unit` as a new prop from the page)
- Test: `app/(app)/running/[id]/page.test.tsx`

- [ ] **Step 1: SplitsSpine consumes wire types.** Replace the `Split`/`IntervalSegment` imports with `import type { RunningIntervalSegment, RunningSplit } from "@/lib/api";` and `import { formatPaceClockOrDash, formatPaceDelta } from "@/lib/pace-format";`. Delete the local `fmtPaceUnit`/`fmtPaceDelta` (lines 22-38). Prop types become `splits: RunningSplit[]` / `intervals: RunningIntervalSegment[] | null`. Field renames in `MilesTable`/`IntervalsTable`:

| old | new |
|---|---|
| `s.avgPaceSecPerUnit` | `s.pace_sec_per_unit` |
| `s.distanceMeters` | `s.distance_meters` |
| `s.durationSec` | `s.duration_seconds` |
| `s.avgHr` | `s.avg_hr_bpm` |
| `s.elevDeltaMeters` | `s.elevation_delta_meters` |
| `fmtPaceUnit(x)` | `formatPaceClockOrDash(x)` |
| `fmtPaceDelta(x)` | `formatPaceDelta(x)` |
| `seg.avgPaceSecPerUnit` / `seg.avgHr` etc. | same snake_case renames |

`s.index`, `s.partial`, `s.fastest`, `s.slowest`, `seg.kind`, `seg.label` are unchanged.

- [ ] **Step 2: PaceRecap header numbers come from the server.** Change the props to:

```ts
export function PaceRecap({
  points,
  stripSummary,
  unit,
}: {
  points: PaceStripPoint[];
  stripSummary: RunningStripSummary | null;
  unit: "km" | "mi";
}) {
```

with `import type { RunningStripSummary } from "@/lib/api";`. Inside:
- `const hasDropout = (stripSummary?.dropout_count ?? 0) > 0;` (replaces the prop)
- Header text uses server values, falling back to the plotted extremes only when the summary is absent: `const headerLo = stripSummary?.fastest_sec_per_unit ?? lo;` / `const headerHi = stripSummary?.slowest_sec_per_unit ?? hi;` and render `Fastest {formatPaceClock(headerLo)} · slowest {formatPaceClock(headerHi)} /{unit}`.
- Dropout phrase count uses `stripSummary?.dropout_count` (`const dropoutCount = stripSummary?.dropout_count ?? gaps.length;` — the caption text now states the server's sample count; the banded windows remain per-gap as drawn). Keep `lo`/`hi` for the y-domain math (chart geometry stays client-side by design).
- The "Fastest" annotation label: `Fastest {formatPaceClock(headerLo)}` anchored at the plotted min point as today.

- [ ] **Step 3: page.tsx becomes render-only.**
- Imports: drop `deriveRunningActivity`; add `buildPaceStrip` from `@/lib/running-splits`.
- Refetch on unit change: add `unit` to the detail-fetch `useEffect` dependency array and pass it: `getRunningSession(token, id, unit)` (deps become `[id, unit, router, handleAuthError]`).
- Replace the derivation memo (lines 96-99) with:

```ts
  // The splits/intervals/summary all arrive server-derived; the only client
  // mapping is trackpoints → chart coordinates.
  const paceStrip = useMemo(
    () => buildPaceStrip(session?.trackpoints ?? [], unit),
    [session, unit],
  );
```

- Best tile (lines 302-308): the value is already per display unit — use `formatPaceClock`:

```ts
              {
                label: "Best",
                value:
                  session.best_pace_sec_per_unit != null
                    ? `${formatPaceClock(session.best_pace_sec_per_unit)} /${unitLabel}`
                    : "—",
              },
```

(add `import { formatPaceClock } from "@/lib/pace-format";`). The Avg pace tile keeps `formatPace(session.avg_pace_sec_per_km)` — still a stored sec/km field.
- Widgets:

```ts
          <SplitsSpine
            splits={session.splits ?? []}
            intervals={
              completesPlan?.run_type === "intervals" ? (session.intervals ?? null) : null
            }
            unitLabel={unitLabel}
            formatDistance={formatDistance}
            hasTargetColumn={targetPace != null}
            targetPaceSecPerUnit={targetPace}
          />
          <PaceRecap
            points={paceStrip}
            stripSummary={session.strip_summary ?? null}
            unit={unit}
          />
```

(The run-type gate stays client-side: the server always computes candidate intervals; the client shows them only when the linked plan says intervals.)
- Calibrate modal: pass `unit` down (`<CalibrateDistanceModal session={session} unit={unit} …/>`) and inside the modal forward it to `calibrateRunningSession(token, id, meters, unit)` so the calibrate response carries unit-correct derived blocks.
- The environment PATCH merge (line 205) is already correct: the summary response omits the derived keys (`omitempty`), so `{ ...s, ...updated, trackpoints: s.trackpoints }` keeps the existing splits/strip/intervals — valid because an environment change never rescales distance.

- [ ] **Step 4: Update page tests.** In `page.test.tsx`: extend the session fixtures with `unit`, `splits` (at least two full + one partial row with `pace_sec_per_unit` equal to `duration_seconds / distance_meters * 1609.344`), `strip_summary`, `best_pace_sec_per_unit`, and `clean_pace: true` on trackpoint fixtures. Update assertions that previously relied on client-derived split values to assert the fixture's server values render verbatim. Add one new test:

```ts
it("refetches the detail when the unit toggles", async () => {
  // Render with unit=mi (default), assert getRunningSession called with "mi";
  // flip the unit context to km (drive the provider's setUnit or re-render
  // with a km-preloaded provider) and assert a second call with "km".
});
```

(Write it concretely against the file's existing provider-wrapping test utilities — the fixture/provider helpers at the top of `page.test.tsx` show how `DistanceUnitProvider` is mounted.)

- [ ] **Step 5: Verify everything**

Run: `npx vitest run` then `npm run typecheck` then `npm run lint`
Expected: all clean — including the calibration describe block at `page.test.tsx:430` (its "header distance matches rescaled splits" test now exercises server-provided splits from the calibrate response fixture; update that fixture to carry rescaled `splits` too).

- [ ] **Step 6: Commit, push, PR**

```bash
git add lib/running-splits.ts lib/running-splits.test.ts "app/(app)/running/[id]" lib/api.ts
git commit -m "feat(running): render server-derived splits/strip/intervals; delete client derivation"
git push -u origin feat/running-detail-render-only
gh pr create --title "feat(running): detail page renders server-derived metrics" --body "Web half of sows/running-detail-metric-alignment.md. Deletes deriveRunningActivity; renders ?unit-aware server splits/strip-summary/intervals/best; refetches on unit toggle; consolidates pace formatting."
```

---

## Repo: prog-strength-docs

### Task D1: Flip the SOW status

- [ ] After both PRs merge: set `sows/running-detail-metric-alignment.md` frontmatter `status: shipped` and the body line to `**Status**: Shipped · **Last updated**: <merge date>`; commit `docs: mark running-detail-metric-alignment as shipped` and PR.

---

## Self-review notes (already applied)

- Spec coverage: policy change (A1), unit param + derived blocks + clean_pace (A5), strip summary + display-unit best (A2), intervals (A3 + client gate in W4), invariant gate I1–I7 (A4, logging in A5), calibrate recompute (A6), property test (A7), render-only web + refetch + formatter consolidation (W1–W4), non-goals untouched (calories/HR-zones/env semantics unchanged).
- The one deliberate divergence from the SOW's response sketch: split DTO uses `elevation_delta_meters` (spec's name) — keep DTO and web type in sync on that exact key.
- Type consistency: `deriveRunning`, `buildSegments`, `buildDerivedSplits`, `buildStripSummary`, `bestRollingPace`, `detectIntervals`, `checkDetailInvariants`, `parseUnitParam`, `isCleanTrackpointPace` are the only new API-package identifiers; web adds `RunningSplit`, `RunningStripSummary`, `RunningIntervalSegment`, `buildPaceStrip`, `formatPaceClockOrDash`, `formatPaceDelta`.
