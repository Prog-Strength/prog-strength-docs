# Customizable Dashboard Tiles Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the fixed seven-card dashboard into a per-user, reorderable, add/remove tile grid backed by a server-persisted layout, and give walking/cycling/hiking their own tiles.

**Architecture:** API PR first, then web PR. The API grows a Go tile catalog (source of truth for the closed tile-id set), a `user_dashboard_layouts` table + repository, a `PUT /dashboard/layout` write path, three new endurance section builders (walking/cycling/hiking), and makes `GET /dashboard/summary` layout-aware (resolves a layout, returns it, and computes only enabled sections). The web PR mirrors the catalog in TypeScript, adds a `putDashboardLayout` wrapper, renders tiles in layout order through an exhaustive `switch`, adds an edit mode (drag/remove/add via `@dnd-kit`), and deletes the fixed KPI strip.

**Tech Stack:** Go 1.25 + chi + SQLite (api); Next.js 16 + React 19 + TypeScript + Tailwind v4 + Vitest, new dep `@dnd-kit/core` + `@dnd-kit/sortable` (web).

**Repos & branches:** both feature branches are named `feat/customizable-dashboard-tiles`.

**Design system:** `scope: in-system`, conforms to design-system v0.4.4. Introduce **no** new palette/accent/type/token. The three new cards mirror the existing `MiniCard`/`BigNum`/`Spark`/`MetaRow` grammar exactly; endurance cards use `--muted` for their spark (there is no walking/cycling discipline hue — only run/lift/hike are defined; hiking may use `--discipline-hike-dot`). Edit-mode chrome (drag handle, remove button, Customize/Done/Cancel) uses the existing form-control tokens (`--accent`, `--surface-2`, hairline `--border`, `--faint` labels).

---

## Global conventions (read before any task)

**API repo (`/workspace/prog-strength-api`):**
- Module path: `github.com/jwallace145/progressive-overload-fitness-tracker`.
- One type/concept per file in domain packages; comments explain *why* not *what*; no emoji.
- Repository pattern: interface + `SQLiteRepository` with `var _ Repository = (*SQLiteRepository)(nil)` compile assertion; `context.Context` first arg; user-scoped; `errors.Is(err, ErrNotFound)`.
- Handlers use the `httpresp` envelope (`OK`, `Created`, `Error`, `ErrorWithCode`, `ServerError`) — never hand-roll JSON. Auth via `auth.UserIDFrom(ctx)`.
- Tests run against ephemeral SQLite via `internal/db/dbtest` (`dbtest.New(t)`). Migrations live in `internal/db/migrations/`, embedded, forward-only; next number is **049**.
- TDD. After each task run the local gate (see "Local gate" below) before committing where practical, and always before the PR.

**Web repo (`/workspace/prog-strength-web`):**
- All API calls go through `lib/api.ts` (one typed wrapper per endpoint). No ad-hoc `fetch` in components.
- Page-private components live in the route's `_components/`. Tailwind + CSS-variable tokens. Match neighbouring surfaces.
- Vitest + Testing Library, co-located `*.test.ts(x)`. New logic arrives with a test.
- Conventional commits enforced by husky commit-msg; husky pre-commit runs lint-staged + typecheck. **Never** `--no-verify`.

**Local gate — API** (run from repo root, must be clean before push):
```
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run --timeout=5m
go vet ./...
go mod tidy && git diff --exit-code go.mod go.sum
go test ./...
```
(golangci-lint v2.12.2 is already on PATH; `golangci-lint run` also works. The pre-push hook is armed.)

**Local gate — Web:**
```
npm run typecheck && npm run lint && npm run format:check && npm run test && npm run build
```

**The tile-id set (closed, 10 ids), in catalog / tray order:**
`running, walking, cycling, hiking, lifting, steps, nutrition, bodyweight, recovery, streak`

**Default layout order** (a user with no stored row), reproduces today's page:
`running, lifting, steps, nutrition, bodyweight, [recovery,] streak` — `recovery` present **only** when the user has a connected Whoop connection.

---

# PART 1 — API PR (`prog-strength-api`, branch `feat/customizable-dashboard-tiles`)

Create the branch first: `git checkout -b feat/customizable-dashboard-tiles`.

## Task A1: Tile catalog + enum contract test

**Files:**
- Create: `internal/dashboard/tiles.go`
- Test: `internal/dashboard/tiles_test.go`

The catalog is the Go source of truth for the closed tile-id set and fixes the tray order. Tile ids are deliberately the **existing summary section keys** (`lifting`, not `strength_training`).

- [ ] **Step 1: Write the failing test** (`internal/dashboard/tiles_test.go`)

```go
package dashboard

import "testing"

func TestCatalog_EveryConstantAppearsExactlyOnce(t *testing.T) {
	all := []TileID{
		TileRunning, TileWalking, TileCycling, TileHiking, TileLifting,
		TileSteps, TileNutrition, TileBodyweight, TileRecovery, TileStreak,
	}
	if len(Catalog) != len(all) {
		t.Fatalf("Catalog has %d entries, expected %d", len(Catalog), len(all))
	}
	seen := map[TileID]int{}
	for _, id := range Catalog {
		seen[id]++
	}
	for _, id := range all {
		if seen[id] != 1 {
			t.Errorf("tile %q appears %d times in Catalog, want exactly 1", id, seen[id])
		}
	}
}

func TestCatalog_Order(t *testing.T) {
	want := []TileID{
		TileRunning, TileWalking, TileCycling, TileHiking, TileLifting,
		TileSteps, TileNutrition, TileBodyweight, TileRecovery, TileStreak,
	}
	for i := range want {
		if Catalog[i] != want[i] {
			t.Errorf("Catalog[%d] = %q, want %q", i, Catalog[i], want[i])
		}
	}
}

func TestValidTileID(t *testing.T) {
	if !ValidTileID("running") {
		t.Error("running should be valid")
	}
	if ValidTileID("strength_training") {
		t.Error("strength_training is not a tile id")
	}
	if ValidTileID("") {
		t.Error("empty is not valid")
	}
}
```

- [ ] **Step 2: Run test, verify it fails to compile** (`go test ./internal/dashboard/ -run TestCatalog`) — expect "undefined: TileID".

- [ ] **Step 3: Implement** (`internal/dashboard/tiles.go`)

```go
package dashboard

// TileID is the closed set of dashboard tile identifiers. Ids are the existing
// summary section keys (lifting, not strength_training) so no field renaming is
// needed. The Go catalog is the source of truth for the set — there is no SQL
// CHECK on stored layouts (see migration 049); the write path validates and the
// read path filters, mirroring migration 042's treatment of activity_type.
type TileID string

const (
	TileRunning    TileID = "running"
	TileWalking    TileID = "walking"
	TileCycling    TileID = "cycling"
	TileHiking     TileID = "hiking"
	TileLifting    TileID = "lifting"
	TileSteps      TileID = "steps"
	TileNutrition  TileID = "nutrition"
	TileBodyweight TileID = "bodyweight"
	TileRecovery   TileID = "recovery"
	TileStreak     TileID = "streak"
)

// Catalog is the ordered set of every tile. Order fixes how tiles appear in the
// web add-tile tray. The contract test (tiles_test.go) and the TS mirror
// (lib/dashboard-tiles.ts) assert this list stays identical across the boundary.
var Catalog = []TileID{
	TileRunning, TileWalking, TileCycling, TileHiking, TileLifting,
	TileSteps, TileNutrition, TileBodyweight, TileRecovery, TileStreak,
}

var catalogSet = func() map[TileID]bool {
	m := make(map[TileID]bool, len(Catalog))
	for _, id := range Catalog {
		m[id] = true
	}
	return m
}()

// ValidTileID reports whether id is a known tile.
func ValidTileID(id TileID) bool { return catalogSet[id] }
```

