# Personal Records — Lifts View — Compact-Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebuild the Personal Records **Lifts view** into the `compact-dashboard` tile grid (every headline lift visible at once, a readiness dot + inline spark that scale by degree), backed by a new additive, nullable estimated-1RM trend field on the existing `GET /personal-records` list response.

**Architecture:** Three independent slices. (1) **API** — `prog-strength-api`: add `recent_estimated_1rm_points` to `personalRecordDTO`, built by downsampling the 1RM history entries the `personalRecords` loop *already fetches* per slug (zero new queries). (2) **Web — readiness**: a pure `readiness.ts` module over `PersonalRecord` + the new nullable type field in `lib/api.ts`. (3) **Web — rebuild**: `LiftsView` becomes the dense tile grid (Tile, dashed untested tile, inline Sparkline, inline expand → `ProgressionChart` + `View workout`), plus a `N due · N/total tested` chrome summary in `page.tsx`. The Running view and page orchestration are untouched.

**Tech Stack:** Go 1.25 (chi, SQLite, stdlib `testing`); Next.js 16 / React 19 / TypeScript / Tailwind v4; Vitest + Testing Library; recharts (existing chart).

**Design system:** `scope: in-system` against design-system **v0.4** (oura-calm). Conform only — **no** token/accent/type change. Use CSS variables: `--surface`, `--border`, `--muted`, `--faint`, `--foreground`, `--warning`, `--success`, `--discipline-lift-dot`. Periwinkle `--accent` stays app-chrome (selection/active) only — never on the readiness/spark signal.

**Decisions on the SOW's Open Questions** (made autonomously; recorded in the PR bodies):
1. **Spark window/cap/shape** — reuse the entries the loop *already fetches* (the baseline window; **no query change**, honoring the hard "no extra DB load / no new query" goal over the softer "lean widen"). Cap **10** points, **even-stride** downsample, bare `[]float64` ascending (oldest→newest), `null` when no entries.
2. **Chart + `View workout`** — the tile is a click target that expands the existing lazy `ProgressionChart` inline (full-width row via `col-span-full`); `View workout →` sits inside the expanded panel. Suppressed for untested lifts.
3. **Per-dumbbell** — `isDumbbell` via `/dumbbell/i` on the exercise name; append a quiet `ea` marker so a tile never implies a combined load.
4. **Untested tiles** — keep **API order** in place (no sinking to the end).

---

## File Structure

**`prog-strength-api`**
- Modify: `internal/workout/handler.go` — add `RecentEstimated1RMPoints` to `personalRecordDTO`; populate it in the `personalRecords` loop from the already-fetched `entries`.
- Modify: `internal/workout/onerepmax.go` — add pure helper `DownsampleEstimated1RM(entries []OneRepMaxEntry, cap int) []float64`.
- Modify: `internal/workout/onerepmax_test.go` — unit-test `DownsampleEstimated1RM`.
- Modify: `internal/workout/handler_test.go` — handler test for the new field (has-history / never-trained / cap cases) and that existing fields are unaffected.

**`prog-strength-web`**
- Modify: `lib/api.ts` — extend `PersonalRecord` with `recent_estimated_1rm_points: number[] | null`.
- Create: `app/(app)/personal-records/_components/readiness.ts` — pure derivation module.
- Create: `app/(app)/personal-records/_components/readiness.test.ts`.
- Create: `app/(app)/personal-records/_components/Sparkline.tsx` — pure inline SVG sparkline.
- Rewrite: `app/(app)/personal-records/_components/LiftsView.tsx` — tile grid + Tile + untested tile + inline expand + skeleton/error/empty.
- Create: `app/(app)/personal-records/_components/LiftsView.test.tsx`.
- Delete: `app/(app)/personal-records/_components/LiftPRCard.tsx` — replaced by the tile grid (its inline gap math moves to `readiness.ts`).
- Modify: `app/(app)/personal-records/page.tsx` — add the `N due · N/total tested` chrome summary (lifts only).
- Modify: `app/(app)/personal-records/page.test.tsx` — add the new field to fixtures; keep the Running/switcher/Customize/one-query-on-expand invariants green against the new tile UI.

`format.ts`, `ExpandChevron.tsx`, `ProgressionChart.tsx`, `ViewSwitcher.tsx`, `RunningView.tsx`, `RunningPRCard.tsx` are **reused unchanged**.

---

## Task 1: API — `DownsampleEstimated1RM` pure helper

**Files:**
- Modify: `internal/workout/onerepmax.go`
- Test: `internal/workout/onerepmax_test.go`

Context: `ListOneRepMaxHistory` returns entries **DESC by `PerformedAt`** (most-recent first), each carrying `MaxEstimated1RM float64` and `Unit`. The spark needs an **ascending** (oldest→newest), rounded, downsampled-to-`cap` slice of the `MaxEstimated1RM` values. `round1` already exists in `progression.go` (package-level). The helper is pure (no IO, no `time.Now`) so it is exhaustively unit-testable, mirroring `RecencyWeightedBaseline`'s style in the same file.

Downsample rule: if `len(entries) <= cap`, take all; else pick exactly `cap` indices by even stride **including the oldest and newest** via `idx = round(i * (n-1) / (cap-1))` for `i` in `0..cap-1`. Return `nil` for empty input or `cap <= 0`.

- [ ] **Step 1: Write the failing tests** in `internal/workout/onerepmax_test.go`

