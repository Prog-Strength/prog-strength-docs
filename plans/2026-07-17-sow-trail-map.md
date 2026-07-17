# Trail Map Rendering Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Capture GPS position from TCX at ingest, persist per-trackpoint lat/lon and a simplified GeoJSON `MultiLineString` route on the activity, expose the route on the activity detail endpoint, and render it as a MapLibre map on the web running-detail page (with the map omitted entirely for indoor / no-GPS runs).

**Architecture:** The API (`prog-strength-api`, the sole DB writer) extends the TCX parser to keep lat/lon, computes a gap-aware Douglas–Peucker–simplified GeoJSON route from the *full raw* positioned series at ingest, persists it on the `activities` row plus per-point lat/lon on the (already downsampled) `activity_trackpoints` rows, and returns a `route` block on `GET /activities/{id}` and `POST /activities/tcx` only. The web client (`prog-strength-web`) adds `maplibre-gl`, a `RunRouteMap` presentational component fitted to the route bounds using the design-system dark basemap + periwinkle `--accent` stroke, mounted on `/running/[id]` only when `session.route` is present.

**Tech Stack:** Go 1.25 (chi, `database/sql` + go-sqlite3, `encoding/xml`, `encoding/json`, hand-rolled RDP — no new Go deps); Next.js 16 / React 19 / TypeScript / Tailwind v4; `maplibre-gl` + OpenFreeMap dark tiles (no API key); vitest + testing-library.

**Design system conformance (web):** Route stroke uses `--accent` (`#9aa6d6`, periwinkle). Map chrome/frame uses the near-black surface tokens (`--surface` `#15171b`, `--border`). Basemap: OpenFreeMap **dark** style `https://tiles.openfreemap.org/styles/dark` (no account/key; verified 200). No new palette tokens are introduced.

---

## Repo split

- **Tasks 1–8** are in `/workspace/prog-strength-api` (branch `feat/sow-trail-map`).
- **Tasks 9–11** are in `/workspace/prog-strength-web` (branch `feat/sow-trail-map`).

Commit after every task. Conventional-commit subjects, lowercase (e.g. `feat: capture tcx gps position at ingest`).

---

## Key reference facts (already verified against the codebases)

**prog-strength-api**
- Parser: `internal/activity/tcx_parser.go`. `parsedTrackpoint` (lines 43–48) has `Time, DistanceMeters, HeartRateBpm *int, AltitudeMeters *float64`. `xmlPosition` (lines 88–91) already binds `LatitudeDegrees *float64` / `LongitudeDegrees *float64`. `parseTCX` sets `p.HasPosition = true` when `tp.Position != nil` (lines 151–153) — **keep that behavior**.
- Summarizer: `internal/activity/tcx_summarizer.go`. `summarize(p *parsedTCX, actType ActivityType) Activity` builds the Activity and calls `downsample(tps, first, actType)` (line 88). `maxTrackpoints = 300`, even-stride `downsample` (lines 186–235) builds `[]Trackpoint`.
- Models: `internal/activity/model.go`. `Activity` struct (21–66), `Trackpoint` struct (72–79).
- Repository: `internal/activity/sqlite_repository.go`. `activityColumns` const (22–27) is the shared select list used by list/range/summary — **do NOT add `route_geojson` to it**. `Create` (54–118) INSERTs the activity row + `insertTrackpointsTx` (120–141) + `insertBestEffortsTx`. `Get` (320–339) selects `activityColumns`, `scanActivity` (777–831), then `loadTrackpoints` (374–413). `ChangeEnvironment` (532+) must NOT touch route.
- Handler/DTO: `internal/activity/handler.go`. `trackpointDTO` (182–193), `activityDTO` (200–232), `toActivityDTO(a Activity, withTrackpoints bool)` (297–332). Detail path (`get`, `buildDetailDTO`) and `uploadTCX` call with `withTrackpoints=true`; `list` and `patch` with `false`.
- Migrations: `internal/db/migrations/`, auto-embedded via `//go:embed migrations/*.sql` (`internal/db/migrate.go`). Highest is `037_max_effort_v2.sql`; next is **`038`**. Pattern to copy: `036_activity_environment.sql`.
- Tests: `internal/activity/*_test.go`. `readFixture(t, "typical_5k.tcx")` loads from `internal/activity/testdata/`. `typical_5k.tcx` = outdoor w/ GPS (600 `<Position>`), `treadmill_5k.tcx` = indoor (no position). Repo/handler tests use `dbtest.New(t)` and `newTestHandler(t)`.

**prog-strength-web**
- Detail page: `app/(app)/running/[id]/page.tsx` (client component). Layout column (`max-w-3xl`, `gap-6`) renders, in order: `RunHeaderBand`, environment/calibrate controls (closes at line 366), then `SplitsSpine` (367), `PaceRecap` (380), `HeartRateZones` (381). Mount the map **between line 366 and `SplitsSpine`** (below the header band, above the splits spine).
- Types + fetch: `lib/api.ts`. `RunningSession` type (~1680–1714), `RunningTrackpoint` (~1750), `getRunningSession(token, id, unit)` (~1842).
- Tokens: `app/globals.css`. `--accent: #9aa6d6`, `--surface: #15171b`, `--surface-2: #191c21`, `--border: rgba(255,255,255,0.06)`, `--foreground`, `--muted`, `--radius-card: 0.875rem`.
- Component pattern: `app/(app)/running/[id]/_components/` — e.g. `PaceRecap.tsx`, `HeartRateZones.tsx` (+ `.test.tsx`). Presentational, prop-driven, tokens via Tailwind arbitrary values (`bg-[var(--surface)]`) and inline `style`. Tests: vitest globals + testing-library, factory fixtures.

---

## API GeoJSON wire shape (target)

`GET /activities/{id}` and `POST /activities/tcx` detail DTO gains a `route` key. Omitted (key absent) when there is no route:

```json
{
  "route": {
    "type": "Feature",
    "geometry": {
      "type": "MultiLineString",
      "coordinates": [[[lng, lat], [lng, lat], ...], ...]
    },
    "properties": {
      "bounds": { "min_lat": 0, "min_lng": 0, "max_lat": 0, "max_lng": 0 }
    }
  }
}
```

The **stored** `route_geojson` TEXT is exactly this serialized `Feature` JSON (bounds baked in at ingest); the API passes it through verbatim as `json.RawMessage`. Coordinate order is `[longitude, latitude]` (RFC 7946). All coordinates truncated to 6 decimals.

---

## Task 1: Migration 038 — additive position + route columns

**Files:**
- Create: `internal/db/migrations/038_activity_route.sql`

- [ ] **Step 1: Write the migration**

Create `internal/db/migrations/038_activity_route.sql`:

