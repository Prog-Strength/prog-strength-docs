# Holistic Training Analysis (Training Snapshot) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `GET /training-snapshot` endpoint to the API, expose it as a `get_training_snapshot` MCP forwarder tool, and rename the agent's `analyze_progress` intent to `analyze_training` (forced to the complex model, prefetching the snapshot for the user's current local week).

**Architecture:** A new `internal/snapshot` package in the Go API composes existing domain repos into one compact, defensively-degrading view (modeled on `internal/dashboard`). The MCP server gains a pure-forwarder tool. The agent renames its analysis intent, threads the client timezone into the prefetch layer, and deterministically forces the complex tier for the analysis intent.

**Tech Stack:** Go 1.25 (chi, SQLite, `internal/daterange`, `internal/httpresp`), Python 3.12 (FastMCP, httpx, respx, pytest), Python 3.12 agent (anthropic SDK, MCP client, pytest).

---

## Cross-repo design notes (read first)

- **Dependency order:** api → mcp → agent. Each repo is a separate branch `feat/analyze-training-snapshot` and a separate PR.
- **Response shape** (the API `data` payload, also what the MCP tool returns unwrapped and what the agent formats):

```jsonc
{
  "period": { "start_date": "2026-06-15", "end_date": "2026-06-21",
              "timezone": "America/Denver", "days": 7 },
  "strength": { "session_count": 3, "total_volume": 48250, "unit": "lb",
    "by_muscle_group": [ { "muscle_group": "chest", "sets": 12, "volume": 9800 } ],
    "sessions": [ { "date": "2026-06-16", "name": "Chest & Back",
      "top_sets": [ { "exercise": "barbell-bench-press", "weight": 265, "reps": 8, "est_1rm": 331 } ],
      "prs": [ { "exercise": "barbell-bench-press", "kind": "weight" } ] } ],
    "headline_prs": [ "335 lb high-bar back squat PR" ] },
  "running": { "run_count": 2, "total_distance_m": 14200, "total_duration_s": 4380,
    "avg_pace_sec_per_km": 308,
    "runs": [ { "date": "2026-06-17", "name": "Easy 5", "distance_m": 8000,
                "duration_s": 2520, "avg_pace_sec_per_km": 315 } ],
    "new_best_efforts": [ { "distance_key": "5k", "time_seconds": 1320 } ] },
  "steps": { "days_logged": 6, "avg": 9120, "total": 54720, "goal": 10000,
    "by_day": [ { "date": "2026-06-15", "steps": 8800 } ] },
  "bodyweight": { "unit": "lb", "start": 184.2, "end": 183.4, "delta": -0.8,
    "readings": [ { "date": "2026-06-15", "weight": 184.2 } ] },
  "nutrition": { "days_logged": 5,
    "avg": { "calories": 2680, "protein_g": 198, "fat_g": 82, "carbs_g": 290 },
    "goals": { "calories": 2700, "protein_g": 200, "fat_g": 80, "carbs_g": 295 },
    "by_day": [ { "date": "2026-06-15", "calories": 2710, "protein_g": 205, "fat_g": 79, "carbs_g": 288 } ] },
  "consistency": { "active_days": 5, "window_days": 7 }
}
```

- **Defensive degradation:** each top-level domain section (`strength`, `running`, `steps`, `bodyweight`, `nutrition`) is a pointer that is `null` **only** when that domain's repo read errors. An empty-but-healthy domain (no runs logged) renders zeros/empty arrays, never null. `period` and `consistency` are always present (value types).
- **PR semantics (decided):** `workout.PersonalRecordEvent` is a heaviest-weight PR break (see `internal/workout/personal_record.go`). So per-session `prs[].kind` is the constant `"weight"`, and `headline_prs` entries are formatted `"<weight> <unit> <exercise name> PR"` (e.g. `"335 lb high-bar back squat PR"`). The SOW example said "est. 1RM PR" but the system tracks weight PRs; `"weight"` is the honest label. Note this in the API PR body.
- **"new best efforts this window" (resolves SOW Open Question 3):** `activity.Repository.GetUserRunningBestEfforts(ctx, userID)` returns each distance's current best with an `ActivityStartTime`. A best effort is "new this window" iff its `ActivityStartTime` falls in `[start, end)`. No heavier query needed.

---

# Repo A — prog-strength-api

Work from: `/workspace/prog-strength-api` on branch `feat/analyze-training-snapshot`.

New package `internal/snapshot/` (sibling to `internal/dashboard/`). No new tables, migrations, or repositories.

**File structure:**
- Create `internal/snapshot/dto.go` — response types only.
- Create `internal/snapshot/aggregate.go` — pure aggregation functions (no IO, no `time.Now`).
- Create `internal/snapshot/aggregate_test.go` — table tests for the pure functions.
- Create `internal/snapshot/service.go` — narrow consumer interfaces + `Service` with defensive `Build`.
- Create `internal/snapshot/service_test.go` — fan-out, defensive-degradation, empty-but-healthy tests using tiny fakes.
- Create `internal/snapshot/handler.go` — `Handler`, `NewHandler`, `Mount`, query parsing + default window.
- Create `internal/snapshot/handler_test.go` — query parsing (date / range / default), DST, auth-required.
- Modify `internal/server/server.go` — mount the handler in the JWT-gated group next to `dashboard`.