- [ ] **Step 4: Run** `go test ./internal/dashboard/ -run 'TestCatalog|TestValidTileID'` — expect PASS.
- [ ] **Step 5: Commit** — `feat(dashboard): add tile catalog and id enum`

---

## Task A2: Migration 049 + layout repository

**Files:**
- Create: `internal/db/migrations/049_dashboard_layout.sql`
- Create: `internal/dashboard/layout.go` (types: `Layout`, `Repository` interface, `ErrNotFound`)
- Create: `internal/dashboard/layout_store.go` (`SQLiteRepository`)
- Test: `internal/dashboard/layout_store_test.go`

The table holds one JSON-array-of-tile-ids row per user. No index beyond the PK; no CHECK on ids. Read filters unknown/retired ids out; write path (Task A3) validates.

- [ ] **Step 1: Write the migration** (`049_dashboard_layout.sql`)

```sql
-- migrations/049_dashboard_layout.sql
-- Per-user dashboard layout: one ordered JSON array of enabled tile ids per
-- user. No row means "never customized" -> the read path resolves the default
-- layout, reproducing today's dashboard. See sows/customizable-dashboard-tiles.md.
--
-- tile_ids carries NO CHECK constraint. Consistent with migration 042's
-- treatment of activity_type, the Go catalog (internal/dashboard/tiles.go) is
-- the source of truth for the closed set: the write path validates and the read
-- path filters unknown ids. Retiring a tile therefore never needs a migration
-- and never breaks a stored layout.
--
-- Keyed by user_id (PK); every read and write is by user_id, so no other index.
-- ON DELETE CASCADE drops a user's layout when the user is hard-deleted.

CREATE TABLE user_dashboard_layouts (
    user_id    TEXT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    tile_ids   TEXT NOT NULL,   -- JSON array of tile ids, in display order
    updated_at TIMESTAMP NOT NULL
);
```

- [ ] **Step 2: Write the failing repo test** (`layout_store_test.go`). Use `dbtest.New(t)`. Mirror `internal/steps/sqlite_repository_test.go` for shape (find it and follow its helper style). Cover: round-trip (`Upsert` then `Get` returns same ordered ids); upsert overwrites (second `Upsert` replaces the array, `updated_at` bumped); unknown-id filtering on read (insert a raw row via `db.Exec` containing `["running","bogus_tile","steps"]`, assert `Get` returns `["running","steps"]`); `ErrNotFound` when no row; empty array round-trips as empty (non-nil) slice; `ON DELETE CASCADE` (insert a user + layout, delete the user, assert `Get` → `ErrNotFound`).

```go
package dashboard

import (
	"context"
	"errors"
	"testing"

	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/db/dbtest"
)

func TestLayoutRepo_RoundTrip(t *testing.T) {
	db := dbtest.New(t)
	repo := NewSQLiteLayoutRepository(db)
	ctx := context.Background()
	mustInsertUser(t, db, "u1")

	want := []TileID{TileRunning, TileHiking, TileSteps, TileStreak}
	if err := repo.Upsert(ctx, "u1", want); err != nil {
		t.Fatalf("upsert: %v", err)
	}
	got, err := repo.Get(ctx, "u1")
	if err != nil {
		t.Fatalf("get: %v", err)
	}
	assertTileIDs(t, got.TileIDs, want)
}

func TestLayoutRepo_UpsertOverwrites(t *testing.T) { /* upsert twice, assert second wins */ }

func TestLayoutRepo_FiltersUnknownOnRead(t *testing.T) {
	db := dbtest.New(t)
	repo := NewSQLiteLayoutRepository(db)
	ctx := context.Background()
	mustInsertUser(t, db, "u1")
	_, err := db.ExecContext(ctx,
		`INSERT INTO user_dashboard_layouts (user_id, tile_ids, updated_at) VALUES (?, ?, ?)`,
		"u1", `["running","bogus_tile","steps"]`, "2026-07-31T00:00:00Z")
	if err != nil {
		t.Fatal(err)
	}
	got, err := repo.Get(ctx, "u1")
	if err != nil {
		t.Fatalf("get: %v", err)
	}
	assertTileIDs(t, got.TileIDs, []TileID{TileRunning, TileSteps})
}

func TestLayoutRepo_NotFound(t *testing.T) {
	db := dbtest.New(t)
	repo := NewSQLiteLayoutRepository(db)
	if _, err := repo.Get(context.Background(), "nobody"); !errors.Is(err, ErrLayoutNotFound) {
		t.Fatalf("want ErrLayoutNotFound, got %v", err)
	}
}

func TestLayoutRepo_CascadeOnUserDelete(t *testing.T) { /* insert user+layout, delete user, expect ErrLayoutNotFound */ }
```

Add small test helpers `mustInsertUser(t, db, id)` (insert into `users` the minimal NOT NULL columns — inspect the `users` table via an existing migration/other test to get required columns; PRAGMA foreign_keys must be ON for the cascade test — `dbtest.New` config; check how other cascade tests do it) and `assertTileIDs(t, got, want)`.

- [ ] **Step 3: Run test, verify it fails** — expect "undefined: NewSQLiteLayoutRepository".

- [ ] **Step 4: Implement types** (`layout.go`)

```go
package dashboard

import (
	"context"
	"errors"
)

// ErrLayoutNotFound is returned by Repository.Get when the user has no stored
// layout row (they have never customized). Callers resolve the default layout.
var ErrLayoutNotFound = errors.New("dashboard: layout not found")

// Layout is a user's persisted dashboard layout: the ordered set of enabled
// tile ids. On read, unknown/retired ids are filtered out (the catalog is the
// source of truth), so TileIDs contains only currently-valid ids.
type Layout struct {
	TileIDs []TileID
}

// Repository persists one dashboard layout per user, keyed by user_id.
type Repository interface {
	// Get returns the stored layout, filtered to currently-valid tile ids.
	// ErrLayoutNotFound when the user has no row.
	Get(ctx context.Context, userID string) (Layout, error)
	// Upsert writes the ordered tile ids for the user (insert or replace).
	Upsert(ctx context.Context, userID string, tileIDs []TileID) error
}
```

- [ ] **Step 5: Implement store** (`layout_store.go`). Marshal `[]TileID` to a JSON array with `encoding/json`; on read, unmarshal to `[]string`, drop any id where `!ValidTileID`, preserve order, always return a non-nil slice (empty slice for `[]`).