```go
func TestDownsampleEstimated1RM(t *testing.T) {
	mk := func(vals ...float64) []OneRepMaxEntry {
		// DESC by PerformedAt, like ListOneRepMaxHistory returns: index 0
		// is newest. Provide values newest-first so the helper must reverse.
		entries := make([]OneRepMaxEntry, len(vals))
		base := time.Date(2026, 1, 1, 0, 0, 0, 0, time.UTC)
		for i, v := range vals {
			entries[i] = OneRepMaxEntry{
				MaxEstimated1RM: v,
				PerformedAt:     base.Add(time.Duration(-i) * 24 * time.Hour),
				Unit:            user.WeightUnitPounds,
			}
		}
		return entries
	}

	t.Run("nil for empty", func(t *testing.T) {
		if got := DownsampleEstimated1RM(nil, 10); got != nil {
			t.Fatalf("want nil, got %v", got)
		}
	})

	t.Run("nil for non-positive cap", func(t *testing.T) {
		if got := DownsampleEstimated1RM(mk(100, 90), 0); got != nil {
			t.Fatalf("want nil, got %v", got)
		}
	})

	t.Run("ascending and rounded under cap", func(t *testing.T) {
		// newest-first input 320, 312.34, 305 -> ascending 305, 312.3, 320.
		got := DownsampleEstimated1RM(mk(320, 312.34, 305), 10)
		want := []float64{305, 312.3, 320}
		if len(got) != len(want) {
			t.Fatalf("len: want %v got %v", want, got)
		}
		for i := range want {
			if got[i] != want[i] {
				t.Fatalf("at %d: want %v got %v (full %v)", i, want[i], got[i], got)
			}
		}
	})

	t.Run("downsamples to cap keeping oldest and newest", func(t *testing.T) {
		// 11 ascending values 100..110; newest-first input is 110..100.
		vals := make([]float64, 11)
		for i := range vals { // newest-first: 110,109,...,100
			vals[i] = float64(110 - i)
		}
		got := DownsampleEstimated1RM(mk(vals...), 5)
		if len(got) != 5 {
			t.Fatalf("want 5 points, got %d (%v)", len(got), got)
		}
		if got[0] != 100 {
			t.Fatalf("want oldest 100 first, got %v", got[0])
		}
		if got[len(got)-1] != 110 {
			t.Fatalf("want newest 110 last, got %v", got[len(got)-1])
		}
		// strictly ascending
		for i := 1; i < len(got); i++ {
			if got[i] <= got[i-1] {
				t.Fatalf("not ascending at %d: %v", i, got)
			}
		}
	})
}
```

- [ ] **Step 2: Run to verify it fails**

Run: `go test ./internal/workout/ -run TestDownsampleEstimated1RM -v`
Expected: FAIL — `undefined: DownsampleEstimated1RM`.

- [ ] **Step 3: Implement the helper** in `internal/workout/onerepmax.go` (append after `RecencyWeightedBaseline`)

```go
// DownsampleEstimated1RM returns a compact ascending (oldest→newest)
// slice of rounded MaxEstimated1RM values for a per-tile trend spark.
//
// entries are expected DESC by PerformedAt (as ListOneRepMaxHistory
// returns them); the result is reversed to ascending. When more than
// cap entries are present they are downsampled to exactly cap points by
// even stride, always retaining the oldest and newest. Returns nil for
// empty input or a non-positive cap so the JSON field renders as null.
//
// Pure: no IO, no time.Now — a downsample of values already in hand, not
// a new series. The estimated-1RM math is unchanged; this only thins it.
func DownsampleEstimated1RM(entries []OneRepMaxEntry, cap int) []float64 {
	if len(entries) == 0 || cap <= 0 {
		return nil
	}

	// Reverse to ascending: entries[0] is newest.
	asc := make([]float64, len(entries))
	for i, e := range entries {
		asc[len(entries)-1-i] = round1(e.MaxEstimated1RM)
	}

	if len(asc) <= cap {
		return asc
	}

	n := len(asc)
	out := make([]float64, cap)
	for i := 0; i < cap; i++ {
		// Even stride across [0, n-1] inclusive of both ends.
		idx := int(math.Round(float64(i) * float64(n-1) / float64(cap-1)))
		out[i] = asc[idx]
	}
	return out
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `go test ./internal/workout/ -run TestDownsampleEstimated1RM -v`
Expected: PASS (all subtests).

- [ ] **Step 5: Commit**

```bash
git add internal/workout/onerepmax.go internal/workout/onerepmax_test.go
git commit -m "feat(personal-records): add DownsampleEstimated1RM trend helper"
```

---

## Task 2: API — emit `recent_estimated_1rm_points` on the list row

**Files:**
- Modify: `internal/workout/handler.go` (DTO at ~812-822; loop at ~879-921)
- Test: `internal/workout/handler_test.go`

Context: `personalRecords` loops per slug, fetches `entries, _ := h.repo.ListOneRepMaxHistory(ctx, userID, slug, &since, &until)` (the **baseline window**), computes `current_estimated_1rm` via `RecencyWeightedBaseline`, and appends a `personalRecordDTO`. We reuse **those same `entries`** for the spark — no new query, no widening (the baseline window is the source per Open Question #1's safe resolution). The field is the **last** field on the DTO and is `null` when there are no in-window entries.

- [ ] **Step 1: Add the DTO field.** In `internal/workout/handler.go`, extend `personalRecordDTO`:

```go
type personalRecordDTO struct {
	ExerciseID          string     `json:"exercise_id"`
	ExerciseName        string     `json:"exercise_name"`
	WorkoutID           *string    `json:"workout_id"`
	Weight              *float64   `json:"weight"`
	Reps                *int       `json:"reps"`
	Unit                *string    `json:"unit"`
	AchievedAt          *time.Time `json:"achieved_at"`
	CurrentEstimated1RM *float64   `json:"current_estimated_1rm"`
	Estimated1RMUnit    *string    `json:"estimated_1rm_unit"`
	// RecentEstimated1RMPoints is a compact, downsampled ascending
	// (oldest→newest) trend of the estimated 1RM, for the list view's
	// per-tile spark. Built from the same in-window history entries used
	// for CurrentEstimated1RM — no extra query. null when no history.
	RecentEstimated1RMPoints []float64 `json:"recent_estimated_1rm_points"`
}
```

- [ ] **Step 2: Populate it in the loop.** In `personalRecords`, immediately after the `if baseline, ok := RecencyWeightedBaseline(...)` block and before `out = append(out, dto)` (around line 918-920), add:

```go
		// Per-tile trend spark: a downsampled, ascending series of the
		// same in-window history entries already fetched for the
		// baseline above. Capped small so a long-history lift doesn't
		// bloat the row; nil (JSON null) when there's no history.
		dto.RecentEstimated1RMPoints = DownsampleEstimated1RM(entries, recentEstimated1RMPointCap)