```sql
-- migrations/038_activity_route.sql
-- Trail Map Rendering: capture GPS route geometry at TCX ingest.
--   activity_trackpoints.latitude / .longitude  WGS84 degrees, truncated to
--                        6 decimals on write (~10 cm). NULL when the source
--                        trackpoint lacked <Position> (indoor / no-GPS, or a
--                        kept downsample sample that had no position).
--   activities.route_geojson  serialized GeoJSON Feature (MultiLineString +
--                        bounds) for the simplified route. NULL when fewer
--                        than two positioned points remain after gap-splitting.
-- Additive-only, same expand-only pattern as 036_activity_environment.sql: no
-- table rebuild, no backfill. Existing rows keep NULL coords / NULL route
-- (correct "no map"). New uploads after deploy populate both.
-- See prog-strength-docs/sows/sow-trail-map.md.

ALTER TABLE activity_trackpoints ADD COLUMN latitude REAL;
ALTER TABLE activity_trackpoints ADD COLUMN longitude REAL;
ALTER TABLE activities ADD COLUMN route_geojson TEXT;
```

- [ ] **Step 2: Verify migrations apply cleanly**

Run: `go test ./internal/db/... ./internal/activity/... 2>&1 | tail -20`
Expected: PASS. The `//go:embed migrations/*.sql` glob auto-includes `038`; `dbtest.New(t)` runs all migrations, so any SQL error surfaces here. (Existing activity/repo tests still pass because the new columns are nullable and unused so far.)

- [ ] **Step 3: Commit**

```bash
git add internal/db/migrations/038_activity_route.sql
git commit -m "feat: add nullable gps position and route_geojson columns"
```

---

## Task 2: Parser — capture lat/lon on each trackpoint

**Files:**
- Modify: `internal/activity/tcx_parser.go` (`parsedTrackpoint` struct ~43–48; `parseTCX` position block ~151–153)
- Test: `internal/activity/tcx_parser_test.go`

- [ ] **Step 1: Write the failing test**

Add to `internal/activity/tcx_parser_test.go`:

```go
func TestParseTCX_CapturesPositionCoords(t *testing.T) {
	p, err := parseTCX(readFixture(t, "typical_5k.tcx"))
	if err != nil {
		t.Fatalf("parseTCX: %v", err)
	}
	if !p.HasPosition {
		t.Fatal("expected HasPosition true for GPS fixture")
	}
	var positioned int
	for _, tp := range p.Trackpoints {
		if tp.Latitude != nil && tp.Longitude != nil {
			positioned++
			if *tp.Latitude < -90 || *tp.Latitude > 90 {
				t.Fatalf("latitude out of range: %v", *tp.Latitude)
			}
			if *tp.Longitude < -180 || *tp.Longitude > 180 {
				t.Fatalf("longitude out of range: %v", *tp.Longitude)
			}
		}
	}
	if positioned == 0 {
		t.Fatal("expected at least one trackpoint with captured lat/lon")
	}
}

func TestParseTCX_TreadmillHasNoCoords(t *testing.T) {
	p, err := parseTCX(readFixture(t, "treadmill_5k.tcx"))
	if err != nil {
		t.Fatalf("parseTCX: %v", err)
	}
	for _, tp := range p.Trackpoints {
		if tp.Latitude != nil || tp.Longitude != nil {
			t.Fatal("treadmill fixture should have no coordinates")
		}
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `go test ./internal/activity/ -run TestParseTCX_CapturesPositionCoords -v`
Expected: FAIL — `tp.Latitude`/`tp.Longitude` undefined (compile error).

- [ ] **Step 3: Add fields to `parsedTrackpoint`**

In `internal/activity/tcx_parser.go`, extend the struct (keep existing fields):

```go
type parsedTrackpoint struct {
	Time           time.Time
	DistanceMeters float64
	HeartRateBpm   *int
	AltitudeMeters *float64
	// Latitude/Longitude are the WGS84 <Position> coordinates when the
	// trackpoint carried one; nil otherwise. Full float64 precision here —
	// truncation to 6 decimals happens on write (summarizer / route).
	Latitude  *float64
	Longitude *float64
}
```

- [ ] **Step 4: Populate lat/lon in `parseTCX`**

Replace the position block (currently `if tp.Position != nil { p.HasPosition = true }`) with:

```go
			if tp.Position != nil {
				p.HasPosition = true
				if tp.Position.LatitudeDegrees != nil && tp.Position.LongitudeDegrees != nil {
					lat := *tp.Position.LatitudeDegrees
					lon := *tp.Position.LongitudeDegrees
					out.Latitude = &lat
					out.Longitude = &lon
				}
			}
```

(`out` is the `parsedTrackpoint` being built; this block is inside the trackpoint loop before `p.Trackpoints = append(p.Trackpoints, out)`.)

- [ ] **Step 5: Run tests to verify they pass**

Run: `go test ./internal/activity/ -run 'TestParseTCX' -v`
Expected: PASS (all parser tests, including the two new ones and the existing `TestParseTCX_HasPosition`).

- [ ] **Step 6: Commit**

```bash
git add internal/activity/tcx_parser.go internal/activity/tcx_parser_test.go
git commit -m "feat: capture tcx gps position coordinates at parse"
```

---

## Task 3: Route geometry — haversine, gap-split, RDP, GeoJSON (pure functions)

This is the algorithmic core, isolated in its own file so it can be understood and tested independently. No DB, no XML — pure functions over `[]parsedTrackpoint`.

**Files:**
- Create: `internal/activity/route.go`
- Test: `internal/activity/route_test.go`

- [ ] **Step 1: Write the failing tests**

Create `internal/activity/route_test.go`:

```go
package activity

import (
	"encoding/json"
	"math"
	"testing"
	"time"
)

func ll(lat, lon float64) parsedTrackpoint {
	la, lo := lat, lon
	return parsedTrackpoint{Latitude: &la, Longitude: &lo}
}

func noPos() parsedTrackpoint { return parsedTrackpoint{} }

func decodeRoute(t *testing.T, s *string) geoFeature {
	t.Helper()
	if s == nil {
		t.Fatal("expected a route, got nil")
	}
	var f geoFeature
	if err := json.Unmarshal([]byte(*s), &f); err != nil {
		t.Fatalf("route is not valid json: %v", err)
	}
	return f
}

func TestBuildRoute_NilWhenNoPositions(t *testing.T) {
	tps := []parsedTrackpoint{noPos(), noPos()}
	if got := buildRoute(tps); got != nil {
		t.Fatalf("expected nil route, got %q", *got)
	}
}

func TestBuildRoute_NilWhenSinglePoint(t *testing.T) {
	tps := []parsedTrackpoint{ll(40.0, -105.0)}
	if got := buildRoute(tps); got != nil {
		t.Fatalf("expected nil route for a single positioned point, got %q", *got)
	}
}

func TestBuildRoute_SingleSegmentIsMultiLineString(t *testing.T) {
	// A straight-ish line of collinear points; RDP should keep the endpoints.
	tps := []parsedTrackpoint{
		ll(40.000000, -105.000000),
		ll(40.001000, -105.000000),
		ll(40.002000, -105.000000),
		ll(40.003000, -105.000000),
	}
	f := decodeRoute(t, buildRoute(tps))
	if f.Type != "Feature" {
		t.Fatalf("type = %q, want Feature", f.Type)
	}
	if f.Geometry.Type != "MultiLineString" {
		t.Fatalf("geometry type = %q, want MultiLineString", f.Geometry.Type)
	}
	if len(f.Geometry.Coordinates) != 1 {
		t.Fatalf("segments = %d, want 1", len(f.Geometry.Coordinates))
	}
	// Coordinates are [lng, lat].
	first := f.Geometry.Coordinates[0][0]
	if math.Abs(first[0]-(-105.0)) > 1e-9 || math.Abs(first[1]-40.0) > 1e-9 {
		t.Fatalf("first coord = %v, want [-105, 40]", first)
	}
}