```go
package dashboard

import (
	"context"
	"database/sql"
	"encoding/json"
	"errors"
	"fmt"
	"time"
)

var _ Repository = (*SQLiteRepository)(nil)

type SQLiteRepository struct {
	db  *sql.DB
	now func() time.Time
}

func NewSQLiteLayoutRepository(db *sql.DB) *SQLiteRepository {
	return &SQLiteRepository{db: db, now: time.Now}
}

func (r *SQLiteRepository) Get(ctx context.Context, userID string) (Layout, error) {
	var raw string
	err := r.db.QueryRowContext(ctx,
		`SELECT tile_ids FROM user_dashboard_layouts WHERE user_id = ?`, userID).Scan(&raw)
	if errors.Is(err, sql.ErrNoRows) {
		return Layout{}, ErrLayoutNotFound
	}
	if err != nil {
		return Layout{}, fmt.Errorf("dashboard: get layout: %w", err)
	}
	var ids []string
	if err := json.Unmarshal([]byte(raw), &ids); err != nil {
		return Layout{}, fmt.Errorf("dashboard: decode layout: %w", err)
	}
	out := make([]TileID, 0, len(ids))
	for _, id := range ids {
		if ValidTileID(TileID(id)) {
			out = append(out, TileID(id))
		}
	}
	return Layout{TileIDs: out}, nil
}

func (r *SQLiteRepository) Upsert(ctx context.Context, userID string, tileIDs []TileID) error {
	if tileIDs == nil {
		tileIDs = []TileID{}
	}
	blob, err := json.Marshal(tileIDs)
	if err != nil {
		return fmt.Errorf("dashboard: encode layout: %w", err)
	}
	_, err = r.db.ExecContext(ctx, `
		INSERT INTO user_dashboard_layouts (user_id, tile_ids, updated_at)
		VALUES (?, ?, ?)
		ON CONFLICT(user_id) DO UPDATE SET
			tile_ids   = excluded.tile_ids,
			updated_at = excluded.updated_at
	`, userID, string(blob), r.now().UTC())
	if err != nil {
		return fmt.Errorf("dashboard: upsert layout: %w", err)
	}
	return nil
}
```

- [ ] **Step 6: Run** `go test ./internal/dashboard/` — expect PASS.
- [ ] **Step 7: Commit** — `feat(dashboard): add user_dashboard_layouts table and repository`

---

## Task A3: Layout write handler — `PUT /dashboard/layout`

**Files:**
- Modify: `internal/dashboard/handler.go` (add `layoutRepo Repository` field + constructor param; add route in `Mount`; add `putLayout` handler)
- Test: `internal/dashboard/layout_handler_test.go`

Body: `{ "tile_ids": ["running", ...] }`. Auth required. Valid ids, no dups → upsert, `204 No Content`. Unknown id → `422` listing valid ids. Duplicate id → `422`. Empty array → accepted (`204`).

- [ ] **Step 1: Write failing handler tests** (`layout_handler_test.go`). Wire a real `dashboard.Handler` over `dbtest.New(t)` with a real layout repo (other section repos can be constructed real or nil — `putLayout` only touches `layoutRepo` and auth). Use `httptest` + an auth context helper. Find how existing dashboard/steps handler tests inject `auth.UserIDFrom` context (look for `authctx` usage in `internal/activity/contract_test.go` and steps handler tests) and mirror it. Cases:
  - `PUT` with `{"tile_ids":["running","steps"]}` → 204, and a subsequent repo `Get` returns those ids.
  - `{"tile_ids":["running","bogus"]}` → 422, body mentions valid ids.
  - `{"tile_ids":["running","running"]}` → 422 (duplicate).
  - `{"tile_ids":[]}` → 204, repo `Get` returns empty slice.
  - no auth context → 500 (missing user) or the pattern the summary handler uses.
- [ ] **Step 2: Run, verify fail.**
- [ ] **Step 3: Implement.** Add to constructor + struct (see Task A6 for the full wiring change; for this task add the field and a new param to `NewHandler`). Register `r.Put("/dashboard/layout", h.putLayout)` in `Mount`. Handler:

```go
func (h *Handler) putLayout(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	userID, ok := auth.UserIDFrom(ctx)
	if !ok {
		httpresp.ServerError(w, ctx, "missing user in context", errors.New("auth middleware not applied"))
		return
	}
	var req struct {
		TileIDs []TileID `json:"tile_ids"`
	}
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		httpresp.Error(w, http.StatusBadRequest, "invalid request body")
		return
	}
	if msg, ok := validateTileIDs(req.TileIDs); !ok {
		httpresp.Error(w, http.StatusUnprocessableEntity, msg)
		return
	}
	if err := h.layoutRepo.Upsert(ctx, userID, req.TileIDs); err != nil {
		httpresp.ServerError(w, ctx, "upsert dashboard layout", err)
		return
	}
	w.WriteHeader(http.StatusNoContent)
}

// validateTileIDs rejects unknown ids and duplicates. An empty (or nil) list is
// valid — an empty dashboard is a legitimate preference.
func validateTileIDs(ids []TileID) (string, bool) {
	seen := make(map[TileID]bool, len(ids))
	for _, id := range ids {
		if !ValidTileID(id) {
			return "unknown tile id " + string(id) + "; valid ids: " + validIDList(), false
		}
		if seen[id] {
			return "duplicate tile id " + string(id), false
		}
		seen[id] = true
	}
	return "", true
}

func validIDList() string {
	parts := make([]string, len(Catalog))
	for i, id := range Catalog {
		parts[i] = string(id)
	}
	return strings.Join(parts, ", ")
}
```

Add `encoding/json`, `strings`, `net/http` imports as needed. Verify `httpresp.Error(w, code, msg)` is the right signature (it is used as `httpresp.Error(w, http.StatusBadRequest, "...")` in the summary handler).

- [ ] **Step 4: Run** `go test ./internal/dashboard/` — PASS.
- [ ] **Step 5: Commit** — `feat(dashboard): add PUT /dashboard/layout write path`

---

## Task A4: Endurance section builders (walking, cycling, hiking)

**Files:**
- Create: `internal/dashboard/endurance_tiles.go` (shared value types + weekly-bucketing helper)
- Create: `internal/dashboard/walking.go`, `cycling.go`, `hiking.go`
- Add DTOs to `internal/dashboard/dto.go`
- Test: `internal/dashboard/walking_test.go`, `cycling_test.go`, `hiking_test.go` (mirror `running_test.go`)

Pure builders over the already-fetched endurance slice, filtered by type. `nil`-on-empty contract: no session of that type ever → `nil` section. Distances render meters (client converts to `distance_unit`, like running). Each carries an 8-week weekly-distance spark. **None** carries a delta.

Per SOW table:
| Tile | Headline | Body | Meta row | Deep link |
| Walking | distance this week | 8-week weekly-distance spark | walks · time · last | `/activities` |
| Cycling | distance this week | 8-week weekly-distance spark | rides · time · last | `/activities` |
| Hiking | distance this week | 8-week weekly-distance spark | hikes · elevation gain · time · last | `/hiking` |

The deep-link href lives in the **web** catalog (Task W1), not the API. The API DTO carries the numbers only.

- [ ] **Step 1: Add DTOs** (`dto.go`). All three share a shape; hiking adds elevation. Keep one type per concept but avoid premature abstraction — three explicit DTOs mirroring `RunningSection`:

```go
// WalkingSection is the walking tile. nil at the Summary level when the user has
// no walking activity at all.
type WalkingSection struct {
	CurrentWeek         EnduranceCurrentWeek `json:"current_week"`
	LatestSession       *EnduranceLatest     `json:"latest_session"`
	WeeklyDistanceSpark []float64            `json:"weekly_distance_spark"`
}

// CyclingSection is the cycling tile. nil when the user has no cycling activity.
type CyclingSection struct {
	CurrentWeek         EnduranceCurrentWeek `json:"current_week"`
	LatestSession       *EnduranceLatest     `json:"latest_session"`
	WeeklyDistanceSpark []float64            `json:"weekly_distance_spark"`
}

// HikingSection is the hiking tile. nil when the user has no hiking activity.
// ElevationGainMeters is this week's summed elevation gain (Activity.ElevationGainMeters).
type HikingSection struct {
	CurrentWeek         EnduranceCurrentWeek `json:"current_week"`
	ElevationGainMeters float64              `json:"elevation_gain_meters"`
	LatestSession       *EnduranceLatest     `json:"latest_session"`
	WeeklyDistanceSpark []float64            `json:"weekly_distance_spark"`
}

// EnduranceCurrentWeek is the shared this-week rollup for the endurance tiles.
type EnduranceCurrentWeek struct {
	DistanceMeters  float64 `json:"distance_meters"`
	SessionCount    int     `json:"session_count"`
	DurationSeconds int     `json:"duration_seconds"`
}

// EnduranceLatest is a thin projection of the most recent session of a type.
type EnduranceLatest struct {
	Name            *string   `json:"name"`
	DistanceMeters  float64   `json:"distance_meters"`
	DurationSeconds int       `json:"duration_seconds"`
	StartTime       time.Time `json:"start_time"`
}
```

- [ ] **Step 2: Write `endurance_tiles.go`** — the shared helper so the DST/timezone arithmetic exists once. Reuse the existing `weeklyBucketStarts`, `localWeekStart`, `weekIndex`, `sparkWeeks` already in the package (buckets.go / running.go). Add a shared computation:

```go
package dashboard

import (
	"time"

	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/activity"
)

// enduranceRollup is the shared computation behind the walking/cycling/hiking
// builders: current-week distance/count/duration and the 8-week weekly-distance
// spark, over the sessions of a single activity type. Pure: now and loc are
// passed in so bucketing is deterministic across timezones and DST.
type enduranceRollup struct {
	current EnduranceCurrentWeek
	spark   []float64
	latest  *EnduranceLatest
	count   int // total sessions of this type in the window (for nil-on-empty)
}

func computeEnduranceRollup(sessions []activity.Activity, t activity.ActivityType, now time.Time, loc *time.Location) enduranceRollup {
	if loc == nil {
		loc = time.UTC
	}
	starts := weeklyBucketStarts(now, loc, sparkWeeks)
	oldest := starts[0]
	current := localWeekStart(now, loc)
	spark := make([]float64, len(starts))
	var roll enduranceRollup
	roll.spark = spark
	var latest *activity.Activity
	for i := range sessions {
		if sessions[i].ActivityType != t {
			continue
		}
		roll.count++
		ws := localWeekStart(sessions[i].StartTime, loc)
		if ws.Equal(current) {
			roll.current.DistanceMeters += sessions[i].DistanceMeters
			roll.current.SessionCount++
			roll.current.DurationSeconds += sessions[i].DurationSeconds
		}
		if !ws.Before(oldest) {
			if idx := weekIndex(starts, ws); idx >= 0 {
				spark[idx] += sessions[i].DistanceMeters
			}
		}
		if latest == nil || sessions[i].StartTime.After(latest.StartTime) {
			latest = &sessions[i]
		}
	}
	if latest != nil {
		roll.latest = &EnduranceLatest{
			Name:            latest.Name,
			DistanceMeters:  latest.DistanceMeters,
			DurationSeconds: latest.DurationSeconds,
			StartTime:       latest.StartTime,
		}
	}
	return roll
}
```

- [ ] **Step 3: Write the three builders.** Each is a thin wrapper. `walking.go`:

```go
package dashboard

import (
	"time"

	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/activity"
)

// buildWalking assembles the walking tile from the already-fetched endurance
// slice. Pure; nil when the user has no walking session at all.
func buildWalking(sessions []activity.Activity, now time.Time, loc *time.Location) *WalkingSection {
	roll := computeEnduranceRollup(sessions, activity.ActivityWalking, now, loc)
	if roll.count == 0 {
		return nil
	}
	return &WalkingSection{
		CurrentWeek:         roll.current,
		LatestSession:       roll.latest,
		WeeklyDistanceSpark: roll.spark,
	}
}
```

`cycling.go` is identical with `activity.ActivityCycling` and `CyclingSection`. `hiking.go` adds elevation — sum `ElevationGainMeters` (nullable `*float64`, skip nil) for **current-week** hikes:

```go
// buildHiking assembles the hiking tile. Like walking/cycling plus this week's
// summed elevation gain from Activity.ElevationGainMeters (nil gains skipped).
func buildHiking(sessions []activity.Activity, now time.Time, loc *time.Location) *HikingSection {
	roll := computeEnduranceRollup(sessions, activity.ActivityHiking, now, loc)
	if roll.count == 0 {
		return nil
	}
	if loc == nil {
		loc = time.UTC
	}
	current := localWeekStart(now, loc)
	var gain float64
	for i := range sessions {
		if sessions[i].ActivityType != activity.ActivityHiking {
			continue
		}
		if !localWeekStart(sessions[i].StartTime, loc).Equal(current) {
			continue
		}
		if sessions[i].ElevationGainMeters != nil {
			gain += *sessions[i].ElevationGainMeters
		}
	}
	return &HikingSection{
		CurrentWeek:         roll.current,
		ElevationGainMeters: gain,
		LatestSession:       roll.latest,
		WeeklyDistanceSpark: roll.spark,
	}
}
```