```

And add the cap constant near the top of the file's other consts or just above the handler (pick the spot matching the file's existing const placement):

```go
// recentEstimated1RMPointCap bounds the per-tile trend spark on the
// /personal-records list row so a long-history lift doesn't bloat the
// response. ~10 points is enough to read as a trend.
const recentEstimated1RMPointCap = 10
```

- [ ] **Step 3: Write the handler test.** In `internal/workout/handler_test.go`, add an envelope type (near the existing `exerciseOneRMHistoryEnvelope`) and the test. Reuse the existing `dbtest`/`SyncCatalog`/`NewHandler` setup pattern and `authctx.WithUserID`.

```go
type personalRecordsEnvelope struct {
	Message string              `json:"message"`
	Data    []personalRecordDTO `json:"data"`
}

func TestPersonalRecords_RecentEstimated1RMPoints(t *testing.T) {
	ctx := context.Background()
	d := dbtest.New(t)
	repo := NewSQLiteRepository(d)
	exRepo := exercise.NewSQLiteRepository(d)
	if err := exRepo.SyncCatalog(ctx, exercise.Catalog); err != nil {
		t.Fatalf("SyncCatalog: %v", err)
	}
	h := NewHandler(repo, exRepo)

	// Pick two real headline exercises from the curated default so they
	// appear in the response; train one, leave the other never-trained.
	trained := HeadlineExercises[0]
	neverTrained := HeadlineExercises[1]

	mkWorkout := func(at time.Time, weight float64) {
		w := &Workout{
			UserID:      "u1",
			Name:        "session",
			PerformedAt: at,
			Exercises: []WorkoutExercise{{
				ExerciseID: trained,
				Order:      0,
				Sets:       []Set{{Reps: 3, Weight: weight, Unit: user.WeightUnitPounds}},
			}},
		}
		if err := repo.Create(ctx, w); err != nil {
			t.Fatalf("create workout: %v", err)
		}
	}
	// Three in-window workouts, ascending weight over time.
	now := time.Now()
	mkWorkout(now.Add(-40*24*time.Hour), 300)
	mkWorkout(now.Add(-20*24*time.Hour), 310)
	mkWorkout(now.Add(-5*24*time.Hour), 320)

	req := httptest.NewRequest("GET", "/personal-records", nil)
	req = req.WithContext(authctx.WithUserID(req.Context(), "u1"))
	w := httptest.NewRecorder()
	h.personalRecords(w, req)

	if w.Code != http.StatusOK {
		t.Fatalf("status: want 200 got %d (%s)", w.Code, w.Body.String())
	}
	var env personalRecordsEnvelope
	if err := json.Unmarshal(w.Body.Bytes(), &env); err != nil {
		t.Fatalf("unmarshal: %v", err)
	}

	byID := make(map[string]personalRecordDTO, len(env.Data))
	for _, r := range env.Data {
		byID[r.ExerciseID] = r
	}

	tr, ok := byID[trained]
	if !ok {
		t.Fatalf("trained exercise %q missing from response", trained)
	}
	if len(tr.RecentEstimated1RMPoints) == 0 {
		t.Fatalf("trained lift: want a non-empty trend, got %v", tr.RecentEstimated1RMPoints)
	}
	if len(tr.RecentEstimated1RMPoints) > recentEstimated1RMPointCap {
		t.Fatalf("trend exceeds cap %d: %v", recentEstimated1RMPointCap, tr.RecentEstimated1RMPoints)
	}
	pts := tr.RecentEstimated1RMPoints
	for i := 1; i < len(pts); i++ {
		if pts[i] < pts[i-1] {
			t.Fatalf("trend not ascending: %v", pts)
		}
	}
	// Existing field still populated and consistent with the trend's end.
	if tr.CurrentEstimated1RM == nil {
		t.Fatalf("current_estimated_1rm should be set for a trained lift")
	}

	nt, ok := byID[neverTrained]
	if !ok {
		t.Fatalf("never-trained exercise %q missing from response", neverTrained)
	}
	if nt.RecentEstimated1RMPoints != nil {
		t.Fatalf("never-trained lift: want nil trend, got %v", nt.RecentEstimated1RMPoints)
	}
	if nt.CurrentEstimated1RM != nil {
		t.Fatalf("never-trained lift: want nil current_estimated_1rm, got %v", *nt.CurrentEstimated1RM)
	}
}
```

NOTE for the implementer: verify the real identifiers before writing — the curated default headline-lift slice may be named `HeadlineExercises` (a `[]string` of slugs) or similar; grep `effectiveHeadlineExerciseSlugs` and its fallback to find the exact exported name and element type, and adjust `trained`/`neverTrained` accordingly. If the default is a slice of structs, use the slug field. Ensure both chosen slugs are real catalog IDs so the catalog lookup resolves.

- [ ] **Step 4: Run the tests**

Run: `go test ./internal/workout/ -run 'TestPersonalRecords_RecentEstimated1RMPoints|TestDownsampleEstimated1RM' -v`
Expected: PASS. Then `go test ./...` — all green.

- [ ] **Step 5: Commit**

```bash
git add internal/workout/handler.go internal/workout/handler_test.go
git commit -m "feat(personal-records): add recent_estimated_1rm_points to list response"
```

---

## Task 3: Web — readiness module + `PersonalRecord` type field

**Files:**
- Modify: `prog-strength-web/lib/api.ts` (`PersonalRecord` type ~171-181)
- Create: `prog-strength-web/app/(app)/personal-records/_components/readiness.ts`
- Test: `prog-strength-web/app/(app)/personal-records/_components/readiness.test.ts`

Context: this consolidates the gap math currently inlined in `LiftPRCard.tsx` into one tested, React-free module the tiles and the chrome summary both consume — so "due" is defined once. Uses a real `new Date()` (injectable for tests). Field shapes mirror the API: `gap = current_estimated_1rm − weight`, `gapPct = gap / weight × 100`, `ready = gapPct ≥ 5`.

- [ ] **Step 1: Extend the type.** In `lib/api.ts`, add the field to `PersonalRecord` (after `estimated_1rm_unit`):

```ts
export type PersonalRecord = {
  exercise_id: string;
  exercise_name: string;
  workout_id: string | null;
  weight: number | null;
  reps: number | null;
  unit: "lb" | "kg" | null;
  achieved_at: string | null;
  current_estimated_1rm: number | null;
  estimated_1rm_unit: "lb" | "kg" | null;
  // Compact ascending (oldest→newest) estimated-1RM trend for the Lifts
  // view spark. Nullable: an older API omits it and the spark just doesn't
  // render. (Additive field — see SOW personal-records-lifts.)
  recent_estimated_1rm_points: number[] | null;
};
```

- [ ] **Step 2: Write the failing tests** `readiness.test.ts`

```ts
/// <reference types="vitest/globals" />
import { deriveReadiness, summarizeReadiness, READY_THRESHOLD_PCT } from "./readiness";
import type { PersonalRecord } from "@/lib/api";

