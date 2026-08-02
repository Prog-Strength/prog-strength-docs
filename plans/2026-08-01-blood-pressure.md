# Blood Pressure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a Blood Pressure feature across the stack: a Go domain + API, a dashboard tile, an MCP module, an agent intent, and a web page that debuts the shared agent chat bar.

**Architecture:** Blood pressure is shaped like bodyweight (timestamped per-entry measurements, soft-deleted, listed by range) with two differences: a reading carries two required numbers plus an optional pulse, and there is no per-user goal — the healthy range is a fixed ACC/AHA classification computed server-side. Each repo mirrors its bodyweight equivalent route-for-route / module-for-module.

**Tech Stack:** Go 1.25 (chi, SQLite), Python 3.12 (FastMCP, FastAPI, pydantic), Next.js 16 / React 19 / TypeScript / Tailwind v4 / recharts.

**Source of truth:** `sows/blood-pressure.md`. Design tokens: `design-system.md`. This plan is derived from both; where they conflict, the SOW wins on behavior and the design system wins on tokens.

---

## Cross-cutting conventions (read once)

- **API** (`prog-strength-api`): domain-oriented packages under `internal/`; one type/concept per file; repository pattern with a single SQLite impl and a compile-time `var _ Repository = (*SQLiteRepository)(nil)`; `context.Context` first param; user scoping at the storage layer (cross-user id → `ErrNotFound`); `errors.Is`; `httpresp.*` envelope only; TDD. Reference package: `internal/bodyweight/`. Reference dashboard tile: `internal/dashboard/bodyweight.go` + `tiles.go`.
- **MCP** (`prog-strength-mcp`): thin forwarder; one module per domain registered in `server.py`; pydantic `Field` bounds only. Reference: `src/prog_strength_mcp/bodyweight.py` + `api_client.py`.
- **Agent** (`prog-strength-agent`): intent registry in `intents.py`; router prompt in `model_router.py`. Reference: the existing `log_bodyweight` intent.
- **Web** (`prog-strength-web`): all API calls via `lib/api.ts`; page-private components in `_components/`, shared in `components/`; Tailwind + CSS-var tokens; recharts with a literal color-mirror module; Vitest tests co-located. Reference page: `app/(app)/bodyweight/` and `app/(app)/dashboard/`.
- **Every task is TDD:** write the failing test, run it red, implement, run it green, commit. Conventional-commit messages, lowercase subject (`feat(bloodpressure): ...`).

## Classification (the one shared algorithm — identical in Go, Python-agent-free, and TS)

Higher category wins; checks MUST be evaluated in descending severity:

```
crisis    if systolic >= 180 or diastolic >= 120
stage_2   if systolic >= 140 or diastolic >=  90
stage_1   if systolic >= 130 or diastolic >=  80
elevated  if systolic >= 120        // diastolic < 80 by fallthrough
normal    otherwise                 // systolic < 120 and diastolic < 80
```

Boundary cases every classification test must assert: `125/85`→stage_1, `135/75`→stage_1, `115/95`→stage_2, `119/79`→normal, `120/79`→elevated, `130/80`→stage_1, `140/90`→stage_2, `180/120`→crisis.

---

# Repo A — prog-strength-api

## Task A1: Migration 050 + `internal/bloodpressure` domain (types, classify, errors, repository)

**Files:**
- Create: `internal/db/migrations/050_blood_pressure.sql`
- Create: `internal/bloodpressure/bloodpressure.go` (Entry + Validate)
- Create: `internal/bloodpressure/category.go` (Classify)
- Create: `internal/bloodpressure/category_test.go`
- Create: `internal/bloodpressure/errors.go`
- Create: `internal/bloodpressure/repository.go`
- Create: `internal/bloodpressure/sqlite_repository.go`
- Test: `internal/bloodpressure/sqlite_repository_test.go`

**Numbering note:** verify `050` is still the next free index (`ls internal/db/migrations/`); if another SOW landed, renumber to the next free one.

- [ ] **Step 1: Migration.** Follow the `bodyweight_entries` conventions in `006_nutrition_and_bodyweight.sql`. No FK on `user_id`. CHECK constraints mirror handler validation.

```sql
-- migrations/050_blood_pressure.sql
-- Blood-pressure readings: one row per measurement, systolic/diastolic
-- required plus an optional pulse. Shaped like bodyweight_entries — no FK on
-- user_id (matches the other per-user content tables), soft delete via
-- deleted_at. CHECK bounds are physiological-plausibility guards against
-- fat-fingered input, not clinical limits. See sows/blood-pressure.md.
--
-- No goal table: the healthy range is a published clinical classification,
-- identical for every user, so it lives as a constant in Go (see
-- internal/bloodpressure/category.go), not as per-user state.

CREATE TABLE IF NOT EXISTS user_blood_pressure (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    systolic INTEGER NOT NULL CHECK (systolic BETWEEN 50 AND 300),
    diastolic INTEGER NOT NULL CHECK (diastolic BETWEEN 30 AND 200),
    pulse INTEGER CHECK (pulse IS NULL OR pulse BETWEEN 20 AND 250),
    measured_at DATETIME NOT NULL,
    created_at DATETIME NOT NULL,
    deleted_at DATETIME,
    -- Systolic is the peak and diastolic the trough of one pressure wave;
    -- equal-or-inverted is a data-entry error every time.
    CHECK (systolic > diastolic)
);

CREATE INDEX idx_bp_user_measured
    ON user_blood_pressure(user_id, measured_at DESC) WHERE deleted_at IS NULL;
```

- [ ] **Step 2: category.go (TDD — write category_test.go first).** Category is a string type with the five constants; `Classify(sys, dia int) Category` implements the descending-severity ladder above. The test is the boundary table from the conventions section.

```go
package bloodpressure

// Category is the ACC/AHA (2017) classification of a systolic/diastolic pair.
// The zero value is intentionally unused; Classify always returns one of the
// five named categories.
type Category string

const (
	CategoryNormal   Category = "normal"
	CategoryElevated Category = "elevated"
	CategoryStage1   Category = "stage_1"
	CategoryStage2   Category = "stage_2"
	CategoryCrisis   Category = "crisis"
)

// Classify maps a reading to its category. Higher category wins — a reading is
// classified by whichever of its two numbers is worse — so the checks run in
// descending severity and the order is load-bearing: testing elevated before
// stage_1 would silently under-report a pair like 125/85 (Stage 1, not
// Elevated, because diastolic already crossed 80).
func Classify(systolic, diastolic int) Category {
	switch {
	case systolic >= 180 || diastolic >= 120:
		return CategoryCrisis
	case systolic >= 140 || diastolic >= 90:
		return CategoryStage2
	case systolic >= 130 || diastolic >= 80:
		return CategoryStage1
	case systolic >= 120:
		return CategoryElevated
	default:
		return CategoryNormal
	}
}
```