func TestBuildRoute_SplitsOnNullPosition(t *testing.T) {
	tps := []parsedTrackpoint{
		ll(40.000000, -105.000000),
		ll(40.001000, -105.000000),
		noPos(), // ends segment 1, excluded
		ll(40.010000, -105.000000),
		ll(40.011000, -105.000000),
	}
	f := decodeRoute(t, buildRoute(tps))
	if len(f.Geometry.Coordinates) != 2 {
		t.Fatalf("segments = %d, want 2 (gap split on null position)", len(f.Geometry.Coordinates))
	}
}

func TestBuildRoute_SplitsOnTeleport(t *testing.T) {
	// Two positioned points >50 m apart end the segment. ~0.001 deg lat ≈ 111 m.
	tps := []parsedTrackpoint{
		ll(40.000000, -105.000000),
		ll(40.000200, -105.000000), // ~22 m, same segment
		ll(40.100000, -105.000000), // ~11 km jump -> new segment
		ll(40.100200, -105.000000),
	}
	f := decodeRoute(t, buildRoute(tps))
	if len(f.Geometry.Coordinates) != 2 {
		t.Fatalf("segments = %d, want 2 (teleport split)", len(f.Geometry.Coordinates))
	}
}

func TestBuildRoute_DropsShortSegments(t *testing.T) {
	// A lone positioned point between two nulls cannot form a >=2-point line.
	tps := []parsedTrackpoint{
		ll(40.000000, -105.000000),
		ll(40.001000, -105.000000),
		noPos(),
		ll(41.000000, -105.000000), // isolated single point
		noPos(),
	}
	f := decodeRoute(t, buildRoute(tps))
	if len(f.Geometry.Coordinates) != 1 {
		t.Fatalf("segments = %d, want 1 (isolated single point dropped)", len(f.Geometry.Coordinates))
	}
}

func TestBuildRoute_SimplifiesAndComputesBounds(t *testing.T) {
	// Dense collinear run of 500 points; RDP should collapse to the 2 endpoints.
	var tps []parsedTrackpoint
	for i := 0; i < 500; i++ {
		tps = append(tps, ll(40.0+float64(i)*0.00001, -105.0))
	}
	f := decodeRoute(t, buildRoute(tps))
	pts := f.Geometry.Coordinates[0]
	if len(pts) > 10 {
		t.Fatalf("simplified line kept %d points, expected a large reduction", len(pts))
	}
	b := f.Properties.Bounds
	if b.MinLat > b.MaxLat || b.MinLng > b.MaxLng {
		t.Fatalf("bounds inverted: %+v", b)
	}
	if math.Abs(b.MinLat-40.0) > 1e-6 {
		t.Fatalf("min_lat = %v, want ~40.0", b.MinLat)
	}
}

func TestBuildRoute_TruncatesToSixDecimals(t *testing.T) {
	tps := []parsedTrackpoint{
		ll(40.1234567, -105.7654321),
		ll(40.2000009, -105.6000001),
	}
	f := decodeRoute(t, buildRoute(tps))
	c := f.Geometry.Coordinates[0][0]
	// truncated lat = 40.123456 (toward zero), lng = -105.765432
	if math.Abs(c[1]-40.123456) > 1e-9 {
		t.Fatalf("lat = %v, want 40.123456", c[1])
	}
	if math.Abs(c[0]-(-105.765432)) > 1e-9 {
		t.Fatalf("lng = %v, want -105.765432", c[0])
	}
}