const base: PersonalRecord = {
  exercise_id: "barbell-bench-press",
  exercise_name: "Barbell Bench Press",
  workout_id: "wk_1",
  weight: 300,
  reps: 3,
  unit: "lb",
  achieved_at: "2026-06-01T00:00:00Z",
  current_estimated_1rm: 330,
  estimated_1rm_unit: "lb",
  recent_estimated_1rm_points: [300, 315, 330],
};

const NOW = new Date("2026-06-19T00:00:00Z");

describe("deriveReadiness", () => {
  it("computes gap, gapPct and ready for a PR'd lift", () => {
    const r = deriveReadiness(base, NOW);
    expect(r.hasPR).toBe(true);
    expect(r.gap).toBe(30);
    expect(r.gapPct).toBeCloseTo(10);
    expect(r.ready).toBe(true); // 10% >= 5%
  });

  it("counts days since the PR against the given now", () => {
    const r = deriveReadiness(base, NOW);
    expect(r.daysSince).toBe(18);
  });

  it("is not ready when the gap is under the threshold (fresh)", () => {
    const r = deriveReadiness({ ...base, current_estimated_1rm: 312 }, NOW); // +4%
    expect(r.ready).toBe(false);
    expect(r.gapPct).toBeCloseTo(4);
  });

  it("returns the null path for a never-PR'd lift", () => {
    const r = deriveReadiness(
      {
        ...base,
        workout_id: null,
        weight: null,
        reps: null,
        achieved_at: null,
        current_estimated_1rm: null,
        recent_estimated_1rm_points: null,
      },
      NOW,
    );
    expect(r.hasPR).toBe(false);
    expect(r.gap).toBeNull();
    expect(r.gapPct).toBeNull();
    expect(r.ready).toBe(false);
    expect(r.daysSince).toBeNull();
  });

  it("detects dumbbell lifts from the name", () => {
    expect(deriveReadiness({ ...base, exercise_name: "Dumbbell Bench Press" }, NOW).isDumbbell).toBe(
      true,
    );
    expect(deriveReadiness(base, NOW).isDumbbell).toBe(false);
  });

  it("computes the gap in kg the same way (unit-agnostic math)", () => {
    const r = deriveReadiness(
      { ...base, unit: "kg", estimated_1rm_unit: "kg", weight: 140, current_estimated_1rm: 154 },
      NOW,
    );
    expect(r.gap).toBe(14);
    expect(r.gapPct).toBeCloseTo(10);
  });

  it("treats threshold exactly at READY_THRESHOLD_PCT as ready", () => {
    const r = deriveReadiness({ ...base, current_estimated_1rm: 315 }, NOW); // +5%
    expect(r.gapPct).toBeCloseTo(READY_THRESHOLD_PCT);
    expect(r.ready).toBe(true);
  });
});

describe("summarizeReadiness", () => {
  it("counts due, tested and total", () => {
    const records: PersonalRecord[] = [
      base, // ready (due), tested
      { ...base, exercise_id: "a", current_estimated_1rm: 305 }, // tested, not due (+1.7%)
      {
        ...base,
        exercise_id: "b",
        workout_id: null,
        weight: null,
        current_estimated_1rm: null,
      }, // untested
    ];
    expect(summarizeReadiness(records, NOW)).toEqual({ due: 1, tested: 2, total: 3 });
  });
});
```

- [ ] **Step 3: Run to verify it fails**

Run (from `prog-strength-web`): `npm run test -- readiness`
Expected: FAIL — cannot resolve `./readiness`.

- [ ] **Step 4: Implement** `readiness.ts`

```ts
/**
 * Pure readiness derivation for the Personal Records Lifts view. The
 * single home for the PR-vs-estimate gap math (formerly inlined in
 * LiftPRCard) — tiles and the chrome summary both consume it so "due" is
 * defined once. React-free; `now` is injectable so the staleness math is
 * deterministic in tests (production passes a real `new Date()`).
 */
import type { PersonalRecord } from "@/lib/api";

/** Gap (%) at or above which a lift is "due" for a new max attempt. This
 * is the shipped 5% rule — kept as the summary's "due" definition; tile
 * magnitude (dot/spark intensity) carries the finer signal. */
export const READY_THRESHOLD_PCT = 5;

const MS_PER_DAY = 1000 * 60 * 60 * 24;
const DUMBBELL_RE = /dumbbell/i;

export type Readiness = {
  hasPR: boolean;
  isDumbbell: boolean;
  gap: number | null;
  gapPct: number | null;
  ready: boolean;
  daysSince: number | null;
};

export function deriveReadiness(record: PersonalRecord, now: Date = new Date()): Readiness {
  const hasPR = record.weight !== null && record.workout_id !== null;
  const isDumbbell = DUMBBELL_RE.test(record.exercise_name);

  let gap: number | null = null;
  let gapPct: number | null = null;
  if (
    hasPR &&
    record.weight !== null &&
    record.weight > 0 &&
    record.current_estimated_1rm !== null
  ) {
    gap = record.current_estimated_1rm - record.weight;
    gapPct = (gap / record.weight) * 100;
  }

  const ready = gapPct !== null && gapPct >= READY_THRESHOLD_PCT;

  let daysSince: number | null = null;
  if (record.achieved_at) {
    const then = new Date(record.achieved_at).getTime();
    if (Number.isFinite(then)) {
      daysSince = Math.floor((now.getTime() - then) / MS_PER_DAY);
    }
  }

  return { hasPR, isDumbbell, gap, gapPct, ready, daysSince };
}