- [ ] **Step 4: Write builder tests** mirroring `running_test.go` (use its `mustLoad`, `ptrF`, `ptrS` helpers already in the package). For each type cover: empty → nil; nil-on-empty when only *other* types present; pass-through of current-week distance/count/duration; latest = max StartTime; weekly spark bucketing including a cross-timezone/DST case (mirror `running_test.go`'s DST test, e.g. `America/Denver` around a spring-forward boundary); hiking elevation gain sums only current-week and skips nil gains. Add a walk/cycle/hike constructor helper analogous to `run(...)`.

- [ ] **Step 5: Run** `go test ./internal/dashboard/` — PASS.
- [ ] **Step 6: Commit** — `feat(dashboard): add walking, cycling, hiking section builders`

---

## Task A5: Make `GET /dashboard/summary` layout-aware

**Files:**
- Modify: `internal/dashboard/handler.go` (`summary` handler; new helpers `resolveLayout`, `defaultLayout`)
- Modify: `internal/dashboard/dto.go` if needed (see note on response shape)
- Test: `internal/dashboard/handler_test.go` (extend) or a new `summary_layout_test.go`

**Behaviour:**
1. Resolve the layout: `layoutRepo.Get(userID)`. On `ErrLayoutNotFound` → default layout. On any **other** error → **log and fall back to default** (never 500) — same resilience principle as `defer1`.
2. The default layout is `running, lifting, steps, nutrition, bodyweight, [recovery,] streak`, with `recovery` present only when the user has a **connected** Whoop connection. Read the Whoop connection once and reuse it for both the default decision and the recovery section.
3. Compute **only enabled sections**. Two reads stay **ungated regardless of layout**: the 53-week `activityRepo.ListInRange` over all types (feeds running/walking/cycling/hiking, strength hydration, and the streak) and the step entries + step goal (the streak credits goal-meeting step days, so disabling Steps must not change the streak).
4. Strength hydration (`hydrateStrengthWorkouts`) runs when lifting **or** streak is enabled (the streak's completion path needs workouts). Gating saves the lifting PR/headline reads, the nutrition/bodyweight/recovery reads, and builder CPU on the endurance tiles.
5. Response: return `layout` (the resolved ordered ids) plus a key per **enabled** tile. A tile enabled but with no data serializes as `null`; a tile not in the layout is **absent** from the response. Because JSON structs can't express "present-null vs absent" via `omitempty`, build the payload as a `map[string]any`.

- [ ] **Step 1: Write failing tests.** Extend the existing handler test harness (find how `handler_test.go` builds a `Handler` with fakes/real repos and pins `now`). Cases:
  - **Default (no row), no Whoop:** response `layout` == `["running","lifting","steps","nutrition","bodyweight","streak"]` (no recovery); keys present exactly for enabled tiles; `recovery` key absent.
  - **Default (no row), connected Whoop:** `layout` includes `recovery` before `streak`; `recovery` key present.
  - **Stored layout gates sections:** store `["running","streak"]`; assert response `layout` == that; `nutrition`/`bodyweight`/`lifting`/`steps`/`walking`/... keys **absent**; `running` present; `streak` present.
  - **Streak unchanged when Steps disabled (the shared-read trap):** with a step day meeting goal that lights the streak, store a layout WITHOUT `steps` but WITH `streak`; assert the streak value is identical to the streak computed when `steps` IS in the layout. (Two subtests comparing the `streak` object.)
  - **Layout-read failure falls back to default, not 500:** inject a layout repo whose `Get` returns a non-NotFound error; assert HTTP 200 and `layout` == default. (Introduce a tiny test double implementing `Repository`; put it in the test file.)
  - **Enabled-but-nodata → null:** enable `nutrition` for a user with no nutrition rows; assert the `nutrition` key is present and JSON `null`.

- [ ] **Step 2: Run, verify fail.**

- [ ] **Step 3: Implement.** Rewrite the tail of `summary` (from the `summary := Summary{...}` assembly onward) to gate and to emit a map. Sketch:

```go
	layout := h.resolveLayout(ctx, r, userID, now, loc) // []TileID, already Whoop-aware for the default
	enabled := make(map[TileID]bool, len(layout))
	for _, id := range layout {
		enabled[id] = true
	}

	// Ungated shared reads (feed the streak and endurance tiles regardless of layout).
	sessions := defer1(...) // unchanged 53-week activity list
	endurance := enduranceOnly(sessions)
	stepEntries := defer1(...) // unchanged
	stepGoal := defer1(...)    // unchanged

	var workouts []strength.Workout
	if enabled[TileLifting] || enabled[TileStreak] {
		workouts = defer1(ctx, r, "workout exercises", func() ([]strength.Workout, error) {
			return h.hydrateStrengthWorkouts(ctx, sessions)
		})
	}

	out := map[string]any{"layout": layout}
	if enabled[TileRunning] {
		out["running"] = h.buildRunningSection(ctx, r, userID, endurance, now, loc)
	}
	if enabled[TileWalking] {
		out["walking"] = buildWalking(endurance, now, loc)
	}
	if enabled[TileCycling] {
		out["cycling"] = buildCycling(endurance, now, loc)
	}
	if enabled[TileHiking] {
		out["hiking"] = buildHiking(endurance, now, loc)
	}
	if enabled[TileLifting] {
		out["lifting"] = h.buildLiftingSection(ctx, r, userID, workouts, unit, now, loc)
	}
	if enabled[TileSteps] {
		out["steps"] = h.buildStepsSection(ctx, r, userID, stepEntries, now, loc)
	}
	if enabled[TileNutrition] {
		out["nutrition"] = h.buildNutritionSection(ctx, r, userID, todayStr, loc)
	}
	if enabled[TileBodyweight] {
		out["bodyweight"] = h.buildBodyweightSection(ctx, r, userID, since8w)
	}
	if enabled[TileRecovery] {
		out["recovery"] = h.buildRecoverySection(ctx, r, userID, now, loc)
	}
	if enabled[TileStreak] {
		out["streak"] = buildStreak(streakDates(endurance, workouts, stepEntries, stepGoal.Goal, loc), now, loc)
	}

	httpresp.OK(w, "dashboard summary", out)
```

IMPORTANT nil-pointer marshaling caveat: `out["running"] = (*RunningSection)(nil)` — a nil pointer stored in an `any` marshals to JSON `null`, which is what we want ("enabled but no data → null"). Confirm the build functions return typed pointers (they do), so this holds.

`resolveLayout` + `defaultLayout`:

```go
// resolveLayout returns the user's stored layout, or the default when none is
// stored. A layout-read failure (other than not-found) is logged and degrades
// to the default rather than failing the request — one flaky table can never
// blank the dashboard, the same principle as the per-section defer1 reads.
func (h *Handler) resolveLayout(ctx context.Context, r *http.Request, userID string, now time.Time, loc *time.Location) []TileID {
	l, err := h.layoutRepo.Get(ctx, userID)
	if err == nil {
		return l.TileIDs
	}
	if !errors.Is(err, ErrLayoutNotFound) {
		log.Printf("dashboard: layout for %s: %v", requestid.FromContext(r.Context()), err)
	}
	return h.defaultLayout(ctx, r, userID)
}

// defaultLayout reproduces today's dashboard: running, lifting, steps,
// nutrition, bodyweight, [recovery,] streak. Recovery is included only when the
// user has a connected Whoop connection (a non-Whoop user should not land on an
// empty Recovery card they never asked for).
func (h *Handler) defaultLayout(ctx context.Context, r *http.Request, userID string) []TileID {
	ids := []TileID{TileRunning, TileLifting, TileSteps, TileNutrition, TileBodyweight}
	if h.hasConnectedWhoop(ctx, r, userID) {
		ids = append(ids, TileRecovery)
	}
	return append(ids, TileStreak)
}

// hasConnectedWhoop reports whether the user has a CONNECTED Whoop connection.
// A missing/errored connection reads as false (the recovery tile stays out of
// the default, and buildRecoverySection independently returns nil).
func (h *Handler) hasConnectedWhoop(ctx context.Context, r *http.Request, userID string) bool {
	conn, err := h.whoopConns.Get(ctx, userID)
	if err != nil {
		if !errors.Is(err, whoopconn.ErrNotFound) {
			log.Printf("dashboard: whoop connection for %s: %v", requestid.FromContext(r.Context()), err)
		}
		return false
	}
	return conn.Status == whoopconn.StatusConnected
}
```

Note the double Whoop read (once for default decision, once in `buildRecoverySection`) is acceptable — it only happens on the default path and `buildRecoverySection`'s own guard is unchanged. Do NOT try to thread the connection through; keep the change small. (If the reviewer objects to the extra read, thread a `*whoopconn.Connection` — but only if asked.)

Remove the now-unused `Summary`, `RunningSection` etc.? **No** — the section DTOs are still used as the map values and by the web contract. Only the top-level `Summary` struct assembly is replaced by the map. Keep the `Summary` struct type only if something else references it; otherwise delete it and its doc. Check with `grep -rn "dashboard.Summary\b\|Summary{" internal/` — if unused after this change, delete the `Summary` type to avoid dead code (golangci may flag it). The section types (`RunningSection`, `StreakSection`, …) stay.

- [ ] **Step 4: Run** `go test ./internal/dashboard/` — PASS. Then `go vet ./...`.
- [ ] **Step 5: Commit** — `feat(dashboard): make summary layout-aware and gate section computation`

---

## Task A6: Wire the layout repository into the server

**Files:**
- Modify: `internal/server/server.go` (construct `dashboard.NewSQLiteLayoutRepository(database)`, pass to `dashboard.NewHandler(...)`)
- Modify: `internal/dashboard/handler.go` `NewHandler` signature (add `layoutRepo Repository` as the final param — do this in Task A3 when the field is introduced; A6 only updates the call site)

- [ ] **Step 1:** Update `NewHandler` call in `server.go` (around line 674) to pass the new layout repo. Construct it near the other dashboard-shared repos.
- [ ] **Step 2:** `go build ./...` then the full local gate: golangci-lint, `go vet ./...`, `go mod tidy` (no drift — no new deps expected), `go test ./...`. Fix anything.
- [ ] **Step 3: Commit** — `feat(dashboard): wire layout repository into server`
- [ ] **Step 4:** Run the **contract test** `go test ./internal/activity/ -run Contract` to confirm the "new types come free" invariant still holds (the summary handler now takes an extra constructor arg — update the contract test's handler construction if it builds `dashboard.NewHandler` directly).

---

## API gate + PR

- [ ] Run the full **Local gate — API** from a clean tree; fix all findings (no `//nolint`, no rule disables).
- [ ] Push `feat/customizable-dashboard-tiles`.
- [ ] `gh pr create` against `main`. Title: `feat(dashboard): customizable tile layout + walking/cycling/hiking tiles`. Body: follow the format of recent merged PRs in this repo (`gh pr list --state merged --limit 5` then `gh pr view <n>`); include a summary, the migration note, the new endpoint, and a test plan.

---

# PART 2 — WEB PR (`prog-strength-web`, branch `feat/customizable-dashboard-tiles`)

Create the branch: `git checkout -b feat/customizable-dashboard-tiles`. Depends on the API contract from Part 1 (the `layout` field + new sections), but web work is independent code and can proceed in parallel — it only needs the agreed shapes.

## Task W1: Install `@dnd-kit`, add the TS tile catalog + exhaustiveness test

**Files:**
- Modify: `package.json` / `package-lock.json` (add `@dnd-kit/core`, `@dnd-kit/sortable`)
- Create: `lib/dashboard-tiles.ts`
- Test: `lib/dashboard-tiles.test.ts`

- [ ] **Step 1:** `npm install @dnd-kit/core @dnd-kit/sortable` (pins into package.json). Confirm versions are recent stable. (These are the SOW-sanctioned new deps.)
- [ ] **Step 2: Write failing test** (`dashboard-tiles.test.ts`): assert `TILE_CATALOG` length is 10, ids in the exact order `running, walking, cycling, hiking, lifting, steps, nutrition, bodyweight, recovery, streak`, every entry has non-empty `title`, `href`, `description`, and that `TILE_IDS` (derived) matches. Add a compile-time exhaustiveness guard test: a helper that maps each `TileId` and would fail typecheck if the union and catalog diverge.
- [ ] **Step 3: Implement** (`lib/dashboard-tiles.ts`):

```ts
/**
 * The dashboard tile catalog — the TypeScript mirror of the Go catalog
 * (internal/dashboard/tiles.go). The Go `Catalog` and this `TILE_CATALOG`
 * must stay identical in id set and order; the API contract test and this
 * file's test both guard that. Order fixes the add-tile tray order.
 */

export type TileId =
  | "running"
  | "walking"
  | "cycling"
  | "hiking"
  | "lifting"
  | "steps"
  | "nutrition"
  | "bodyweight"
  | "recovery"
  | "streak";

export type TileCatalogEntry = {
  id: TileId;
  title: string;
  href: string; // deep link into the tile's full page
  description: string; // one-line tray description
};

export const TILE_CATALOG: readonly TileCatalogEntry[] = [
  { id: "running", title: "Running", href: "/activities?view=running", description: "Weekly running distance and pace." },
  { id: "walking", title: "Walking", href: "/activities", description: "Weekly walking distance and time." },
  { id: "cycling", title: "Cycling", href: "/activities", description: "Weekly cycling distance and time." },
  { id: "hiking", title: "Hiking", href: "/hiking", description: "Weekly hiking distance and elevation." },
  { id: "lifting", title: "Lifting", href: "/workouts", description: "This week's lifting volume and PRs." },
  { id: "steps", title: "Steps", href: "/activities?view=steps", description: "Daily steps against your goal." },
  { id: "nutrition", title: "Nutrition", href: "/nutrition", description: "Today's calories and macros." },
  { id: "bodyweight", title: "Bodyweight", href: "/bodyweight", description: "Bodyweight trend and goal." },
  { id: "recovery", title: "Recovery", href: "/recovery", description: "Whoop recovery and resting HR." },
  { id: "streak", title: "Streak", href: "/activities", description: "Your weekly training streak." },
] as const;

export const TILE_IDS: readonly TileId[] = TILE_CATALOG.map((t) => t.id);

const CATALOG_BY_ID: Record<TileId, TileCatalogEntry> = Object.fromEntries(
  TILE_CATALOG.map((t) => [t.id, t]),
) as Record<TileId, TileCatalogEntry>;

export function tileEntry(id: TileId): TileCatalogEntry {
  return CATALOG_BY_ID[id];
}
```

(Confirm the running/steps deep-links match the existing `DEEP_LINKS` map in `page.tsx`; keep them consistent. `/hiking` is the hiking view — verify it exists in the app router; if not, fall back to `/activities`.)

- [ ] **Step 4: Run** `npm run test -- dashboard-tiles` and `npm run typecheck` — PASS.
- [ ] **Step 5: Commit** — `feat(dashboard): add tile catalog and dnd-kit dependency`

---

## Task W2: API wrapper + adapter layout wiring

**Files:**
- Modify: `lib/api.ts` (add `layout` + walking/cycling/hiking to `DashboardSummary`; add `putDashboardLayout`)
- Modify: `lib/dashboard.ts` (`adaptDashboard` returns `layout: TileId[]`; add `WalkingView`/`CyclingView`/`HikingView` + adapters; `DashboardData` gains `layout`, `walking`, `cycling`, `hiking`)
- Test: extend `lib/dashboard.test.ts` and `lib/api.test.ts`

- [ ] **Step 1 (api.ts types):** Add the new section types + extend `DashboardSummary`:

```ts
export type DashboardEnduranceCurrentWeek = {
  distance_meters: number;
  session_count: number;
  duration_seconds: number;
};
export type DashboardEnduranceLatest = {
  name: string | null;
  distance_meters: number;
  duration_seconds: number;
  start_time: string;
} | null;
export type DashboardWalking = {
  current_week: DashboardEnduranceCurrentWeek;
  latest_session: DashboardEnduranceLatest;
  weekly_distance_spark: number[];
};
export type DashboardCycling = DashboardWalking;
export type DashboardHiking = DashboardWalking & { elevation_gain_meters: number };
```

Extend `DashboardSummary`: add `layout: TileId[]` (import `TileId` from `./dashboard-tiles`) and make walking/cycling/hiking/recovery/etc. optional. Because the API now omits disabled keys and sends `null` for enabled-no-data, type each section as `Section | null | undefined` — i.e. `running?: DashboardRunning | null;`. Add `walking?`, `cycling?`, `hiking?`. Keep existing keys but mark optional. `streak` becomes `streak?: DashboardStreak` (present only if enabled) — the adapter must tolerate its absence.

- [ ] **Step 2 (api.ts wrapper):** Add `putDashboardLayout` mirroring `putStepsGoal` (PUT, JSON body `{ tile_ids }`, 204 → no payload; treat a non-2xx as an error via the existing `fetch`/status pattern — `unwrap` expects a body, so for 204 check `resp.ok` directly and throw on failure):

```ts
export async function putDashboardLayout(token: string, tileIds: TileId[]): Promise<void> {
  const resp = await fetch(`${config.apiUrl}/dashboard/layout`, {
    method: "PUT",
    headers: { "Content-Type": "application/json", Authorization: `Bearer ${token}` },
    body: JSON.stringify({ tile_ids: tileIds }),
  });
  if (!resp.ok) {
    throw new Error(`PUT /dashboard/layout failed: ${resp.status}`);
  }
}
```

- [ ] **Step 3 (dashboard.ts):** Add view models + adapters for walking/cycling/hiking (mirror `RunningView`/`adaptRunning`; distances → display unit via `formatDistanceValue`/`distanceToDisplay`; endurance cards carry `unit: DistanceUnit`). Add `walking`, `cycling`, `hiking` to `DashboardData` as `Section<...>`, and add `layout: TileId[]`. `adaptDashboard` reads `summary.layout ?? []`, adapts each **present-and-non-null** section to `{present:true,...}` else `{present:false}`, and for streak uses `summary.streak ? adaptStreak(...) : {weeks:0,...,isNew:true}`. The null-summary branch returns `layout: []` and all sections absent.

Design note: the page renders strictly from `layout`; `DashboardData` sections still exist so the renderer can look each up. Keep the `Section<T>` discriminated-union pattern.

- [ ] **Step 4: Tests.** Extend `dashboard.test.ts`: a summary with `layout:["running","hiking","steps","streak"]`, `hiking` populated, `steps:null` → adapted `data.layout` equals the array, `data.hiking.present` true, `data.steps.present` false, `data.running.present` false (absent key). Add a walking/cycling/hiking distance-conversion assertion (mi vs km). Extend `api.test.ts` for `putDashboardLayout` (mock fetch: 204 resolves; 422/500 throws).
- [ ] **Step 5: Run** `npm run test -- dashboard` `npm run typecheck`.
- [ ] **Step 6: Commit** — `feat(dashboard): adapter + api wrapper for layout and endurance tiles`

---

## Task W3: Pure reorder / add / remove functions

**Files:**
- Create: `app/(app)/dashboard/_components/layout-ops.ts`
- Test: `app/(app)/dashboard/_components/layout-ops.test.ts`

Reorder/add/remove are pure functions over `TileId[]`, unit-tested independently of the DnD library.

- [ ] **Step 1: Write failing tests.** `reorderTiles(ids, fromId, toId)` moves `fromId` to `toId`'s position preserving the rest; `addTile(ids, id)` appends if absent (no-op if present); `removeTile(ids, id)` drops it; `availableTiles(ids)` returns catalog entries whose id is not in `ids`, in catalog order. Cover edge cases (move to same position; add duplicate; remove absent).
- [ ] **Step 2: Implement** `layout-ops.ts` (use `arrayMove` semantics by index, or a small hand-rolled splice; keep DnD-agnostic — take ids, not dnd events). Provide `availableTiles` using `TILE_CATALOG`.
- [ ] **Step 3: Run** tests — PASS.
- [ ] **Step 4: Commit** — `feat(dashboard): pure reorder/add/remove layout operations`

---

## Task W4: The three new cards + Recovery empty state

**Files:**
- Create: `app/(app)/dashboard/_components/walking-card.tsx`, `cycling-card.tsx`, `hiking-card.tsx`
- Modify: `app/(app)/dashboard/_components/whoop-card.tsx` (add an empty-state variant for enabled-but-not-connected)

Mirror the existing `RunningCard` structure but as standalone components taking their view + href. Use `MiniCard`/`BigNum`/`Spark`/`MetaRow`. Spark colour: walking/cycling use `text-[var(--muted)]`; hiking may use `text-[var(--discipline-hike-dot)]`. Empty state uses `MiniCardEmpty` with per-type CTA ("Log a walk to start tracking", "Log a ride…", "Log a hike…"). Hiking meta row: hikes · elevation gain · time · last (render elevation gain via a shared meters→display formatter; check `lib/format.ts`/`hiking-stats.ts` for an existing elevation formatter and reuse it).

- [ ] **Step 1:** Write `walking-card.tsx` (component `WalkingCard({ section, href })` where `section: Section<WalkingView>`), mirroring `RunningCard` in `page.tsx`. Meta row: walks · time · last.
- [ ] **Step 2:** `cycling-card.tsx` (rides · time · last).
- [ ] **Step 3:** `hiking-card.tsx` (hikes · elevation gain · time · last).
- [ ] **Step 4:** Recovery empty state. The recovery tile can now be deliberately enabled without a Whoop connection. Add a `RecoveryCardEmpty` (or an empty branch) rendering *"Connect Whoop to see recovery"* via `MiniCardEmpty`, so the renderer can show it when `recovery` is enabled but `present:false`.
- [ ] **Step 5:** Tests — a render test per card (populated + empty) asserting headline/meta text and the deep-link href. Mirror `whoop-card.test.tsx`.
- [ ] **Step 6: Run** `npm run test` (the new files) + `npm run typecheck`.
- [ ] **Step 7: Commit** — `feat(dashboard): walking, cycling, hiking cards and recovery empty state`

---

## Task W5: Tile renderer + grid + tray + edit bar

**Files:**
- Create: `app/(app)/dashboard/_components/tile-renderer.tsx` (exhaustive `switch (id)` → card)
- Create: `app/(app)/dashboard/_components/tile-grid.tsx` (`DndContext` + `SortableContext`; sortable wrappers only in edit mode)
- Create: `app/(app)/dashboard/_components/add-tile-tray.tsx`
- Create: `app/(app)/dashboard/_components/edit-bar.tsx`
- Tests: `tile-renderer.test.tsx` optional (compiler enforces exhaustiveness), `tile-grid.test.tsx`, `add-tile-tray.test.tsx`

- [ ] **Step 1: `tile-renderer.tsx`.** A component `TileCard({ id, data })` with `switch (id)` over `TileId` mapping each id to its card, passing the matching `data.<section>` and the catalog href. The `default` branch does `const _exhaustive: never = id;` so adding a `TileId` without a case is a **compile error** (SOW goal). Reuse the existing in-`page.tsx` card bodies by extracting `RunningCard`, `LiftingCard`, `StepsCard`, `NutritionCard`, `BodyweightCard`, `StreakCard` into this module (move them out of `page.tsx`) alongside the Task W4 cards, so every tile renders through one switch.
- [ ] **Step 2: `tile-grid.tsx`.** Props: `layout: TileId[]`, `data`, `mode`, and edit callbacks (`onReorder(next: TileId[])`, `onRemove(id)`). In view mode render the tiles in `layout` order in the existing `CardGrid` (grid-cols responsive). In edit mode wrap each tile in a sortable wrapper (dnd-kit `useSortable`) with a labelled drag handle and a `Remove {title}` button (accessible name). Use `DndContext` with a `PointerSensor` (activation distance 8px) + `KeyboardSensor` (`sortableKeyboardCoordinates`); `SortableContext` with `rectSortingStrategy`. `onDragEnd` computes the reordered id list (via `layout-ops.reorderTiles` / `arrayMove`) and calls `onReorder`. Keep DnD wiring thin; the reorder math is the tested pure function.
- [ ] **Step 3: `add-tile-tray.tsx`.** Inline tray below the grid listing `availableTiles(draft)` (catalog entries not in the draft), each with title + description + an "Add" affordance calling `onAdd(id)`. Design-system form tokens; hairline panel; no modal. When the draft is full (all ten enabled) render a quiet "All tiles added" note.
- [ ] **Step 4: `edit-bar.tsx`.** View mode: a `Customize` control (secondary pill). Edit mode: `Cancel` (secondary) + `Done` (primary `--accent` pill), with a slot for an inline save error beside `Done` and a `saving` disabled state.
- [ ] **Step 5: Tests.** `add-tile-tray.test.tsx`: renders only not-enabled tiles, Add fires callback. `tile-grid.test.tsx`: view mode renders layout order; edit mode shows remove buttons with accessible names; remove fires callback. (Drag is hard to unit-test; rely on the pure `reorderTiles` test for the ordering logic and a light render assertion here.)
- [ ] **Step 6: Run** tests + typecheck.
- [ ] **Step 7: Commit** — `feat(dashboard): tile renderer, sortable grid, add-tile tray, edit bar`

---

## Task W6: Rewire `page.tsx` into edit-mode state machine; delete the KPI strip

**Files:**
- Modify: `app/(app)/dashboard/page.tsx`
- Delete: `app/(app)/dashboard/_components/kpi.tsx`
- Modify: `app/(app)/dashboard/page.test.tsx`

- [ ] **Step 1:** Rework `page.tsx`. State: `data` (adapted summary incl. `layout`), `mode: "view" | "edit"`, `draft: TileId[]`, `saving: boolean`, `saveError: string | null`. The command bar stays pinned above the grid (not a tile, not draggable/removable). Remove `<KpiStrip>`, `<KpiStripSkeleton>`, the `Kpi` import, and the `pctDelta` helper. Grid renders via `<TileGrid>`.
  - `Customize` → copy `data.layout` into `draft`, `mode="edit"`.
  - Drag/remove/add mutate `draft` only (via the pure ops).
  - `Done` → `putDashboardLayout(token, draft)`, then **refetch** the summary (newly enabled tiles have no data in the current payload), then `mode="view"`. On failure: stay in `edit`, keep `draft`, set `saveError` (retry is one click).
  - `Cancel` → discard `draft`, `mode="view"`.
  - Empty dashboard (`layout` empty) → render an empty-state CTA that opens the tray / enters edit mode.
  - Loading: keep the skeleton grid (drop the KPI skeleton).
- [ ] **Step 2:** Delete `kpi.tsx`. Grep for other importers of `Kpi`/`KpiDelta` (`grep -rn "kpi" app lib components`) and clean up. The `KpiStrip`/`KpiStripSkeleton` functions live inside `page.tsx` — delete them there.
- [ ] **Step 3:** Rewrite `page.test.tsx` (mirror existing structure, mock `getMe`/`getDashboardSummary`/`putDashboardLayout`): enter edit mode via Customize; `Cancel` discards a change; `Done` calls `putDashboardLayout` then refetches and exits edit mode; a failed save keeps edit mode + draft + shows the inline error; empty dashboard shows the CTA. Assert the KPI strip is gone (no `Fuel`/`Weight` KPI labels).
- [ ] **Step 4:** `RunningView`/`LiftingView` delta: `RunningView.currentWeek.deltaPct` may stay in the type (API still sends it) but is no longer read; the SOW says the web adapter simply stops reading the delta and `pctDelta` is deleted. Confirm nothing else references `deltaPct`.
- [ ] **Step 5: Run** the full **Local gate — Web**. Fix all findings.
- [ ] **Step 6: Commit** — `feat(dashboard): customizable edit mode; remove fixed KPI strip`

---

## Web gate + PR

- [ ] Run **Local gate — Web** from a clean tree; all green (typecheck, lint, prettier, vitest, build). No `--no-verify`, no disabled rules.
- [ ] Push `feat/customizable-dashboard-tiles`.
- [ ] `gh pr create` against `main`. Follow the format of recent merged PRs (`gh pr list --state merged --limit 5`, `gh pr view <n>`). Title: `feat(dashboard): customizable tile layout with drag/add/remove`. Body: summary, the new dep note (`@dnd-kit`, ~12kB), the deleted KPI strip, test plan.

---

# PART 3 — Docs status flip (handled by the controller, not a subagent)

After both PRs are open, in `/workspace/prog-strength-docs`:
- Branch `feat/customizable-dashboard-tiles`; edit `sows/customizable-dashboard-tiles.md`: frontmatter `status: shipped`; header `**Status**: Shipped`; `**Last updated**: 2026-07-31`.
- Commit `docs: mark customizable-dashboard-tiles as shipped`.
- Open the docs PR using the required operator template (SOW intro summary, implementation PR bullets, deployment order api→web, verification steps).

---

## Self-review checklist (controller, before dispatch)

- Every SOW section maps to a task: catalog (A1), migration+repo (A2), write path (A3), three sections + shared helper (A4), layout-aware read/default/resilience/gating (A5), wiring (A6); TS catalog+exhaustiveness (W1), adapter+api (W2), pure ops (W3), cards+recovery empty (W4), renderer/grid/tray/bar (W5), page rewire + KPI deletion (W6). ✅
- Non-goals respected: no `other` tile; one tile per subject; no resize/spans; no new subjects; no mobile; no live tray preview; one layout per user. ✅
- Design system: no new tokens; cards mirror existing grammar; edit chrome uses form-control tokens. ✅
- Type names consistent across tasks: `TileID` (Go) / `TileId` (TS); `EnduranceCurrentWeek`/`EnduranceLatest`; `WalkingSection`/`CyclingSection`/`HikingSection`; `DashboardWalking`/`Cycling`/`Hiking`; `putDashboardLayout`; `resolveLayout`/`defaultLayout`. ✅