func TestBuildRoute_RealTrailFixtureReduces(t *testing.T) {
	p, err := parseTCX(readFixture(t, "typical_5k.tcx"))
	if err != nil {
		t.Fatalf("parseTCX: %v", err)
	}
	var raw int
	for _, tp := range p.Trackpoints {
		if tp.Latitude != nil {
			raw++
		}
	}
	f := decodeRoute(t, buildRoute(p.Trackpoints))
	var simplified int
	for _, seg := range f.Geometry.Coordinates {
		simplified += len(seg)
	}
	if simplified == 0 || simplified >= raw {
		t.Fatalf("simplified=%d raw=%d, expected a material reduction", simplified, raw)
	}
	_ = time.Now // keep time import if unused elsewhere
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `go test ./internal/activity/ -run TestBuildRoute -v`
Expected: FAIL — `buildRoute`, `geoFeature`, etc. undefined (compile error).

- [ ] **Step 3: Implement `internal/activity/route.go`**

```go
package activity

import (
	"encoding/json"
	"math"
)

// Route simplification tuning. See sows/sow-trail-map.md → Algorithms.
const (
	// routeGapMeters ends a segment when two consecutive positioned points
	// are farther apart than this (teleport / GPS acquisition glitch).
	routeGapMeters = 50.0
	// routeSimplifyEpsilon is the Douglas–Peucker tolerance, in DEGREES, on
	// (lat, lon) treated as a local planar approximation. ~5.6 m N–S; drops
	// ~94% of points on a real trail file at 1 Hz.
	routeSimplifyEpsilon = 5e-5
	// routeCoordScale truncates coordinates to 6 decimal places (~10 cm) on
	// write. Raw float64 fidelity survives in the archived S3 TCX.
	routeCoordScale = 1e6
)

// latLon is a positioned point used by the simplifier. Coordinates are full
// float64 precision until serialization truncates them.
type latLon struct {
	Lat float64
	Lon float64
}

// geoFeature is the serialized route: a GeoJSON Feature wrapping a
// MultiLineString whose coordinates are [longitude, latitude] pairs
// (RFC 7946), with the pre-computed bounding box in properties.
type geoFeature struct {
	Type       string        `json:"type"`
	Geometry   geoGeometry   `json:"geometry"`
	Properties geoProperties `json:"properties"`
}

type geoGeometry struct {
	Type        string        `json:"type"`
	Coordinates [][][2]float64 `json:"coordinates"`
}

type geoProperties struct {
	Bounds geoBounds `json:"bounds"`
}

type geoBounds struct {
	MinLat float64 `json:"min_lat"`
	MinLng float64 `json:"min_lng"`
	MaxLat float64 `json:"max_lat"`
	MaxLng float64 `json:"max_lng"`
}

// buildRoute derives the simplified GeoJSON route from the FULL raw
// positioned trackpoint series (before the ~300-point chart downsample). It
// gap-splits into segments (never bridging a NULL Position or a >50 m
// teleport), runs Douglas–Peucker on each, truncates coordinates to 6
// decimals, and serializes a MultiLineString Feature with bounds. Returns
// nil when fewer than two positioned points remain after splitting (no
// renderable route — the caller stores NULL and the map is omitted).
func buildRoute(tps []parsedTrackpoint) *string {
	segments := splitSegments(tps)

	coords := make([][][2]float64, 0, len(segments))
	first := true
	var b geoBounds
	for _, seg := range segments {
		simplified := rdp(seg, routeSimplifyEpsilon)
		if len(simplified) < 2 {
			continue
		}
		line := make([][2]float64, 0, len(simplified))
		for _, p := range simplified {
			lat := truncateCoord(p.Lat)
			lon := truncateCoord(p.Lon)
			line = append(line, [2]float64{lon, lat})
			if first {
				b = geoBounds{MinLat: lat, MinLng: lon, MaxLat: lat, MaxLng: lon}
				first = false
				continue
			}
			b.MinLat = math.Min(b.MinLat, lat)
			b.MaxLat = math.Max(b.MaxLat, lat)
			b.MinLng = math.Min(b.MinLng, lon)
			b.MaxLng = math.Max(b.MaxLng, lon)
		}
		coords = append(coords, line)
	}

	if len(coords) == 0 {
		return nil
	}

	f := geoFeature{
		Type: "Feature",
		Geometry: geoGeometry{
			Type:        "MultiLineString",
			Coordinates: coords,
		},
		Properties: geoProperties{Bounds: b},
	}
	raw, err := json.Marshal(f)
	if err != nil {
		// The struct is fixed-shape and all-finite; marshaling cannot fail in
		// practice. Treat a failure as "no route" rather than panicking ingest.
		return nil
	}
	s := string(raw)
	return &s
}

// splitSegments walks the series and cuts a new segment whenever a NULL
// Position appears (the null point is excluded) or two consecutive
// positioned points are more than routeGapMeters apart. Segments of any
// length are returned; the caller drops those that simplify below 2 points.
func splitSegments(tps []parsedTrackpoint) [][]latLon {
	var segments [][]latLon
	var cur []latLon
	var prev *latLon

	flush := func() {
		if len(cur) > 0 {
			segments = append(segments, cur)
			cur = nil
		}
		prev = nil
	}

	for _, tp := range tps {
		if tp.Latitude == nil || tp.Longitude == nil {
			flush()
			continue
		}
		p := latLon{Lat: *tp.Latitude, Lon: *tp.Longitude}
		if prev != nil && haversineMeters(prev.Lat, prev.Lon, p.Lat, p.Lon) > routeGapMeters {
			flush()
		}
		cur = append(cur, p)
		last := p
		prev = &last
	}
	flush()
	return segments
}

// haversineMeters is the great-circle distance between two WGS84 points.
func haversineMeters(lat1, lon1, lat2, lon2 float64) float64 {
	const earthRadiusM = 6371000.0
	rad := math.Pi / 180
	dLat := (lat2 - lat1) * rad
	dLon := (lon2 - lon1) * rad
	a := math.Sin(dLat/2)*math.Sin(dLat/2) +
		math.Cos(lat1*rad)*math.Cos(lat2*rad)*math.Sin(dLon/2)*math.Sin(dLon/2)
	return earthRadiusM * 2 * math.Atan2(math.Sqrt(a), math.Sqrt(1-a))
}

// rdp is Douglas–Peucker line simplification working in degrees on the
// (lat, lon) plane. epsilon is the max perpendicular deviation a point may
// have from the segment chord before it must be kept. Endpoints are always
// retained, so a run of >=2 input points yields >=2 output points.
func rdp(points []latLon, epsilon float64) []latLon {
	if len(points) < 3 {
		return points
	}
	first, last := points[0], points[len(points)-1]
	maxDist := 0.0
	idx := 0
	for i := 1; i < len(points)-1; i++ {
		d := perpendicularDistance(points[i], first, last)
		if d > maxDist {
			maxDist = d
			idx = i
		}
	}
	if maxDist <= epsilon {
		return []latLon{first, last}
	}
	left := rdp(points[:idx+1], epsilon)
	right := rdp(points[idx:], epsilon)
	// Drop the duplicated join point (idx appears in both halves).
	return append(left[:len(left)-1], right...)
}

// perpendicularDistance is the distance from p to the line through a and b,
// treating lat/lon as planar (x=lon, y=lat). When a == b it degenerates to
// the point distance.
func perpendicularDistance(p, a, b latLon) float64 {
	dx := b.Lon - a.Lon
	dy := b.Lat - a.Lat
	if dx == 0 && dy == 0 {
		return math.Hypot(p.Lon-a.Lon, p.Lat-a.Lat)
	}
	// |cross product| / |chord length|.
	num := math.Abs(dy*(p.Lon-a.Lon) - dx*(p.Lat-a.Lat))
	return num / math.Hypot(dx, dy)
}

// truncateCoord truncates a coordinate toward zero to 6 decimal places
// (~10 cm). Truncation, not rounding, matches the DB write rule.
func truncateCoord(v float64) float64 {
	return math.Trunc(v*routeCoordScale) / routeCoordScale
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `go test ./internal/activity/ -run 'TestBuildRoute|TestParseTCX' -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/activity/route.go internal/activity/route_test.go
git commit -m "feat: add gap-aware rdp route geometry builder"
```

---

## Task 4: Models + summarizer — carry lat/lon onto trackpoints, compute route

**Files:**
- Modify: `internal/activity/model.go` (`Activity` struct; `Trackpoint` struct)
- Modify: `internal/activity/tcx_summarizer.go` (`summarize`, `downsample`)
- Test: `internal/activity/tcx_summarizer_test.go`

- [ ] **Step 1: Write the failing test**

Add to `internal/activity/tcx_summarizer_test.go`:

```go
func TestSummarize_OutdoorRunBuildsRoute(t *testing.T) {
	p, err := parseTCX(readFixture(t, "typical_5k.tcx"))
	if err != nil {
		t.Fatalf("parseTCX: %v", err)
	}
	a := summarize(p, ActivityRunning)
	if a.RouteGeoJSON == nil {
		t.Fatal("expected a route for the outdoor GPS fixture")
	}
	if !strings.Contains(*a.RouteGeoJSON, "MultiLineString") {
		t.Fatalf("route should be a MultiLineString feature, got: %.80s", *a.RouteGeoJSON)
	}
	// At least one downsampled trackpoint carries coordinates.
	var withCoords int
	for _, tp := range a.Trackpoints {
		if tp.Latitude != nil && tp.Longitude != nil {
			withCoords++
			// Stored coords are truncated to 6 decimals.
			if *tp.Latitude != math.Trunc(*tp.Latitude*1e6)/1e6 {
				t.Fatalf("stored latitude not truncated to 6 decimals: %v", *tp.Latitude)
			}
		}
	}
	if withCoords == 0 {
		t.Fatal("expected downsampled trackpoints to carry lat/lon")
	}
}

func TestSummarize_IndoorRunHasNoRoute(t *testing.T) {
	p, err := parseTCX(readFixture(t, "treadmill_5k.tcx"))
	if err != nil {
		t.Fatalf("parseTCX: %v", err)
	}
	a := summarize(p, ActivityRunning)
	if a.RouteGeoJSON != nil {
		t.Fatalf("expected no route for indoor run, got %q", *a.RouteGeoJSON)
	}
	for _, tp := range a.Trackpoints {
		if tp.Latitude != nil || tp.Longitude != nil {
			t.Fatal("indoor trackpoints should have no coordinates")
		}
	}
}
```

Ensure the test file imports `math` and `strings` (add to the import block if missing).

- [ ] **Step 2: Run test to verify it fails**

Run: `go test ./internal/activity/ -run TestSummarize_OutdoorRunBuildsRoute -v`
Expected: FAIL — `a.RouteGeoJSON` / `tp.Latitude` undefined (compile error).

- [ ] **Step 3: Add fields to the models**

In `internal/activity/model.go`, add to `Activity` (after `TCXS3Key` / near the derived fields — place it right after `ElevationGainMeters`):

```go
	// RouteGeoJSON is the serialized GeoJSON Feature (MultiLineString +
	// bounds) for the simplified GPS route, computed at ingest from the raw
	// positioned series. nil when the activity had fewer than two positioned
	// points (indoor / no-GPS). Loaded only on the detail path, never on
	// list/summary reads.
	RouteGeoJSON *string
```

Add to `Trackpoint`:

```go
	// Latitude/Longitude are the WGS84 coordinates of this kept sample,
	// truncated to 6 decimals, when the source trackpoint had a <Position>;
	// nil otherwise. The map renders from Activity.RouteGeoJSON, not these —
	// they are stored for future pace↔map correlation / FIT parity.
	Latitude  *float64
	Longitude *float64
```

- [ ] **Step 4: Carry lat/lon in `downsample` and compute the route in `summarize`**

In `internal/activity/tcx_summarizer.go`, inside `downsample`, when building each `Trackpoint tp`, add coordinate carry-over (truncated) after the existing field assignments and before the pace block:

```go
		if rp.Latitude != nil && rp.Longitude != nil {
			lat := truncateCoord(*rp.Latitude)
			lon := truncateCoord(*rp.Longitude)
			tp.Latitude = &lat
			tp.Longitude = &lon
		}
```

In `summarize`, set the route from the FULL raw series (`tps`) — add just before `a.Trackpoints = downsample(...)`:

```go
	a.RouteGeoJSON = buildRoute(tps)
```

(Note: `buildRoute` runs for every sport with positions — it keys off position presence, not `environment` or sport, matching the SOW's source-agnostic map layer. Indoor/no-GPS files yield nil.)

- [ ] **Step 5: Run tests to verify they pass**

Run: `go test ./internal/activity/ -run 'TestSummarize' -v`
Expected: PASS (new tests plus existing `TestSummarize_*`).

- [ ] **Step 6: Commit**

```bash
git add internal/activity/model.go internal/activity/tcx_summarizer.go internal/activity/tcx_summarizer_test.go
git commit -m "feat: persist route geojson and per-point coords in summarizer"
```

---

## Task 5: Repository — write/read lat/lon and route_geojson

**Files:**
- Modify: `internal/activity/sqlite_repository.go` (`Create` activities INSERT; `insertTrackpointsTx`; `loadTrackpoints`; `Get`)
- Test: `internal/activity/sqlite_repository_test.go`

- [ ] **Step 1: Write the failing test**

Add to `internal/activity/sqlite_repository_test.go` (match existing helpers in that file; if a helper to build+create an activity already exists, reuse it — otherwise this self-contained test works):

```go
func TestRepository_RoundTripsRouteAndCoords(t *testing.T) {
	repo := NewSQLiteRepository(dbtest.New(t), NewMemoryArchiver())
	ctx := context.Background()

	lat, lon := 40.123456, -105.654321
	route := `{"type":"Feature","geometry":{"type":"MultiLineString","coordinates":[[[-105.654321,40.123456],[-105.654000,40.124000]]]},"properties":{"bounds":{"min_lat":40.123456,"min_lng":-105.654321,"max_lat":40.124,"max_lng":-105.654}}}`
	a := &Activity{
		UserID:           "u1",
		ActivityType:     ActivityRunning,
		IngestSource:     IngestManualTCX,
		SourceActivityID: "route-test-1",
		StartTime:        time.Now().UTC(),
		Environment:      EnvironmentOutdoor,
		RouteGeoJSON:     &route,
		Trackpoints: []Trackpoint{
			{Sequence: 0, ElapsedSeconds: 0, DistanceMeters: 0, Latitude: &lat, Longitude: &lon},
			{Sequence: 1, ElapsedSeconds: 1, DistanceMeters: 5},
		},
	}
	if err := repo.Create(ctx, a, []byte("<xml/>")); err != nil {
		t.Fatalf("Create: %v", err)
	}

	got, err := repo.Get(ctx, "u1", a.ID)
	if err != nil {
		t.Fatalf("Get: %v", err)
	}
	if got.RouteGeoJSON == nil || *got.RouteGeoJSON != route {
		t.Fatalf("route not round-tripped: %v", got.RouteGeoJSON)
	}
	if got.Trackpoints[0].Latitude == nil || *got.Trackpoints[0].Latitude != lat {
		t.Fatalf("trackpoint latitude not round-tripped: %+v", got.Trackpoints[0])
	}
	if got.Trackpoints[1].Latitude != nil {
		t.Fatal("second trackpoint should have NULL latitude")
	}
}
```

(If the test file already imports `context`/`time`, don't duplicate. `NewMemoryArchiver` is the in-repo archiver used by `newTestHandler`; confirm its constructor name in the test package and match it.)

- [ ] **Step 2: Run test to verify it fails**

Run: `go test ./internal/activity/ -run TestRepository_RoundTripsRouteAndCoords -v`
Expected: FAIL — columns not written/read (route nil, or SQL error on unknown binding).

- [ ] **Step 3: Write route_geojson in `Create`**

In the `INSERT INTO activities (...)` statement, add `route_geojson` to the column list and one `?` to VALUES, and append `a.RouteGeoJSON` as the final arg:

```go
	if _, err := tx.ExecContext(ctx, `
		INSERT INTO activities (
			id, user_id, activity_type, ingest_source, source_activity_id,
			start_time, name, distance_meters, duration_seconds,
			avg_pace_sec_per_km, best_pace_sec_per_km,
			avg_heart_rate_bpm, max_heart_rate_bpm, total_calories, elevation_gain_meters,
			tcx_s3_key, created_at, environment, raw_distance_meters, route_geojson
		) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
	`, a.ID, a.UserID, a.ActivityType, a.IngestSource, a.SourceActivityID,
		a.StartTime, a.Name, a.DistanceMeters, a.DurationSeconds,
		a.AvgPaceSecPerKm, a.BestPaceSecPerKm,
		a.AvgHeartRateBpm, a.MaxHeartRateBpm, a.TotalCalories, a.ElevationGainMeters,
		a.TCXS3Key, a.CreatedAt, a.Environment, a.RawDistanceMeters, a.RouteGeoJSON); err != nil {
```

- [ ] **Step 4: Write lat/lon in `insertTrackpointsTx`**

Extend the prepared INSERT and the per-point Exec:

```go
	stmt, err := tx.PrepareContext(ctx, `
		INSERT INTO activity_trackpoints (
			activity_id, sequence, elapsed_seconds, distance_meters,
			heart_rate_bpm, pace_sec_per_km, elevation_meters, latitude, longitude
		) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
	`)
	if err != nil {
		return err
	}
	defer stmt.Close()
	for _, p := range points {
		if _, err := stmt.ExecContext(ctx, activityID, p.Sequence, p.ElapsedSeconds,
			p.DistanceMeters, p.HeartRateBpm, p.PaceSecPerKm, p.ElevationMeters,
			p.Latitude, p.Longitude); err != nil {
			return err
		}
	}
```

- [ ] **Step 5: Read lat/lon in `loadTrackpoints`**

Extend the SELECT, the null scan targets, and the population:

```go
	rows, err := r.db.QueryContext(ctx, `
		SELECT sequence, elapsed_seconds, distance_meters,
		       heart_rate_bpm, pace_sec_per_km, elevation_meters, latitude, longitude
		FROM activity_trackpoints
		WHERE activity_id = ?
		ORDER BY sequence ASC
	`, activityID)
```

Add `lat, lon sql.NullFloat64` to the per-row var block, extend the `rows.Scan(...)` call with `&lat, &lon`, and after the existing `el.Valid` block:

```go
		if lat.Valid {
			v := lat.Float64
			p.Latitude = &v
		}
		if lon.Valid {
			v := lon.Float64
			p.Longitude = &v
		}
```

- [ ] **Step 6: Load route_geojson in `Get`**

`Get` selects the shared `activityColumns` (which must stay route-free). After `a.Trackpoints = points`, load the route with a dedicated by-PK query and set it:

```go
	a.Trackpoints = points

	var route sql.NullString
	if err := r.db.QueryRowContext(ctx, `
		SELECT route_geojson FROM activities
		WHERE id = ? AND user_id = ? AND deleted_at IS NULL
	`, activityID, userID).Scan(&route); err != nil {
		return nil, err
	}
	if route.Valid {
		a.RouteGeoJSON = &route.String
	}
	return a, nil
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `go test ./internal/activity/ -v 2>&1 | tail -30`
Expected: PASS across the package (new round-trip test + all existing repository/handler/summarizer tests — the `activityColumns` scan order is unchanged, so `List`/`ChangeEnvironment` etc. are unaffected).

- [ ] **Step 8: Commit**

```bash
git add internal/activity/sqlite_repository.go internal/activity/sqlite_repository_test.go
git commit -m "feat: persist and load route geojson and trackpoint coords"
```

---

## Task 6: API DTO — expose `route` on the detail responses only

**Files:**
- Modify: `internal/activity/handler.go` (`activityDTO`; `toActivityDTO`)
- Test: `internal/activity/handler_test.go`

- [ ] **Step 1: Write the failing test**

Add to `internal/activity/handler_test.go`. Use the existing import/upload helpers in that file (`newTestHandler`, `doImport`, `readFixture`, the `activityEnvelope`/`listEnvelope` wrappers). The route arrives as a nested JSON object; decode the raw detail body to assert on it:

```go
func TestUploadTCX_OutdoorReturnsRoute(t *testing.T) {
	h, _, _ := newTestHandler(t)
	w := doImport(t, h, readFixture(t, "typical_5k.tcx"))
	if w.Code != http.StatusCreated {
		t.Fatalf("status = %d, want 201; body=%s", w.Code, w.Body.String())
	}
	var env struct {
		Data struct {
			Route *struct {
				Type     string `json:"type"`
				Geometry struct {
					Type        string          `json:"type"`
					Coordinates json.RawMessage `json:"coordinates"`
				} `json:"geometry"`
				Properties struct {
					Bounds struct {
						MinLat float64 `json:"min_lat"`
						MaxLat float64 `json:"max_lat"`
					} `json:"bounds"`
				} `json:"properties"`
			} `json:"route"`
		} `json:"data"`
	}
	if err := json.Unmarshal(w.Body.Bytes(), &env); err != nil {
		t.Fatalf("decode: %v", err)
	}
	if env.Data.Route == nil {
		t.Fatal("expected route on outdoor upload response")
	}
	if env.Data.Route.Type != "Feature" || env.Data.Route.Geometry.Type != "MultiLineString" {
		t.Fatalf("unexpected route shape: %+v", env.Data.Route)
	}
	if env.Data.Route.Properties.Bounds.MinLat > env.Data.Route.Properties.Bounds.MaxLat {
		t.Fatal("bounds inverted")
	}
}

func TestUploadTCX_IndoorOmitsRoute(t *testing.T) {
	h, _, _ := newTestHandler(t)
	w := doImport(t, h, readFixture(t, "treadmill_5k.tcx"))
	if w.Code != http.StatusCreated {
		t.Fatalf("status = %d, want 201", w.Code)
	}
	var env struct {
		Data struct {
			Route json.RawMessage `json:"route"`
		} `json:"data"`
	}
	if err := json.Unmarshal(w.Body.Bytes(), &env); err != nil {
		t.Fatalf("decode: %v", err)
	}
	if len(env.Data.Route) != 0 {
		t.Fatalf("expected no route key for indoor run, got %s", env.Data.Route)
	}
}
```

(Match the real response-envelope key: existing handler tests decode via typed envelopes — if the wrapper key is not `data`, mirror whatever those tests use. `httpresp.OK/Created` shape is visible in `handler_test.go`.)

- [ ] **Step 2: Run test to verify it fails**

Run: `go test ./internal/activity/ -run 'TestUploadTCX_OutdoorReturnsRoute|TestUploadTCX_IndoorOmitsRoute' -v`
Expected: FAIL — no `route` key emitted.

- [ ] **Step 3: Add `Route` to `activityDTO`**

Add the field (import `encoding/json` in `handler.go` if not already imported). Place it with the detail-only derived blocks:

```go
	// Route is the simplified GPS route as a GeoJSON Feature (MultiLineString
	// + bounds), passed through verbatim from storage. Detail-only; omitted
	// (key absent) when the activity has no route. The map consumes this; the
	// per-trackpoint DTO deliberately stays coordinate-free.
	Route json.RawMessage `json:"route,omitempty"`
```

- [ ] **Step 4: Populate it in `toActivityDTO`**

After the `if withTrackpoints { ... }` block, before `return dto`:

```go
	if a.RouteGeoJSON != nil {
		dto.Route = json.RawMessage(*a.RouteGeoJSON)
	}
```

(`RouteGeoJSON` is only ever non-nil on detail-load / fresh-ingest Activities — list/summary reads never select `route_geojson` — so this naturally stays off list and PATCH responses without an extra gate.)

- [ ] **Step 5: Run the full package test suite**

Run: `go test ./internal/activity/ -v 2>&1 | tail -30`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add internal/activity/handler.go internal/activity/handler_test.go
git commit -m "feat: expose route geojson on activity detail responses"
```

---

## Task 7: API — full local CI gate

No new code; this task makes CI-green a checkpoint before the web work.

**Files:** none (verification only)

- [ ] **Step 1: Tidy and verify no module drift**

Run:
```bash
go mod tidy
git diff --exit-code go.mod go.sum
```
Expected: no diff (no new imports were added — RDP/GeoJSON use only stdlib `math`, `encoding/json`).

- [ ] **Step 2: Vet**

Run: `go vet ./...`
Expected: no output.

- [ ] **Step 3: Lint at the CI-pinned version**

Run:
```bash
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run
```
Expected: `0 issues`. If `gosec` flags anything, fix the code — do **not** add `//nolint`. (Watch for: unchecked errors, integer conversions. The route code checks the one `json.Marshal` error path.)

- [ ] **Step 4: Full test suite**

Run: `go test ./...`
Expected: all packages PASS.

- [ ] **Step 5: Commit (only if `go mod tidy` changed anything)**

If Step 1 produced a diff, investigate and fix; only commit an intentional tidy:
```bash
git add go.mod go.sum && git commit -m "chore: go mod tidy"
```
Otherwise skip — nothing to commit.

---

## Task 8: Web — add maplibre-gl dependency and the `route` type

**Files:**
- Modify: `package.json` / `package-lock.json` (via `npm install`)
- Modify: `lib/api.ts` (`RunningSession` type)

- [ ] **Step 1: Install maplibre-gl**

Run (from `/workspace/prog-strength-web`):
```bash
npm install maplibre-gl@^5
npm ls maplibre-gl
```
Expected: `maplibre-gl` added to `dependencies` in `package.json` and locked in `package-lock.json`. (maplibre-gl ships its own TypeScript types — no separate `@types` package.)

- [ ] **Step 2: Add the `RouteFeature` type and `route` field**

In `lib/api.ts`, define a route type near `RunningSession` (place the type just above the `RunningSession` type):

```ts
// A simplified GPS route as a GeoJSON Feature (RFC 7946): a MultiLineString
// whose coordinates are [longitude, latitude] pairs, plus a pre-computed
// bounding box the map fits its camera to. Present on the activity-detail
// response only for runs recorded with GPS; absent for indoor / no-GPS runs.
export type RouteBounds = {
  min_lat: number;
  min_lng: number;
  max_lat: number;
  max_lng: number;
};

export type RouteFeature = {
  type: "Feature";
  geometry: {
    type: "MultiLineString";
    coordinates: number[][][]; // [segment][point][lng, lat]
  };
  properties: {
    bounds: RouteBounds;
  };
};
```

Add to the `RunningSession` type, alongside the other detail-only optional fields (e.g. after `heart_rate_zones?`):

```ts
  // Simplified GPS route for the map; present only on the detail GET for
  // GPS-recorded runs. Absent for indoor / no-GPS activities.
  route?: RouteFeature;
```

- [ ] **Step 3: Typecheck**

Run: `npm run typecheck`
Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add package.json package-lock.json lib/api.ts
git commit -m "feat: add maplibre-gl dependency and route detail type"
```

---

## Task 9: Web — `RunRouteMap` presentational component

**Files:**
- Create: `app/(app)/running/[id]/_components/RunRouteMap.tsx`
- Test: `app/(app)/running/[id]/_components/RunRouteMap.test.tsx`

**Design tokens:** route stroke `var(--accent)`; frame `bg-[var(--surface)]`, `border-[var(--border)]`, `rounded-[var(--radius-card)]`. Basemap style URL: `https://tiles.openfreemap.org/styles/dark` (OpenFreeMap, no key/account). MapLibre CSS is imported in the component (`import "maplibre-gl/dist/maplibre-gl.css";`).

- [ ] **Step 1: Write the failing test**

MapLibre's `Map` touches WebGL, which jsdom lacks — so the test mocks `maplibre-gl` and asserts the component's *contract* (renders a container when a route is present, renders nothing when absent, wires the accent stroke + bounds into the map calls). Create `app/(app)/running/[id]/_components/RunRouteMap.test.tsx`:

```tsx
/// <reference types="vitest/globals" />
import { render } from "@testing-library/react";
import type { RouteFeature } from "@/lib/api";

const addSource = vi.fn();
const addLayer = vi.fn();
const fitBounds = vi.fn();
const on = vi.fn((event: string, cb: () => void) => {
  if (event === "load") cb();
});
const remove = vi.fn();
const mapCtor = vi.fn(() => ({ addSource, addLayer, fitBounds, on, remove }));

vi.mock("maplibre-gl", () => ({
  default: { Map: mapCtor },
  Map: mapCtor,
}));
vi.mock("maplibre-gl/dist/maplibre-gl.css", () => ({}));

import { RunRouteMap } from "./RunRouteMap";

function route(): RouteFeature {
  return {
    type: "Feature",
    geometry: {
      type: "MultiLineString",
      coordinates: [
        [
          [-105.0, 40.0],
          [-105.001, 40.001],
        ],
      ],
    },
    properties: {
      bounds: { min_lat: 40.0, min_lng: -105.001, max_lat: 40.001, max_lng: -105.0 },
    },
  };
}

beforeEach(() => {
  vi.clearAllMocks();
});

it("renders nothing when route is undefined", () => {
  const { container } = render(<RunRouteMap route={undefined} />);
  expect(container.firstChild).toBeNull();
});

it("mounts a maplibre map and fits the route bounds when a route is present", () => {
  const { container } = render(<RunRouteMap route={route()} />);
  expect(container.firstChild).not.toBeNull();
  expect(mapCtor).toHaveBeenCalledTimes(1);
  // Camera fit to [ [min_lng,min_lat], [max_lng,max_lat] ].
  expect(fitBounds).toHaveBeenCalledWith(
    [
      [-105.001, 40.0],
      [-105.0, 40.001],
    ],
    expect.objectContaining({ padding: expect.any(Number) }),
  );
  // Route line drawn with the accent stroke.
  const layerArg = addLayer.mock.calls[0][0];
  expect(layerArg.type).toBe("line");
  expect(layerArg.paint["line-color"]).toBe("var(--accent)");
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm run test -- RunRouteMap`
Expected: FAIL — `./RunRouteMap` module not found.

- [ ] **Step 3: Implement `RunRouteMap.tsx`**

```tsx
"use client";

import { useEffect, useRef } from "react";
import maplibregl from "maplibre-gl";
import "maplibre-gl/dist/maplibre-gl.css";
import type { RouteFeature } from "@/lib/api";

// OpenFreeMap's dark first-party style — no account, no API key. Chosen to sit
// under the app's near-black chrome; the periwinkle --accent route stroke reads
// clearly on it. See sows/sow-trail-map.md.
const BASEMAP_STYLE_URL = "https://tiles.openfreemap.org/styles/dark";
const FIT_PADDING = 32;

type RunRouteMapProps = {
  route: RouteFeature | undefined;
};

// RunRouteMap renders the simplified GPS route on a MapLibre map fitted to the
// route bounds. It renders nothing when there is no route (indoor / no-GPS
// runs), so the detail page is byte-for-byte unchanged for those. Purely
// presentational: it owns the map lifecycle for the route prop it is given.
export function RunRouteMap({ route }: RunRouteMapProps) {
  const containerRef = useRef<HTMLDivElement | null>(null);

  useEffect(() => {
    if (!route || !containerRef.current) return;

    const { bounds } = route.properties;
    const map = new maplibregl.Map({
      container: containerRef.current,
      style: BASEMAP_STYLE_URL,
      attributionControl: { compact: true },
      interactive: true,
    });

    map.on("load", () => {
      map.addSource("route", { type: "geojson", data: route });
      map.addLayer({
        id: "route-line",
        type: "line",
        source: "route",
        layout: { "line-cap": "round", "line-join": "round" },
        paint: { "line-color": "var(--accent)", "line-width": 3 },
      });
      map.fitBounds(
        [
          [bounds.min_lng, bounds.min_lat],
          [bounds.max_lng, bounds.max_lat],
        ],
        { padding: FIT_PADDING, duration: 0 },
      );
    });

    return () => map.remove();
  }, [route]);

  if (!route) return null;

  return (
    <div className="overflow-hidden rounded-[var(--radius-card)] border border-[var(--border)] bg-[var(--surface)]">
      <div ref={containerRef} className="h-64 w-full" aria-label="Run route map" />
    </div>
  );
}
```

Note on the test's `line-color` assertion: MapLibre paint values are read back through the mocked `addLayer` call, so the literal `"var(--accent)"` string is what the test sees. In the real browser, MapLibre does not resolve CSS custom properties in paint expressions — so if the stroke renders as default black at runtime, resolve `--accent` to its hex at mount instead. To stay robust, read the computed value at mount and pass a concrete color:

Replace the `paint` line and add a resolver so the accent survives both the test and the browser:

```tsx
      const accent =
        getComputedStyle(document.documentElement).getPropertyValue("--accent").trim() ||
        "#9aa6d6";
      map.addLayer({
        id: "route-line",
        type: "line",
        source: "route",
        layout: { "line-cap": "round", "line-join": "round" },
        paint: { "line-color": accent, "line-width": 3 },
      });
```

If you make this change, update the test assertion to accept the resolved value:

```tsx
  expect(layerArg.paint["line-color"]).toMatch(/#9aa6d6|9aa6d6|var\(--accent\)/i);
```

Pick ONE approach (resolver recommended, since it renders correctly in a real browser) and make the test and component agree. Document the chosen basemap URL and the stroke-resolution decision in the PR description.

- [ ] **Step 4: Run test to verify it passes**

Run: `npm run test -- RunRouteMap`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "app/(app)/running/[id]/_components/RunRouteMap.tsx" "app/(app)/running/[id]/_components/RunRouteMap.test.tsx"
git commit -m "feat(running): add RunRouteMap route map component"
```

---

## Task 10: Web — mount the map on the running detail page

**Files:**
- Modify: `app/(app)/running/[id]/page.tsx` (import; mount between the calibrate controls block and `SplitsSpine`)

- [ ] **Step 1: Import the component**

Add to the imports at the top of `app/(app)/running/[id]/page.tsx` (alongside the other `_components` imports):

```tsx
import { RunRouteMap } from "./_components/RunRouteMap";
```

- [ ] **Step 2: Mount it conditionally above the splits spine**

Immediately after the environment/calibrate controls block closes (the `)}` at line ~366) and before `<SplitsSpine`, add:

```tsx
          <RunRouteMap route={session.route} />