/** Counts for the `N due · N/total tested` chrome summary. */
export function summarizeReadiness(
  records: PersonalRecord[],
  now: Date = new Date(),
): { due: number; tested: number; total: number } {
  let due = 0;
  let tested = 0;
  for (const r of records) {
    const d = deriveReadiness(r, now);
    if (d.hasPR) tested += 1;
    if (d.ready) due += 1;
  }
  return { due, tested, total: records.length };
}
```

- [ ] **Step 5: Run to verify it passes**

Run: `npm run test -- readiness`
Expected: PASS. Then `npm run typecheck`.

- [ ] **Step 6: Commit**

```bash
git add lib/api.ts "app/(app)/personal-records/_components/readiness.ts" "app/(app)/personal-records/_components/readiness.test.ts"
git commit -m "feat(personal-records): add pure readiness module and trend field type"
```

---

## Task 4: Web — Sparkline component

**Files:**
- Create: `prog-strength-web/app/(app)/personal-records/_components/Sparkline.tsx`

Context: a tiny inline SVG trend line for a tile, fed by `recent_estimated_1rm_points`. Pure/presentational. Renders nothing when fewer than 2 points (a single point isn't a trend). `color` is passed by the tile (`--warning` when not fresh, `--discipline-lift-dot` when fresh). No axes, no labels — a spark.

- [ ] **Step 1: Implement** `Sparkline.tsx`

```tsx
/**
 * A minimal inline trend spark — a normalized SVG polyline with no axes
 * or labels, sized to sit inside a compact dashboard tile. Renders null
 * for an empty/absent series or a single point (nothing to trend). The
 * stroke color is supplied by the caller so the tile can tie it to the
 * readiness signal (warm when not fresh, calm steel when fresh).
 */