- [ ] **Step 3: bloodpressure.go — Entry + Validate.**

```go
// Package bloodpressure tracks blood-pressure readings keyed by user +
// timestamp. It mirrors internal/bodyweight route-for-route; the differences
// are two required numbers plus an optional pulse, and no per-user goal (the
// healthy range is a fixed classification, see category.go).
// See prog-strength-docs/sows/blood-pressure.md.
package bloodpressure

import "time"

// Entry is one blood-pressure reading. Pulse is optional (nil when the user
// didn't record it). Category is derived, not stored — Classify computes it on
// read so no client re-derives a stored reading's category.
type Entry struct {
	ID         string
	UserID     string
	Systolic   int
	Diastolic  int
	Pulse      *int
	MeasuredAt time.Time
	CreatedAt  time.Time
	DeletedAt  *time.Time
}

// Category returns the reading's classification.
func (e *Entry) Category() Category { return Classify(e.Systolic, e.Diastolic) }

// Validate enforces the same invariants the schema CHECKs do, run handler-side
// so the caller gets a clean 400 instead of a 500 from a constraint violation.
func (e *Entry) Validate() error {
	if e.Systolic < 50 || e.Systolic > 300 {
		return ErrSystolicRange
	}
	if e.Diastolic < 30 || e.Diastolic > 200 {
		return ErrDiastolicRange
	}
	if e.Systolic <= e.Diastolic {
		return ErrSystolicNotAboveDiastolic
	}
	if e.Pulse != nil && (*e.Pulse < 20 || *e.Pulse > 250) {
		return ErrPulseRange
	}
	if e.MeasuredAt.IsZero() {
		return ErrMeasuredAtRequired
	}
	return nil
}
```

- [ ] **Step 4: errors.go.**

```go
package bloodpressure

import "errors"

var (
	// ErrNotFound is returned when an entry does not exist, was soft-deleted,
	// or belongs to a different user. Cross-user lookups surface as ErrNotFound
	// (not Forbidden) so a probing client can't distinguish "exists for someone
	// else" from "does not exist".
	ErrNotFound = errors.New("bloodpressure: not found")

	ErrSystolicRange             = errors.New("bloodpressure: systolic must be between 50 and 300")
	ErrDiastolicRange            = errors.New("bloodpressure: diastolic must be between 30 and 200")
	ErrPulseRange                = errors.New("bloodpressure: pulse must be between 20 and 250")
	ErrSystolicNotAboveDiastolic = errors.New("bloodpressure: systolic must be greater than diastolic")
	ErrMeasuredAtRequired        = errors.New("bloodpressure: measured_at is required")
)
```

- [ ] **Step 5: repository.go.** Interface with `Create`, `Get`, `List(ctx, userID, since, until *time.Time)`, `UpdateEntry`, `Delete`. Copy the doc-comment style from `bodyweight/repository.go` but drop the goal methods. Note: unlike bodyweight, there IS an UpdateEntry (the SOW specifies `PUT /blood-pressure/{id}`).

- [ ] **Step 6: sqlite_repository.go.** Structural copy of `bodyweight/sqlite_repository.go` minus goal methods, with the pulse-nullable scan (`sql.NullInt64`). `Create` sets `ID = id.New()`, `CreatedAt = now().UTC()`. `List` uses the same static-clause branching (`user_id`, `deleted_at IS NULL`, optional `measured_at >= ?` / `< ?`), `ORDER BY measured_at DESC`. `UpdateEntry` sets `systolic, diastolic, pulse, measured_at` scoped to `id AND user_id AND deleted_at IS NULL`, returns `ErrNotFound` on 0 rows. `Delete` soft-deletes. Add the `var _ Repository = (*SQLiteRepository)(nil)` assertion. Validate() runs in Create and UpdateEntry before the DB call.

```go
// pulse scan helper pattern:
var pulse sql.NullInt64
// ... Scan(..., &pulse, ...)
if pulse.Valid { p := int(pulse.Int64); e.Pulse = &p }
// insert bind: convert *int -> any (nil or value):
var pulseArg any
if e.Pulse != nil { pulseArg = *e.Pulse }
```

