# Weather in the Agent — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give the chat agent one MCP tool, `get_weather_forecast`, that reads the same OpenWeather forecast the dashboard tile reads, so it can answer "when should I schedule my long run this week?" with the week's conditions in hand.

**Architecture:** `GET /weather` in `prog-strength-api` gains two optional query parameters — `place` (free text, resolved against the user's saved labels first and the geocoder second) and `source` (`tile` | `agent`, a metrics label only). `api_weather_requests_total` gains a matching `source` label so one shared provider budget stays attributable per surface. `prog-strength-mcp` adds a transparent forwarder module in the mold of `nutrition_lookup.py`, translating the endpoint's two expected 404s into structured data instead of tool errors. `prog-strength-agent` joins the tool to `_TZ_AWARE_TOOLS` and teaches the `plan_workout` intent how weather bears on outdoor training. `prog-strength-infra` gains one Grafana panel splitting request outcomes by `source`. No migration, no config, no new table, no client change.

**Tech Stack:** Go 1.25 (chi, Prometheus client) · Python 3.12 (FastMCP 2, httpx, pytest, ruff) · Grafana provisioned dashboard JSON.

---

## Repository and branch map

Every repo below is worked on a branch named `feat/weather-agent-tool`, cut from `main`.

| Repo | Tasks | What lands |
| --- | --- | --- |
| `prog-strength-api` | 1, 2, 3 | `source` label + `source`/`place` query params on `GET /weather` |
| `prog-strength-mcp` | 4, 5 | `APIClient.get_weather_forecast` + the `get_weather_forecast` tool |
| `prog-strength-agent` | 6 | timezone injection + `plan_workout` weather rules |
| `prog-strength-infra` | 7 | one Grafana panel |
| `prog-strength-docs` | 8 | SOW status flip |

## Environment notes (this box)

The Go module needs `sqlite3.h` for the `sqlite-vec` cgo bindings and the box has
no `libsqlite3-dev`. Headers matching the installed `sqlite-libs` 3.40.0 have been
placed at `/home/developer/sqlite-include`. Every Go command in this plan must run
with:

```bash
export CGO_CFLAGS="-I/home/developer/sqlite-include"
```

The Python repos need `uv`, installed at `~/.local/bin/uv`. Prefix Python commands
with `export PATH="$HOME/.local/bin:$PATH"`.

## File Structure

**`prog-strength-api`**

- Create `internal/weather/source.go` — the `Source` type and its query-param parser. Its own file because this package follows one-type-per-file, and `Source` is a wire *and* metric concept that both the handler and the service depend on.
- Create `internal/weather/source_test.go` — `ParseSource` table test.
- Modify `internal/weather/metrics.go` — add the `source` label to `requestsTotal`.
- Modify `internal/weather/service.go` — thread `Source` through `Readings`, `Search`, `Reverse`, `geocode`.
- Modify `internal/weather/handler.go` — parse `source` and `place`, resolve a place, pass `Source` to the service.
- Modify `internal/weather/service_test.go`, `internal/weather/handler_test.go`, `internal/weather/activity_capture_test.go` (only if it references the changed signatures).

**`prog-strength-mcp`**

- Modify `src/prog_strength_mcp/api_client.py` — `APIError.code`, and a `get_weather_forecast` method.
- Create `src/prog_strength_mcp/weather.py` — the tool module (mirrors `nutrition_lookup.py`).
- Modify `src/prog_strength_mcp/server.py` — import + `weather.register(mcp, api)`.
- Create `tests/test_weather_tools.py`.

**`prog-strength-agent`**

- Modify `src/prog_strength_agent/model_harness.py` — `_TZ_AWARE_TOOLS`.
- Modify `src/prog_strength_agent/intents.py` — `_PLAN_WORKOUT_RULES`.
- Modify `tests/test_model_harness.py`, `tests/test_intents.py`.

**`prog-strength-infra`**

- Modify `monitoring/grafana/dashboards/weather.json` — one new panel, and a `y` shift for everything below it.

---

## Task 1: `source` label on `api_weather_requests_total` (prog-strength-api)

Introduce the `Source` type and thread it through every `requestsTotal`
increment. This task does **not** read the query parameter yet — every handler
call site passes `SourceTile`, so behavior is unchanged and the diff is purely
the plumbing. Task 2 wires the parameter.

**Files:**
- Create: `/workspace/prog-strength-api/internal/weather/source.go`
- Create: `/workspace/prog-strength-api/internal/weather/source_test.go`
- Modify: `/workspace/prog-strength-api/internal/weather/metrics.go` (the `requestsTotal` var, ~line 21)
- Modify: `/workspace/prog-strength-api/internal/weather/service.go` (`Readings`, `Search`, `Reverse`, `geocode`)
- Modify: `/workspace/prog-strength-api/internal/weather/handler.go` (`readings`, `search`, `reverse`)
- Modify: `/workspace/prog-strength-api/internal/weather/service_test.go` (call sites + the `requestsTotal` assertions at ~lines 465, 758, 823)

- [ ] **Step 1: Cut the branch**

```bash
cd /workspace/prog-strength-api
git checkout -b feat/weather-agent-tool
```

- [ ] **Step 2: Write the failing test for `ParseSource`**

Create `internal/weather/source_test.go`:

```go
package weather

import "testing"

func TestParseSource(t *testing.T) {
	tests := []struct {
		name string
		raw  string
		want Source
		ok   bool
	}{
		{"absent defaults to tile", "", SourceTile, true},
		{"explicit tile", "tile", SourceTile, true},
		{"agent", "agent", SourceAgent, true},
		{"unknown value rejected", "mobile", "", false},
		{"case sensitive", "Agent", "", false},
		{"whitespace is not trimmed away into a default", " ", "", false},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, ok := ParseSource(tt.raw)
			if ok != tt.ok {
				t.Fatalf("ParseSource(%q) ok = %v, want %v", tt.raw, ok, tt.ok)
			}
			if got != tt.want {
				t.Fatalf("ParseSource(%q) = %q, want %q", tt.raw, got, tt.want)
			}
		})
	}
}
```

- [ ] **Step 3: Run the test to verify it fails**

```bash
export CGO_CFLAGS="-I/home/developer/sqlite-include"
cd /workspace/prog-strength-api && go test ./internal/weather/ -run TestParseSource
```

Expected: FAIL — `undefined: Source`, `undefined: ParseSource`, `undefined: SourceTile`.

- [ ] **Step 4: Create `internal/weather/source.go`**

```go
package weather

// Source names the surface a weather request came from. The set is closed
// because the value becomes a Prometheus label: the tile and the agent share
// one provider budget, one cache, and one daily ceiling, and the only new
// operational question this feature creates is "is chat eating the tile's
// budget?".
type Source string

const (
	SourceTile  Source = "tile"
	SourceAgent Source = "agent"
)

// ParseSource resolves the ?source= query value, reporting false for anything
// outside the closed set. An absent value is the tile: that surface shipped
// first, so defaulting this way is what lets the existing web client keep
// working unedited. An unrecognized value is rejected rather than coerced to
// the default — a metric label taken from arbitrary caller input is an
// unbounded-cardinality hazard, and a silent coercion would also hide a
// client bug behind plausible-looking data.
func ParseSource(raw string) (Source, bool) {
	switch Source(raw) {
	case "":
		return SourceTile, true
	case SourceTile, SourceAgent:
		return Source(raw), true
	default:
		return "", false
	}
}
```

- [ ] **Step 5: Run the test to verify it passes**

```bash
export CGO_CFLAGS="-I/home/developer/sqlite-include"
cd /workspace/prog-strength-api && go test ./internal/weather/ -run TestParseSource
```

Expected: PASS.

- [ ] **Step 6: Add the label to the metric**

In `internal/weather/metrics.go`, change the `requestsTotal` declaration. Replace:

```go
var requestsTotal = prometheus.NewCounterVec(
	prometheus.CounterOpts{
		Name: "api_weather_requests_total",
		Help: "Weather tile requests by final disposition (cache_hit/served/served_stale/budget_exhausted/disabled/failed).",
	},
	[]string{"outcome"},
)
```

with:

```go
var requestsTotal = prometheus.NewCounterVec(
	prometheus.CounterOpts{
		Name: "api_weather_requests_total",
		Help: "Weather requests by final disposition (cache_hit/served/served_stale/budget_exhausted/disabled/failed) and requesting surface (tile/agent).",
	},
	[]string{"outcome", "source"},
)
```

Also extend the doc comment above it. The existing comment starts
`// requestsTotal counts weather tile requests by final disposition:` — change
that opening line to:

```go
// requestsTotal counts weather requests by final disposition and by the
// surface that asked (see source.go). The two surfaces share one budget and
// one cache, so `source` splits attribution without pretending they are
// isolated; every other series in this file stays unlabelled for that reason.
//
// Disposition:
```

- [ ] **Step 7: Thread `Source` through the service**

In `internal/weather/service.go`:

1. `func (s *Service) Readings(ctx context.Context, lat, lon float64) Reading` becomes
   `func (s *Service) Readings(ctx context.Context, lat, lon float64, source Source) Reading`.
2. `func (s *Service) Search(ctx context.Context, query string, limit int) ([]GeoResult, Status, error)` becomes
   `func (s *Service) Search(ctx context.Context, query string, limit int, source Source) ([]GeoResult, Status, error)`, and passes `source` to `s.geocode`.
3. `func (s *Service) Reverse(ctx context.Context, lat, lon float64) ([]GeoResult, Status, error)` becomes
   `func (s *Service) Reverse(ctx context.Context, lat, lon float64, source Source) ([]GeoResult, Status, error)`, and passes `source` to `s.geocode`.
4. `func (s *Service) geocode(ctx context.Context, key string, endpoint Endpoint, limit int, call func(ctx context.Context) ([]GeoResult, error))` gains a `source Source` parameter placed immediately before `call`.
5. Every `requestsTotal.WithLabelValues("<outcome>").Inc()` becomes
   `requestsTotal.WithLabelValues("<outcome>", string(source)).Inc()`. There are
   six in `Readings` and six in `geocode` — do not miss the one inside the
   `serveStale` closure.

- [ ] **Step 8: Update the handler call sites**

In `internal/weather/handler.go`:

- `readings`: `reading := h.svc.Readings(ctx, loc.Lat, loc.Lon)` → `reading := h.svc.Readings(ctx, loc.Lat, loc.Lon, SourceTile)`. Task 2 replaces the constant with the parsed value.
- `search`: `results, status, err := h.svc.Search(ctx, q, limit)` → `h.svc.Search(ctx, q, limit, SourceTile)`.
- `reverse`: `results, status, err := h.svc.Reverse(ctx, lat, lon)` → `h.svc.Reverse(ctx, lat, lon, SourceTile)`.

Add this comment once, above the `search` call site, so the constants read as a
decision rather than a stub:

```go
	// /weather/search and /weather/reverse are the dashboard popover's
	// location picker; there is no agent tool for either, so they are
	// attributed to the tile rather than growing a ?source= of their own.
```

- [ ] **Step 9: Update the existing tests for the new signatures**

`internal/weather/service_test.go` calls `svc.Readings(...)`, `svc.Search(...)`
and `svc.Reverse(...)` in many places. Add `SourceTile` as the trailing argument
to every one of them. The three `testutil.ToFloat64(requestsTotal.WithLabelValues(...))`
reads (around lines 465, 758 and 823) need the matching second label value —
`requestsTotal.WithLabelValues("served_stale", "tile")` and, in the
`TestReadings_OutcomeMetric`-style helper around line 823,
`requestsTotal.WithLabelValues(label, "tile")`.

Find every call site with:

```bash
cd /workspace/prog-strength-api
grep -rn "\.Readings(\|\.Search(\|\.Reverse(\|requestsTotal.WithLabelValues" internal/
```

- [ ] **Step 10: Add a test asserting the label is populated**

Append to `internal/weather/source_test.go`:

```go
// The label is what makes the shared budget attributable, so assert it lands
// on the counter rather than trusting the call sites.
func TestReadingsLabelsTheRequestingSource(t *testing.T) {
	before := func(source Source) float64 {
		return testutil.ToFloat64(requestsTotal.WithLabelValues("served", string(source)))
	}
	agentBefore, tileBefore := before(SourceAgent), before(SourceTile)

	f := newServiceFixture(t, svcCfg())
	f.svc.Readings(context.Background(), testLat, testLon, SourceAgent)

	if got := before(SourceAgent) - agentBefore; got != 1 {
		t.Fatalf("agent-sourced served count delta = %v, want 1", got)
	}
	if got := before(SourceTile) - tileBefore; got != 0 {
		t.Fatalf("tile served count moved by %v on an agent request, want 0", got)
	}
}
```

Imports needed at the top of `source_test.go`: `context`, `testing`, and
`github.com/prometheus/client_golang/prometheus/testutil`.

`newServiceFixture` / `svcCfg` / `testLat` / `testLon` are the existing helpers
in `service_test.go` — read that file and use the real names and signature; if
the fixture constructor differs, adapt this test to it rather than adding a
second fixture.

- [ ] **Step 11: Run the full package suite**

```bash
export CGO_CFLAGS="-I/home/developer/sqlite-include"
cd /workspace/prog-strength-api && go build ./... && go test ./internal/weather/
```

Expected: `ok github.com/Prog-Strength/prog-strength-api/internal/weather`.

- [ ] **Step 12: Commit**

```bash
cd /workspace/prog-strength-api
git add internal/weather/
git commit -m "feat(weather): attribute weather requests to their calling surface"
```

---

## Task 2: `source` query parameter on `GET /weather` (prog-strength-api)

**Files:**
- Modify: `/workspace/prog-strength-api/internal/weather/handler.go` (`readings`)
- Modify: `/workspace/prog-strength-api/internal/weather/handler_test.go`

- [ ] **Step 1: Write the failing tests**

Append to `internal/weather/handler_test.go`:

```go
func TestReadingsRejectsUnknownSource(t *testing.T) {
	f := newHandlerFixture(t, handlerCfg(), user.DistanceUnitMiles)
	f.seed(t, denverLocation())

	w := f.do(t, http.MethodGet, "/weather?timezone=America/Denver&source=mobile", "")

	wantError(t, w, http.StatusBadRequest, "source")
}

func TestReadingsAgentSourceIsAccepted(t *testing.T) {
	f := newHandlerFixture(t, handlerCfg(), user.DistanceUnitMiles)
	f.seed(t, denverLocation())

	w := f.do(t, http.MethodGet, "/weather?timezone=America/Denver&source=agent", "")

	if w.Code != http.StatusOK {
		t.Fatalf("status = %d, want 200 (body: %s)", w.Code, w.Body.String())
	}
	var got readingsResponse
	decodeData(t, w, &got)
	if got.Status != StatusOK {
		t.Fatalf("status = %q, want %q", got.Status, StatusOK)
	}
}

// An absent ?source= must keep working untouched — the shipped web tile does
// not send one and this feature is explicitly a no-op for it.
func TestReadingsDefaultsSourceToTile(t *testing.T) {
	f := newHandlerFixture(t, handlerCfg(), user.DistanceUnitMiles)
	f.seed(t, denverLocation())

	w := f.do(t, http.MethodGet, "/weather?timezone=America/Denver", "")

	if w.Code != http.StatusOK {
		t.Fatalf("status = %d, want 200 (body: %s)", w.Code, w.Body.String())
	}
}
```

`wantError` is the existing helper in `handler_test.go` — read its signature
(around line 120) before writing these and match it exactly.

- [ ] **Step 2: Run to verify the rejection test fails**

```bash
export CGO_CFLAGS="-I/home/developer/sqlite-include"
cd /workspace/prog-strength-api && go test ./internal/weather/ -run TestReadingsRejectsUnknownSource
```

Expected: FAIL — the handler currently ignores `source` and answers 200.

- [ ] **Step 3: Parse the parameter in the handler**

In `internal/weather/handler.go`, inside `readings`, immediately after the
timezone validation block (`if _, err := time.LoadLocation(tz); err != nil { … }`)
and **before** the `if !h.cfg.Enabled` short-circuit, insert:

```go
	// Validated at the boundary, ahead of the kill switch: the value becomes a
	// metric label, so a caller sending a bad one should hear about it whether
	// or not the feature happens to be on today.
	src, ok := ParseSource(r.URL.Query().Get("source"))
	if !ok {
		httpresp.Error(w, http.StatusBadRequest, `source must be "tile" or "agent"`)
		return
	}
```

Then change `reading := h.svc.Readings(ctx, loc.Lat, loc.Lon, SourceTile)` to
`reading := h.svc.Readings(ctx, loc.Lat, loc.Lon, src)`.

Note `ok` is already used as the name of the `auth.UserIDFrom` result earlier in
this function; reuse of the name via `:=` on a two-value assignment where `src`
is new is legal Go and shadows nothing, but if `golangci-lint`'s shadow check
objects, rename to `srcOK`.

- [ ] **Step 4: Run the tests**

```bash
export CGO_CFLAGS="-I/home/developer/sqlite-include"
cd /workspace/prog-strength-api && go test ./internal/weather/
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd /workspace/prog-strength-api
git add internal/weather/
git commit -m "feat(weather): accept a source parameter on GET /weather"
```

---

## Task 3: `place` resolution on `GET /weather` (prog-strength-api)

**Files:**
- Modify: `/workspace/prog-strength-api/internal/weather/handler.go` (`readings`, plus a new `resolvePlace` method)
- Modify: `/workspace/prog-strength-api/internal/weather/handler_test.go`

- [ ] **Step 1: Write the failing tests**

Append to `internal/weather/handler_test.go`:

```go
// A user who says "Denver" and has Denver saved gets THEIR Denver — the
// coordinates they curated — and it costs no provider call at all.
func TestReadingsPlaceMatchesSavedLabelWithoutGeocoding(t *testing.T) {
	f := newHandlerFixture(t, handlerCfg(), user.DistanceUnitMiles)
	saved := f.seed(t, denverLocation())

	w := f.do(t, http.MethodGet, "/weather?timezone=America/Denver&place=denver", "")

	if w.Code != http.StatusOK {
		t.Fatalf("status = %d, want 200 (body: %s)", w.Code, w.Body.String())
	}
	var got readingsResponse
	decodeData(t, w, &got)
	if got.Location.ID != saved[0].ID {
		t.Fatalf("location id = %q, want the saved location %q", got.Location.ID, saved[0].ID)
	}
	if n := f.provider.calls[EndpointGeocodeDirect]; n != 0 {
		t.Fatalf("geocode calls = %d, want 0 for a place the user already saved", n)
	}
}

func TestReadingsPlaceFallsThroughToGeocoding(t *testing.T) {
	f := newHandlerFixture(t, handlerCfg(), user.DistanceUnitMiles)
	f.seed(t, denverLocation())
	f.provider.direct = []GeoResult{{Name: "Boulder", State: "Colorado", Country: "US", Lat: testLat, Lon: testLon}}

	w := f.do(t, http.MethodGet, "/weather?timezone=America/Denver&place=Boulder", "")

	if w.Code != http.StatusOK {
		t.Fatalf("status = %d, want 200 (body: %s)", w.Code, w.Body.String())
	}
	var got readingsResponse
	decodeData(t, w, &got)
	if got.Location.ID != "" {
		t.Fatalf("location id = %q, want an empty string for an unsaved place", got.Location.ID)
	}
	if got.Location.Label != "Boulder" {
		t.Fatalf("location label = %q, want %q", got.Location.Label, "Boulder")
	}
	if got.Location.State == nil || *got.Location.State != "Colorado" {
		t.Fatalf("location state = %v, want Colorado", got.Location.State)
	}
	if got.Location.Country != "US" {
		t.Fatalf("location country = %q, want US", got.Location.Country)
	}
	if n := f.provider.calls[EndpointGeocodeDirect]; n != 1 {
		t.Fatalf("geocode calls = %d, want 1", n)
	}
}

// An empty `id` must survive JSON encoding as "" rather than being dropped, or
// a client that reads location.id gets undefined instead of a known-empty value.
func TestReadingsAdHocPlaceKeepsAnEmptyIDInTheBody(t *testing.T) {
	f := newHandlerFixture(t, handlerCfg(), user.DistanceUnitMiles)
	f.seed(t, denverLocation())
	f.provider.direct = []GeoResult{{Name: "Moab", Country: "US", Lat: testLat, Lon: testLon}}

	w := f.do(t, http.MethodGet, "/weather?timezone=America/Denver&place=Moab", "")

	if !strings.Contains(w.Body.String(), `"id":""`) {
		t.Fatalf("body does not carry an empty id: %s", w.Body.String())
	}
}

func TestReadingsUnknownPlaceIs404(t *testing.T) {
	f := newHandlerFixture(t, handlerCfg(), user.DistanceUnitMiles)
	f.seed(t, denverLocation())
	f.provider.direct = nil

	w := f.do(t, http.MethodGet, "/weather?timezone=America/Denver&place=Xyzzyville", "")

	if w.Code != http.StatusNotFound {
		t.Fatalf("status = %d, want 404 (body: %s)", w.Code, w.Body.String())
	}
	if !strings.Contains(w.Body.String(), "place_not_found") {
		t.Fatalf("body missing the place_not_found code: %s", w.Body.String())
	}
}

// A geocode that degraded (budget spent, provider down) is not evidence the
// place does not exist. Saying "no such place" there would have the agent deny
// a real city, so the disposition is reported instead.
func TestReadingsDegradedGeocodeReportsStatusNotPlaceNotFound(t *testing.T) {
	cfg := handlerCfg()
	cfg.CountGeocodingCalls = true
	cfg.DailyCallBudget = 0
	f := newHandlerFixture(t, cfg, user.DistanceUnitMiles)
	f.seed(t, denverLocation())

	w := f.do(t, http.MethodGet, "/weather?timezone=America/Denver&place=Boulder", "")

	if w.Code != http.StatusOK {
		t.Fatalf("status = %d, want 200 (body: %s)", w.Code, w.Body.String())
	}
	var got readingsResponse
	decodeData(t, w, &got)
	if got.Status != StatusBudgetExhausted {
		t.Fatalf("status = %q, want %q", got.Status, StatusBudgetExhausted)
	}
}

func TestReadingsPlaceAndLocationIDTogetherIs400(t *testing.T) {
	f := newHandlerFixture(t, handlerCfg(), user.DistanceUnitMiles)
	saved := f.seed(t, denverLocation())

	target := "/weather?timezone=America/Denver&place=Boulder&location_id=" + saved[0].ID
	w := f.do(t, http.MethodGet, target, "")

	wantError(t, w, http.StatusBadRequest, "location_id")
}

// A user with no saved locations can still ask about a place by name — that is
// the whole point of the parameter, so `place` must not be gated on the
// no_locations check that guards the default path.
func TestReadingsPlaceWorksWithNoSavedLocations(t *testing.T) {
	f := newHandlerFixture(t, handlerCfg(), user.DistanceUnitMiles)

	w := f.do(t, http.MethodGet, "/weather?timezone=America/Denver&place=Boulder", "")

	if w.Code != http.StatusOK {
		t.Fatalf("status = %d, want 200 (body: %s)", w.Code, w.Body.String())
	}
}
```

Check the real names of the config fields used in
`TestReadingsDegradedGeocodeReportsStatusNotPlaceNotFound` against
`config.WeatherConfig` and `svcCfg()` before running — `CountGeocodingCalls` and
`DailyCallBudget` are the expected names, and `activeCeiling(cfg)` in
`service.go` shows how the ceiling is derived. Set whatever combination
actually makes `budget.Reserve` return `ErrBudgetExhausted`.

- [ ] **Step 2: Run to verify they fail**

```bash
export CGO_CFLAGS="-I/home/developer/sqlite-include"
cd /workspace/prog-strength-api && go test ./internal/weather/ -run 'TestReadings(Place|UnknownPlace|AdHoc|Degraded)'
```

Expected: FAIL — `place` is currently ignored, so every one of these gets the
first saved location or a `no_locations` 404.

- [ ] **Step 3: Add the place cap constant**

In `internal/weather/handler.go`, next to the existing
`maxLocationLabelLen` / `maxLocationRegionLen` block, add:

```go
// maxPlaceQueryLen caps the free-text ?place= at the same length
// /weather/search caps `q`: the value reaches the geocoder and becomes part of
// a cache key, so it is bounded at the boundary rather than downstream.
const maxPlaceQueryLen = 200
```

- [ ] **Step 4: Rework location resolution in `readings`**

Replace the existing resolution block in `readings` — the one that currently reads:

```go
	locs, err := h.locations.List(ctx, userID)
	if err != nil {
		httpresp.ServerError(w, ctx, "list weather locations", err)
		return
	}
	var loc *Location
	if wantID := r.URL.Query().Get("location_id"); wantID != "" {
		for i := range locs {
			if locs[i].ID == wantID {
				loc = &locs[i]
				break
			}
		}
		if loc == nil {
			httpresp.ErrorWithCode(w, http.StatusNotFound, "location not found", "location_not_found")
			return
		}
	} else {
		if len(locs) == 0 {
			httpresp.ErrorWithCode(w, http.StatusNotFound, "no saved locations", "no_locations")
			return
		}
		loc = &locs[0]
	}
```

with:

```go
	locs, err := h.locations.List(ctx, userID)
	if err != nil {
		httpresp.ServerError(w, ctx, "list weather locations", err)
		return
	}
	var loc *Location
	switch wantID := r.URL.Query().Get("location_id"); {
	case wantID != "":
		for i := range locs {
			if locs[i].ID == wantID {
				loc = &locs[i]
				break
			}
		}
		if loc == nil {
			httpresp.ErrorWithCode(w, http.StatusNotFound, "location not found", "location_not_found")
			return
		}
	case place != "":
		var resolved bool
		if loc, resolved = h.resolvePlace(w, r, place, locs, src); !resolved {
			return
		}
	default:
		if len(locs) == 0 {
			httpresp.ErrorWithCode(w, http.StatusNotFound, "no saved locations", "no_locations")
			return
		}
		loc = &locs[0]
	}
```

And directly after the `ParseSource` block added in Task 2, insert the `place`
validation (it must run before the kill-switch short-circuit, same reasoning):

```go
	// Two location selectors in one request is a caller bug; silently
	// preferring one would hide it.
	place := strings.TrimSpace(r.URL.Query().Get("place"))
	if place != "" && r.URL.Query().Get("location_id") != "" {
		httpresp.Error(w, http.StatusBadRequest, "place and location_id are mutually exclusive")
		return
	}
	if utf8.RuneCountInString(place) > maxPlaceQueryLen {
		httpresp.Error(w, http.StatusBadRequest,
			fmt.Sprintf("place is too long (max %d characters)", maxPlaceQueryLen))
		return
	}
```

- [ ] **Step 5: Add `resolvePlace`**

Add this method to `internal/weather/handler.go`, immediately after `readings`:

```go
// resolvePlace turns free text into a location. Saved labels win: a user who
// says "Denver" and has Denver saved should get the coordinates they curated on
// the dashboard, not whatever the geocoder returns for the string — and that
// match costs no provider call, no budget reservation, and no cache lookup.
// Only a name the account does not already know reaches the geocoder.
//
// It reports false when it has already written the response.
func (h *Handler) resolvePlace(w http.ResponseWriter, r *http.Request, place string, saved []Location, src Source) (*Location, bool) {
	ctx := r.Context()
	for i := range saved {
		if strings.EqualFold(saved[i].Label, place) {
			return &saved[i], true
		}
	}

	// limit 1: the caller named one place and gets one answer. Offering
	// candidates is /weather/search's job, and it is the surface with a human
	// to choose between them.
	results, status, err := h.svc.Search(ctx, place, 1, src)
	if err != nil {
		httpresp.ServerError(w, ctx, "weather place lookup", err)
		return nil, false
	}
	if len(results) == 0 {
		if status != StatusOK && status != StatusStale {
			// The geocode degraded — feature off, budget spent, provider down.
			// That is not evidence the place does not exist, and answering
			// "no such place" would have the caller deny a real city. Report
			// the disposition instead and let it say what actually happened.
			httpresp.OK(w, "weather readings", readingsResponse{Status: status})
			return nil, false
		}
		httpresp.ErrorWithCode(w, http.StatusNotFound, "place not found", "place_not_found")
		return nil, false
	}

	g := results[0]
	// No ID: this place is not a row anywhere. It stays an empty string rather
	// than becoming a pointer field on locationRefPayload, so the response
	// shape is identical for saved and ad-hoc locations — the MCP forwarder
	// has no branch and no existing consumer sees a type change.
	loc := Location{Label: g.Name, Country: g.Country, Lat: g.Lat, Lon: g.Lon}
	if g.State != "" {
		state := g.State
		loc.State = &state
	}
	return &loc, true
}
```

- [ ] **Step 6: Run the tests**

```bash
export CGO_CFLAGS="-I/home/developer/sqlite-include"
cd /workspace/prog-strength-api && go build ./... && go test ./internal/weather/
```

Expected: PASS, all tests in the package.

- [ ] **Step 7: Run the whole local gate**

```bash
export CGO_CFLAGS="-I/home/developer/sqlite-include"
cd /workspace/prog-strength-api
gofmt -l internal/ cmd/
go vet ./...
golangci-lint run
go mod tidy && git diff --exit-code go.mod go.sum
go test ./...
```

Expected: `gofmt -l` prints nothing; `go vet` silent; golangci-lint reports
`0 issues`; no `go.mod`/`go.sum` diff; all packages `ok`. Fix any finding in the
code — do not add `//nolint`, do not disable a rule, do not skip a test.

- [ ] **Step 8: Commit**

```bash
cd /workspace/prog-strength-api
git add internal/weather/
git commit -m "feat(weather): resolve a free-text place on GET /weather"
```

---

## Task 4: `APIClient.get_weather_forecast` (prog-strength-mcp)

**Files:**
- Modify: `/workspace/prog-strength-mcp/src/prog_strength_mcp/api_client.py` (`APIError`, plus a new method)
- Create: `/workspace/prog-strength-mcp/tests/test_weather_tools.py`

- [ ] **Step 1: Cut the branch**

```bash
cd /workspace/prog-strength-mcp
git checkout -b feat/weather-agent-tool
```

- [ ] **Step 2: Write the failing client tests**

Create `tests/test_weather_tools.py`:

```python
"""Weather forwarder tests: the client's request shape and the tool's
translation of the two expected 404s into structured data.

Mirrors tests/test_nutrition_lookup.py — the same forwarder pattern, so
the same boundaries are worth pinning.
"""

import httpx
import pytest
import respx

from prog_strength_mcp.api_client import APIClient, APIError

BASE = "http://api.test"


@pytest.fixture
async def client():
    api = APIClient(base_url=BASE)
    yield api
    await api.aclose()


@respx.mock
async def test_get_weather_forecast_sends_agent_source_and_timezone(client):
    route = respx.get(f"{BASE}/weather").mock(
        return_value=httpx.Response(
            200,
            json={"data": {"status": "ok", "location": {"id": "", "label": "Boulder"}}},
            headers={"X-Request-ID": "req-1"},
        )
    )

    out = await client.get_weather_forecast(
        "Bearer t", timezone="America/Denver", place="Boulder"
    )

    params = route.calls[0].request.url.params
    assert params["timezone"] == "America/Denver"
    assert params["source"] == "agent"
    assert params["place"] == "Boulder"
    assert route.calls[0].request.headers["authorization"] == "Bearer t"
    assert out["status"] == "ok"
    assert out["request_id"] == "req-1"


@respx.mock
async def test_get_weather_forecast_omits_place_when_absent(client):
    route = respx.get(f"{BASE}/weather").mock(
        return_value=httpx.Response(200, json={"data": {"status": "ok"}})
    )

    await client.get_weather_forecast("Bearer t", timezone="America/Denver")

    assert "place" not in route.calls[0].request.url.params


@respx.mock
async def test_get_weather_forecast_raises_with_the_envelope_code(client):
    respx.get(f"{BASE}/weather").mock(
        return_value=httpx.Response(
            404,
            json={"error": "no saved locations", "code": "no_locations"},
            headers={"X-Request-ID": "req-2"},
        )
    )

    with pytest.raises(APIError) as excinfo:
        await client.get_weather_forecast("Bearer t", timezone="America/Denver")

    assert excinfo.value.status_code == 404
    assert excinfo.value.code == "no_locations"
    assert excinfo.value.request_id == "req-2"


@respx.mock
async def test_get_weather_forecast_survives_a_non_json_error_body(client):
    respx.get(f"{BASE}/weather").mock(return_value=httpx.Response(502, text="bad gateway"))

    with pytest.raises(APIError) as excinfo:
        await client.get_weather_forecast("Bearer t", timezone="America/Denver")

    assert excinfo.value.status_code == 502
    assert excinfo.value.code == ""
```

Read `tests/test_nutrition_lookup.py` first and match its fixture style — if it
constructs the client differently (e.g. without an async fixture), follow that
file rather than the sketch above.

- [ ] **Step 3: Run to verify they fail**

```bash
export PATH="$HOME/.local/bin:$PATH"
cd /workspace/prog-strength-mcp && uv run pytest tests/test_weather_tools.py -q
```

Expected: FAIL — `AttributeError: 'APIClient' object has no attribute 'get_weather_forecast'`.

- [ ] **Step 4: Add `code` to `APIError`**

In `src/prog_strength_mcp/api_client.py`, replace the `APIError` class body with:

```python
class APIError(RuntimeError):
    """Raised when the API returns a non-2xx response.

    `request_id` is the API's X-Request-ID for the failed call when the
    caller captured it — threaded through so even failures are traceable
    in CloudWatch.

    `code` is the API envelope's machine-readable `code` when it carried
    one. Only callers that must branch on the precise reason populate it:
    the weather tool has to tell "you have saved no locations" apart from
    "I don't know that place", and both arrive as a 404 whose human
    message is not a contract.
    """

    def __init__(self, status_code: int, message: str, request_id: str = "", code: str = ""):
        super().__init__(f"api returned {status_code}: {message}")
        self.status_code = status_code
        self.message = message
        self.request_id = request_id
        self.code = code
```

- [ ] **Step 5: Add the client method**

Add to `APIClient`, in its own section at the end of the class (after
`get_training_snapshot`), with the section banner comment matching the file's
existing style:

```python
    # --- Weather ------------------------------------------------------

    async def get_weather_forecast(
        self,
        auth_header: str,
        *,
        timezone: str,
        place: str | None = None,
    ) -> dict[str, Any]:
        """GET /weather with `source=agent`. The Go API owns the OpenWeather
        provider, the shared daily call budget, and the coordinate-keyed
        cache; this client just forwards. `source` splits the shared budget's
        Grafana attribution between the dashboard tile and chat — it changes
        no behavior.

        `place` is optional free text; the API matches it against the user's
        saved location labels before spending a geocoding call, and falls
        back to their first saved location when it is omitted.

        Returns the readings object under `data` with the API's `request_id`
        attached. A failure raises APIError carrying the envelope's
        machine-readable `code` (`no_locations`, `place_not_found`) so the
        tool layer can turn an ordinary state into structured data instead
        of a tool error.
        """
        params: dict[str, str] = {"timezone": timezone, "source": "agent"}
        if place:
            params["place"] = place
        resp = await self._client.get(
            "/weather",
            params=params,
            headers={"Authorization": auth_header},
        )
        # Captured before the status check so failed reads stay traceable.
        request_id = resp.headers.get("x-request-id", "")
        if resp.status_code >= 400:
            try:
                body = resp.json()
            except ValueError:
                body = None
            if isinstance(body, dict):
                detail, code = body.get("error", resp.text), body.get("code", "")
            else:
                detail, code = resp.text, ""
            raise APIError(resp.status_code, detail, request_id=request_id, code=code)
        data = resp.json().get("data")
        out = data if isinstance(data, dict) else {}
        if request_id:
            out["request_id"] = request_id
        return out
```

- [ ] **Step 6: Run the tests**

```bash
export PATH="$HOME/.local/bin:$PATH"
cd /workspace/prog-strength-mcp && uv run pytest tests/test_weather_tools.py -q
```

Expected: 4 passed.

- [ ] **Step 7: Commit**

```bash
cd /workspace/prog-strength-mcp
git add src/prog_strength_mcp/api_client.py tests/test_weather_tools.py
git commit -m "feat(weather): forward GET /weather from the API client"
```

---

## Task 5: the `get_weather_forecast` tool (prog-strength-mcp)

**Files:**
- Create: `/workspace/prog-strength-mcp/src/prog_strength_mcp/weather.py`
- Modify: `/workspace/prog-strength-mcp/src/prog_strength_mcp/server.py`
- Modify: `/workspace/prog-strength-mcp/tests/test_weather_tools.py`

- [ ] **Step 1: Write the failing tool tests**

Append to `tests/test_weather_tools.py`. Read `tests/test_nutrition_lookup.py`
for how it drives a registered FastMCP tool (it builds a `FastMCP` instance,
calls `register`, and reaches the function through the tool registry) and mirror
that mechanism exactly — the sketch below shows the assertions, not the
plumbing:

```python
# --- tool boundary ----------------------------------------------------


class _StubAPI:
    """Records the forwarded call, or raises the APIError it was given."""

    def __init__(self, result=None, error=None):
        self.result = result
        self.error = error
        self.calls = []

    async def get_weather_forecast(self, auth_header, *, timezone, place=None):
        self.calls.append({"auth": auth_header, "timezone": timezone, "place": place})
        if self.error:
            raise self.error
        return self.result


async def test_tool_forwards_timezone_and_place(monkeypatch):
    api = _StubAPI(result={"status": "ok", "location": {"id": "", "label": "Boulder"}})
    tool = _weather_tool(api, auth="Bearer t")

    out = await tool(timezone="America/Denver", place="Boulder")

    assert api.calls == [{"auth": "Bearer t", "timezone": "America/Denver", "place": "Boulder"}]
    assert out["status"] == "ok"


async def test_tool_translates_no_locations_into_data():
    api = _StubAPI(error=APIError(404, "no saved locations", request_id="r-1", code="no_locations"))
    tool = _weather_tool(api, auth="Bearer t")

    out = await tool(timezone="America/Denver")

    assert out["error"] == "no_saved_locations"
    assert out["request_id"] == "r-1"


async def test_tool_translates_place_not_found_and_echoes_the_place():
    api = _StubAPI(error=APIError(404, "place not found", code="place_not_found"))
    tool = _weather_tool(api, auth="Bearer t")

    out = await tool(timezone="America/Denver", place="Xyzzyville")

    assert out["error"] == "place_not_found"
    assert out["place"] == "Xyzzyville"


async def test_tool_passes_degraded_statuses_straight_through():
    # disabled / budget_exhausted / stale / unavailable are 200s carrying a
    # status the model can speak to — the tool must not reinterpret them.
    for status in ("disabled", "budget_exhausted", "stale", "unavailable"):
        api = _StubAPI(result={"status": status})
        tool = _weather_tool(api, auth="Bearer t")
        out = await tool(timezone="America/Denver")
        assert out == {"status": status}


async def test_tool_raises_on_a_non_404_error():
    api = _StubAPI(error=APIError(500, "boom"))
    tool = _weather_tool(api, auth="Bearer t")

    with pytest.raises(RuntimeError, match="500"):
        await tool(timezone="America/Denver")


async def test_tool_raises_before_calling_the_api_without_an_auth_header():
    api = _StubAPI(result={"status": "ok"})
    tool = _weather_tool(api, auth="")

    with pytest.raises(RuntimeError, match="Authorization"):
        await tool(timezone="America/Denver")

    assert api.calls == []


async def test_tool_schema_requires_timezone_and_leaves_place_optional():
    # The harness injects timezone; place is the model's own choice.
    ...
```

`_weather_tool(api, auth)` is a helper you write in this file: it registers
`weather.register` on a fresh `FastMCP`, patches
`prog_strength_mcp.weather.get_http_headers` to return
`{"authorization": auth}` (or `{}` when auth is empty), and returns the
registered coroutine. Base it on however `test_nutrition_lookup.py` does the
equivalent; do not invent a second mechanism.

Fill in `test_tool_schema_requires_timezone_and_leaves_place_optional` with real
assertions against the registered tool's JSON schema — `timezone` in
`required`, `place` not in `required`.

- [ ] **Step 2: Run to verify they fail**

```bash
export PATH="$HOME/.local/bin:$PATH"
cd /workspace/prog-strength-mcp && uv run pytest tests/test_weather_tools.py -q
```

Expected: FAIL — `ModuleNotFoundError: No module named 'prog_strength_mcp.weather'`.

- [ ] **Step 3: Create `src/prog_strength_mcp/weather.py`**

```python
"""Weather domain: one MCP tool exposing the shipped GET /weather forecast
to the agent.

Transparent forwarder in the mold of nutrition_lookup — the Go API owns the
OpenWeather provider, the durable daily call budget, and the coordinate-keyed
cache; this module is plumbing. It deliberately holds no opinion about what
weather makes a good training session: this server is shared by every client,
and training judgment lives in the agent's prompt.

See prog-strength-docs/sows/weather-agent-tool.md.
"""

from typing import Annotated, Any

from fastmcp import FastMCP
from fastmcp.server.dependencies import get_http_headers
from pydantic import Field

from prog_strength_mcp.api_client import APIClient, APIError


def _auth_header_or_raise() -> str:
    """Pull the inbound Authorization header. The endpoint is auth-gated
    (it reads the user's saved locations and spends shared provider
    quota), so the tool forwards the user's Bearer token like every
    other domain tool.
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
    """Register the forecast tool on `mcp`, backed by `api`."""

    @mcp.tool
    async def get_weather_forecast(
        timezone: Annotated[
            str,
            Field(
                min_length=1,
                max_length=64,
                description=(
                    "The user's IANA timezone, e.g. 'America/Denver'. The "
                    "agent supplies this automatically — never guess one."
                ),
            ),
        ],
        place: Annotated[
            str | None,
            Field(
                default=None,
                max_length=200,
                description=(
                    "Optional free-text place the user named — 'Boulder', "
                    "'Moab, UT'. Omit it for the user's own location, which "
                    "is what they mean unless they say otherwise."
                ),
            ),
        ] = None,
    ) -> dict[str, Any]:
        """Get the weather forecast for the user's saved location, or for a
        place they name.

        Returns `current` conditions, an `hourly` strip, `today`'s high/low
        and sun times, and a `daily` multi-day forecast.

        Horizons differ and matter: `hourly` covers roughly the next 20
        hours only, while `daily` covers about 8 days. There is no hour-level
        data past the end of `hourly`.

        Field semantics: `units` states which temperature and wind units every
        number in the payload is already in (they follow the user's distance
        preference) — the values are not convertible without it, and nothing
        here needs converting. `precip_chance` is a percentage, not a
        probability. `sunrise` and `sunset` are RFC3339 UTC instants that bound
        when a session is possible in daylight. `location.id` is empty for a
        place that is not one of the user's saved locations. `request_id` is
        operational tracing metadata — never read it aloud.

        `status` is always present. `"ok"` means fresh. `"stale"` means the
        numbers came from an expired cache and may be dated — say so.
        `"disabled"` means the weather feature is switched off,
        `"budget_exhausted"` means today's provider call allowance is spent,
        and `"unavailable"` means the provider could not be reached; in those
        three the forecast fields are absent or thin.

        Returns `{"error": "no_saved_locations"}` when the user has saved no
        location and named no place — they can add one on their dashboard, or
        just say which city they are in. Returns
        `{"error": "place_not_found", "place": …}` when the named place cannot
        be resolved.
        """
        auth = _auth_header_or_raise()
        try:
            return await api.get_weather_forecast(auth, timezone=timezone, place=place)
        except APIError as e:
            # Both 404s are ordinary states, not faults. A user who has saved
            # no location is in a perfectly normal place, and a name the
            # geocoder does not know is something to say back — the model can
            # only do either if it receives a fact instead of a tool error.
            # This mirrors how nutrition_lookup adapts the API's 503.
            if e.status_code == 404 and e.code in ("no_locations", "place_not_found"):
                out: dict[str, Any] = {}
                if e.code == "no_locations":
                    out["error"] = "no_saved_locations"
                else:
                    out["error"] = "place_not_found"
                    out["place"] = place
                if e.request_id:
                    out["request_id"] = e.request_id
                return out
            raise RuntimeError(f"API error ({e.status_code}): {e.message}") from e
```

- [ ] **Step 4: Register it in `server.py`**

Add `weather` to the alphabetically-ordered `from prog_strength_mcp import (…)`
block (it goes after `training_snapshot` and before `whoop`), and add
`weather.register(mcp, api)` to the registration list in the same relative
position the other modules use.

- [ ] **Step 5: Run the tests**

```bash
export PATH="$HOME/.local/bin:$PATH"
cd /workspace/prog-strength-mcp
uv run pytest -q
uv run ruff check src tests
uv run ruff format --check src tests
```

Expected: all tests pass (149 before this branch, more now); ruff clean on both.
If `ruff format --check` flags files this branch did not touch, leave them alone
and only format what you changed.

- [ ] **Step 6: Commit**

```bash
cd /workspace/prog-strength-mcp
git add src/prog_strength_mcp/weather.py src/prog_strength_mcp/server.py tests/test_weather_tools.py
git commit -m "feat(weather): add the get_weather_forecast tool"
```

---

## Task 6: timezone injection and `plan_workout` weather rules (prog-strength-agent)

**Files:**
- Modify: `/workspace/prog-strength-agent/src/prog_strength_agent/model_harness.py:83`
- Modify: `/workspace/prog-strength-agent/src/prog_strength_agent/intents.py` (`_PLAN_WORKOUT_RULES`, ~line 238)
- Modify: `/workspace/prog-strength-agent/tests/test_model_harness.py`
- Modify: `/workspace/prog-strength-agent/tests/test_intents.py`

- [ ] **Step 1: Cut the branch**

```bash
cd /workspace/prog-strength-agent
git checkout -b feat/weather-agent-tool
```

- [ ] **Step 2: Write the failing tests**

Append to `tests/test_model_harness.py`, following the existing style in that
file (each test imports `_maybe_inject_timezone` locally):

```python
def test_inject_timezone_into_get_weather_forecast():
    from prog_strength_agent.model_harness import _maybe_inject_timezone

    out = _maybe_inject_timezone(
        "get_weather_forecast", {"place": "Boulder"}, "America/Denver"
    )
    assert out["timezone"] == "America/Denver"
    assert out["place"] == "Boulder"


def test_model_supplied_timezone_not_overwritten_for_weather():
    from prog_strength_agent.model_harness import _maybe_inject_timezone

    out = _maybe_inject_timezone(
        "get_weather_forecast",
        {"timezone": "Europe/London"},
        "America/Denver",
    )
    assert out["timezone"] == "Europe/London"
```

Append to `tests/test_intents.py`:

```python
def test_plan_workout_rules_carry_outdoor_weather_guidance():
    """The forecast is only useful if the model knows when to fetch it and
    what its horizon is; both live in the intent rules, not in MCP."""
    from prog_strength_agent.intents import IntentRegistry

    rules = IntentRegistry.rules_for("plan_workout")

    assert "get_weather_forecast" in rules
    lowered = rules.lower()
    assert "outdoor" in lowered
    assert "humidity" in lowered
    assert "sunrise" in lowered
    assert "stale" in lowered
    # The horizon asymmetry is the thing the model gets wrong by default:
    # inventing an hourly temperature for a day outside the hourly window.
    assert "20 hours" in lowered
```

`IntentRegistry.rules_for` may not exist — read `intents.py` and
`tests/test_intents.py` first and reach the rules string the way the existing
`test_plan_workout_prefetch_includes_catalog_and_recent_5` test does (it takes
`rules` off `await IntentRegistry.run("plan_workout", session)`). Use that
mechanism rather than adding an accessor.

- [ ] **Step 3: Run to verify they fail**

```bash
export PATH="$HOME/.local/bin:$PATH"
cd /workspace/prog-strength-agent
uv run pytest tests/test_model_harness.py tests/test_intents.py -q
```

Expected: FAIL — the tool is not in `_TZ_AWARE_TOOLS` and the rules say nothing
about weather.

- [ ] **Step 4: Add the tool to `_TZ_AWARE_TOOLS`**

In `src/prog_strength_agent/model_harness.py`, replace line 83:

```python
_TZ_AWARE_TOOLS = {"list_nutrition_log", "get_daily_macros", "get_training_snapshot"}
```

with:

```python
# get_weather_forecast is here for the same reason as the others: the client
# sends an IANA name, the server does every conversion, and the model never
# constructs a window itself.
_TZ_AWARE_TOOLS = {
    "list_nutrition_log",
    "get_daily_macros",
    "get_training_snapshot",
    "get_weather_forecast",
}
```

- [ ] **Step 5: Extend `_PLAN_WORKOUT_RULES`**

In `src/prog_strength_agent/intents.py`, replace `_PLAN_WORKOUT_RULES` with:

```python
_PLAN_WORKOUT_RULES = """\
The user wants to plan FUTURE training (not log a completed session). \
Use create_planned_workout once per training day, building the schedule \
in the user's timezone with RFC3339 windows; look up exercise slugs from \
the catalog below for any target agenda; space rest days sensibly. Only \
push to Google Calendar (schedule_workout_to_calendar) if the user \
explicitly asks.

OUTDOOR SESSIONS. Call get_weather_forecast when the session under \
discussion happens outdoors — running, cycling, hiking, outdoor \
conditioning — and especially when the user is asking WHICH DAY to do it. \
Do not call it for indoor lifting, machine cardio, or treadmill work. \
Pass `place` only when the user names somewhere other than where they \
train normally.

READING THE FORECAST. Heat and humidity together are the dominant limiters \
on endurance work — a humid 80F morning is harder than a dry 90F one. Wind \
costs pace into a headwind and inflates perceived effort. precip_chance is \
about safety and footing more than comfort. sunrise and sunset bound when a \
session is possible in daylight, so name a start time relative to them.

RESPECT THE HORIZON. The hourly strip covers roughly the next 20 hours. \
Past that, recommend a DAY and reason about time of day from that day's \
low, high, and sunrise — and say that is what you are doing. Never invent \
an hourly temperature for a day outside the hourly window. Name the date \
you mean. If status is "stale", say the numbers may be dated.

THE FORECAST IS ONE INPUT. Recovery, planned distance, and how \
heat-acclimated the user is are already in your context; the mildest day \
is not automatically the right day.\
"""
```

- [ ] **Step 6: Run the tests**

```bash
export PATH="$HOME/.local/bin:$PATH"
cd /workspace/prog-strength-agent
uv run pytest -q
uv run ruff check src tests evals
```

Expected: all tests pass (204 before this branch, more now). `ruff check` reports
**only** the 6 pre-existing `E501` findings in `tests/test_intents.py` lines
56/57/82 that are already on `main` — verify with
`git stash && uv run ruff check src tests evals; git stash pop` if in doubt. Do
not fix them here (unrelated, and out of this branch's scope) and do not add any
new finding.

- [ ] **Step 7: Commit**

```bash
cd /workspace/prog-strength-agent
git add src/prog_strength_agent/model_harness.py src/prog_strength_agent/intents.py tests/
git commit -m "feat(agent): teach plan_workout to read the weather forecast"
```

---

## Task 7: Grafana panel splitting request outcomes by source (prog-strength-infra)

The alert rules were checked first: `grep -n api_weather_requests_total
monitoring/grafana/provisioning/alerting/rules-weather.yml` returns nothing —
no provisioned rule matches on this series, so adding a label breaks no alert.
The one existing dashboard query is `sum by (outcome) (…)`, which aggregates
over the new label and is unaffected. Both facts belong in the PR body.

**Files:**
- Modify: `/workspace/prog-strength-infra/monitoring/grafana/dashboards/weather.json`

- [ ] **Step 1: Cut the branch**

```bash
cd /workspace/prog-strength-infra
git checkout -b feat/weather-agent-tool
```

- [ ] **Step 2: Read the neighbouring panel**

Read panel `id: 7` ("Request outcomes") in
`monitoring/grafana/dashboards/weather.json`. The new panel copies its
datasource, `fieldConfig`, `options`, and legend shape verbatim so the two read
as siblings. Note the file is **hand-maintained JSON with mixed formatting** —
edit it with targeted text edits, never a `json.load`/`json.dump` round-trip,
which would reformat the whole file and make the diff unreviewable.

- [ ] **Step 3: Insert the new panel**

Insert a new panel object immediately after panel `id: 9` ("Provider latency
p50/p95"), i.e. as the last panel of the Integration Health row, with
`"gridPos": {"h": 8, "w": 24, "x": 0, "y": 34}`:

```json
    {
      "id": 16,
      "type": "timeseries",
      "title": "Request outcomes by source",
      "description": "The dashboard tile and the chat agent share one provider budget, one cache, and one daily ceiling — this is the only panel that separates them. Read it when the budget row looks hot and you need to know which surface is spending it. `cache_hit` costs nothing; `served` and `served_stale` are the bands that consume calls. The `source` label is set by the API handler from the request's ?source= parameter and is a closed two-value set.",
      "gridPos": {"h": 8, "w": 24, "x": 0, "y": 34},
      "targets": [
        {
          "expr": "sum by (source, outcome) (rate(api_weather_requests_total[5m]))",
          "legendFormat": "{{source}} · {{outcome}}",
          "refId": "A"
        }
      ]
    }
```

Fill in `datasource`, `fieldConfig` and `options` by copying panel 7's, adjusting
only the title/description/expr/legendFormat above. Keep the file's existing
indentation and quoting style.

- [ ] **Step 4: Shift everything below it down by 8**

Every panel currently at `y >= 34` moves down 8 rows so the new panel does not
overlap. Make these exact edits:

| Panel | `y` before | `y` after |
| --- | --- | --- |
| row 102 "Cache Efficiency" | 34 | 42 |
| 10 "Cache hit rate (24h)" | 35 | 43 |
| 11 "Cache events by type" | 35 | 43 |
| 12 "Cache write errors" | 43 | 51 |
| row 103 "Activity Capture" | 51 | 59 |
| 13 "Activities with weather" | 52 | 60 |
| 14 "Pending capture" | 52 | 60 |
| 15 "Captures by result (24h)" | 52 | 60 |

- [ ] **Step 5: Verify the JSON and the layout**

```bash
cd /workspace/prog-strength-infra
python3 -c "
import json
d = json.load(open('monitoring/grafana/dashboards/weather.json'))
seen = []
for p in d['panels']:
    g = p['gridPos']
    for q in seen:
        h = q['gridPos']
        if g['y'] < h['y'] + h['h'] and h['y'] < g['y'] + g['h'] and \
           g['x'] < h['x'] + h['w'] and h['x'] < g['x'] + g['w']:
            raise SystemExit(f\"overlap: {p['title']!r} and {q['title']!r}\")
    seen.append(p)
print('panels:', len(d['panels']), '- no overlaps')
"
```

Expected: `panels: 17 - no overlaps`.

- [ ] **Step 6: Run the local gate**

```bash
cd /workspace/prog-strength-infra
pre-commit run --all-files
pre-commit run --all-files --hook-stage pre-push
```

Expected: every hook passes (the JSON/YAML hygiene hooks and the Grafana alert
rule validator are the ones this change can trip; `terraform_validate` and
`tflint` touch nothing here). If `pre-commit` cannot install its environments
offline, at minimum run the JSON check and the alert-rule validator directly and
say so in the PR body.

- [ ] **Step 7: Commit**

```bash
cd /workspace/prog-strength-infra
git add monitoring/grafana/dashboards/weather.json
git commit -m "feat(monitoring): split weather request outcomes by source"
```

---

## Task 8: mark the SOW shipped (prog-strength-docs)

Handled by the controller, not a subagent.

- [ ] **Step 1: Branch, edit, commit**

```bash
cd /workspace/prog-strength-docs
git checkout -b feat/weather-agent-tool
```

In `sows/weather-agent-tool.md`:
- frontmatter `status: draft` → `status: shipped`
- body `**Status**: Draft · **Last updated**: 2026-08-12` → `**Status**: Shipped · **Last updated**: 2026-08-12`

```bash
git add sows/weather-agent-tool.md plans/2026-08-12-weather-agent-tool.md
git commit -m "docs: mark weather-agent-tool as shipped"
```

`sows/weather-dashboard-tile.md` needs **no** edit: its "No MCP tool or agent
capability" non-goal already carries the `**Superseded** — the agent gets
weather in weather-agent-tool.md` amendment.

---

## Verification before opening PRs

Run each repo's own gate, in the repo, before pushing anything:

```bash
# prog-strength-api
export CGO_CFLAGS="-I/home/developer/sqlite-include"
cd /workspace/prog-strength-api
gofmt -l internal/ cmd/ && go vet ./... && golangci-lint run \
  && go mod tidy && git diff --exit-code go.mod go.sum && go test ./...

# prog-strength-mcp
export PATH="$HOME/.local/bin:$PATH"
cd /workspace/prog-strength-mcp && uv run pytest -q && uv run ruff check src tests

# prog-strength-agent
cd /workspace/prog-strength-agent && uv run pytest -q && uv run ruff check src tests evals

# prog-strength-infra
cd /workspace/prog-strength-infra && pre-commit run --all-files \
  && pre-commit run --all-files --hook-stage pre-push
```

The `prog-strength-api` push runs a pre-push hook if `pre-commit install` was
run in that clone. Never pass `--no-verify`. If a check fails, fix the code.