export function Sparkline({
  points,
  color,
  width = 64,
  height = 18,
}: {
  points: number[] | null | undefined;
  color: string;
  width?: number;
  height?: number;
}) {
  if (!points || points.length < 2) return null;

  const min = Math.min(...points);
  const max = Math.max(...points);
  const span = max - min || 1; // flat series => a centered horizontal line
  const stepX = width / (points.length - 1);
  const pad = 1; // keep the stroke off the top/bottom edge

  const coords = points.map((v, i) => {
    const x = i * stepX;
    const y = pad + (1 - (v - min) / span) * (height - 2 * pad);
    return `${x.toFixed(2)},${y.toFixed(2)}`;
  });

  return (
    <svg
      width={width}
      height={height}
      viewBox={`0 0 ${width} ${height}`}
      fill="none"
      aria-hidden="true"
      className="overflow-visible"
    >
      <polyline
        points={coords.join(" ")}
        stroke={color}
        strokeWidth={1.5}
        strokeLinecap="round"
        strokeLinejoin="round"
      />
    </svg>
  );
}
```

- [ ] **Step 2: Typecheck**

Run: `npm run typecheck`
Expected: PASS (no test needed for a pure presentational primitive — it's exercised via the LiftsView tests in Task 5).

- [ ] **Step 3: Commit**

```bash
git add "app/(app)/personal-records/_components/Sparkline.tsx"
git commit -m "feat(personal-records): add inline Sparkline primitive"
```

---

## Task 5: Web — rebuild LiftsView into the compact-dashboard grid

**Files:**
- Rewrite: `prog-strength-web/app/(app)/personal-records/_components/LiftsView.tsx`
- Delete: `prog-strength-web/app/(app)/personal-records/_components/LiftPRCard.tsx`
- Create: `prog-strength-web/app/(app)/personal-records/_components/LiftsView.test.tsx`

Context: the dense tile grid is the heart of the SOW. Keep `LiftsView`'s `{ records, isPending, error }` contract from `page.tsx`. Renders in **API order** (no re-rank). Each real lift is a `Tile`; the never-PR'd lift is a dashed `UntestedTile`. A tile is a click target (`<button aria-expanded>`) that toggles an inline full-width (`col-span-full`) expanded panel with the existing lazy `ProgressionChart` (`kind="lifts"`, `exerciseId`) + `View workout →`. Reuses `format.ts` (`formatWeight`, `formatDate`), `readiness.ts`, `Sparkline`, `ProgressionChart`. Delete `LiftPRCard.tsx` (its gap math now lives in `readiness.ts`).

Visual spec (conform to v0.4 tokens — no raw hex that duplicates a token; use `var(--…)`):
- Grid: `grid grid-cols-2 gap-2 sm:grid-cols-3 lg:grid-cols-4`.
- Tile: hairline `border-[var(--border)]`, `bg-[var(--surface)]`, `rounded-[14px]` (the 14px panel radius), `p-3`, left-aligned text, `text-left`; full width button. Tight vertical rhythm (`flex flex-col gap-1`).
- Name: `truncate text-xs font-medium`, `title={exercise_name}` for the tooltip.
- PR figure: `weight` big (`text-xl font-semibold tabular-nums tracking-tight`), `unit × reps` small/muted; append `ea` when `isDumbbell` (so it never implies a combined load).
- Readiness dot: an 8px `rounded-full` span. When `ready`-magnitude (`gapPct >= READY_THRESHOLD_PCT`): `background: var(--warning)` with `opacity` scaling `min(gapPct, 30) / 30` floored at ~0.35. When fresh (`hasPR` and `gapPct < 5`): `background: var(--discipline-lift-dot)`, full opacity. No dot when untested.
- Delta: when `gapPct >= 5` → `+{round(gap)}` in `text-[var(--success)]` plus faint `est {round(est)}` in `text-[var(--faint)]`; when fresh → a calm `·` in `text-[var(--faint)]`.
- Recency line: `formatDate`-adjacent `fmtAgo`-style — use the existing `formatDate(achieved_at)` prefixed/styled as `text-[10px] text-[var(--muted)]` (no new formatter needed; `format.ts` is the shared formatter). Render only when `hasPR`.
- Spark: `<Sparkline points={record.recent_estimated_1rm_points} color={fresh ? "var(--discipline-lift-dot)" : "var(--warning)"} />`. Omitted automatically when the series is null/empty/<2.
- Untested tile: `border-dashed border-[var(--border)] bg-transparent`, muted `—` and `untested` label, no dot/delta/spark/expand. Not a button.
- Skeleton (`isPending`): a grid of ~8 `animate-pulse` `bg-[var(--surface-2)]` tiles matching the grid.
- Error: keep the existing inline danger panel markup.
- Empty (`records.length === 0`): keep `No headline lifts configured.`

Accessibility: the tile button's accessible name must include the exercise name and its expand state — use `aria-expanded` and `aria-label={`${name}, ${expanded ? "hide" : "show"} progression`}`. Only one tile expanded at a time (track `expandedId`).

- [ ] **Step 1: Write the failing component tests** `LiftsView.test.tsx`

```tsx
/// <reference types="vitest/globals" />
import { render, screen, fireEvent, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { PersonalRecord } from "@/lib/api";

// ProgressionChart fetches history lazily; spy the api fn it calls so we
// can assert "exactly one query per lift on expand".
const historyMock = vi.hoisted(() => vi.fn(async () => ({
  exercise_id: "barbell-bench-press",
  exercise_name: "Barbell Bench Press",
  unit: "lb",
  points: [
    { workout_id: "w1", performed_at: "2026-01-01T00:00:00Z", estimated_1rm: 300 },
    { workout_id: "w2", performed_at: "2026-03-01T00:00:00Z", estimated_1rm: 320 },
  ],
})));

vi.mock("@/lib/auth", () => ({ getToken: () => "test-token" }));
vi.mock("@/lib/api", async (orig) => ({
  ...(await orig<typeof import("@/lib/api")>()),
  getExerciseOneRMHistory: historyMock,
}));

import { getExerciseOneRMHistory } from "@/lib/api";
import { LiftsView } from "./LiftsView";

const READY: PersonalRecord = {
  exercise_id: "barbell-bench-press",
  exercise_name: "Barbell Bench Press",
  workout_id: "wk_1",
  weight: 300,
  reps: 3,
  unit: "lb",
  achieved_at: "2026-04-01T00:00:00Z",
  current_estimated_1rm: 360, // +20% => due, warm
  estimated_1rm_unit: "lb",
  recent_estimated_1rm_points: [300, 330, 360],
};
const FRESH: PersonalRecord = {
  exercise_id: "back-squat",
  exercise_name: "Back Squat",
  workout_id: "wk_2",
  weight: 405,
  reps: 2,
  unit: "lb",
  achieved_at: "2026-04-15T00:00:00Z",
  current_estimated_1rm: 410, // +1.2% => fresh, no spark? has series
  estimated_1rm_unit: "lb",
  recent_estimated_1rm_points: [400, 405, 410],
};
const NO_SERIES: PersonalRecord = {
  ...FRESH,
  exercise_id: "deadlift",
  exercise_name: "Deadlift",
  recent_estimated_1rm_points: null,
};
const UNTESTED: PersonalRecord = {
  exercise_id: "overhead-press",
  exercise_name: "Overhead Press",
  workout_id: null,
  weight: null,
  reps: null,
  unit: null,
  achieved_at: null,
  current_estimated_1rm: null,
  estimated_1rm_unit: null,
  recent_estimated_1rm_points: null,
};

function renderView(records: PersonalRecord[]) {
  const client = new QueryClient({
    defaultOptions: { queries: { retry: false, staleTime: Infinity } },
  });
  return render(
    <QueryClientProvider client={client}>
      <LiftsView records={records} isPending={false} error={null} />
    </QueryClientProvider>,
  );
}

beforeEach(() => historyMock.mockClear());

describe("LiftsView (compact dashboard)", () => {
  it("renders a tile per record including the dashed untested tile", () => {
    renderView([READY, UNTESTED]);
    expect(screen.getByText("Barbell Bench Press")).toBeInTheDocument();
    expect(screen.getByText("Overhead Press")).toBeInTheDocument();
    expect(screen.getByText("untested")).toBeInTheDocument();
  });

  it("omits the spark when the series is null", () => {
    const { container } = renderView([NO_SERIES]);
    // No svg polyline rendered for a null series.
    expect(container.querySelector("polyline")).toBeNull();
  });

  it("renders a spark when the series is present", () => {
    const { container } = renderView([READY]);
    expect(container.querySelector("polyline")).not.toBeNull();
  });

  it("shows the +delta for a due lift", () => {
    renderView([READY]);
    expect(screen.getByText(/\+60/)).toBeInTheDocument(); // 360 - 300
  });

  it("expands a tile and fires exactly one history query, reusing cache on re-expand", async () => {
    renderView([READY]);
    const tile = screen.getByRole("button", { name: /Barbell Bench Press/ });
    fireEvent.click(tile);
    await waitFor(() => expect(getExerciseOneRMHistory).toHaveBeenCalledTimes(1));
    // View workout link is inside the expanded panel.
    expect(screen.getByRole("link", { name: /View workout/ })).toBeInTheDocument();
    // Collapse + re-expand: cached, no second call.
    fireEvent.click(screen.getByRole("button", { name: /Barbell Bench Press/ }));
    fireEvent.click(screen.getByRole("button", { name: /Barbell Bench Press/ }));
    await waitFor(() => expect(getExerciseOneRMHistory).toHaveBeenCalledTimes(1));
  });

  it("does not make the untested tile expandable", () => {
    renderView([UNTESTED]);
    expect(screen.queryByRole("button", { name: /Overhead Press/ })).toBeNull();
  });

  it("renders the skeleton grid while pending", () => {
    const { container } = render(
      <QueryClientProvider client={new QueryClient()}>
        <LiftsView records={undefined} isPending={true} error={null} />
      </QueryClientProvider>,
    );
    expect(container.querySelectorAll(".animate-pulse").length).toBeGreaterThan(0);
  });

  it("shows the empty message when there are no records", () => {
    renderView([]);
    expect(screen.getByText("No headline lifts configured.")).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npm run test -- LiftsView`
Expected: FAIL (new tiles/markup not implemented).

- [ ] **Step 3: Implement** the new `LiftsView.tsx` (full rewrite). Implementer: write production-quality JSX per the visual spec above; structure below is the required shape (fill body to satisfy the tests and the spec — dot scaling, delta, recency, spark, inline expand). Keep it one file with `Tile` and `UntestedTile` local components.

```tsx
"use client";

/**
 * The Lifts view as a compact-dashboard tile grid: every headline lift
 * visible at once, each a mini-stat (PR figure, a readiness dot and est-1RM
 * delta that scale with the gap, recency, and an inline trend spark). The
 * never-PR'd lift gets a quiet dashed "untested" tile. A tile is a click
 * target that expands the lazy ProgressionChart inline with View workout →.
 * Conforms to design-system v0.4 — neutral by default, --warning only on the
 * scaling readiness signal, --success on the positive delta; the periwinkle
 * accent stays app-chrome and is never used here as selection.
 */

import { useState } from "react";
import Link from "next/link";
import type { PersonalRecord } from "@/lib/api";
import { ProgressionChart } from "./ProgressionChart";
import { Sparkline } from "./Sparkline";
import { deriveReadiness, READY_THRESHOLD_PCT } from "./readiness";
import { formatWeight, formatDate } from "./format";

const DOT_GAP_CAP = 30; // gapPct at which the readiness dot is fully saturated
const DOT_MIN_OPACITY = 0.35;
const SKELETON_TILES = 8;

export function LiftsView({
  records,
  isPending,
  error,
}: {
  records: PersonalRecord[] | undefined;
  isPending: boolean;
  error: Error | null;
}) {
  const [expandedId, setExpandedId] = useState<string | null>(null);

  if (error) {
    return (
      <div className="rounded-md border border-[var(--danger)]/40 bg-[var(--danger)]/10 px-3 py-2 text-sm text-[var(--danger)]">
        {error.message}
      </div>
    );
  }

  if (isPending || !records) {
    return (
      <div className="grid grid-cols-2 gap-2 sm:grid-cols-3 lg:grid-cols-4">
        {Array.from({ length: SKELETON_TILES }).map((_, i) => (
          <div
            key={i}
            className="h-28 animate-pulse rounded-[14px] border border-[var(--border)] bg-[var(--surface-2)]"
            aria-hidden="true"
          />
        ))}
      </div>
    );
  }

  if (records.length === 0) {
    return <p className="text-sm text-[var(--muted)]">No headline lifts configured.</p>;
  }

  return (
    <div className="grid grid-cols-2 gap-2 sm:grid-cols-3 lg:grid-cols-4">
      {records.map((r) => {
        const d = deriveReadiness(r);
        if (!d.hasPR) return <UntestedTile key={r.exercise_id} record={r} />;
        const expanded = expandedId === r.exercise_id;
        return (
          <Tile
            key={r.exercise_id}
            record={r}
            expanded={expanded}
            onToggle={() => setExpandedId((cur) => (cur === r.exercise_id ? null : r.exercise_id))}
          />
        );
      })}
    </div>
  );
}

function Tile({
  record,
  expanded,
  onToggle,
}: {
  record: PersonalRecord;
  expanded: boolean;
  onToggle: () => void;
}) {
  const d = deriveReadiness(record);
  const fresh = d.gapPct !== null && d.gapPct < READY_THRESHOLD_PCT;
  const dueByMagnitude = d.gapPct !== null && d.gapPct >= READY_THRESHOLD_PCT;

  const dotColor = fresh ? "var(--discipline-lift-dot)" : "var(--warning)";
  const dotOpacity = dueByMagnitude
    ? Math.max(DOT_MIN_OPACITY, Math.min(d.gapPct as number, DOT_GAP_CAP) / DOT_GAP_CAP)
    : 1;

  const sparkColor = fresh ? "var(--discipline-lift-dot)" : "var(--warning)";

  return (
    <>
      <button
        type="button"
        onClick={onToggle}
        aria-expanded={expanded}
        aria-label={`${record.exercise_name}, ${expanded ? "hide" : "show"} progression`}
        title={record.exercise_name}
        className="flex flex-col gap-1 rounded-[14px] border border-[var(--border)] bg-[var(--surface)] p-3 text-left transition hover:border-[var(--foreground)]/20"
      >
        <div className="flex items-center justify-between gap-1">
          <span className="truncate text-xs font-medium">{record.exercise_name}</span>
          <span
            className="h-2 w-2 shrink-0 rounded-full"
            style={{ backgroundColor: dotColor, opacity: dotOpacity }}
            aria-hidden="true"
          />
        </div>

        <p className="text-xl font-semibold tabular-nums tracking-tight">
          {formatWeight(record.weight, record.unit)}
          <span className="ml-1 text-[11px] font-normal text-[var(--muted)]">
            {d.isDumbbell ? "ea " : ""}× {record.reps}
          </span>
        </p>

        <div className="flex items-center justify-between gap-1 text-[11px] tabular-nums">
          {dueByMagnitude && d.gap !== null ? (
            <span className="text-[var(--success)]">
              +{Math.round(d.gap)}
              {record.current_estimated_1rm !== null && (
                <span className="ml-1 text-[var(--faint)]">
                  est {Math.round(record.current_estimated_1rm)}
                </span>
              )}
            </span>
          ) : (
            <span className="text-[var(--faint)]">·</span>
          )}
          <Sparkline points={record.recent_estimated_1rm_points} color={sparkColor} />
        </div>

        <p className="text-[10px] text-[var(--muted)]">{formatDate(record.achieved_at)}</p>
      </button>

      {expanded && (
        <div className="col-span-full rounded-[14px] border border-[var(--border)] bg-[var(--surface)] p-3">
          <ProgressionChart kind="lifts" exerciseId={record.exercise_id} />
          {record.workout_id && (
            <Link
              href={`/workouts/${record.workout_id}`}
              className="mt-2 inline-block text-xs text-[var(--accent)] hover:underline"
            >
              View workout →
            </Link>
          )}
        </div>
      )}
    </>
  );
}

function UntestedTile({ record }: { record: PersonalRecord }) {
  return (
    <div
      title={record.exercise_name}
      className="flex flex-col gap-1 rounded-[14px] border border-dashed border-[var(--border)] bg-transparent p-3"
    >
      <span className="truncate text-xs font-medium text-[var(--muted)]">
        {record.exercise_name}
      </span>
      <p className="text-xl font-semibold tabular-nums text-[var(--faint)]">—</p>
      <p className="text-[10px] text-[var(--faint)]">untested</p>
    </div>
  );
}
```

- [ ] **Step 4: Delete the old card.**

```bash
git rm "app/(app)/personal-records/_components/LiftPRCard.tsx"
```

- [ ] **Step 5: Run the tests + typecheck + lint**

Run: `npm run test -- LiftsView readiness` then `npm run typecheck` then `npm run lint`
Expected: PASS. Fix any unused-import / a11y lint findings by fixing code (never disabling rules).

- [ ] **Step 6: Commit**

```bash
git add -A "app/(app)/personal-records/_components/"
git commit -m "feat(personal-records): rebuild Lifts view into compact-dashboard tile grid"
```

---

## Task 6: Web — page chrome summary + keep page.test.tsx green

**Files:**
- Modify: `prog-strength-web/app/(app)/personal-records/page.tsx`
- Modify: `prog-strength-web/app/(app)/personal-records/page.test.tsx`

Context: add the `N due · N/total tested` summary to the Lifts header (hidden on Running), derived from `liftsQuery.data` via `summarizeReadiness`. The existing `page.test.tsx` fixtures must gain `recent_estimated_1rm_points`, and its expand assertions must target the new tile button (the chevron is gone) while keeping the Running/switcher/Customize/one-query invariants.

- [ ] **Step 1: Add the summary to `page.tsx`.** Import `summarizeReadiness`, and in the lifts header (after the title/ViewSwitcher row or beside the intro `<p>`), render when `view === "lifts" && liftsQuery.data`:

```tsx
// near the other imports
import { summarizeReadiness } from "./_components/readiness";
```

Inside the component, compute and render (place the summary in the header, e.g. right after the intro `<p>` or beside the ViewSwitcher — keep it small, `text-xs text-[var(--muted)]`, tabular):

```tsx
{view === "lifts" && liftsQuery.data && liftsQuery.data.length > 0 && (() => {
  const s = summarizeReadiness(liftsQuery.data);
  return (
    <p className="text-xs tabular-nums text-[var(--muted)]">
      <span className="text-[var(--foreground)]">{s.due}</span> due ·{" "}
      <span className="text-[var(--foreground)]">
        {s.tested}/{s.total}
      </span>{" "}
      tested
    </p>
  );
})()}
```

- [ ] **Step 2: Update `page.test.tsx` fixtures and expand assertions.** Add `recent_estimated_1rm_points` to each `LIFTS` fixture (e.g. `[305, 312, 320]` for bench, `[395, 405, 410]` for squat). Replace the chevron-based expand test so it clicks the tile button by name and asserts exactly one history call + cache reuse:

```tsx
it("fires exactly one history query on expand and reuses the cache on re-expand", async () => {
  renderPage();
  await screen.findByText("Barbell Bench Press");

  const tile = screen.getByRole("button", { name: /Barbell Bench Press/ });
  fireEvent.click(tile);
  await waitFor(() => expect(getExerciseOneRMHistory).toHaveBeenCalledTimes(1));

  fireEvent.click(screen.getByRole("button", { name: /Barbell Bench Press/ }));
  fireEvent.click(screen.getByRole("button", { name: /Barbell Bench Press/ }));
  await waitFor(() => expect(getExerciseOneRMHistory).toHaveBeenCalledTimes(1));
});
```

Also add an assertion that the summary renders on lifts:

```tsx
it("shows the readiness summary on lifts", async () => {
  renderPage();
  await screen.findByText("Barbell Bench Press");
  expect(screen.getByText(/tested/)).toBeInTheDocument();
});
```

Keep the existing Running / Customize-visibility / does-not-fetch-running tests as-is (they don't depend on card internals). Implementer: if any existing assertion referenced `LiftPRCard`-only copy (e.g. "Set on", "Current estimated 1RM", "Time for a max?"), update it to the new tile semantics — verify by reading the current file before editing.

- [ ] **Step 3: Run the full web suite + gates**

Run: `npm run test` then `npm run typecheck` then `npm run lint` then `npm run build`
Expected: all PASS.

- [ ] **Step 4: Commit**

```bash
git add "app/(app)/personal-records/page.tsx" "app/(app)/personal-records/page.test.tsx"
git commit -m "feat(personal-records): add readiness summary to Lifts header"
```

---

## Self-Review (controller checklist, done before dispatch)

- **Spec coverage:** API trend field (T1–T2); readiness module + type (T3); spark (T4); tile grid incl. dashed untested tile, dot/delta/spark, inline expand + View workout, skeleton/error/empty (T5); chrome summary + preserved switcher/Customize/one-query invariant (T6). Running view untouched. ✔
- **No design-system change:** all colors via `var(--…)` tokens; periwinkle only as app-chrome; `--warning`/`--success`/`--discipline-lift-dot` per the SOW's color discipline. ✔
- **No extra DB query (API):** reuses the loop's already-fetched `entries`. ✔
- **Type consistency:** `recent_estimated_1rm_points` (API JSON ↔ `number[] | null` web), `deriveReadiness`/`summarizeReadiness`/`READY_THRESHOLD_PCT`, `DownsampleEstimated1RM`, `recentEstimated1RMPointCap` used consistently across tasks. ✔
- **Open Questions:** all four resolved and recorded above. ✔