> First: `cd /workspace/prog-strength-api && git checkout -b feat/analyze-training-snapshot`. Confirm `GOTOOLCHAIN=auto` (the repo's go.mod requires go 1.25.x; the toolchain auto-downloads).

---

### Task A1: Snapshot DTOs + pure aggregation helpers

**Files:**
- Create: `internal/snapshot/dto.go`
- Create: `internal/snapshot/aggregate.go`
- Test: `internal/snapshot/aggregate_test.go`

- [ ] **Step 1: Write `dto.go`** (types only — no test for plain structs)

```go
package snapshot

// Snapshot is the agent-facing, date-windowed, pre-aggregated view across
// every training domain. Each domain section is a pointer: nil ONLY when
// that domain's repository read failed (defensive degradation). An
// empty-but-healthy domain renders zeros/empty slices, never nil.
type Snapshot struct {
	Period      Period             `json:"period"`
	Strength    *StrengthSection   `json:"strength"`
	Running     *RunningSection    `json:"running"`
	Steps       *StepsSection      `json:"steps"`
	Bodyweight  *BodyweightSection `json:"bodyweight"`
	Nutrition   *NutritionSection  `json:"nutrition"`
	Consistency Consistency        `json:"consistency"`
}

type Period struct {
	StartDate string `json:"start_date"`
	EndDate   string `json:"end_date"`
	Timezone  string `json:"timezone"`
	Days      int    `json:"days"`
}

type StrengthSection struct {
	SessionCount  int                 `json:"session_count"`
	TotalVolume   float64             `json:"total_volume"`
	Unit          string              `json:"unit"`
	ByMuscleGroup []MuscleGroupVolume `json:"by_muscle_group"`
	Sessions      []StrengthSession   `json:"sessions"`
	HeadlinePRs   []string            `json:"headline_prs"`
}

type MuscleGroupVolume struct {
	MuscleGroup string  `json:"muscle_group"`
	Sets        int     `json:"sets"`
	Volume      float64 `json:"volume"`
}

type StrengthSession struct {
	Date    string      `json:"date"`
	Name    string      `json:"name"`
	TopSets []TopSet    `json:"top_sets"`
	PRs     []SessionPR `json:"prs"`
}

type TopSet struct {
	Exercise string  `json:"exercise"`
	Weight   float64 `json:"weight"`
	Reps     int     `json:"reps"`
	Est1RM   float64 `json:"est_1rm"`
}

type SessionPR struct {
	Exercise string `json:"exercise"`
	Kind     string `json:"kind"`
}

type RunningSection struct {
	RunCount        int          `json:"run_count"`
	TotalDistanceM  float64      `json:"total_distance_m"`
	TotalDurationS  int          `json:"total_duration_s"`
	AvgPaceSecPerKm *float64     `json:"avg_pace_sec_per_km"`
	Runs            []RunSummary `json:"runs"`
	NewBestEfforts  []BestEffort `json:"new_best_efforts"`
}

type RunSummary struct {
	Date            string   `json:"date"`
	Name            string   `json:"name"`
	DistanceM       float64  `json:"distance_m"`
	DurationS       int      `json:"duration_s"`
	AvgPaceSecPerKm *float64 `json:"avg_pace_sec_per_km"`
}

type BestEffort struct {
	DistanceKey string `json:"distance_key"`
	TimeSeconds int    `json:"time_seconds"`
}

type StepsSection struct {
	DaysLogged int        `json:"days_logged"`
	Avg        int        `json:"avg"`
	Total      int        `json:"total"`
	Goal       int        `json:"goal"`
	ByDay      []StepsDay `json:"by_day"`
}

type StepsDay struct {
	Date  string `json:"date"`
	Steps int    `json:"steps"`
}

type BodyweightSection struct {
	Unit     string              `json:"unit"`
	Start    float64             `json:"start"`
	End      float64             `json:"end"`
	Delta    float64             `json:"delta"`
	Readings []BodyweightReading `json:"readings"`
}

type BodyweightReading struct {
	Date   string  `json:"date"`
	Weight float64 `json:"weight"`
}

type NutritionSection struct {
	DaysLogged int            `json:"days_logged"`
	Avg        MacroSet       `json:"avg"`
	Goals      MacroSet       `json:"goals"`
	ByDay      []NutritionDay `json:"by_day"`
}

type MacroSet struct {
	Calories int `json:"calories"`
	ProteinG int `json:"protein_g"`
	FatG     int `json:"fat_g"`
	CarbsG   int `json:"carbs_g"`
}

type NutritionDay struct {
	Date     string `json:"date"`
	Calories int    `json:"calories"`
	ProteinG int    `json:"protein_g"`
	FatG     int    `json:"fat_g"`
	CarbsG   int    `json:"carbs_g"`
}

type Consistency struct {
	ActiveDays int `json:"active_days"`
	WindowDays int `json:"window_days"`
}
```

- [ ] **Step 2: Write the failing aggregation tests** in `aggregate_test.go`

Cover: `aggregateStrength` (volume sum, per-muscle-group rollup attributing each exercise's sets+volume to every muscle group it targets, top sets ranked by Epley est-1RM capped at 3, session PRs from events, headline PR strings); `aggregateRunning` (filters to running type, totals, section avg pace = totalDuration / (totalDistance/1000), nil when no distance, new-best-efforts filtered by `ActivityStartTime` in window); `aggregateSteps` (days_logged counts entries with steps>0, avg over logged days, total, goal); `aggregateBodyweight` (oldest reading = start, newest = end, delta = end-start, readings oldest→newest); `aggregateNutrition` (days_logged = len(rows), avg rounded to int over logged days, goals passthrough, by_day rounded); `countActiveDays` (union of strength/running/steps dates).

Use small constructed inputs. Example shape (engineer fills the rest analogously):

```go
package snapshot

import (
	"testing"
	"time"

	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/activity"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/exercise"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/workout"
)

func mustLoc(t *testing.T) *time.Location {
	t.Helper()
	loc, err := time.LoadLocation("America/Denver")
	if err != nil {
		t.Fatal(err)
	}
	return loc
}

func TestAggregateStrength_VolumeMuscleGroupsTopSetsPRs(t *testing.T) {
	loc := mustLoc(t)
	at := time.Date(2026, 6, 16, 18, 0, 0, 0, time.UTC)
	bench := exercise.Exercise{ID: "barbell-bench-press", Name: "barbell bench press",
		MuscleGroups: []exercise.MuscleGroup{exercise.MuscleChest, exercise.MuscleTriceps}}
	w := workout.Workout{ID: "w1", Name: "Chest & Back", PerformedAt: at,
		Exercises: []workout.WorkoutExercise{{ExerciseID: "barbell-bench-press", Sets: []workout.Set{
			{Reps: 8, Weight: 225}, {Reps: 8, Weight: 265},
		}}}}
	prs := []workout.PersonalRecordEvent{{ExerciseID: "barbell-bench-press", Weight: 265, Reps: 8, WorkoutID: "w1"}}
	got := aggregateStrength([]workout.Workout{w}, prs, []exercise.Exercise{bench}, "lb", loc)

	if got.SessionCount != 1 {
		t.Fatalf("session_count = %d, want 1", got.SessionCount)
	}
	if got.TotalVolume != 225*8+265*8 {
		t.Fatalf("total_volume = %v", got.TotalVolume)
	}
	// chest and triceps each get the exercise's 2 sets and full volume.
	if len(got.ByMuscleGroup) != 2 {
		t.Fatalf("by_muscle_group = %+v", got.ByMuscleGroup)
	}
	if got.Sessions[0].TopSets[0].Exercise != "barbell-bench-press" ||
		got.Sessions[0].TopSets[0].Weight != 265 {
		t.Fatalf("top set = %+v", got.Sessions[0].TopSets[0])
	}
	if len(got.Sessions[0].PRs) != 1 || got.Sessions[0].PRs[0].Kind != "weight" {
		t.Fatalf("session prs = %+v", got.Sessions[0].PRs)
	}
	if len(got.HeadlinePRs) != 1 {
		t.Fatalf("headline prs = %+v", got.HeadlinePRs)
	}
}

func TestAggregateRunning_TotalsPaceAndNewBests(t *testing.T) {
	loc := mustLoc(t)
	start := time.Date(2026, 6, 15, 6, 0, 0, 0, time.UTC)
	end := time.Date(2026, 6, 22, 6, 0, 0, 0, time.UTC)
	runStart := time.Date(2026, 6, 17, 13, 0, 0, 0, time.UTC)
	pace := 315.0
	runs := []activity.Activity{{
		ActivityType: activity.ActivityRunning, StartTime: runStart, DistanceMeters: 8000,
		DurationSeconds: 2520, AvgPaceSecPerKm: &pace, Name: strptr("Easy 5"),
	}}
	bests := []activity.RunningBestEffort{
		{DistanceKey: "5k", DurationSeconds: 1320, ActivityStartTime: runStart}, // in window → new
		{DistanceKey: "10k", DurationSeconds: 3000, ActivityStartTime: time.Date(2026, 1, 1, 0, 0, 0, 0, time.UTC)}, // old
	}
	got := aggregateRunning(runs, bests, start, end, loc)
	if got.RunCount != 1 || got.TotalDistanceM != 8000 || got.TotalDurationS != 2520 {
		t.Fatalf("totals = %+v", got)
	}
	if got.AvgPaceSecPerKm == nil || *got.AvgPaceSecPerKm != 315 {
		t.Fatalf("avg pace = %v", got.AvgPaceSecPerKm)
	}
	if len(got.NewBestEfforts) != 1 || got.NewBestEfforts[0].DistanceKey != "5k" ||
		got.NewBestEfforts[0].TimeSeconds != 1320 {
		t.Fatalf("new bests = %+v", got.NewBestEfforts)
	}
}

func strptr(s string) *string { return &s }
```

(Also add `TestAggregateSteps_*`, `TestAggregateBodyweight_*`, `TestAggregateNutrition_*`, `TestCountActiveDays_*` following the same pattern, asserting the rules in Step 2's description.)

- [ ] **Step 3: Run tests, verify they FAIL to compile** — Run: `GOTOOLCHAIN=auto go test ./internal/snapshot/...` → expect undefined `aggregateStrength` etc.

- [ ] **Step 4: Write `aggregate.go`** — pure functions. Reference helpers already in the repo: `workout.EpleyOneRM(weight, reps)`, `exercise.MuscleGroup` constants, `activity.ActivityRunning`.

```go
package snapshot

import (
	"fmt"
	"math"
	"sort"
	"time"

	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/activity"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/exercise"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/nutrition"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/steps"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/workout"
)

const maxTopSetsPerSession = 3

// aggregateStrength rolls up the window's workouts into the strength
// section. Always returns a non-nil section (empty input → zeroed
// section) so an empty-but-healthy domain renders, not nulls.
func aggregateStrength(
	workouts []workout.Workout,
	prEvents []workout.PersonalRecordEvent,
	exercises []exercise.Exercise,
	unit string,
	loc *time.Location,
) *StrengthSection {
	muscleByExercise := map[string][]exercise.MuscleGroup{}
	nameByExercise := map[string]string{}
	for _, e := range exercises {
		muscleByExercise[e.ID] = e.MuscleGroups
		nameByExercise[e.ID] = e.Name
	}
	prsByWorkout := map[string][]workout.PersonalRecordEvent{}
	for _, ev := range prEvents {
		prsByWorkout[ev.WorkoutID] = append(prsByWorkout[ev.WorkoutID], ev)
	}

	sec := &StrengthSection{Unit: unit, ByMuscleGroup: []MuscleGroupVolume{},
		Sessions: []StrengthSession{}, HeadlinePRs: []string{}}
	sec.SessionCount = len(workouts)

	type mg struct {
		sets   int
		volume float64
	}
	mgTotals := map[exercise.MuscleGroup]*mg{}

	for _, w := range workouts {
		session := StrengthSession{
			Date: w.PerformedAt.In(loc).Format("2006-01-02"),
			Name: w.Name, TopSets: []TopSet{}, PRs: []SessionPR{},
		}
		var bestByExercise []TopSet
		for _, we := range w.Exercises {
			setCount := len(we.Sets)
			var exVol float64
			var best TopSet
			haveBest := false
			for _, s := range we.Sets {
				exVol += s.Weight * float64(s.Reps)
				sec.TotalVolume += s.Weight * float64(s.Reps)
				est := workout.EpleyOneRM(s.Weight, s.Reps)
				if !haveBest || est > best.Est1RM {
					best = TopSet{Exercise: we.ExerciseID, Weight: s.Weight, Reps: s.Reps, Est1RM: roundTo(est, 0)}
					haveBest = true
				}
			}
			if haveBest {
				bestByExercise = append(bestByExercise, best)
			}
			for _, g := range muscleByExercise[we.ExerciseID] {
				if mgTotals[g] == nil {
					mgTotals[g] = &mg{}
				}
				mgTotals[g].sets += setCount
				mgTotals[g].volume += exVol
			}
		}
		sort.SliceStable(bestByExercise, func(i, j int) bool {
			return bestByExercise[i].Est1RM > bestByExercise[j].Est1RM
		})
		if len(bestByExercise) > maxTopSetsPerSession {
			bestByExercise = bestByExercise[:maxTopSetsPerSession]
		}
		session.TopSets = append(session.TopSets, bestByExercise...)
		for _, ev := range prsByWorkout[w.ID] {
			session.PRs = append(session.PRs, SessionPR{Exercise: ev.ExerciseID, Kind: "weight"})
			name := nameByExercise[ev.ExerciseID]
			if name == "" {
				name = ev.ExerciseID
			}
			sec.HeadlinePRs = append(sec.HeadlinePRs,
				fmt.Sprintf("%s %s %s PR", trimFloat(ev.Weight), unit, name))
		}
		sec.Sessions = append(sec.Sessions, session)
	}

	for g, t := range mgTotals {
		sec.ByMuscleGroup = append(sec.ByMuscleGroup, MuscleGroupVolume{
			MuscleGroup: string(g), Sets: t.sets, Volume: t.volume,
		})
	}
	sort.SliceStable(sec.ByMuscleGroup, func(i, j int) bool {
		return sec.ByMuscleGroup[i].Volume > sec.ByMuscleGroup[j].Volume
	})
	return sec
}

func aggregateRunning(
	activities []activity.Activity,
	bests []activity.RunningBestEffort,
	start, end time.Time,
	loc *time.Location,
) *RunningSection {
	sec := &RunningSection{Runs: []RunSummary{}, NewBestEfforts: []BestEffort{}}
	var totalDist float64
	var totalDur int
	for _, a := range activities {
		if a.ActivityType != activity.ActivityRunning {
			continue
		}
		sec.RunCount++
		totalDist += a.DistanceMeters
		totalDur += a.DurationSeconds
		name := ""
		if a.Name != nil {
			name = *a.Name
		}
		sec.Runs = append(sec.Runs, RunSummary{
			Date: a.StartTime.In(loc).Format("2006-01-02"), Name: name,
			DistanceM: a.DistanceMeters, DurationS: a.DurationSeconds,
			AvgPaceSecPerKm: a.AvgPaceSecPerKm,
		})
	}
	sec.TotalDistanceM = totalDist
	sec.TotalDurationS = totalDur
	if totalDist > 0 {
		pace := float64(totalDur) / (totalDist / 1000.0)
		sec.AvgPaceSecPerKm = &pace
	}
	for _, b := range bests {
		if !b.ActivityStartTime.Before(start) && b.ActivityStartTime.Before(end) {
			sec.NewBestEfforts = append(sec.NewBestEfforts, BestEffort{
				DistanceKey: b.DistanceKey, TimeSeconds: int(math.Round(b.DurationSeconds)),
			})
		}
	}
	return sec
}

func aggregateSteps(entries []steps.Entry, goal int, loc *time.Location) *StepsSection {
	sec := &StepsSection{Goal: goal, ByDay: []StepsDay{}}
	var sum int
	for _, e := range entries {
		sec.ByDay = append(sec.ByDay, StepsDay{Date: e.Date, Steps: e.Steps})
		if e.Steps > 0 {
			sec.DaysLogged++
			sum += e.Steps
		}
		sec.Total += e.Steps
	}
	sort.SliceStable(sec.ByDay, func(i, j int) bool { return sec.ByDay[i].Date < sec.ByDay[j].Date })
	if sec.DaysLogged > 0 {
		sec.Avg = int(math.Round(float64(sum) / float64(sec.DaysLogged)))
	}
	return sec
}

func aggregateBodyweight(entries []bodyweightReading) *BodyweightSection {
	sec := &BodyweightSection{Readings: []BodyweightReading{}}
	if len(entries) == 0 {
		return sec
	}
	sorted := make([]bodyweightReading, len(entries))
	copy(sorted, entries)
	sort.SliceStable(sorted, func(i, j int) bool { return sorted[i].measuredAt.Before(sorted[j].measuredAt) })
	sec.Unit = sorted[0].unit
	sec.Start = sorted[0].weight
	sec.End = sorted[len(sorted)-1].weight
	sec.Delta = roundTo(sec.End-sec.Start, 1)
	for _, r := range sorted {
		sec.Readings = append(sec.Readings, BodyweightReading{
			Date: r.measuredAt.Format("2006-01-02"), Weight: r.weight,
		})
	}
	return sec
}

func aggregateNutrition(days []nutrition.DailyMacros, goals nutrition.MacroGoals) *NutritionSection {
	sec := &NutritionSection{ByDay: []NutritionDay{},
		Goals: MacroSet{Calories: goals.Calories, ProteinG: goals.ProteinG, FatG: goals.FatG, CarbsG: goals.CarbsG}}
	sec.DaysLogged = len(days)
	var c, p, f, cb float64
	for _, d := range days {
		c += d.Calories
		p += d.ProteinG
		f += d.FatG
		cb += d.CarbsG
		sec.ByDay = append(sec.ByDay, NutritionDay{Date: d.Date,
			Calories: int(math.Round(d.Calories)), ProteinG: int(math.Round(d.ProteinG)),
			FatG: int(math.Round(d.FatG)), CarbsG: int(math.Round(d.CarbsG))})
	}
	sort.SliceStable(sec.ByDay, func(i, j int) bool { return sec.ByDay[i].Date < sec.ByDay[j].Date })
	if sec.DaysLogged > 0 {
		n := float64(sec.DaysLogged)
		sec.Avg = MacroSet{Calories: int(math.Round(c / n)), ProteinG: int(math.Round(p / n)),
			FatG: int(math.Round(f / n)), CarbsG: int(math.Round(cb / n))}
	}
	return sec
}

func countActiveDays(strength *StrengthSection, running *RunningSection, stepsSec *StepsSection) int {
	active := map[string]bool{}
	if strength != nil {
		for _, s := range strength.Sessions {
			active[s.Date] = true
		}
	}
	if running != nil {
		for _, r := range running.Runs {
			active[r.Date] = true
		}
	}
	if stepsSec != nil {
		for _, d := range stepsSec.ByDay {
			if d.Steps > 0 {
				active[d.Date] = true
			}
		}
	}
	return len(active)
}

func roundTo(v float64, places int) float64 {
	p := math.Pow(10, float64(places))
	return math.Round(v*p) / p
}

func trimFloat(v float64) string {
	return fmt.Sprintf("%g", roundTo(v, 1))
}
```

Note: `bodyweightReading` is a small internal projection (`measuredAt time.Time`, `weight float64`, `unit string`) defined in `service.go` (Task A2) so `aggregate.go` stays free of the bodyweight repo import; the service maps `bodyweight.Entry` → `bodyweightReading`. Define it in service.go, but if writing aggregate.go first causes an undefined type, add the struct to aggregate.go instead — keep it in ONE file.

- [ ] **Step 5: Run tests, verify they PASS** — Run: `GOTOOLCHAIN=auto go test ./internal/snapshot/...`

- [ ] **Step 6: Lint + commit**

```bash
GOTOOLCHAIN=auto go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run ./internal/snapshot/...
git add internal/snapshot/dto.go internal/snapshot/aggregate.go internal/snapshot/aggregate_test.go
git commit -m "feat(snapshot): pure aggregation helpers and response DTOs"
```

---

### Task A2: Snapshot Service with defensive degradation

**Files:**
- Create: `internal/snapshot/service.go`
- Test: `internal/snapshot/service_test.go`

- [ ] **Step 1: Write the failing service tests** in `service_test.go`

Define tiny fakes implementing the narrow interfaces (below). Cover:
1. **Fan-out happy path** — all repos return data; assert every section non-nil with expected rollups, `period` (start/end/days/timezone), and `consistency.active_days`/`window_days`.
2. **Defensive degradation** — make ONLY the workout repo return an error; assert `snap.Strength == nil` but every other section non-nil. Repeat conceptually for at least one more domain (e.g. nutrition error → `snap.Nutrition == nil`, others non-nil).
3. **Empty-but-healthy** — all repos return empty slices with no error; assert sections are **non-nil** with zero counts / empty arrays (NOT nil).

```go
package snapshot

import (
	"context"
	"errors"
	"testing"
	"time"

	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/activity"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/bodyweight"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/exercise"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/nutrition"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/steps"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/user"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/workout"
)

type fakeWorkout struct {
	workouts []workout.Workout
	prs      []workout.PersonalRecordEvent
	err      error
}

func (f fakeWorkout) ListByUser(_ context.Context, _ string, _ workout.ListOptions) ([]workout.Workout, error) {
	return f.workouts, f.err
}
func (f fakeWorkout) ListPersonalRecordEventsByWorkouts(_ context.Context, _ []string) ([]workout.PersonalRecordEvent, error) {
	return f.prs, f.err
}

// ... analogous fakeExercise, fakeActivity, fakeSteps, fakeBodyweight,
// fakeNutrition, fakeUser implementing the narrow interfaces below ...

func TestBuild_DefensiveDegradation_WorkoutError(t *testing.T) {
	loc, _ := time.LoadLocation("America/Denver")
	start := time.Date(2026, 6, 15, 6, 0, 0, 0, time.UTC)
	end := time.Date(2026, 6, 22, 6, 0, 0, 0, time.UTC)
	svc := NewService(
		fakeWorkout{err: errors.New("boom")},
		fakeExercise{}, fakeActivity{}, fakeSteps{}, fakeBodyweight{}, fakeNutrition{}, fakeUser{},
	)
	snap := svc.Build(context.Background(), "u1", start, end, loc)
	if snap.Strength != nil {
		t.Fatalf("strength should be nil on workout repo error")
	}
	if snap.Running == nil || snap.Steps == nil || snap.Bodyweight == nil || snap.Nutrition == nil {
		t.Fatalf("other sections should survive one domain's failure")
	}
}

func TestBuild_EmptyButHealthy_RendersNonNil(t *testing.T) {
	loc, _ := time.LoadLocation("America/Denver")
	start := time.Date(2026, 6, 15, 6, 0, 0, 0, time.UTC)
	end := time.Date(2026, 6, 22, 6, 0, 0, 0, time.UTC)
	svc := NewService(fakeWorkout{}, fakeExercise{}, fakeActivity{}, fakeSteps{}, fakeBodyweight{}, fakeNutrition{}, fakeUser{u: &user.User{WeightUnit: user.WeightUnitPounds}})
	snap := svc.Build(context.Background(), "u1", start, end, loc)
	if snap.Strength == nil || snap.Running == nil || snap.Steps == nil || snap.Bodyweight == nil || snap.Nutrition == nil {
		t.Fatalf("empty-but-healthy domains must render non-nil")
	}
	if snap.Period.Days != 7 || snap.Consistency.WindowDays != 7 {
		t.Fatalf("days = %d", snap.Period.Days)
	}
}
```

(Add the happy-path test asserting concrete rollups, and the nutrition-error variant. Implement all fakes.)

- [ ] **Step 2: Run, verify FAIL** — `GOTOOLCHAIN=auto go test ./internal/snapshot/...` → undefined `NewService`.

- [ ] **Step 3: Write `service.go`**

```go
package snapshot

import (
	"context"
	"log"
	"time"

	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/activity"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/bodyweight"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/exercise"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/nutrition"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/requestid"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/steps"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/user"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/workout"
)

// Narrow consumer interfaces: only the methods the snapshot needs, so
// tests can supply tiny fakes. The concrete domain repositories satisfy
// these. (Idiomatic Go: accept the interface you use.)
type workoutReader interface {
	ListByUser(ctx context.Context, userID string, opts workout.ListOptions) ([]workout.Workout, error)
	ListPersonalRecordEventsByWorkouts(ctx context.Context, workoutIDs []string) ([]workout.PersonalRecordEvent, error)
}
type exerciseReader interface {
	List(ctx context.Context, opts exercise.ListOptions) ([]exercise.Exercise, error)
}
type activityReader interface {
	ListInRange(ctx context.Context, userID string, since, until *time.Time) ([]activity.Activity, error)
	GetUserRunningBestEfforts(ctx context.Context, userID string) ([]activity.RunningBestEffort, error)
}
type stepsReader interface {
	List(ctx context.Context, userID string, since, until *string, limit int, before *string) ([]steps.Entry, string, error)
	GetGoal(ctx context.Context, userID string) (steps.Goal, error)
}
type bodyweightReader interface {
	List(ctx context.Context, userID string, since, until *time.Time) ([]bodyweight.Entry, error)
}
type nutritionReader interface {
	DailyMacros(ctx context.Context, userID string, since, until time.Time, loc *time.Location) ([]nutrition.DailyMacros, error)
	GetMacroGoals(ctx context.Context, userID string) (nutrition.MacroGoals, error)
}
type userReader interface {
	GetByID(ctx context.Context, id string) (*user.User, error)
}

// bodyweightReading is the snapshot's internal projection of a bodyweight
// entry, keeping aggregate.go free of the bodyweight repo import.
type bodyweightReading struct {
	measuredAt time.Time
	weight     float64
	unit       string
}

type Service struct {
	workoutRepo    workoutReader
	exerciseRepo   exerciseReader
	activityRepo   activityReader
	stepsRepo      stepsReader
	bodyweightRepo bodyweightReader
	nutritionRepo  nutritionReader
	userRepo       userReader
}

func NewService(w workoutReader, e exerciseReader, a activityReader, s stepsReader,
	b bodyweightReader, n nutritionReader, u userReader) *Service {
	return &Service{w, e, a, s, b, n, u}
}

// Build composes the snapshot over [start, end) (UTC half-open) localized
// by loc. Each domain section is read defensively: a repo error nulls that
// section only (logged with the request id), never the whole response.
func (s *Service) Build(ctx context.Context, userID string, start, end time.Time, loc *time.Location) Snapshot {
	days := countLocalDays(start, end, loc)
	snap := Snapshot{
		Period: Period{
			StartDate: start.In(loc).Format("2006-01-02"),
			EndDate:   end.In(loc).AddDate(0, 0, -1).Format("2006-01-02"),
			Timezone:  loc.String(),
			Days:      days,
		},
		Consistency: Consistency{WindowDays: days},
	}

	unit := "lb"
	if u, err := s.userRepo.GetByID(ctx, userID); err == nil && u != nil {
		unit = string(u.WeightUnit)
	}

	snap.Strength = sectionOrNil(ctx, "strength", func() (*StrengthSection, error) {
		ws, err := s.workoutRepo.ListByUser(ctx, userID, workout.ListOptions{Since: &start, Until: &end})
		if err != nil {
			return nil, err
		}
		ids := make([]string, len(ws))
		for i, w := range ws {
			ids[i] = w.ID
		}
		prs, err := s.workoutRepo.ListPersonalRecordEventsByWorkouts(ctx, ids)
		if err != nil {
			return nil, err
		}
		exs, err := s.exerciseRepo.List(ctx, exercise.ListOptions{})
		if err != nil {
			return nil, err
		}
		return aggregateStrength(ws, prs, exs, unit, loc), nil
	})

	snap.Running = sectionOrNil(ctx, "running", func() (*RunningSection, error) {
		acts, err := s.activityRepo.ListInRange(ctx, userID, &start, &end)
		if err != nil {
			return nil, err
		}
		bests, err := s.activityRepo.GetUserRunningBestEfforts(ctx, userID)
		if err != nil {
			return nil, err
		}
		return aggregateRunning(acts, bests, start, end, loc), nil
	})

	snap.Steps = sectionOrNil(ctx, "steps", func() (*StepsSection, error) {
		since := start.In(loc).Format("2006-01-02")
		until := end.In(loc).AddDate(0, 0, -1).Format("2006-01-02")
		entries, _, err := s.stepsRepo.List(ctx, userID, &since, &until, 0, nil)
		if err != nil {
			return nil, err
		}
		goal, err := s.stepsRepo.GetGoal(ctx, userID)
		if err != nil {
			return nil, err
		}
		return aggregateSteps(entries, goal.Goal, loc), nil
	})

	snap.Bodyweight = sectionOrNil(ctx, "bodyweight", func() (*BodyweightSection, error) {
		entries, err := s.bodyweightRepo.List(ctx, userID, &start, &end)
		if err != nil {
			return nil, err
		}
		readings := make([]bodyweightReading, len(entries))
		for i, e := range entries {
			readings[i] = bodyweightReading{measuredAt: e.MeasuredAt.In(loc), weight: e.Weight, unit: string(e.Unit)}
		}
		return aggregateBodyweight(readings), nil
	})

	snap.Nutrition = sectionOrNil(ctx, "nutrition", func() (*NutritionSection, error) {
		days, err := s.nutritionRepo.DailyMacros(ctx, userID, start, end, loc)
		if err != nil {
			return nil, err
		}
		goals, err := s.nutritionRepo.GetMacroGoals(ctx, userID)
		if err != nil {
			return nil, err
		}
		return aggregateNutrition(days, goals), nil
	})

	snap.Consistency.ActiveDays = countActiveDays(snap.Strength, snap.Running, snap.Steps)
	return snap
}

// sectionOrNil runs fn and, on error, logs (tagged with op + request id)
// and returns nil — the section degrades to JSON null. Mirrors the
// dashboard handler's defer1 defensive wrapper.
func sectionOrNil[T any](ctx context.Context, op string, fn func() (*T, error)) *T {
	v, err := fn()
	if err != nil {
		log.Printf("snapshot: %s for %s: %v", op, requestid.FromContext(ctx), err)
		return nil
	}
	return v
}

// countLocalDays counts whole local calendar days in [start, end). DST-safe
// (steps one local day at a time rather than dividing by 24h).
func countLocalDays(start, end time.Time, loc *time.Location) int {
	d := start.In(loc)
	cur := time.Date(d.Year(), d.Month(), d.Day(), 0, 0, 0, 0, loc)
	n := 0
	for cur.Before(end) {
		n++
		cur = cur.AddDate(0, 0, 1)
	}
	return n
}
```

- [ ] **Step 4: Run, verify PASS** — `GOTOOLCHAIN=auto go test ./internal/snapshot/...`

- [ ] **Step 5: Lint + commit**

```bash
GOTOOLCHAIN=auto go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run ./internal/snapshot/...
git add internal/snapshot/service.go internal/snapshot/service_test.go
git commit -m "feat(snapshot): defensive multi-repo composition service"
```

---

### Task A3: Snapshot Handler, route, and server wiring

**Files:**
- Create: `internal/snapshot/handler.go`
- Test: `internal/snapshot/handler_test.go`
- Modify: `internal/server/server.go`

- [ ] **Step 1: Write the failing handler tests** in `handler_test.go`

Cover: (a) default window (no date params) → 200, `period.days == 7`, `period.timezone` echoed; (b) explicit `start_date`/`end_date` range honored in `period`; (c) single `date` honored; (d) missing `timezone` → 400; (e) invalid timezone → 400; (f) missing auth context → 500/server error. Use `httptest`, inject auth via the same context key the middleware uses, and pin `now` to a fixed time. Inspect how `internal/dashboard/handler_test.go` seeds the authed user into the request context and pins `now`; mirror it exactly.

```go
package snapshot

import (
	"net/http"
	"net/http/httptest"
	"testing"
	"time"
	// import auth context + httpresp envelope as dashboard's test does
)

func newTestHandler() *Handler {
	svc := NewService(fakeWorkout{}, fakeExercise{}, fakeActivity{}, fakeSteps{},
		fakeBodyweight{}, fakeNutrition{}, fakeUser{u: &user.User{WeightUnit: user.WeightUnitPounds}})
	h := NewHandler(svc)
	h.now = func() time.Time { return time.Date(2026, 6, 21, 12, 0, 0, 0, time.UTC) }
	return h
}

func TestSnapshot_DefaultWindowIsTrailing7(t *testing.T) {
	h := newTestHandler()
	req := httptest.NewRequest(http.MethodGet, "/training-snapshot?timezone=America/Denver", nil)
	req = req.WithContext(withUser(req.Context(), "u1")) // mirror dashboard test helper
	rec := httptest.NewRecorder()
	h.snapshot(rec, req)
	if rec.Code != http.StatusOK {
		t.Fatalf("status = %d body=%s", rec.Code, rec.Body.String())
	}
	// decode envelope.data.period.days == 7
}

func TestSnapshot_MissingTimezone400(t *testing.T) {
	h := newTestHandler()
	req := httptest.NewRequest(http.MethodGet, "/training-snapshot", nil)
	req = req.WithContext(withUser(req.Context(), "u1"))
	rec := httptest.NewRecorder()
	h.snapshot(rec, req)
	if rec.Code != http.StatusBadRequest {
		t.Fatalf("status = %d", rec.Code)
	}
}
```

(Implement `withUser` using the real auth context seam — check `internal/auth` / `internal/dashboard/handler_test.go` for the exact helper, e.g. `authctx.WithUserID`. Add the range/date/invalid-tz/no-auth tests.)

- [ ] **Step 2: Run, verify FAIL** — `GOTOOLCHAIN=auto go test ./internal/snapshot/...` → undefined `NewHandler`.

- [ ] **Step 3: Write `handler.go`**

```go
package snapshot

import (
	"errors"
	"net/http"
	"time"

	"github.com/go-chi/chi/v5"

	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/auth"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/daterange"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/httpresp"
)

type Handler struct {
	svc *Service
	now func() time.Time
}

func NewHandler(svc *Service) *Handler {
	return &Handler{svc: svc, now: time.Now}
}

func (h *Handler) Mount(r chi.Router) {
	r.Route("/training-snapshot", func(r chi.Router) {
		r.Get("/", h.snapshot)
	})
}

func (h *Handler) snapshot(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	userID, ok := auth.UserIDFrom(ctx)
	if !ok {
		httpresp.ServerError(w, ctx, "missing user in context", errors.New("auth middleware not applied"))
		return
	}

	q := r.URL.Query()
	var start, end time.Time
	var loc *time.Location
	var err error

	if q.Get("date") == "" && q.Get("start_date") == "" && q.Get("end_date") == "" {
		// Default window: trailing 7 local days ending today.
		tz := q.Get("timezone")
		if tz == "" {
			httpresp.Error(w, http.StatusBadRequest, "timezone is required")
			return
		}
		loc, err = daterange.LoadTimezone(tz)
		if err != nil {
			httpresp.Error(w, http.StatusBadRequest, "invalid timezone "+tz)
			return
		}
		now := h.now().In(loc)
		endLocal := time.Date(now.Year(), now.Month(), now.Day(), 0, 0, 0, 0, loc).AddDate(0, 0, 1)
		startLocal := endLocal.AddDate(0, 0, -7)
		start, end = startLocal.UTC(), endLocal.UTC()
	} else {
		start, end, loc, err = daterange.ParseQuery(q)
		if err != nil {
			httpresp.Error(w, http.StatusBadRequest, err.Error())
			return
		}
	}

	snap := h.svc.Build(ctx, userID, start, end, loc)
	httpresp.OK(w, "training snapshot", snap)
}
```

- [ ] **Step 4: Run, verify PASS** — `GOTOOLCHAIN=auto go test ./internal/snapshot/...`

- [ ] **Step 5: Wire into `server.go`** — locate the `dashboard.NewHandler(...).Mount(r)` line inside the `r.Group(func(r chi.Router){ r.Use(auth.RequireUser(jwtSecret)) ... })` block (around line 470). Immediately after it, add:

```go
		// Training snapshot — the agent-facing holistic read across every
		// domain (GET /training-snapshot). Separate surface from the web
		// dashboard; composes the same domain repos defensively.
		snapshot.NewHandler(snapshot.NewService(
			workoutRepo, exerciseRepo, activityRepo, stepsRepo, bodyweightRepo, nutritionRepo, userRepo,
		)).Mount(r)
```

Add the import `"github.com/jwallace145/progressive-overload-fitness-tracker/internal/snapshot"`. Confirm the variable names (`workoutRepo`, `exerciseRepo`, `activityRepo`, `stepsRepo`, `bodyweightRepo`, `nutritionRepo`, `userRepo`) match those in scope where dashboard mounts — they are the same repos dashboard receives.

- [ ] **Step 6: Full gate + commit**

```bash
GOTOOLCHAIN=auto go build ./...
GOTOOLCHAIN=auto go vet ./...
GOTOOLCHAIN=auto go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run
GOTOOLCHAIN=auto go mod tidy && git diff --exit-code go.mod go.sum
GOTOOLCHAIN=auto go test ./...
git add internal/snapshot/handler.go internal/snapshot/handler_test.go internal/server/server.go
git commit -m "feat(snapshot): GET /training-snapshot endpoint and route"
```

---

# Repo B — prog-strength-mcp

Work from: `/workspace/prog-strength-mcp` on branch `feat/analyze-training-snapshot`. Run everything via `uv` (PATH includes `~/.local/bin`).

**File structure:**
- Modify `src/prog_strength_mcp/api_client.py` — add `get_training_snapshot`.
- Create `src/prog_strength_mcp/training_snapshot.py` — the forwarder tool module.
- Modify `src/prog_strength_mcp/server.py` — register the new module.
- Create `tests/test_training_snapshot_tools.py` — respx-mocked, modeled on `tests/test_steps_tools.py`.

> First: `cd /workspace/prog-strength-mcp && git checkout -b feat/analyze-training-snapshot`.

---

### Task B1: APIClient.get_training_snapshot

**Files:**
- Modify: `src/prog_strength_mcp/api_client.py`
- Test: `tests/test_training_snapshot_tools.py`

- [ ] **Step 1: Write the failing client tests** (start the new test file)

```python
import httpx
import pytest
import respx

from prog_strength_mcp import training_snapshot
from prog_strength_mcp.api_client import APIClient, APIError

BASE_URL = "http://api.test"
AUTH = "Bearer test-token"

_SAMPLE = {
    "period": {"start_date": "2026-06-15", "end_date": "2026-06-21",
               "timezone": "America/Denver", "days": 7},
    "strength": {"session_count": 3, "total_volume": 48250, "unit": "lb",
                 "by_muscle_group": [], "sessions": [], "headline_prs": []},
    "running": None,
    "steps": {"days_logged": 6, "avg": 9120, "total": 54720, "goal": 10000, "by_day": []},
    "bodyweight": None,
    "nutrition": None,
    "consistency": {"active_days": 5, "window_days": 7},
}


@respx.mock
async def test_get_training_snapshot_forwards_params_and_unwraps():
    route = respx.get(f"{BASE_URL}/training-snapshot").mock(
        return_value=httpx.Response(200, json={"data": _SAMPLE})
    )
    async with APIClient(base_url=BASE_URL) as api:
        result = await api.get_training_snapshot(
            AUTH, timezone="America/Denver", start_date="2026-06-15", end_date="2026-06-21"
        )
    assert result == _SAMPLE
    req = route.calls.last.request
    assert req.headers["Authorization"] == AUTH
    assert req.url.params["timezone"] == "America/Denver"
    assert req.url.params["start_date"] == "2026-06-15"
    assert req.url.params["end_date"] == "2026-06-21"


@respx.mock
async def test_get_training_snapshot_non_dict_data_yields_empty():
    respx.get(f"{BASE_URL}/training-snapshot").mock(
        return_value=httpx.Response(200, json={"data": None})
    )
    async with APIClient(base_url=BASE_URL) as api:
        result = await api.get_training_snapshot(AUTH, timezone="UTC")
    assert result == {}


@respx.mock
async def test_get_training_snapshot_surfaces_api_error():
    respx.get(f"{BASE_URL}/training-snapshot").mock(
        return_value=httpx.Response(500, json={"error": "db exploded"})
    )
    async with APIClient(base_url=BASE_URL) as api:
        with pytest.raises(APIError) as excinfo:
            await api.get_training_snapshot(AUTH, timezone="UTC")
    assert excinfo.value.status_code == 500
```

- [ ] **Step 2: Run, verify FAIL** — `uv run pytest tests/test_training_snapshot_tools.py -q` → AttributeError / ImportError.

- [ ] **Step 3: Add `get_training_snapshot` to `APIClient`** (mirror `get_steps`)

```python
    async def get_training_snapshot(
        self,
        auth_header: str,
        *,
        timezone: str,
        date: str | None = None,
        start_date: str | None = None,
        end_date: str | None = None,
    ) -> dict[str, Any]:
        """GET /training-snapshot. `timezone` is a required IANA name;
        supply either `date` or `start_date`/`end_date` (YYYY-MM-DD), or
        omit dates for the trailing 7 local days. Returns the snapshot
        object under `data`.
        """
        params: dict[str, str] = {"timezone": timezone}
        if date:
            params["date"] = date
        if start_date:
            params["start_date"] = start_date
        if end_date:
            params["end_date"] = end_date
        resp = await self._client.get(
            "/training-snapshot",
            params=params,
            headers={"Authorization": auth_header},
        )
        _raise_for_status(resp)
        data = resp.json().get("data")
        return data if isinstance(data, dict) else {}
```

- [ ] **Step 4: Run, verify PASS** — `uv run pytest tests/test_training_snapshot_tools.py -q` (the 3 client tests pass; tool-boundary tests come in B2).

- [ ] **Step 5: Commit**

```bash
git add src/prog_strength_mcp/api_client.py tests/test_training_snapshot_tools.py
git commit -m "feat(snapshot): APIClient.get_training_snapshot forwarder"
```

---

### Task B2: training_snapshot tool module + registration

**Files:**
- Create: `src/prog_strength_mcp/training_snapshot.py`
- Modify: `src/prog_strength_mcp/server.py`
- Test: `tests/test_training_snapshot_tools.py` (append tool-boundary tests)

- [ ] **Step 1: Append the failing tool-boundary tests** (mirror `test_steps_tools.py`'s `_ExplodingAPI` / `_FailingAPI` patterns)

```python
async def test_tool_requires_auth(monkeypatch):
    from fastmcp import FastMCP

    monkeypatch.setattr(training_snapshot, "get_http_headers", lambda **_: {})

    class _ExplodingAPI:
        async def get_training_snapshot(self, *a, **k):  # pragma: no cover
            raise AssertionError("HTTP forwarding must not happen on missing auth")

    mcp = FastMCP("test")
    training_snapshot.register(mcp, _ExplodingAPI())
    tool = await mcp.get_tool("get_training_snapshot")
    with pytest.raises(RuntimeError, match="Authorization"):
        await tool.fn(timezone="UTC")


async def test_tool_maps_api_error(monkeypatch):
    from fastmcp import FastMCP

    monkeypatch.setattr(training_snapshot, "get_http_headers", lambda **_: {"authorization": AUTH})

    class _FailingAPI:
        async def get_training_snapshot(self, *a, **k):
            raise APIError(500, "db exploded")

    mcp = FastMCP("test")
    training_snapshot.register(mcp, _FailingAPI())
    tool = await mcp.get_tool("get_training_snapshot")
    with pytest.raises(RuntimeError, match="500"):
        await tool.fn(timezone="UTC")


async def test_tool_forwards_all_params(monkeypatch):
    from fastmcp import FastMCP

    monkeypatch.setattr(training_snapshot, "get_http_headers", lambda **_: {"authorization": AUTH})
    captured = {}

    class _CaptureAPI:
        async def get_training_snapshot(self, auth, *, timezone, date=None, start_date=None, end_date=None):
            captured.update(auth=auth, timezone=timezone, date=date, start_date=start_date, end_date=end_date)
            return _SAMPLE

    mcp = FastMCP("test")
    training_snapshot.register(mcp, _CaptureAPI())
    tool = await mcp.get_tool("get_training_snapshot")
    result = await tool.fn(timezone="America/Denver", start_date="2026-06-15", end_date="2026-06-21")
    assert result == _SAMPLE
    assert captured["auth"] == AUTH
    assert captured["timezone"] == "America/Denver"
    assert captured["start_date"] == "2026-06-15"
```

- [ ] **Step 2: Run, verify FAIL** — `uv run pytest tests/test_training_snapshot_tools.py -q` → ImportError (`training_snapshot` has no `register`).

- [ ] **Step 3: Write `training_snapshot.py`** (mirror `steps.py` exactly: module docstring, `_auth_header_or_raise`, `register`)

```python
"""Training-snapshot domain: a single MCP tool that forwards to the API's
GET /training-snapshot — a holistic, pre-aggregated view across strength,
running, steps, bodyweight, and nutrition for a date window.

Pure forwarder, following the steps.py / nutrition.py pattern: it reads
the inbound Authorization header, calls APIClient.get_training_snapshot,
unwraps the `{service, message, data}` envelope, and surfaces APIError as
RuntimeError. No aggregation happens here — the API owns it.
"""

from typing import Annotated, Any

from fastmcp import FastMCP
from fastmcp.server.dependencies import get_http_headers
from pydantic import Field

from prog_strength_mcp.api_client import APIClient, APIError


def _auth_header_or_raise() -> str:
    """Pull the inbound Authorization header. Tools that require auth
    call this before forwarding to the API; missing/empty header is
    surfaced to Claude as an error rather than letting the API 401.
    """
    headers = get_http_headers(include={"authorization"})
    auth = headers.get("authorization", "")
    if not auth:
        raise RuntimeError(
            "missing Authorization header on the MCP request — the agent "
            "must open the MCP session with the user's Bearer token."
        )
    return auth


def register(mcp: FastMCP, api: APIClient) -> None:
    """Register the training-snapshot tool on `mcp`, backed by `api`."""

    @mcp.tool
    async def get_training_snapshot(
        timezone: Annotated[
            str,
            Field(description="IANA timezone name, e.g. 'America/Denver'. Required."),
        ],
        date: Annotated[
            str | None,
            Field(default=None, description="Single day as YYYY-MM-DD. Mutually exclusive with the range."),
        ] = None,
        start_date: Annotated[
            str | None,
            Field(default=None, description="Range start (YYYY-MM-DD); use with end_date."),
        ] = None,
        end_date: Annotated[
            str | None,
            Field(default=None, description="Range end (YYYY-MM-DD); use with start_date."),
        ] = None,
    ) -> dict[str, Any]:
        """Holistic training snapshot for a date window — strength, running,
        steps, bodyweight, and nutrition, pre-aggregated for analysis. Supply an
        IANA `timezone` plus either a single `date` or a `start_date`/`end_date`
        range (YYYY-MM-DD); omit dates for the trailing 7 days. Use this for any
        "how did my training go" question over a period before reaching for the
        per-domain list tools.
        """
        auth = _auth_header_or_raise()
        try:
            return await api.get_training_snapshot(
                auth, timezone=timezone, date=date, start_date=start_date, end_date=end_date
            )
        except APIError as e:
            raise RuntimeError(f"API error ({e.status_code}): {e.message}") from e
```

- [ ] **Step 4: Register in `server.py`** — add `training_snapshot` to the `from prog_strength_mcp import (...)` block (keep alphabetical) and add `training_snapshot.register(mcp, api)` alongside the other `*.register(mcp, api)` calls.

- [ ] **Step 5: Run full suite + lint + verify PASS**

```bash
uv run pytest -q
uv run ruff check src/prog_strength_mcp/training_snapshot.py tests/test_training_snapshot_tools.py src/prog_strength_mcp/api_client.py src/prog_strength_mcp/server.py
uv run ruff format src/prog_strength_mcp/training_snapshot.py tests/test_training_snapshot_tools.py
```

- [ ] **Step 6: Commit**

```bash
git add src/prog_strength_mcp/training_snapshot.py src/prog_strength_mcp/server.py tests/test_training_snapshot_tools.py
git commit -m "feat(snapshot): get_training_snapshot forwarder tool"
```

---

# Repo C — prog-strength-agent

Work from: `/workspace/prog-strength-agent` on branch `feat/analyze-training-snapshot`.

**File structure:**
- Modify `src/prog_strength_agent/intents.py` — thread timezone into prefetch; rename intent; new prefetch/format/rules.
- Modify `src/prog_strength_agent/model_harness.py` — pass `client_timezone` into `IntentRegistry.run`; add `get_training_snapshot` to the tz-injected tool set.
- Modify `src/prog_strength_agent/model_router.py` — rename intent in prompt; deterministic always-complex override.
- Modify `tests/test_intents.py` and `tests/test_model_router.py` — update to the renamed intent + new behavior.

> First: `cd /workspace/prog-strength-agent && git checkout -b feat/analyze-training-snapshot`. Run via `uv`.

---

### Task C1: Rename intent, thread timezone, snapshot prefetch + format + rules

**Files:**
- Modify: `src/prog_strength_agent/intents.py`
- Modify: `src/prog_strength_agent/model_harness.py`
- Test: `tests/test_intents.py`

Background: `IntentRegistry.run(intent, session)` calls each spec's `prefetch(session)`. The old `_analyze_progress_prefetch` used `datetime.now(UTC)` (no timezone). The new intent needs the user's local week, so we thread `client_timezone` from the harness into the prefetch. `PrefetchFn` becomes `Callable[[Any, str | None], Awaitable[dict]]`; ALL existing prefetch functions gain a second parameter (most ignore it). `build_intent_aware_prompt` already receives the data it needs; we extend it + its caller to pass `client_timezone`.

- [ ] **Step 1: Update the failing tests** in `tests/test_intents.py`

- Change the `test_known_intents_enum` set: replace `"analyze_progress"` with `"analyze_training"`.
- Replace `test_analyze_progress_prefetch_*` with the test below.

```python
@pytest.mark.asyncio
async def test_analyze_training_prefetch_calls_snapshot_for_current_week():
    from typing import Any

    captured: dict[str, Any] = {}

    class _CaptureSession:
        async def call_tool(self, name: str, args: dict):
            captured[name] = args
            return _FakeMCPResult(
                '{"period":{"start_date":"2026-06-15","end_date":"2026-06-21",'
                '"timezone":"America/Denver","days":7},'
                '"strength":{"session_count":3,"total_volume":48250,"unit":"lb",'
                '"by_muscle_group":[],"sessions":[],"headline_prs":["335 lb squat PR"]},'
                '"running":null,"steps":{"days_logged":6,"avg":9120,"total":54720,"goal":10000,"by_day":[]},'
                '"bodyweight":null,"nutrition":null,"consistency":{"active_days":5,"window_days":7}}'
            )

    rules, data, failed = await IntentRegistry.run(
        "analyze_training", _CaptureSession(), "America/Denver"
    )
    assert failed is False
    assert "get_training_snapshot" in captured
    assert captured["get_training_snapshot"]["timezone"] == "America/Denver"
    # current local week = 7-day window ending today
    assert captured["get_training_snapshot"]["start_date"] is not None
    assert captured["get_training_snapshot"]["end_date"] is not None
    # format renders compact specifics from the snapshot
    assert "48250" in data or "335 lb squat PR" in data
    assert "9120" in data  # steps avg present


@pytest.mark.asyncio
async def test_analyze_training_prefetch_fails_soft_without_timezone_ok():
    # When timezone is None, the prefetch still produces a snapshot call
    # using a UTC fallback rather than raising.
    captured = {}

    class _S:
        async def call_tool(self, name, args):
            captured[name] = args
            return _FakeMCPResult("{}")

    rules, data, failed = await IntentRegistry.run("analyze_training", _S(), None)
    assert "get_training_snapshot" in captured
    assert captured["get_training_snapshot"]["timezone"] == "UTC"
```

(If `IntentRegistry.run` is called elsewhere in the test file with two args, update those call sites to pass a timezone, e.g. `"UTC"`.)

- [ ] **Step 2: Run, verify FAIL** — `uv run pytest tests/test_intents.py -q`.

- [ ] **Step 3: Edit `intents.py`**

1. In `KNOWN_INTENTS`, rename `"analyze_progress"` → `"analyze_training"`.
2. Change the type alias: `PrefetchFn = Callable[[Any, str | None], Awaitable[dict[str, Any]]]`.
3. Add `from datetime import datetime, timedelta` and `from zoneinfo import ZoneInfo` (keep existing `UTC` import; remove it only if now unused — check).
4. Give EVERY existing prefetch function a second parameter. Rename `session` usage stays; add `timezone: str | None` (ignored where unused). E.g. `async def _log_nutrition_prefetch(session: Any, timezone: str | None) -> dict[str, Any]:`. Do this for all of them so the registry can call uniformly.
5. Replace `_analyze_progress_prefetch`, `_analyze_progress_format`, `_ANALYZE_PROGRESS_RULES`, and the registration block:

```python
async def _analyze_training_prefetch(session: Any, timezone: str | None) -> dict[str, Any]:
    tz_name = timezone or "UTC"
    try:
        tz = ZoneInfo(tz_name)
    except Exception:  # noqa: BLE001 — unknown tz falls back to UTC
        tz_name, tz = "UTC", ZoneInfo("UTC")
    today = datetime.now(tz).date()
    start = today - timedelta(days=6)
    res = await session.call_tool(
        "get_training_snapshot",
        {"timezone": tz_name, "start_date": start.isoformat(), "end_date": today.isoformat()},
    )
    snap = _decode_tool_result(res)
    return {"snapshot": snap if isinstance(snap, dict) else {}}


def _analyze_training_format(data: dict[str, Any]) -> str:
    snap = data.get("snapshot") or {}
    if not isinstance(snap, dict) or not snap:
        return "TRAINING SNAPSHOT: (unavailable)"
    lines: list[str] = ["TRAINING SNAPSHOT (pre-aggregated for the period):"]
    period = snap.get("period") or {}
    lines.append(
        f"- period: {period.get('start_date', '?')} → {period.get('end_date', '?')} "
        f"({period.get('days', '?')} days, {period.get('timezone', '?')})"
    )
    cons = snap.get("consistency") or {}
    lines.append(f"- active days: {cons.get('active_days', '?')}/{cons.get('window_days', '?')}")

    strength = snap.get("strength")
    if strength is None:
        lines.append("- strength: unavailable")
    else:
        lines.append(
            f"- strength: {strength.get('session_count', 0)} session(s), "
            f"volume {strength.get('total_volume', 0)} {strength.get('unit', '')}"
        )
        for pr in strength.get("headline_prs") or []:
            lines.append(f"  - PR: {pr}")

    running = snap.get("running")
    if running is None:
        lines.append("- running: unavailable")
    else:
        lines.append(
            f"- running: {running.get('run_count', 0)} run(s), "
            f"{running.get('total_distance_m', 0)} m, {running.get('total_duration_s', 0)} s"
        )

    steps = snap.get("steps")
    if steps is None:
        lines.append("- steps: unavailable")
    else:
        lines.append(
            f"- steps: avg {steps.get('avg', 0)} over {steps.get('days_logged', 0)} day(s), "
            f"goal {steps.get('goal', 0)}"
        )

    bodyweight = snap.get("bodyweight")
    if bodyweight is None:
        lines.append("- bodyweight: unavailable")
    else:
        lines.append(
            f"- bodyweight: {bodyweight.get('start', '?')} → {bodyweight.get('end', '?')} "
            f"{bodyweight.get('unit', '')} (delta {bodyweight.get('delta', '?')})"
        )

    nutrition = snap.get("nutrition")
    if nutrition is None:
        lines.append("- nutrition: unavailable")
    else:
        avg = nutrition.get("avg") or {}
        goals = nutrition.get("goals") or {}
        lines.append(
            f"- nutrition: {nutrition.get('days_logged', 0)} day(s) logged; "
            f"avg {avg.get('calories', 0)} kcal P/F/C "
            f"{avg.get('protein_g', 0)}/{avg.get('fat_g', 0)}/{avg.get('carbs_g', 0)}; "
            f"goal {goals.get('calories', 0)} kcal"
        )
    return "\n".join(lines)


_ANALYZE_TRAINING_RULES = """\
The user wants a holistic read on their training over a period. A pre-aggregated \
TRAINING SNAPSHOT across strength, running, steps, bodyweight, and nutrition is \
below. Give a read across ALL modalities present in the snapshot, not just \
lifting. Cite specifics from the snapshot (volumes, paces, averages, deltas, \
active days) rather than generalizing. Use remembered goals, constraints, and \
injuries from the Background block when relevant. If the user asked about a \
period other than the prefetched week (e.g. "last month"), call \
get_training_snapshot again with explicit start_date/end_date. Reach for the \
per-domain detail tools only when you need more than the snapshot carries. Never \
fabricate data for a domain the snapshot reports as empty or unavailable.\
"""

_register(IntentSpec(
    intent="analyze_training",
    rules=_ANALYZE_TRAINING_RULES,
    prefetch=_analyze_training_prefetch,
    format=_analyze_training_format,
))
```

6. Update `IntentRegistry.run` to accept and forward the timezone:

```python
    @classmethod
    async def run(cls, intent: str, session: Any, client_timezone: str | None = None) -> tuple[str, str, bool]:
        ...
        try:
            data = await spec.prefetch(session, client_timezone)
        except Exception:  # noqa: BLE001 — broad by design
            ...
```

- [ ] **Step 4: Thread timezone through `model_harness.py`**

- In `build_intent_aware_prompt`, add `client_timezone: str | None = None` parameter and pass it: `rules, data, failed = await IntentRegistry.run(intent, session, client_timezone)`.
- At the call site (around line 197, inside `stream_chat`), pass `client_timezone=client_timezone` (the variable is in scope there).
- Add `get_training_snapshot` to the timezone-injected tool set so model-driven re-calls also get the tz: rename/extend `_NUTRITION_TZ_TOOLS`. Keep it minimal — add `"get_training_snapshot"` to the set (rename the constant to `_TZ_AWARE_TOOLS` and update its one usage and the comment, OR just add the entry and broaden the comment). Pick the smallest clean change.

- [ ] **Step 5: Run, verify PASS** — `uv run pytest tests/test_intents.py -q`. Then run the full suite to catch call-site breakage: `uv run pytest -q`. Fix any other test that referenced `IntentRegistry.run(intent, session)` with the old arity or the old intent name.

- [ ] **Step 6: Commit**

```bash
git add src/prog_strength_agent/intents.py src/prog_strength_agent/model_harness.py tests/test_intents.py
git commit -m "feat(agent): analyze_training intent prefetches the training snapshot"
```

---

### Task C2: Router rename + force complex

**Files:**
- Modify: `src/prog_strength_agent/model_router.py`
- Test: `tests/test_model_router.py`

- [ ] **Step 1: Update/add failing tests** in `tests/test_model_router.py`

- Update the existing test that asserts `t.intent == "analyze_progress"` to `"analyze_training"` (and `_tool_use_block("complex", "analyze_progress")` → `"analyze_training"`).
- Add a test proving the deterministic override: even when the router LLM returns `simple` for `analyze_training`, the decision is forced to `complex`.

```python
@pytest.mark.asyncio
async def test_analyze_training_is_forced_complex_even_if_model_says_simple():
    client = SimpleNamespace(
        messages=SimpleNamespace(
            create=AsyncMock(return_value=_resp([_tool_use_block("simple", "analyze_training")]))
        )
    )
    router = ModelRouter(client=client, router_model="claude-haiku-4-5-20251001")
    from prog_strength_agent.telemetry import TurnInstrumentation
    t = TurnInstrumentation.new(user_id="u-1", session_id=None)
    decision = await router.route(
        messages=[{"role": "user", "content": "how did my training go this week?"}], telemetry=t
    )
    assert decision.tier == "complex"
    assert decision.intent == "analyze_training"
    assert t.routed_tier == "complex"
```

(Confirm `router.route` returns the decision; if it returns something else, assert via `t.routed_tier`/`t.intent` which the existing test already uses.)

- [ ] **Step 2: Run, verify FAIL** — `uv run pytest tests/test_model_router.py -q`.

- [ ] **Step 3: Edit `model_router.py`**

1. In `ROUTER_SYSTEM_PROMPT`, replace the `analyze_progress` bullet with:
   `- analyze_training — the user wants to assess, review, or reflect on their training over a period ("how did I do this week?", "how's my training going", "review my month"). Always routes to the complex tier.`
   And in the `tier` section, update the parenthetical that mentions plan_workout to also mention analyze_training, e.g. "(a) the intent is plan_workout or analyze_training — composing or reviewing a block of training is multi-step".
2. Add the deterministic override constant and apply it in `_parse_decision` right before returning the decision:

```python
# Intents that must always run on the complex tier regardless of what the
# router model returns — period analysis and planning are inherently
# multi-step and deserve the stronger model.
_ALWAYS_COMPLEX = {"plan_workout", "analyze_training"}
```

In `_parse_decision`, where it currently does `return RouterDecision(tier=tier, intent=intent)`:

```python
        if intent in _ALWAYS_COMPLEX:
            tier = "complex"
        return RouterDecision(tier=tier, intent=intent)
```

(`KNOWN_INTENTS` is imported from `intents.py`, so the enum already reflects the rename — no separate list to edit there.)

- [ ] **Step 4: Run, verify PASS** — `uv run pytest tests/test_model_router.py -q`, then `uv run pytest -q`.

- [ ] **Step 5: Lint + commit**

```bash
uv run ruff check src tests evals
git add src/prog_strength_agent/model_router.py tests/test_model_router.py
git commit -m "feat(agent): force complex tier for analyze_training in router"
```

---

## Final per-repo gate (before pushing each repo)

- **prog-strength-api:** `GOTOOLCHAIN=auto go build ./... && GOTOOLCHAIN=auto go vet ./... && GOTOOLCHAIN=auto go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run && GOTOOLCHAIN=auto go mod tidy && git diff --exit-code go.mod go.sum && GOTOOLCHAIN=auto go test ./...`
- **prog-strength-mcp:** `uv run pytest -q && uv run ruff check src tests`
- **prog-strength-agent:** `uv run pytest -q && uv run ruff check src tests evals`

## Self-review notes (spec coverage)

- API endpoint, defensive degradation, empty-but-healthy, compact shape, drill-down preserved (single-item tools untouched) → Tasks A1–A3. ✓
- MCP pure forwarder following steps.py pattern, registered in server.py → Tasks B1–B2. ✓
- Agent rename, force complex, snapshot prefetch for current local week, rules cite memories + all modalities, fail-soft, tests updated → Tasks C1–C2. ✓
- Running list-for-a-period (no tool today) folded into snapshot → A1/A2 running section. ✓
- Memory injection (PR #18) consumed, not modified → C1 leaves `memory.format_memory_block` untouched; rules reference the Background block. ✓
- Non-goals respected: `/dashboard/summary` untouched; no migrations; no new vector-memory infra; no raw sets/trackpoints in payload. ✓