```

`RunRouteMap` renders `null` when `session.route` is undefined, so indoor / no-GPS runs are unchanged (no empty frame, no placeholder). No extra guard is needed here — the conditional lives inside the component.

- [ ] **Step 3: Typecheck, lint, format, test, build**

Run:
```bash
npm run typecheck
npm run lint
npm run format:check
npm run test
```
Expected: all pass. `session.route` is typed as `RouteFeature | undefined` and matches the component prop.

- [ ] **Step 4: Commit**

```bash
git add "app/(app)/running/[id]/page.tsx"
git commit -m "feat(running): render route map on activity detail when present"
```

---

## Task 11: Web — full local CI gate

No new code; makes CI-green a checkpoint before opening the PR.

**Files:** none (verification only)

- [ ] **Step 1: Run the exact CI gate**

Run (from `/workspace/prog-strength-web`):
```bash
npm run lint
npm run format:check
npm run typecheck
npm run test
npm run build
```
Expected: every step exits 0. If `format:check` fails, run `npm run format` and re-check; commit the formatting as `style: format`. If `build` surfaces a maplibre SSR issue (it should not — the component is `"use client"` and the map is created inside `useEffect`), confirm no top-level browser-only access exists and fix by keeping all `maplibregl` usage inside the effect.

- [ ] **Step 2: Commit any formatting fixes**

```bash
git add -A && git commit -m "style: format"
```
(Skip if nothing changed.)

---

## Self-Review notes (spec coverage)

- **Extract `<Position>` lat/lon, persist nullable columns on `activity_trackpoints`** → Tasks 1, 2, 4, 5.
- **Compute + persist gap-aware RDP GeoJSON `MultiLineString` on the activity row** → Tasks 1, 3, 4, 5.
- **Expose route on detail endpoint; leave list/summary unchanged** → Task 6 (Route on `activityDTO` via `toActivityDTO`; `activityColumns` untouched; nil on list).
- **`POST /activities/tcx` includes route** → Task 6 (uploadTCX uses `withTrackpoints=true`; Activity from IngestTCX carries RouteGeoJSON).
- **Render on `/running/[id]` with MapLibre + OpenFreeMap when route exists; omit entirely otherwise** → Tasks 8–10 (`RunRouteMap` returns null without a route).
- **Outdoor → map fitted to bounds** → Task 9 (`fitBounds` on `properties.bounds`).
- **Indoor / no-GPS → identical page, no map/error** → Tasks 4 (nil route), 9/10 (null render).
- **Mid-run gap / teleport → discontinuous segments, never a straight line across the gap** → Task 3 (`splitSegments` on NULL and >50 m; always `MultiLineString`).
- **Materially lower simplified point count** → Task 3 (RDP ε = 5e-5; `TestBuildRoute_RealTrailFixtureReduces`).
- **Additive-only migration, no backfill** → Task 1.
- **PATCH environment does not recompute/clear route** → verified: `ChangeEnvironment` never touches `route_geojson`; no change made there.
- **No new Go dependency** → Task 3 (stdlib only); Task 7 asserts `go.mod`/`go.sum` no-drift.
- **No new palette tokens; accent stroke, dark basemap** → Task 9 (uses existing `--accent`/`--surface`/`--border`).
- **CI green in both repos** → Tasks 7, 11.

**Non-goals honored:** no mobile, no FIT, no map matching, no OSM enrichment, no backfill, no timeline/feed maps, no elevation overlay, no basemap accounts/keys. `run-route-geometry-capture.md` supersession + the SOW status flip are handled by the operator workflow (docs PR), not this plan's code tasks.