- [ ] **Step 7: sqlite_repository_test.go (TDD — ideally written alongside step 6).** Use `internal/db/dbtest` (`dbtest.New(t)`) exactly as `bodyweight` handler/repo tests do — inspect `internal/bodyweight/*_test.go` and `internal/testutil` for the exact helper names. Cover: create→get round-trip (pulse present and nil); list-by-range newest-first; soft delete then get→ErrNotFound; partial update via UpdateEntry; ownership scoping (user B cannot get/update/delete user A's row → ErrNotFound); every CHECK boundary including `systolic > diastolic` rejection (Create returns the validation error, no DB write).

- [ ] **Step 8: Run** `go test ./internal/bloodpressure/... ./internal/db/...` green.
- [ ] **Step 9: Commit** `feat(bloodpressure): add domain package, classification, and migration 050`.

**Context for implementer:** Read `internal/bodyweight/{bodyweight.go,errors.go,repository.go,sqlite_repository.go,sqlite_repository_test.go}` first — this task is that shape minus the goal, plus pulse-nullable and category. The migration runner embeds `internal/db/migrations/*.sql` via `embed.FS` and applies on startup; nothing else registers a migration.

## Task A2: `internal/bloodpressure` HTTP handler + routes + server wiring

**Files:**
- Create: `internal/bloodpressure/handler.go`
- Test: `internal/bloodpressure/handler_test.go`
- Modify: `internal/server/server.go` (repo construction + Mount)

- [ ] **Step 1: handler.go.** Mirror `bodyweight/handler.go` but: no `userRepo` dependency (`NewHandler(repo Repository)`), no goal routes, and the entry DTO carries `category`. Routes:

```go
func (h *Handler) Mount(r chi.Router) {
	r.Route("/blood-pressure", func(r chi.Router) {
		r.Get("/", h.list)
		r.Post("/", h.create)
		r.Put("/{id}", h.update)
		r.Delete("/{id}", h.delete)
	})
}
```

DTO and requests:

```go
type entryDTO struct {
	ID         string    `json:"id"`
	Systolic   int       `json:"systolic"`
	Diastolic  int       `json:"diastolic"`
	Pulse      *int      `json:"pulse"`
	Category   Category  `json:"category"`
	MeasuredAt time.Time `json:"measured_at"`
	CreatedAt  time.Time `json:"created_at"`
}

func toDTO(e Entry) entryDTO {
	return entryDTO{
		ID: e.ID, Systolic: e.Systolic, Diastolic: e.Diastolic, Pulse: e.Pulse,
		Category: e.Category(), MeasuredAt: e.MeasuredAt, CreatedAt: e.CreatedAt,
	}
}

type createRequest struct {
	Systolic   int        `json:"systolic"`
	Diastolic  int        `json:"diastolic"`
	Pulse      *int       `json:"pulse"`
	MeasuredAt *time.Time `json:"measured_at"`
}

type updateRequest struct {
	Systolic   *int       `json:"systolic"`
	Diastolic  *int       `json:"diastolic"`
	Pulse      *int       `json:"pulse"`
	MeasuredAt *time.Time `json:"measured_at"`
}
```

Behavior, mirroring bodyweight's handler:
- `create`: read `auth.UserIDFrom`; decode; `measuredAt := time.Now().UTC()` unless supplied; build Entry; on `repo.Create` err, map validation errors (`isValidationError`) to 400 else ServerError; `httpresp.Created(w, "logged blood pressure", toDTO(*entry))`.
- `list`: `parseSinceUntil(r)` (copy the helper into this package — it's private to bodyweight; a small duplicate is correct here, do NOT export from bodyweight), `repo.List`, map to `[]entryDTO`, `httpresp.OK`.
- `update`: `Get` existing (404 on ErrNotFound), overlay only supplied pointer fields, validating each supplied field inline for a clean 400 (systolic/diastolic/pulse ranges; note the `systolic > diastolic` invariant is enforced by `Validate()` inside `UpdateEntry`, which returns the right error — map it via isValidationError), `repo.UpdateEntry`, `httpresp.OK`.
- `delete`: `repo.Delete`, 404 on ErrNotFound, `httpresp.OK(w, "deleted blood pressure reading", nil)`.
- `isValidationError`: true for the five validation sentinels.

**Note on update + `systolic > diastolic`:** when only one of the pair is supplied, the overlay merges it onto the existing row, then `Validate()` inside `UpdateEntry` checks the combined pair — so a PUT that would invert the pair returns 400 via the ErrSystolicNotAboveDiastolic mapping. Add an explicit handler test for this.

- [ ] **Step 2: handler_test.go (TDD).** HTTP-level tests against the SQLite repo + a temp DB (copy the setup from `bodyweight/handler_test.go`). Cover: POST defaults `measured_at` and returns 201 with `category`; POST out-of-range → 400; POST inverted pair (`120/130`) → 400; PUT overlays only supplied fields; PUT to a cross-user id → 404; PUT that inverts the pair → 400; DELETE → then GET list omits it; every returned entry has a non-empty `category`.

- [ ] **Step 3: server.go wiring.** After `bodyweightRepo = ...` add `bloodPressureRepo := bloodpressure.NewSQLiteRepository(database)` (declare a `var bloodPressureRepo bloodpressure.Repository` near the other repo vars, matching style; or the short-decl if only used locally — match how sibling repos that aren't swapped for a memory impl are declared; `bodyweightRepo` uses a package-level interface var, but a local `:=` is fine since there's a single impl. Prefer the local `:=` to minimize the diff, placed right after `bodyweightRepo` is assigned). Add the import. In the JWT-gated group, after `bodyweight.NewHandler(...).Mount(r)`, add:

```go
// Blood pressure — its own package, same JWT-gated group as bodyweight.
// No user repository needed (no unit to default); the healthy-range
// classification is a code constant, not per-user state.
bloodpressure.NewHandler(bloodPressureRepo).Mount(r)
```

The `bloodPressureRepo` is also consumed by the dashboard handler in Task A3, so declare it where both the mount above and the `dashboard.NewHandler(...)` call (same function) can see it.

- [ ] **Step 4: Run** `go test ./internal/bloodpressure/... ./internal/server/...` and `go build ./...` green.
- [ ] **Step 5: Commit** `feat(bloodpressure): add HTTP handler and mount /blood-pressure routes`.

## Task A3: Dashboard Blood Pressure tile

**Files:**
- Modify: `internal/dashboard/tiles.go` (id + Catalog placement)
- Modify: `internal/dashboard/tiles_test.go` (add id to the two hard-coded lists)
- Modify: `internal/dashboard/dto.go` (BloodPressureSection + nested types)
- Create: `internal/dashboard/bloodpressure.go` (`buildBloodPressure`)
- Test: `internal/dashboard/bloodpressure_test.go`
- Modify: `internal/dashboard/handler.go` (add repo field, ctor param, `buildBloodPressureSection`, gated call in `summary`)
- Modify: `internal/dashboard/layout_resolve.go` — **unchanged** (assert in test)
- Modify: `internal/dashboard/summary_layout_test.go` or add a test that default layout excludes `blood_pressure`
- Modify: `internal/server/server.go` (`dashboard.NewHandler(...)` gains the bp repo arg)

- [ ] **Step 1: tiles.go.** Add `TileBloodPressure TileID = "blood_pressure"` in the const block after `TileBodyweight`. In `Catalog`, insert `TileBloodPressure` **after `TileBodyweight` and before `TileRecovery`** (the health-metric cluster).

- [ ] **Step 2: tiles_test.go.** Both `TestCatalog_EveryConstantAppearsExactlyOnce` and `TestCatalog_Order` hard-code the full tile list — add `TileBloodPressure` in the same position (after `TileBodyweight`, before `TileRecovery`) to both.

- [ ] **Step 3: dto.go — section types.**

```go
// BloodPressureSection is the blood-pressure tile. nil at the Summary level
// when the user has logged no readings. The two sparks are computed from ONE
// day-bucketed series so their indices align (the card draws them as two lines
// on a shared x-axis).
type BloodPressureSection struct {
	Latest   BloodPressureLatest  `json:"latest"`
	Category Category             `json:"category"`
	Avg30    *BloodPressureAvg    `json:"avg_30d"` // nil when the trailing-30d window is empty
	// SystolicSpark and DiastolicSpark are ~8 daily-average points, oldest→
	// newest, same length and same day set.
	SystolicSpark  []float64 `json:"systolic_spark"`
	DiastolicSpark []float64 `json:"diastolic_spark"`
}

type BloodPressureLatest struct {
	Systolic   int       `json:"systolic"`
	Diastolic  int       `json:"diastolic"`
	MeasuredAt time.Time `json:"measured_at"`
}

type BloodPressureAvg struct {
	Systolic  int `json:"systolic"`
	Diastolic int `json:"diastolic"`
}
```

Use `Category` = `bloodpressure.Category` (import the package and alias the field type as `bloodpressure.Category`). Add `BloodPressure *BloodPressureSection json:"blood_pressure,omitempty"` to the `Summary` struct is NOT needed — the handler builds a `map[string]any`, so no Summary field is required; but if a `Summary` struct field is used elsewhere for docs, mirror the recovery `omitempty` convention. (Confirm by reading how `Recovery` flows: it IS on Summary with omitempty AND set via the map. Match that: add `BloodPressure *BloodPressureSection json:"blood_pressure,omitempty"` to Summary for symmetry.)

- [ ] **Step 4: bloodpressure.go — pure builder.**

```go
package dashboard

import (
	"sort"
	"time"

	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/bloodpressure"
)

const bpSparkMax = 8

// buildBloodPressure assembles the blood-pressure tile from the user's
// readings. Pure (no DB, no clock beyond the passed now-window). loc buckets
// readings into local calendar days for the daily-average sparks and the
// 30-day average. Returns nil when there are no readings.
//
// entries are the trailing window the handler fetched (newest-first by
// contract, but this does not rely on order). now is the reference instant for
// the trailing-30-day average window.
func buildBloodPressure(entries []bloodpressure.Entry, now time.Time, loc *time.Location) *BloodPressureSection {
	if len(entries) == 0 {
		return nil
	}
	if loc == nil {
		loc = time.UTC
	}

	// newest by MeasuredAt (don't trust order).
	newest := entries[0]
	for i := range entries {
		if entries[i].MeasuredAt.After(newest.MeasuredAt) {
			newest = entries[i]
		}
	}

	// One day-bucketed series feeds both sparks so their indices align.
	// Bucket key = local calendar day; value = mean systolic & diastolic.
	type dayAgg struct{ sysSum, diaSum, n int }
	byDay := map[string]*dayAgg{}
	var order []string
	for _, e := range entries {
		key := e.MeasuredAt.In(loc).Format("2006-01-02")
		a := byDay[key]
		if a == nil {
			a = &dayAgg{}
			byDay[key] = a
			order = append(order, key)
		}
		a.sysSum += e.Systolic
		a.diaSum += e.Diastolic
		a.n++
	}
	sort.Strings(order) // oldest→newest

	sysDaily := make([]float64, len(order))
	diaDaily := make([]float64, len(order))
	for i, k := range order {
		a := byDay[k]
		sysDaily[i] = roundHalf(float64(a.sysSum) / float64(a.n))
		diaDaily[i] = roundHalf(float64(a.diaSum) / float64(a.n))
	}

	// Trailing-30-day average over the raw readings (not the daily series),
	// nil when the window holds nothing.
	cutoff := now.AddDate(0, 0, -30)
	var sSum, dSum, n int
	for _, e := range entries {
		if e.MeasuredAt.Before(cutoff) {
			continue
		}
		sSum += e.Systolic
		dSum += e.Diastolic
		n++
	}
	var avg30 *BloodPressureAvg
	if n > 0 {
		avg30 = &BloodPressureAvg{
			Systolic:  int(roundHalf(float64(sSum) / float64(n))),
			Diastolic: int(roundHalf(float64(dSum) / float64(n))),
		}
	}

	return &BloodPressureSection{
		Latest: BloodPressureLatest{
			Systolic: newest.Systolic, Diastolic: newest.Diastolic, MeasuredAt: newest.MeasuredAt,
		},
		Category:       newest.Category(),
		Avg30:          avg30,
		SystolicSpark:  downsampleFloats(sysDaily, bpSparkMax),
		DiastolicSpark: downsampleFloats(diaDaily, bpSparkMax),
	}
}

// roundHalf rounds to the nearest integer, ties away from zero (values are
// always positive here).
func roundHalf(x float64) float64 { return float64(int(x + 0.5)) }
```

Note: `downsampleFloats` already exists in `buckets.go`. Both sparks pass through the same-length `order` slice → identical length and day alignment (the SOW's required invariant). Do NOT add a second `roundHalf` if one already exists in the package — grep first; if `math.Round` is idiomatic here, use it instead.

- [ ] **Step 5: bloodpressure_test.go.** Assert: nil section on empty input; sparks equal-length and day-aligned (feed a 3-readings-on-Tuesday + 1-on-Thursday set, expect 2 daily points each, systolic and diastolic same length); 30-day average excludes an older reading (one reading 40 days ago, one today → avg30 == today's values); `Category` is the newest reading's category; daily average rounds (two readings 120 and 121 on one day → 121 or 120 per rounding rule — assert the exact expected value). Use a fixed `now` and a fixed `loc` (e.g. `time.UTC` and an America/New_York case to prove local bucketing).

- [ ] **Step 6: handler.go wiring.** Add `bloodPressureRepo bloodpressure.Repository` field + ctor param (append to `NewHandler` signature after `bodyweightRepo`; update all call sites — there is one in server.go and any in `handler_test.go`). Add:

```go
func (h *Handler) buildBloodPressureSection(ctx context.Context, r *http.Request, userID string, since time.Time, now time.Time, loc *time.Location) *BloodPressureSection {
	entries := defer1(ctx, r, "blood pressure", func() ([]bloodpressure.Entry, error) {
		return h.bloodPressureRepo.List(ctx, userID, &since, nil)
	})
	return buildBloodPressure(entries, now, loc)
}
```

In `summary`, add a gated call mirroring bodyweight — use a window that covers both the sparks and the 30-day average. `since8w` (56 days) already exists and comfortably covers 30 days + ~8 daily points; reuse it:

```go
if enabled[TileBloodPressure] {
	out[string(TileBloodPressure)] = h.buildBloodPressureSection(ctx, r, userID, since8w, now, loc)
}
```

**Also update `handler_test.go`** where `NewHandler` is constructed — add a blood-pressure repo argument (a real `bloodpressure.NewSQLiteRepository(db)` against the test DB, or the existing test's repo-construction pattern). Grep the dashboard tests for `NewHandler(` and fix every call site.

- [ ] **Step 7: default-layout test.** In the dashboard test that asserts the default layout (see `summary_layout_test.go` / `layout_*_test.go`), add an assertion that `defaultLayout(...)` does NOT contain `TileBloodPressure`. `layout_resolve.go` itself is unchanged.

- [ ] **Step 8: server.go.** Update the `dashboard.NewHandler(...)` call to pass `bloodPressureRepo` in the new argument position (matching the ctor change). The repo var was created in Task A2.

- [ ] **Step 9: Run** `go test ./internal/dashboard/... ./internal/server/...` and `go build ./...` green.
- [ ] **Step 10: Commit** `feat(dashboard): add blood-pressure tile, section builder, and catalog entry`.

## Task A4: API green-gate

- [ ] Run the full local CI gate for the API (see the Gate section at the end). Fix any lint/vet/tidy/test failure by changing code, never by silencing. Commit any formatting/tidy fixups.

---

# Repo B — prog-strength-mcp

## Task B1: `blood_pressure` MCP module

**Files:**
- Modify: `src/prog_strength_mcp/api_client.py` (`log_blood_pressure`, `list_blood_pressure`)
- Create: `src/prog_strength_mcp/blood_pressure.py`
- Modify: `src/prog_strength_mcp/server.py` (import + `blood_pressure.register(mcp, api)`)
- Create: `tests/test_blood_pressure_tools.py`

- [ ] **Step 1: api_client.py.** Add two methods mirroring `log_bodyweight`/`list_bodyweight` (read them first). `log_blood_pressure(auth, *, systolic, diastolic, pulse=None, measured_at=None)` → `POST /blood-pressure`, body `{systolic, diastolic}` plus `pulse`/`measured_at` only when non-None, returns `data` dict-or-`{}`. `list_blood_pressure(auth, *, since=None, until=None)` → `GET /blood-pressure`, params only when set, returns `data` list-or-`[]`.

- [ ] **Step 2: blood_pressure.py.** Copy the structure of `bodyweight.py` (module docstring, `_auth_header_or_raise`, `register(mcp, api)`). Two tools:

```python
@mcp.tool
async def log_blood_pressure(
    systolic: Annotated[int, Field(ge=50, le=300, description=(
        "Systolic pressure — the HIGHER of the two numbers, the pressure "
        "during a heartbeat. In '122 over 78' this is 122."))],
    diastolic: Annotated[int, Field(ge=30, le=200, description=(
        "Diastolic pressure — the LOWER of the two numbers, the pressure "
        "between beats. In '122 over 78' this is 78."))],
    pulse: Annotated[int | None, Field(default=None, ge=20, le=250, description=(
        "Heart rate in beats per minute, if the cuff reported one. Omit "
        "when the user didn't mention it."))] = None,
    measured_at: Annotated[str | None, Field(default=None, description=(
        "RFC3339 UTC timestamp of when the reading was taken. Omit to "
        "default to now. The agent resolves relative phrases like 'this "
        "morning' before calling."))] = None,
) -> dict[str, Any]:
    """Log a single blood-pressure reading (systolic/diastolic, optional pulse)."""
    auth = _auth_header_or_raise()
    try:
        return await api.log_blood_pressure(
            auth, systolic=systolic, diastolic=diastolic, pulse=pulse, measured_at=measured_at)
    except APIError as e:
        raise RuntimeError(f"API error ({e.status_code}): {e.message}") from e
```

`list_blood_pressure(since=None, until=None)` mirrors `list_bodyweight`. Note the `pulse` default: pydantic `Field(default=None, ...)` with the `= None` param default, matching the `bodyweight.py` `measured_at` pattern exactly. Do NOT put classification/averaging here — the MCP is a pure forwarder.

- [ ] **Step 3: server.py.** Add `blood_pressure` to the `from prog_strength_mcp import (...)` block and add `blood_pressure.register(mcp, api)` next to `bodyweight.register(mcp, api)`.

- [ ] **Step 4: tests.** Copy `tests/test_steps_tools.py` structure exactly. API-client tests with `respx`: `log_blood_pressure` PUTs/POSTs the right body (`{"systolic":122,"diastolic":78}` with pulse/measured_at omitted when None; included when set), forwards auth, unwraps `data`, non-dict `data`→`{}`, surfaces `APIError`. `list_blood_pressure` forwards params and unwraps list. Tool-boundary tests: missing auth → `RuntimeError` matching "Authorization" (via `_ExplodingAPI`); `APIError` → `RuntimeError` matching "500" (via `_FailingAPI`).

- [ ] **Step 5: Run** `uv run pytest tests/test_blood_pressure_tools.py` then the full `uv run pytest` and `uv run ruff check src tests`. Green.
- [ ] **Step 6: Commit** `feat(blood_pressure): add MCP module forwarding /blood-pressure`.

**Context:** No AGENTS.md/CONTRIBUTING in this repo; the gate is `uv run pytest` + `uv run ruff check src tests`. Check `pyproject.toml` for the ruff line-length. There are no git hooks — run the gate yourself.

---

# Repo C — prog-strength-agent

## Task C1: `log_blood_pressure` intent

**Files:**
- Modify: `src/prog_strength_agent/intents.py` (KNOWN_INTENTS + prefetch/format/rules + register)
- Modify: `src/prog_strength_agent/model_router.py` (intent line in the router prompt)
- Modify: `tests/test_intents.py` (KNOWN_INTENTS enum set + a prefetch/format test)

- [ ] **Step 1: KNOWN_INTENTS.** Add `"log_blood_pressure"` to the `KNOWN_INTENTS` tuple in `intents.py` (place it right after `"log_bodyweight"`).

- [ ] **Step 2: prefetch + format + rules.** Model on `_log_bodyweight_prefetch`/`_log_bodyweight_format`/`_LOG_BODYWEIGHT_RULES`:

```python
async def _log_blood_pressure_prefetch(session: Any, _timezone: str | None) -> dict[str, Any]:
    since = (datetime.now(UTC) - timedelta(days=14)).isoformat().replace("+00:00", "Z")
    res = await session.call_tool("list_blood_pressure", {"since": since})
    return {"entries": _decode_tool_result(res)}


def _log_blood_pressure_format(data: dict[str, Any]) -> str:
    entries = data.get("entries") or []
    if not entries:
        return "RECENT BLOOD PRESSURE (last 14 days): (no entries yet)"
    lines = ["RECENT BLOOD PRESSURE (last 14 days, most recent first):"]
    for e in entries:
        pulse = e.get("pulse")
        pulse_str = f" · pulse {pulse}" if pulse else ""
        lines.append(
            f"- {e.get('measured_at', '?')} · "
            f"{e.get('systolic', '?')}/{e.get('diastolic', '?')} "
            f"({e.get('category', '?')}){pulse_str}"
        )
    return "\n".join(lines)


_LOG_BLOOD_PRESSURE_RULES = """\
The user is logging a blood-pressure reading. The first/larger number they \
say is systolic, the second/smaller is diastolic (e.g. "122 over 78" is \
systolic 122, diastolic 78). If the phrasing was relative ("this morning"), \
confirm the resolved timestamp back to them. State the reading's category as \
a plain fact only when it is outside the normal range, and reference the \
recent trend only when the new reading is a meaningful departure from it.

Safety, non-negotiable: do NOT diagnose, do NOT interpret a reading as a \
medical condition, and NEVER discuss medication — dosage, timing, starting, \
or stopping. On a crisis-range reading, state the classification factually \
and suggest contacting a clinician; do not escalate, alarm, or speculate \
about the cause. Report numbers and their published classification only.\
"""


_register(IntentSpec(
    intent="log_blood_pressure",
    rules=_LOG_BLOOD_PRESSURE_RULES,
    prefetch=_log_blood_pressure_prefetch,
    format=_log_blood_pressure_format,
))
```

- [ ] **Step 3: model_router.py.** In the router prompt's `intent:` list, add a line after the `log_bodyweight` line: `- log_blood_pressure — the user is logging a blood-pressure reading (e.g. "122 over 78", "my BP is 130/85").`

- [ ] **Step 4: tests.** In `tests/test_intents.py`: add `"log_blood_pressure"` to the `test_known_intents_enum` expected set. Add a test mirroring `test_log_bodyweight_prefetch_calls_list_with_14_day_window` that asserts the prefetch calls `list_blood_pressure` with a `since`, and that `rules` mentions the safety boundary (e.g. `"medication" in rules.lower()` and `"blood" in rules.lower()`). Add a formatter test: empty history renders the "(no entries yet)" line; a populated entry renders `122/78 (elevated)` and pulse when present.

- [ ] **Step 5: Run** `uv run pytest` and `uv run ruff check src tests evals`. Green. (No new LLM spend — these are unit tests, no eval runs.)
- [ ] **Step 6: Commit** `feat(agent): add log_blood_pressure intent with safety rules`.

**Context:** The agent already ships `log_bodyweight` — this is a near-mechanical parallel. The router classifier enum is built from `KNOWN_INTENTS` automatically (`model_router.py` line ~41), so step 1 is what makes it routable; step 3 just teaches the model when to pick it. PR title must be `feat(...)` so it deploys (see AGENTS.md).

---

# Repo D — prog-strength-web

> **Critical:** this is the "NOT the Next.js you know" fork. Before writing page/route code, read `node_modules/next/dist/docs/` and mirror the validated patterns in `app/(app)/bodyweight/` and `app/(app)/dashboard/`. Run `npm ci` first (node_modules is not present in the clone).

## Task D1: Promote `CommandBar` to a shared component

**Files:**
- Create: `components/command-bar.tsx` (moved)
- Delete: `app/(app)/dashboard/_components/command-bar.tsx`
- Modify: `app/(app)/dashboard/page.tsx` (import path)

- [ ] **Step 1:** Move `app/(app)/dashboard/_components/command-bar.tsx` verbatim to `components/command-bar.tsx` (content unchanged — it already accepts a `placeholder` prop). Use `git mv` so the move is legible in the diff.
- [ ] **Step 2:** Update the dashboard import from `"./_components/command-bar"` to `"@/components/command-bar"`.
- [ ] **Step 3:** Grep the repo for any other importer of the old path and update it (there should be only the dashboard).
- [ ] **Step 4: Run** `npm run typecheck && npm run test && npm run lint`. The dashboard's existing CommandBar tests (if any) must pass unchanged. If a test imported the old path, update the import only — do not change assertions.
- [ ] **Step 5: Commit** `refactor(command-bar): promote to shared components/ for reuse`.

**Note:** behavior is unchanged — a move plus an import change. `refactor:` prefix is correct (no behavior change).

## Task D2: Web lib layer — helpers, chart colors, API client, tile mirror

**Files:**
- Create: `lib/blood-pressure.ts` + `lib/blood-pressure.test.ts`
- Create: `lib/blood-pressure-chart-colors.ts`
- Modify: `lib/api.ts` (BP entry types + 4 wrappers + `DashboardBloodPressure` DTO + `DashboardSummary.blood_pressure` field)
- Modify: `lib/dashboard-tiles.ts` (+ `lib/dashboard-tiles.test.ts`)
- Modify: `lib/dashboard.ts` (BloodPressureView + adapter + DashboardData field)

- [ ] **Step 1: lib/blood-pressure.ts (TDD).** Exports:
  - `BP_CATEGORIES` metadata: for each of `normal|elevated|stage_1|stage_2|crisis`, a display `label` and a design-system status tone token: `normal`→`--success`, `elevated`/`stage_1`→`--warning`, `stage_2`/`crisis`→`--danger`.
  - Threshold constants and `classify(systolic, diastolic): BpCategory` — the exact descending ladder from the conventions section (mirror of the Go `Classify`). This is used ONLY for values the server never classified: a daily average and the log-modal live preview. For a stored reading, callers use the DTO's `category`.
  - `dailyAverages(entries, timeZone)`: bucket readings by **local calendar day** in the given IANA tz, return `{ day: string; systolic: number; diastolic: number; count: number; category: BpCategory }[]` oldest→newest, arithmetic mean rounded to nearest int, category from `classify(avgSys, avgDia)`. Gap days are simply absent (the chart renders gaps).
  - `lib/blood-pressure.test.ts`: the boundary table (every case in the conventions section) for `classify`; daily averaging over a multi-reading day, a gap day (absent, not zero), and a tz-bucketing case (a reading at 23:30 local on the boundary lands in the right local day, not the UTC day).

- [ ] **Step 2: lib/blood-pressure-chart-colors.ts.** **Do NOT copy `lib/bodyweight-chart-colors.ts`** — it holds the retired violet `#8b7cf6`. Mirror the CURRENT tokens per the SOW table:

```ts
/**
 * Chart SVG colors for the blood-pressure trend chart. Recharts writes
 * stroke/fill as SVG attributes where CSS var(--token) does not resolve, so
 * these literals MIRROR the current design-system tokens (app/globals.css /
 * design-system.md). Not a new palette — keep in sync with those tokens.
 *
 * NB: intentionally NOT sourced from bodyweight-chart-colors.ts, which still
 * carries the retired violet accent (pre-v0.4 re-tone). These use the current
 * periwinkle/neutral/success tokens.
 */
export const BP_CHART_COLORS = {
  systolic: "#9aa6d6", // --accent (periwinkle) — the primary series
  diastolic: "#7d818c", // --muted — subordinate, neutral
  bandFill: "rgba(134, 179, 159, 0.10)", // --success at low alpha — healthy band
  reference: "#7d818c", // --muted, dashed — the 130 / 80 reference lines
  grid: "rgba(255, 255, 255, 0.06)", // --border hairline
  axis: "#7d818c", // --muted — axis ticks + text
  tooltipSurface: "#15171b", // --surface
  tooltipBorder: "rgba(255, 255, 255, 0.10)", // --border
} as const;
```

(Confirm `--surface` = `#15171b` against `app/globals.css`; the design-system doc lists surface `#15171b`. Use the value in globals.css if it differs.)

- [ ] **Step 3: lib/api.ts.** Mirror the bodyweight block (lines ~1163–1284):
  - `BloodPressureCategory = "normal" | "elevated" | "stage_1" | "stage_2" | "crisis"`.
  - `BloodPressureEntry = { id; systolic; diastolic; pulse: number | null; category: BloodPressureCategory; measured_at: string; created_at: string }`.
  - `CreateBloodPressurePayload = { systolic; diastolic; pulse?: number | null; measured_at?: string }`.
  - `listBloodPressure(token, {since?, until?})`, `createBloodPressureEntry(token, payload)`, `updateBloodPressureEntry(token, id, {systolic?; diastolic?; pulse?; measured_at?})`, `deleteBloodPressureEntry(token, id)` — copy the bodyweight wrappers' shape (fetch + `unwrap`, same error handling on delete).
  - `DashboardBloodPressure` DTO: `{ latest: { systolic; diastolic; measured_at }; category: BloodPressureCategory; avg_30d: { systolic; diastolic } | null; systolic_spark: number[]; diastolic_spark: number[] }`. Add `blood_pressure?: DashboardBloodPressure | null;` to `DashboardSummary`.

- [ ] **Step 4: lib/dashboard-tiles.ts + test.** Add `"blood_pressure"` to the `TileId` union **after `"bodyweight"`, before `"recovery"`**, and the `TILE_CATALOG` entry in the same position:

```ts
{
  id: "blood_pressure",
  title: "Blood Pressure",
  href: "/blood-pressure",
  description: "Latest reading and trend against the healthy range.",
},
```

In `lib/dashboard-tiles.test.ts`: bump `"has exactly 10 tiles"` → 11; add `"blood_pressure"` to the `ids are in the Go catalog order` array (same position); add `blood_pressure: true` to the exhaustive `ALL_TILE_IDS` Record.

- [ ] **Step 5: lib/dashboard.ts.** Add `BloodPressureView` (mirror `BodyweightView` shape: `latest`, `category`, `avg30` nullable, `systolicSpark`, `diastolicSpark`), an `adaptBloodPressure(bp: DashboardBloodPressure): BloodPressureView` (sanitize sparks via the existing helper), a `bloodPressure: Section<BloodPressureView>` field on `DashboardData`, the `{ present: false }` entry in the empty-state object, and the `summary.blood_pressure ? {present:true, ...adapt} : {present:false}` line in `adaptDashboard`.

- [ ] **Step 6: Run** `npm run typecheck && npm run test && npm run lint`. Green.
- [ ] **Step 7: Commit** `feat(blood-pressure): add web lib helpers, api client, and tile mirror`.

## Task D3: Blood Pressure page + sidebar + page components

**Files:**
- Modify: `components/sidebar.tsx` (NAV entry + `HeartPulseIcon`)
- Create: `app/(app)/blood-pressure/page.tsx`
- Create: `app/(app)/blood-pressure/_components/bp-trend-card.tsx` (+ test)
- Create: `app/(app)/blood-pressure/_components/bp-log-modal.tsx`
- Create: `app/(app)/blood-pressure/_components/bp-readings-timeline.tsx`
- Create: `app/(app)/blood-pressure/_components/bp-about.tsx` (+ test)

- [ ] **Step 1: sidebar.tsx.** Add a `NAV` entry `{ href: "/blood-pressure", label: "Blood Pressure", icon: <HeartPulseIcon /> }` **after the Bodyweight entry, before Recovery**. Add a `HeartPulseIcon()` inline SVG in the same file matching the neighbors' stroke weight (1.75) and 16px-in-20px box — a heart outline with a pulse/ECG notch. Prefix-match highlight is the default; no `exact` needed.

- [ ] **Step 2: bp-about.tsx (+ test).** Static, factual explainer card (`--radius-card` panel, hairline border). Content per the SOW: what BP is; systolic (top/larger = heart contracting) vs diastolic (bottom = between beats); why monitoring matters (silent, noisy single readings, signal is the average/trend); the classification table (`<120/<80` Normal, `120–129/<80` Elevated, `130–139 or 80–89` Stage 1, `≥140 or ≥90` Stage 2, `>180 or >120` Crisis) with the source named (2017 ACC/AHA); a closing "general information, not medical advice — talk to a clinician" line. Category rows use the status-tone tokens from `lib/blood-pressure.ts` `BP_CATEGORIES`. Test: renders the "not medical advice" line and all five category labels.

- [ ] **Step 3: bp-log-modal.tsx.** Systolic, diastolic, optional pulse, date/time defaulting to now. Same modal chrome / focus trap / Escape handling as the bodyweight action sheet (read `app/(app)/bodyweight/page.tsx` for the modal it embeds and mirror it — but keep this in its own file, do NOT inline into page.tsx). Client-side validation mirrors the API bounds AND the `systolic > diastolic` invariant, showing an inline message (not relying on the 400). A live category preview using `classify()` from `lib/blood-pressure.ts`. Emits `onSubmit({systolic, diastolic, pulse, measuredAt})` and `onClose`.

- [ ] **Step 4: bp-readings-timeline.tsx.** Day-grouped reading rail — mirror the bodyweight readings timeline (`app/(app)/bodyweight/` — the SOW references a timeline rail; read its implementation). `READINGS_PER_PAGE = 20`, days never split across a page boundary. Each reading shows `122/78`, its category chip (status tone), pulse when present, and the time. Edit/delete emitted as callbacks so the page owns mutations.

- [ ] **Step 5: bp-trend-card.tsx (+ test).** The hero: a recharts `ComposedChart`, back to front: a shaded band across the healthy region (an `Area` or `ReferenceArea` filled `BP_CHART_COLORS.bandFill` between the healthy bounds), dashed `ReferenceLine`s at **130** and **80** (`BP_CHART_COLORS.reference`), then systolic (heavier stroke, `BP_CHART_COLORS.systolic`) and diastolic (`BP_CHART_COLORS.diastolic`) `type="monotone"` lines over daily averages with **gaps for missing days** (use `connectNulls={false}` and null y-values for gap days — never zero, never interpolate across holes). Requirements:
  - **Y-axis domain must always include both reference lines (80 and 130) and the band** — use a domain function like bodyweight's `[dataMin => Math.floor(Math.min(dataMin, 80) - N), dataMax => Math.ceil(Math.max(dataMax, 130) + N)]`. Test this: readings entirely at 110/70 must still keep 130 within the domain.
  - Tooltip shows both numbers `122/78`, the day, the reading count contributing to that day's average, and the day's category.
  - A legend distinguishes the two series.
  - Below `sm:`, follow the bodyweight chart's `matchMedia`-driven responsive y-axis width/label handling.
  - Stat tiles tucked inside the same bordered box below the chart, via `components/stat-tile.tsx`: **Average** (`sys/dia` over range), **Latest** (reading + category chip), **In normal range** (% of readings classified `normal`), **Readings** (count + days covered). Computed over the selected timeframe from props.
  - Test (`bp-trend-card.test.tsx`): renders both reference lines and keeps them inside the y-axis domain when readings sit far below them (assert via the computed domain or presence of the 130 reference); renders the four stat tiles.

- [ ] **Step 6: page.tsx.** Owns data fetching, mutations, and layout ONLY (do not let it collect modal bodies — the SOW explicitly calls this out; keep it well under bodyweight's ~1250 lines). Flow top→bottom:
  1. Header — `h1` "Blood Pressure" + one-line subtitle, `border-b px-3 py-4 sm:px-6` (match dashboard/bodyweight).
  2. **Chat bar** — `<CommandBar onSubmit={handleCommand} placeholder='Ask your coach, or log a reading — "122 over 78 this morning"' />` as the FIRST element of the scroll body. `handleCommand` routes to `/chat?prompt=${encodeURIComponent(value)}` (same as dashboard).
  3. Range tabs — 30/60/90/All, the same `RangeKey` shape the bodyweight page uses; `border-b` doubles as separator.
  4. Hero card — `<BpTrendCard>`.
  5. Log toolbar — pencil "Log" button (match bodyweight) opening `<BpLogModal>`.
  6. Reading rail — `<BpReadingsTimeline>`.
  7. About card — `<BpAbout>` at the BOTTOM in the normal state; in the EMPTY state (no readings) promote `<BpAbout>` ABOVE the log CTA (it's the only useful content then).

  Data: `listBloodPressure` on mount + range change; create/update/delete via the `lib/api.ts` wrappers; use the DTO's `category` for stored readings, and `classify()`/`dailyAverages()` from `lib/blood-pressure.ts` for the chart's daily averages and the modal preview. Follow the auth/token + `fetch`+`useState` pattern of the bodyweight page. No ad-hoc `fetch` — only `lib/api.ts`.

- [ ] **Step 7: Run** `npm run typecheck && npm run test && npm run lint && npm run build`. Green.
- [ ] **Step 8: Commit** `feat(blood-pressure): add page, chat bar, chart, log modal, rail, and explainer`.

## Task D4: Dashboard `BloodPressureCard` + dual sparkline

**Files:**
- Modify: `app/(app)/dashboard/_components/spark.tsx` (optional second series) — OR create `dual-spark.tsx`
- Modify: `app/(app)/dashboard/_components/tile-renderer.tsx` (BloodPressureCard + switch case)
- Modify/Add: `tile-renderer.test.tsx` (or a dedicated test) for the empty + populated branches

- [ ] **Step 1: dual sparkline.** Prefer adding an optional second `points2` series (+ `accent2`) to `spark.tsx` (render a second `<polyline>`, normalizing BOTH series against a shared min/max so they read on one scale). If that muddies the component, create a sibling `dual-spark.tsx` — but NOT a copy-paste fork with a second `<path>` bolted on. Add a small test for the two-series render.

- [ ] **Step 2: BloodPressureCard.** In `tile-renderer.tsx`, add a `BloodPressureCard({ section, href })` mirroring `BodyweightCard`: `!section.present` → `<MiniCard title="Blood Pressure" href={href}><MiniCardEmpty cta="Log your first reading" /></MiniCard>`. Present → `<MiniCard>` with `<BigNum value={`${v.latest.systolic}/${v.latest.diastolic}`} />`, the dual sparkline (systolicSpark + diastolicSpark, tinted `--accent` / `--muted`), and a `<MetaRow>` of category (display label) / 30-day average (`sys/dia` or null) / time since last reading. Add the `case "blood_pressure": return <BloodPressureCard section={data.bloodPressure} href={href} />;` to the exhaustive `switch` (the `never` default makes omitting it a compile error). Import `BloodPressureView` from `@/lib/dashboard`.

- [ ] **Step 3: test.** Cover the empty branch (CTA) and the populated branch (renders `122/78`, category, dual spark). Follow `tile-renderer.test.tsx` patterns.

- [ ] **Step 4: Run** `npm run typecheck && npm run test && npm run lint && npm run build`. Green.
- [ ] **Step 5: Commit** `feat(dashboard): add blood-pressure tile card with dual sparkline`.

## Task D5: Web green-gate

- [ ] Run `npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build`. Fix any failure by changing code. If `format:check` fails, run `npm run format` and commit. Never `--no-verify`.

---

# Repo E — prog-strength-docs

## Task E1: Flip SOW status to shipped (done at PR time, see workflow step 4)

- [ ] On branch `feat/blood-pressure`, edit `sows/blood-pressure.md`: frontmatter `status: shipped`; body `**Status**: Shipped`; `**Last updated**: 2026-08-01`. Commit `docs: mark blood-pressure as shipped`. Also commit this plan file.

---

## Local CI Gate per repo (run BEFORE pushing)

- **prog-strength-api:** `go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run` (CI-pinned) · `go vet ./...` · `go mod tidy` (no go.mod/go.sum drift) · `go test ./...`. All green.
- **prog-strength-mcp:** `uv run pytest` · `uv run ruff check src tests`.
- **prog-strength-agent:** `uv run pytest` · `uv run ruff check src tests evals`. (No LLM/eval spend.)
- **prog-strength-web:** `npm run lint` · `npm run format:check` · `npm run typecheck` · `npm run test` · `npm run build`.
- **prog-strength-docs:** no code gate; the docs PR body follows the required template.

## Rollout / merge order (for the docs PR body)

1. **prog-strength-api** — migration 050 + `/blood-pressure` routes + dashboard tile. Merges/deploys FIRST: until the API is live, every MCP/agent/web call 4xxs and the tile has no data source.
2. **prog-strength-mcp** and **prog-strength-agent** — can merge in parallel with each other (the agent calls MCP tools at runtime, not build time), but AFTER the API deploys so the forwarded calls resolve.
3. **prog-strength-web** — last: the page, sidebar entry, tile card, and TS mirror. The Go↔TS catalog contract test only passes once both catalogs carry `blood_pressure`, so web deploys after the API.

## Self-review checklist (run after writing, before executing)

- Spec coverage: every SOW "Goals" bullet maps to a task (log form+agent → A2/B1/C1/D3; timestamped no-limit entries → A1; server classification on DTO → A1/A2; page with chat bar/tabs/hero/tiles/log/rail/explainer → D3; dashboard tile from tray → A3/D4; CommandBar promotion → D1; MCP module + agent intent → B1/C1). ✓
- Non-goals honored: no goal table, no notifications, no device sync, no derived analytics, no chat-bar retrofit to other pages, no mobile, no backfill. ✓
- Classification ladder identical in Go (A1) and TS (D2); both tested against the same boundary table. ✓
- Chart colors mirror CURRENT tokens, not the stale bodyweight module. ✓
- Default dashboard layout unchanged; test asserts `blood_pressure` absent from default. ✓
