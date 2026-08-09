# Activity Weather Conditions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Capture and durably store the weather conditions at the start of every outdoor
GPS activity — at import time going forward, and for the existing history through a
paced, resumable `cmd/weather-backfill` — then render it as a shared `WeatherRecap`
widget on `/running/[id]` and `/hiking/[id]`, expose the full activity detail (weather
included) to the chat agent through a new `get_activity` MCP tool, and add an "Activity
Capture" row to the shipped Grafana `weather.json` dashboard.

**Architecture:** Almost entirely additive on top of the shipped
`sows/weather-dashboard-tile.md` stack. One new table (`activity_weather`, 1:1 on
`activities`, migration `057`), one new method on the existing `weather.Provider` seam
(`Historical`), one new `Endpoint` constant that the existing metrics labels pick up for
free, and the existing `BudgetLedger` / kill switch / `Exporter` reused unchanged.
Historical readings are **immutable** — stored, never cached, never refreshed. A failed
import-time capture writes **no row**, which makes the backfill command the single
repair path for both history and import-time misses.

**Tech Stack:** Go 1.25 / chi / go-sqlite3 / prometheus client (API) · Next.js 16 /
React 19 / vitest (web) · FastMCP 2 / httpx / pytest (mcp) · Grafana dashboard JSON
(infra).

**Repos & branches:** `feat/activity-weather-conditions` in `prog-strength-api`,
`prog-strength-web`, `prog-strength-mcp`, `prog-strength-infra`, `prog-strength-docs`.

**Design system:** conforms to `design-system.md` § *Activity session-recap*. Existing
tokens only. **No new hues** — condition icons render in `--foreground` / `--muted`
neutrals; `--accent` stays edit/focus chrome; section kickers use the discipline hue
(`--discipline-run-dot` / `--discipline-hike-dot`). Beats with no data are omitted whole.

**Source spec:** `prog-strength-docs/sows/activity-weather-conditions.md`. Where this
plan and the SOW disagree, the SOW wins.

---

## Open Questions — resolutions adopted by this plan

The SOW leaves four open questions. Three take their tentative lean. The first cannot be
answered from inside this environment and is handled explicitly rather than silently.

1. **One Call 4.0 historical path and response shape.** The SOW says this "must be
   captured from a live call, not assumed". **There is no OpenWeather key available to
   this implementation run, so no live capture is possible.** The plan therefore:
   - Implements the path as `oneCallBasePath + "/timemachine"` with a `dt` unix-seconds
     parameter — the One Call *timemachine* surface, consistent with the sibling paths
     already in `openweather.go` (`/onecall/current`, `/onecall/timeline/1h`,
     `/onecall/timeline/1day`) and with the documented One Call historical endpoint.
   - Puts the path and the `dt` parameter behind **one constant each**, so correcting
     them after a live probe is a one-line change (this is the whole reason
     `oneCallBasePath` exists — see the file header).
   - Marks the fixtures in `openweather_test.go` **explicitly as hand-authored to the
     documented shape, NOT live-captured**, with a `RE-CAPTURE BEFORE ROLLOUT` note.
     Claiming a live capture that did not happen is the exact failure mode PR #124
     documents ("fixtures authored to match what the provider code assumed... the suite
     was confirming the implementation against itself").
   - Fails loudly on an empty or absent `data` array, exactly as `Daily` does, so a wrong
     shape is a visible error and never a plausible 0 °C reading.
   - Surfaces this in the API PR body as a **pre-merge verification item for the
     operator**: probe the endpoint once with the production key and re-capture the
     fixtures. `--dry-run` plus a single `--limit 1` backfill run is the cheap proof.
2. **History window and billing rate.** Assume the same rate (conservative for a spend
   cap). A definitive provider negative for an out-of-window timestamp becomes a terminal
   `unavailable` row. Concretely: a **400** response from the historical endpoint is
   mapped to `ErrNoHistoricalData` (definitive); every other failure is transient.
3. **Start point vs. route centroid.** Start point. `activity_weather.lat`/`lon` record
   which coordinate was used, so changing the rule later is a re-backfill.
4. **`precip_mm` when dry.** Hide. Zero precipitation is omitted from the metric strip.

---

## Global conventions (read before any task)

- **API repo** (`/workspace/prog-strength-api`): module
  `github.com/Prog-Strength/prog-strength-api`. Local gate before push:
  `gofmt -l .` (must be empty) `&& go build ./... && go vet ./... &&
  golangci-lint run --timeout=5m && go mod tidy` (no `go.mod`/`go.sum` drift) `&&
  go test ./...`. golangci-lint is CI-pinned at **v2.12.2**. DB tests use
  `internal/db/dbtest.New(t)`. Services take a `now func() time.Time` for injected
  clocks. IDs come from `internal/id.New()`. Responses use `internal/httpresp`.
  Comments explain *why*, never *what*. **No new third-party dependencies.**
  `pre-commit install --install-hooks --hook-type pre-commit --hook-type commit-msg
  --hook-type pre-push` before the first commit; never `--no-verify`.
- **Web repo** (`/workspace/prog-strength-web`): npm. Local gate:
  `npm run lint && npm run format:check && npm run typecheck && npm run test &&
  npm run build`. All API types live in `lib/api.ts`. Icons are hand-rolled
  `currentColor` SVGs (no icon library). Tests co-located `*.test.tsx`, vitest +
  Testing Library. Husky arms `pre-commit` (lint-staged + typecheck) and `commit-msg`
  (commitlint); never bypass them.
- **MCP repo** (`/workspace/prog-strength-mcp`): local gate: `ruff check src tests`,
  `ruff format --check src tests`, `pytest tests`. Requires Python ≥ 3.12; if the host
  only has an older interpreter, install one with `uv python install 3.12` and run
  through `uv run`. If the suite genuinely cannot be run locally, say so plainly in the
  PR body rather than claiming a green gate.
- **Infra repo** (`/workspace/prog-strength-infra`): local gate:
  `python3 monitoring/grafana/provisioning/alerting/validate_rules.py` and
  `python3 -c "import json;json.load(open('monitoring/grafana/dashboards/weather.json'))"`,
  plus `tflint --recursive` if any `.tf` changes (none expected here). `pre-commit
  install` arms commit + push hooks.
- **Commit messages:** conventional commits, lowercase subject
  (`feat(weather): …`, `feat(activity-detail): …`, `feat(mcp): …`,
  `feat(monitoring): …`, `refactor(weather): …`, `docs: …`).
- **Canonical units:** metric everywhere in storage and on the wire. The provider is
  called with `units=metric`; wind converts m/s → km/h **in the provider layer**.
  Weather follows the **raw-measurement convention** of the activity DTO (like
  `distance_meters`): stored metric, formatted client-side. This deliberately diverges
  from `GET /weather`, which converts server-side — the SOW explains why.
- **Never** touch `weather_cache` from the historical path. `activity_weather` is the
  durable store.

---

## Repo 1 — `prog-strength-api`

### Task 1: Migration `057_activity_weather.sql`

- [ ] Add `internal/db/migrations/057_activity_weather.sql`:

```sql
-- migrations/057_activity_weather.sql
-- Activity Weather Conditions: the conditions at the start of one outdoor GPS
-- activity, captured once and stored forever. See
-- sows/activity-weather-conditions.md.
--
-- One row per ATTEMPTED activity, not per successful reading: `status` is what
-- stops the import path and the backfill from re-spending budget on an activity
-- that has no answer. Historical weather is immutable, so this is a store and
-- not a cache — no TTL column, no eviction, and deliberately not weather_cache.
--
-- A 1:1 table rather than fourteen columns on `activities`: they are meaningful
-- only for outdoor GPS activities, would sit NULL on every lift and treadmill
-- run, and `activities` is already wide and read on every list page.
--
-- No index beyond the primary key. Every read is a point lookup by activity_id,
-- and the backfill's "what is left" query is an anti-join that the PK serves.
CREATE TABLE activity_weather (
    activity_id  TEXT PRIMARY KEY REFERENCES activities(id) ON DELETE CASCADE,
    status       TEXT NOT NULL CHECK(status IN ('ok','no_coordinates','unavailable')),
    lat          REAL,      -- the coordinate actually used; NULL for no_coordinates
    lon          REAL,
    observed_at  TIMESTAMP, -- the provider's observation hour, UTC; NULL when not ok
    temp_c       REAL,
    feels_like_c REAL,
    dew_point_c  REAL,
    humidity     INTEGER,
    wind_kmh     REAL,
    wind_deg     INTEGER,
    precip_mm    REAL,
    condition    TEXT,
    icon         TEXT,
    fetched_at   TIMESTAMP NOT NULL  -- when we asked, as distinct from observed_at
);
```

- [ ] Test (`internal/db/migrate_test.go` or a new
      `internal/weather/activity_repository_test.go` using `dbtest.New(t)`): the table
      exists after migration, the `status` CHECK rejects an unknown value, and deleting
      the parent activity cascades the weather row away.

**Verify:** `go test ./internal/db/... ./internal/weather/...`

---

### Task 2: `weather.Observation`, `EndpointHistorical`, `ErrNoHistoricalData`

- [ ] `internal/weather/models.go`: add `EndpointHistorical Endpoint = "historical"` to
      the const block, and the `Observation` struct:

```go
// Observation is one historical reading at a specific hour, metric units.
// Deliberately NOT Current: Current is the tile's shape and carries no dew
// point, wind direction, or precipitation, and no observation timestamp —
// for a forecast "now" is implicit, for a stored reading it is the point.
type Observation struct {
	ObservedAt time.Time `json:"observed_at"`
	TempC      float64   `json:"temp_c"`
	FeelsLikeC float64   `json:"feels_like_c"`
	DewPointC  float64   `json:"dew_point_c"`
	Humidity   int       `json:"humidity"`
	WindKMH    float64   `json:"wind_kmh"`
	WindDeg    int       `json:"wind_deg"`
	PrecipMM   float64   `json:"precip_mm"`
	Condition  string    `json:"condition"`
	Icon       string    `json:"icon"`
}
```

- [ ] `internal/weather/provider.go`: add to the `Provider` interface

```go
	Historical(ctx context.Context, lat, lon float64, at time.Time) (Observation, error)
```

  and, in the same file (beside the seam it belongs to):

```go
// ErrNoHistoricalData is the provider's DEFINITIVE negative: there is no
// reading for this time and there never will be (the timestamp predates the
// history window). It is the one terminal failure — the backfill records it as
// `unavailable` and never retries without --retry-unavailable. Every other
// error is transient and leaves no row.
var ErrNoHistoricalData = errors.New("weather: provider has no historical data for the requested time")
```

- [ ] Update the fake providers in `internal/weather/service_test.go` and
      `internal/weather/handler_test.go` so the package still compiles.

**Verify:** `go build ./... && go vet ./...`

---

### Task 3: `OpenWeatherProvider.Historical`

- [ ] `internal/weather/openweather.go`: add the timemachine path constant and the method.
      Extend the file-header comment block with the historical surface's own trap note.

```go
// historicalPath is the One Call 4.0 historical ("timemachine") surface. It is
// a named constant for the same reason oneCallBasePath is: the /onecall
// omission that 404'd every reading in production came from inlined copies.
//
// NOT YET VERIFIED AGAINST A LIVE CALL — see the plan's Open Question 1. The
// fixtures pinning this shape are hand-authored to the documented envelope, not
// captured. Probe once with the production key and re-capture before rollout.
const historicalPath = oneCallBasePath + "/timemachine"

func (p *OpenWeatherProvider) Historical(ctx context.Context, lat, lon float64, at time.Time) (Observation, error) {
	params := metricParams(lat, lon)
	params.Set("dt", strconv.FormatInt(at.UTC().Unix(), 10))
	var payload struct {
		Data []struct {
			DT        int64          `json:"dt"`
			Temp      float64        `json:"temp"`
			FeelsLike float64        `json:"feels_like"`
			DewPoint  float64        `json:"dew_point"`
			Humidity  int            `json:"humidity"`
			WindSpeed float64        `json:"wind_speed"`
			WindDeg   int            `json:"wind_deg"`
			Rain      struct{ OneH float64 `json:"1h"` } `json:"rain"`
			Snow      struct{ OneH float64 `json:"1h"` } `json:"snow"`
			Weather   []owWeatherTag `json:"weather"`
		} `json:"data"`
	}
	if err := p.get(ctx, historicalPath, params, &payload); err != nil {
		return Observation{}, err
	}
	if len(payload.Data) == 0 {
		// Same guard as Current and Daily, and the reason this file's header
		// calls the envelope load-bearing: decoding the wrong shape succeeds
		// silently and would store a plausible 0 °C reading as a fact.
		return Observation{}, fmt.Errorf("openweather historical: response carried no data entries")
	}
	e := payload.Data[0]
	out := Observation{
		ObservedAt: unixUTC(e.DT),
		TempC:      e.Temp,
		FeelsLikeC: e.FeelsLike,
		DewPointC:  e.DewPoint,
		Humidity:   e.Humidity,
		WindKMH:    e.WindSpeed * 3.6, // metric units are m/s; the canonical model is km/h
		WindDeg:    e.WindDeg,
		// Rain and snow are reported separately and are both "water that fell
		// in this hour"; the model carries one total.
		PrecipMM: e.Rain.OneH + e.Snow.OneH,
	}
	if len(e.Weather) > 0 {
		out.Condition = e.Weather[0].Main
		out.Icon = e.Weather[0].Icon
	}
	return out, nil
}
```

- [ ] Teach `get` to distinguish the definitive negative. Keep it narrow — one status
      code, documented:

```go
	if resp.StatusCode == http.StatusBadRequest {
		// OpenWeather answers 400 for a timestamp outside its history window.
		// That is the one failure worth recording as terminal; everything else
		// (5xx, timeouts, 429) is transient and must leave no row.
		return fmt.Errorf("openweather %s: %w", path, ErrNoHistoricalData)
	}
```

- [ ] Tests in `internal/weather/openweather_test.go`, against an `httptest` server:
  - the request path is `/data/4.0/onecall/timemachine` and carries
    `dt`, `units=metric`, `lat`, `lon`, `appid`;
  - a full fixture decodes into the expected `Observation`, **wind converted m/s → km/h**;
  - an empty `data` array returns an error (not a zero-value observation);
  - a body with the readings at the ROOT (no `data` wrapper) also errors — the PR #124
    regression guard;
  - `rain.1h` + `snow.1h` sum into `PrecipMM`; both absent ⇒ 0;
  - a missing `weather` tag degrades to empty condition/icon rather than failing;
  - a 400 response returns an error satisfying `errors.Is(err, ErrNoHistoricalData)`;
  - a 500 response does **not** satisfy it.
- [ ] Add a header comment above the new fixtures stating they are hand-authored to the
      documented shape and must be re-captured from a live response.

**Verify:** `go test ./internal/weather/...`

---

### Task 4: `activity_weather` repository

- [ ] `internal/weather/activity_weather.go` — the row type, the status constants, and
      the repository interface:

```go
// ActivityStatus is the disposition of one capture attempt. The set is closed
// and mirrors the migration's CHECK.
type ActivityStatus string

const (
	ActivityStatusOK             ActivityStatus = "ok"
	ActivityStatusNoCoordinates  ActivityStatus = "no_coordinates" // terminal: position cannot be re-derived
	ActivityStatusUnavailable    ActivityStatus = "unavailable"    // terminal: provider has no answer
)

// ActivityWeather is one stored reading. Reading fields are nil unless
// Status is ActivityStatusOK.
type ActivityWeather struct {
	ActivityID string
	Status     ActivityStatus
	Lat, Lon   *float64
	ObservedAt *time.Time
	Observation *Observation // nil unless Status is ok
	FetchedAt  time.Time
}

type ActivityWeatherRepository interface {
	Get(ctx context.Context, activityID string) (*ActivityWeather, error)
	Put(ctx context.Context, row ActivityWeather) error
	// CountOK / CountPending feed the two new gauges and --dry-run.
	CountOK(ctx context.Context) (int, error)
	CountPending(ctx context.Context) (PendingCounts, error)
	// ListPending returns eligible outdoor activities with no row, newest
	// first — recent runs are the ones the user is most likely to open.
	ListPending(ctx context.Context, limit int) ([]PendingActivity, error)
	// ClearUnavailable drops terminal 'unavailable' rows so they become
	// eligible again. no_coordinates rows are deliberately NOT cleared.
	ClearUnavailable(ctx context.Context) (int, error)
}
```

  `PendingActivity` carries `ActivityID`, `StartTime`, and `Lat`/`Lon` (`*float64`, nil
  when the activity has no positioned trackpoint). `PendingCounts` carries
  `Eligible`, `NoCoordinates`, and is what `--dry-run` prints.

- [ ] `internal/weather/activity_weather_sqlite.go` — the SQLite implementation, shaped
      like `sqlite_repository.go`. The selection query is the anti-join across the five
      endurance detail tables (`environment` lives on the per-type detail row, not on
      `activities` — migration 042):

```sql
SELECT a.id, a.start_time,
       (SELECT tp.latitude  FROM activity_trackpoints tp
         WHERE tp.activity_id = a.id AND tp.latitude IS NOT NULL AND tp.longitude IS NOT NULL
         ORDER BY tp.sequence LIMIT 1),
       (SELECT tp.longitude FROM activity_trackpoints tp
         WHERE tp.activity_id = a.id AND tp.latitude IS NOT NULL AND tp.longitude IS NOT NULL
         ORDER BY tp.sequence LIMIT 1)
  FROM activities a
  JOIN (
        SELECT activity_id, environment FROM activity_run_details
  UNION ALL SELECT activity_id, environment FROM activity_walk_details
  UNION ALL SELECT activity_id, environment FROM activity_cycle_details
  UNION ALL SELECT activity_id, environment FROM activity_hike_details
  UNION ALL SELECT activity_id, environment FROM activity_other_details
       ) d ON d.activity_id = a.id
 WHERE a.deleted_at IS NULL
   AND d.environment = 'outdoor'
   AND NOT EXISTS (SELECT 1 FROM activity_weather w WHERE w.activity_id = a.id)
 ORDER BY a.start_time DESC
```

  Put the UNION in one unexported `outdoorDetailsUnion` const so `ListPending` and
  `CountPending` cannot drift apart. `LIMIT` is appended by `ListPending` only.
  `CountPending` runs the same shape twice — once counting all pending, once counting
  those whose first-positioned-trackpoint subquery is NULL — so `--dry-run` can print
  the "eligible / no GPS / already captured" split without a second pass.

- [ ] Tests (`activity_weather_sqlite_test.go`, `dbtest.New(t)`):
  round-trip an `ok` row with every field; round-trip a `no_coordinates` row with NULL
  readings; `Get` returns `nil, nil` for an absent row; indoor activities are excluded
  from `ListPending`; soft-deleted activities are excluded; an activity WITH a row is
  excluded; ordering is newest-first; `ListPending` reports nil coordinates for a
  GPS-less activity; `ClearUnavailable` removes `unavailable` rows and leaves
  `no_coordinates` and `ok` alone; `CountOK`/`CountPending` agree with the fixtures.

**Verify:** `go test ./internal/weather/...`

---

### Task 5: The capture service

- [ ] `internal/weather/activity_capture.go` — `ActivityCapturer`, the one place that
      turns "an activity and a coordinate" into a stored row:

```go
// ActivityCapturer captures and stores the conditions at one activity's start.
// It is the only writer of activity_weather 'ok' rows on the import path, and
// the backfill drives the same method so there is one code path to trust.
type ActivityCapturer struct {
	cfg      config.WeatherConfig
	repo     ActivityWeatherRepository
	ledger   *BudgetLedger
	provider Provider
	log      *slog.Logger
	now      func() time.Time
}
```

  - `Enabled()` — `cfg.Enabled && cfg.CaptureActivityWeather && provider.Configured()`.
    The master switch gates the capture knob, per the SOW's config comment.
  - `Capture(ctx, activityID string, lat, lon float64, at time.Time) error`:
    1. `if !c.Enabled() { return ErrCaptureDisabled }` — no reservation, no row.
    2. `c.ledger.Reserve(ctx, 1, activeCeiling(c.cfg))` — the SAME ledger and ceiling as
       the tile. No separate allowance.
    3. `c.provider.Historical(ctx, lat, lon, at.UTC().Round(time.Hour))` wrapped in the
       same latency/`providerCallsTotal` instrumentation the tile uses, labelled
       `EndpointHistorical`. Factor the timing wrapper out of `Service.timedFetch` into a
       package-level `observeCall(endpoint, started, err, now)` so both call sites share
       it rather than duplicating the two metric lines.
    4. On success, `repo.Put` an `ok` row (`observed_at` from the provider's answer, not
       from the request) and `activityCapturesTotal.WithLabelValues("ok").Inc()`.
    5. On `ErrNoHistoricalData`, return it **without writing** — only the backfill turns
       it into a terminal row (SOW: "written *only* by the backfill command").
    6. On any other error, count `failed` and return; **no row**.
  - `RecordNoCoordinates(ctx, activityID)` — writes the terminal `no_coordinates` row,
    no provider call, counts `no_coordinates`.
  - `RecordUnavailable(ctx, activityID, lat, lon)` — writes the terminal `unavailable`
    row, counts `unavailable`.
  - `Get(ctx, activityID)` — passthrough to the repository, so the activity handler
    needs exactly one seam.
- [ ] Rounding: `at.UTC().Round(time.Hour)` — nearest hour, the resolution historical
      data is published at. Document why in a comment.
- [ ] Tests: a fake provider and a real SQLite repo. Assert the ledger is consulted
      before the provider (a fake ledger at ceiling 0 ⇒ provider never called, no row);
      `ErrBudgetExhausted` writes no row; a transient error writes no row;
      `ErrNoHistoricalData` from `Capture` writes no row; success stores every field and
      uses the provider's `observed_at`; disabled config makes no reservation and no
      call; the start time is rounded to the nearest hour (assert 14:07 → 14:00 and
      14:40 → 15:00 through the fake provider's recorded argument).

**Verify:** `go test ./internal/weather/...`

---

### Task 6: Metrics and the exporter row

- [ ] `internal/weather/metrics.go` — three additions, registered in `init`:
  - `activityCapturesTotal` — `CounterVec{Name: "api_weather_activity_captures_total"}`,
    label `result` ∈ `ok | no_coordinates | unavailable | failed`.
  - `activitiesWithWeatherGauge` — `api_weather_activities_with_weather`.
  - `activitiesPendingCaptureGauge` — `api_weather_activities_pending_capture`.
  Both gauges get the "published by the Exporter from durable SQLite" doc comment, and
  the pending gauge's help text names it as backfill progress.
- [ ] `internal/weather/collector.go` — `Exporter` gains an
      `activityWeather ActivityWeatherRepository` field, read in `refresh` alongside the
      existing reads and published only after every read succeeds (the existing
      all-or-nothing contract). `NewExporter` grows one parameter; update `cmd/api`.
  - Guard: when the repository is nil the exporter skips the two gauges rather than
    panicking, so the constructor stays usable from tests that do not care.
- [ ] Add a comment in `collector.go` recording the SOW's caveat: activity captures
      deliberately do **not** advance `api_weather_last_success_timestamp_seconds`,
      because they never touch `weather_cache`. That gauge measures *tile* liveness.
- [ ] Test: `collector_test.go` gains a case asserting both new gauges publish the
      repository's counts, and that a repository error leaves every gauge unchanged.

**Verify:** `go test ./internal/weather/...`

---

### Task 7: Config knob

- [ ] `internal/config`: add `CaptureActivityWeather bool` to `WeatherConfig`, parsed
      from `capture_activity_weather` in the `[weather]` block, defaulting **true**
      (matching config.toml, and matching the shipped default-on posture of the block).
- [ ] `config.toml`: append to the existing `[weather]` block, with the SOW's comment:

```toml
# Capture historical conditions on activity import. false ⇒ no import-time
# provider calls and no new activity_weather rows; existing rows still render.
# The master `enabled` switch above gates this too — false there disables
# capture regardless of this value. Separate from `enabled` because the failure
# modes differ: `enabled = false` is "the vendor integration is off", this is
# "stop capturing but keep the tile working" — what an operator wants during a
# backfill that needs the whole day's budget.
capture_activity_weather = true
```

- [ ] No new secret ⇒ **no change** to `REQUIRED_ENV_KEYS` or the Secrets Manager path.
      Confirm with a grep that nothing else needs touching.
- [ ] Test: the config test asserts the field parses and that the documented default
      holds when the key is absent.

**Verify:** `go test ./internal/config/...`

---

### Task 8: Detached import-time capture

- [ ] `internal/activity/weather.go` — the narrow seam and the first-position helper:

```go
// WeatherCapturer is the seam onto internal/weather's activity capture. The
// activity package owns WHEN a capture happens; the weather package owns the
// budget, the provider, and the store.
type WeatherCapturer interface {
	Capture(ctx context.Context, activityID string, lat, lon float64, at time.Time) error
	Get(ctx context.Context, activityID string) (*weather.ActivityWeather, error)
}

// firstPosition returns the coordinate of the first positioned trackpoint —
// where the activity started. A 10 km loop sits inside a single weather grid
// cell, so the start point is not a lossy approximation of the run; it is the
// run's weather.
func firstPosition(tps []Trackpoint) (lat, lon float64, ok bool)
```

- [ ] `Handler.SetWeatherCapturer(WeatherCapturer)`, nil-guarded like `SetCalendarSyncer`.
- [ ] `Handler.captureWeather(a Activity)` — called from `uploadTCX` immediately after
      the existing best-effort hooks (`publish`, `syncCalendar`, `matchSession`) in the
      `err == nil` branch:

```go
// captureWeather spawns the detached conditions capture. Three properties are
// load-bearing and each has a test:
//
//   - It runs on context.Background(), NOT r.Context(). The request context is
//     cancelled the instant the upload response is written, so a fetch
//     inheriting it would be cancelled essentially always — a feature that
//     silently never works, the exact shape of the WHOOP webhook outage.
//   - It is bounded, so a hanging provider cannot leak a goroutine per upload.
//   - It never blocks the response and never surfaces a failure to the
//     uploader. TCX import is already the slowest user-facing write in the
//     product; a third-party call on that path would make a slow OpenWeather
//     day look like a broken uploader.
//
// A failure leaves NO ROW, so the activity stays indistinguishable from one
// never attempted and cmd/weather-backfill is the single repair path.
```

  Guard on `h.weather != nil`, `a.Environment == EnvironmentOutdoor`, and
  `firstPosition(a.Trackpoints)` before spawning.
- [ ] `weatherCaptureTimeout` — a package constant (30s), commented as "generous
      relative to the 8s HTTP client timeout so a retry-free slow call still lands, tight
      enough that a hung provider cannot accumulate goroutines".
- [ ] Tests in `internal/activity/` — **the highest-value tests in this SOW**:
  - **Detached context**: upload a TCX through `httptest`; the fake capturer records
    `ctx.Err()` at call time and signals a channel. After `ServeHTTP` returns (and
    therefore after the request context is cancelled), the test waits on the channel and
    asserts the recorded `ctx.Err()` is nil and that the context carries a deadline in
    the expected range. Wiring to the request context fails this test.
  - **Non-fatal**: three sub-cases — the capturer returns a generic error, a timeout, and
    `weather.ErrBudgetExhausted`. The upload responds 201 in all three and
    `activity_weather` is empty in all three.
  - **Indoor**: an indoor activity spawns no capture at all (fake records zero calls).
  - **No GPS**: an outdoor activity with no positioned trackpoint spawns no capture.
  - Use a `sync.WaitGroup`/channel with a bounded `select` timeout so the tests are
    deterministic under `-race` and never hang CI.

**Verify:** `go test -race ./internal/activity/...`

---

### Task 9: The DTO field

- [ ] `internal/activity/handler.go` — `activityDTO` gains, beside `Route`:

```go
	// Weather is the stored conditions at the activity's start. omitempty:
	// the key is ABSENT when the activity has no row, the same "omit the beat
	// entirely" contract route already has.
	Weather *weatherDTO `json:"weather,omitempty"`
```

  and the DTO itself (metric, raw-measurement convention — see the SOW):

```go
type weatherDTO struct {
	Status     string     `json:"status"`
	ObservedAt *time.Time `json:"observed_at,omitempty"`
	TempC      *float64   `json:"temp_c,omitempty"`
	FeelsLikeC *float64   `json:"feels_like_c,omitempty"`
	DewPointC  *float64   `json:"dew_point_c,omitempty"`
	Humidity   *int       `json:"humidity,omitempty"`
	WindKMH    *float64   `json:"wind_kmh,omitempty"`
	WindDeg    *int       `json:"wind_deg,omitempty"`
	PrecipMM   *float64   `json:"precip_mm,omitempty"`
	Condition  string     `json:"condition,omitempty"`
	Icon       string     `json:"icon,omitempty"`
}
```

  `precip_mm` is a pointer so a genuine 0.0 mm still serializes for an `ok` row (the
  *client* hides the zero cell; the API does not lie about having measured it).
- [ ] `attachWeather(ctx, activityID, dto)` in `unified_handler.go`, called from
      `buildDetailDTO` after `attachVideos`. Returns the repository error, consistent
      with its sibling `attach*` methods. No-ops when the seam is unwired.
- [ ] Tests (`handler_test.go`): the key is **absent** with no row; an `ok` row
      serializes every field; a `no_coordinates` row serializes `{"status":"..."}` with
      every reading field dropped; a row survives an `outdoor → indoor` retag (the
      response still carries it — hiding is the client's job).

**Verify:** `go test ./internal/activity/...`

---

### Task 10: `cmd/weather-backfill`

- [ ] `cmd/weather-backfill/main.go`, on the `cmd/memory-backfill` precedent: its own
      `main.go`, flags parsed in `main`, and a `run`/`backfill` split with injectable
      dependencies so the loop is testable without a real provider.
- [ ] Flags: `--dry-run`, `--limit N` (0 = unlimited), `--rate F` (calls/sec, default 1),
      `--retry-unavailable`.
- [ ] Order of operations:
  1. `--retry-unavailable` ⇒ `ClearUnavailable` first, and print how many were cleared.
  2. `CountPending` ⇒ print
     `N activities eligible (N provider calls), M with no GPS (free), K already captured`.
  3. `--dry-run` ⇒ stop here. **Zero provider calls, zero writes.** Exit 0.
  4. Otherwise iterate `ListPending` newest-first:
     - no coordinate ⇒ `RecordNoCoordinates`, **no call**, continue (does not count
       against `--limit`);
     - otherwise pace with a `time.Ticker` at `--rate`, then `Capture`;
     - `errors.Is(err, weather.ErrBudgetExhausted)` ⇒ print what remains and
       `resume tomorrow`, **exit 0** (a budget stop is a clean stop, not a failure);
     - `errors.Is(err, weather.ErrNoHistoricalData)` ⇒ `RecordUnavailable`, continue;
     - any other error ⇒ log with the activity ID, write nothing, continue;
     - success ⇒ the row is written by `Capture`.
  5. Honour `SIGINT` through a cancellable context so a Ctrl-C stops between activities
     and leaves a consistent database.
- [ ] The command **must not** invent a second budget: it calls the same
      `ActivityCapturer` the import path uses.
- [ ] Doc comment at the top explaining why this is a command and not a worker (the SOW's
      Algorithms section), and the scale boundary (low thousands; > ~800 spans a UTC day).
- [ ] Tests (`cmd/weather-backfill/main_test.go`) driving the extracted `backfill`
      function over `dbtest` + a fake provider:
  - `--dry-run` makes zero provider calls and writes zero rows;
  - `--limit` is respected (n provider calls, remainder untouched);
  - budget exhaustion mid-run exits cleanly and leaves the already-written rows intact;
  - re-running resumes and never re-spends on a resolved activity;
  - `no_coordinates` and `unavailable` rows are never re-attempted;
  - `--retry-unavailable` clears `unavailable` and **not** `no_coordinates`;
  - an indoor activity is skipped, and becomes eligible after a retag to outdoor;
  - a transient provider error leaves no row and the run continues.

**Verify:** `go test ./cmd/...`

---

### Task 11: Wiring in `cmd/api`

- [ ] Construct `weatherActivityRepo := weather.NewSQLiteActivityWeatherRepository(database)`
      beside the existing weather repos.
- [ ] Construct the `ActivityCapturer` with the same cfg / ledger / provider the tile
      uses (share the one `*OpenWeatherProvider` instance — the HTTP client is already
      timeout-bounded).
- [ ] `activityHandler.SetWeatherCapturer(weatherCapturer)`.
- [ ] Pass `weatherActivityRepo` into `weather.NewExporter`.
- [ ] Extend the existing boot log line so an operator can see the new knob:
      `weather: enabled=%v provider_configured=%v capture_activity_weather=%v`.
- [ ] Update `CHANGELOG`/docs only if the repo's conventions require it (semantic-release
      owns CHANGELOG.md — do not hand-edit).

**Verify:** full API gate (`gofmt -l .`, `go build ./...`, `go vet ./...`,
`golangci-lint run --timeout=5m`, `go mod tidy` + no drift, `go test ./...`).

---

## Repo 2 — `prog-strength-web`

### Task 12: Promote the weather icon set (zero-behaviour-change)

- [ ] `git mv app/(app)/dashboard/_components/weather-icons.tsx components/weather/icons.tsx`.
- [ ] Rewrite the tile's import to `@/components/weather/icons`.
- [ ] Grep for any other importer and update it.
- [ ] The existing `weather-tile.test.tsx` must stay green **unchanged** — it is the
      regression guard on this refactor. If it needs editing, the move was not
      behaviour-neutral.
- [ ] Commit this on its own (`refactor(weather): promote the weather icon set out of
      the dashboard route`) so the diff reads as a move.

**Verify:** `npm run lint && npm run typecheck && npm run test`

---

### Task 13: Types and unit formatters

- [ ] `lib/api.ts`: add

```ts
export type ActivityWeather = {
  status: "ok" | "no_coordinates" | "unavailable";
  observed_at?: string; // RFC3339, the provider's observation hour, UTC
  temp_c?: number;
  feels_like_c?: number;
  dew_point_c?: number;
  humidity?: number;
  wind_kmh?: number;
  wind_deg?: number;
  precip_mm?: number;
  condition?: string;
  icon?: string;
};
```

  and `weather?: ActivityWeather;` on `RunningSession` (beside `route`), with a comment
  noting it is metric like `distance_meters` and formatted client-side.
- [ ] `lib/distance-unit-context.tsx`: add beside `formatElevationValue`

```ts
export function formatTemperature(celsius: number | null, unit: DistanceUnit): string
export function formatWindSpeed(kmh: number | null, unit: DistanceUnit): string
```

  `mi` ⇒ °F / mph, `km` ⇒ °C / km/h. Whole numbers, `"—"` for null/non-finite,
  matching `formatElevationValue`'s contract exactly. Expose them on the
  `useDistanceUnit()` value as `formatTemperature` / `formatWindSpeed` so the component
  reads the same way the page already reads `formatElevation`.
- [ ] Tests in the existing `lib/distance-unit-context.test.tsx` (or a new one):
      0 °C ⇒ `32°F` / `0°C`; a negative temperature rounds correctly in both systems;
      10 km/h ⇒ `6 mph` / `10 km/h`; null ⇒ `—`.

**Verify:** `npm run typecheck && npm run test`

---

### Task 14: `WeatherRecap`

- [ ] `components/activity-detail/WeatherRecap.tsx`, beside `MapView` / `HeartRateRecap`
      / `ElevationRecap`, and its co-located `WeatherRecap.test.tsx`.
- [ ] Props: `{ weather?: ActivityWeather; environment: "outdoor" | "indoor";
      discipline?: "run" | "hike" }`.
- [ ] Renders `null` when `weather` is absent, when `weather.status !== "ok"`, or when
      `environment !== "outdoor"`. No skeleton, no empty frame, no "weather unavailable"
      line — beats with no data are omitted whole.
- [ ] Composition:
  - `<SectionKicker discipline={discipline}>Conditions</SectionKicker>`;
  - headline: `<WeatherIcon icon={weather.icon} />`, the temperature at a large numeric
    scale, and the condition word — `31°C · Clear`;
  - metric strip: the same `border-y` definition-list idiom the detail page already uses
    for distance/time/pace — **feels like, dew point, humidity, wind, precipitation**;
  - wind shows speed plus a compass direction derived from `wind_deg` (a 16-point
    `degreesToCompass` helper local to the file — meteorological degrees name the
    direction the wind comes *from*, worth a one-line comment);
  - precipitation is **omitted when zero** rather than rendered as `0 mm`;
  - any individual absent field drops its cell rather than rendering `—`.
- [ ] Tokens only: `--surface`, hairline `--border`, `--radius-card`, `--muted`/`--faint`,
      `--foreground`. **No new hues**, no `--accent` as a section or series hue.
- [ ] Tests: renders null for each non-`ok` status, for a missing `weather` key, and for
      `environment: "indoor"`; renders the headline and every present metric for an `ok`
      row; unit formatting flips with `useDistanceUnit` (render inside the provider with
      each unit); zero precipitation is omitted; a `wind_deg` of 210 renders `SSW`;
      the icon renders (an `svg` is present).

**Verify:** `npm run test`

---

### Task 15: Wire both detail routes

- [ ] `app/(app)/running/[id]/page.tsx`: `<WeatherRecap … discipline="run" />`
      immediately after `<MapView …>` and before the *The Miles* section.
- [ ] `app/(app)/hiking/[id]/page.tsx`: the equivalent slot — after `<MapView …>` and
      before the elevation-profile section — with `discipline="hike"`.
- [ ] Neither page refetches: the reading arrives on the detail response both pages
      already load.
- [ ] Tests: each route's existing page test (or a new focused one) asserts the widget
      renders in the right slot for an `ok` outdoor fixture, and is absent for an indoor
      fixture.

**Verify:** full web gate (`npm run lint && npm run format:check && npm run typecheck &&
npm run test && npm run build`).

---

## Repo 3 — `prog-strength-mcp`

### Task 16: `get_activity`

- [ ] `src/prog_strength_mcp/activities.py`: a new `get_activity(activity_id)` tool
      beside `list_activities`, backed by the **existing** `APIClient.get_activity`.
- [ ] **Trim before the model sees it.** Drop `trackpoints` and `route` — both are
      rendering data, meaningless to a language model, and would dominate the response
      (and on a long run crowd out the conversation). Keep the aggregates the agent can
      reason about (`splits`, `strip_summary`, `heart_rate_zones`, `intervals`). Put the
      dropped keys in a module-level constant with a comment explaining the reasoning.
- [ ] **Weather in both unit systems.** When `weather` is present and `status == "ok"`,
      add `temp_f`, `feels_like_f`, `dew_point_f`, `wind_mph` alongside the metric
      fields. Two multiplications that remove a class of quiet arithmetic error from the
      model's answer. The rest of the DTO passes through in its existing units,
      unchanged, consistent with every other tool.
- [ ] Degrade gracefully, following the `nutrition_lookup` precedent: a 404 surfaces as a
      clean `RuntimeError("API error (404): …")` message rather than a stack trace, and
      an activity with no `weather` key simply has none — the tool must not invent a
      "no weather recorded" error where absence is the answer.
- [ ] The docstring is the agent-facing contract: say what the tool returns, say that
      trackpoints and the route are omitted deliberately, and say weather is dual-unit.
- [ ] Tests in `tests/test_activities_tools.py`:
  - requires auth (mirrors the existing guard tests);
  - strips `trackpoints` **and** `route` from a fixture that contains both;
  - keeps `splits` / `strip_summary` / `heart_rate_zones` / `intervals`;
  - returns weather in both unit systems with correct arithmetic;
  - an activity with no `weather` key returns a payload with no `weather` key and no
    error;
  - a non-`ok` weather status is passed through without invented Fahrenheit fields;
  - a 404 maps to a `RuntimeError` carrying the status code.

**Verify:** `ruff check src tests && ruff format --check src tests && pytest tests`

---

## Repo 4 — `prog-strength-infra`

### Task 17: The "Activity Capture" dashboard row

- [ ] `monitoring/grafana/dashboards/weather.json`: a **fourth** collapsible row,
      `"Activity Capture"`, after "Cache Efficiency", matching the existing three rows'
      shape (row panel id 103, `gridPos.h = 1, w = 24`, a `description`).
- [ ] Its description must carry the SOW's caveat verbatim in substance: activity
      captures deliberately do not advance
      `api_weather_last_success_timestamp_seconds`, because historical readings never
      touch `weather_cache`. A backfill can run all day while "Time since last successful
      call" climbs — that is correct, and the row says so before anyone files it as a bug.
- [ ] Three panels, new ids starting at 13 (the current max data-panel id is 12; row ids
      are 100–102, so the new row is 103):
  - **Activities with weather** — stat, `api_weather_activities_with_weather`.
  - **Pending capture** — stat, `api_weather_activities_pending_capture`, with a
    description saying that trending to zero *is* backfill progress.
  - **Captures by result (24h)** — timeseries, stacked,
    `increase(api_weather_activity_captures_total[24h])` by `result`, with the
    description noting that a rising `failed` band during a backfill is the signal to
    stop and look.
- [ ] **No new alerts.** `rules-weather.yml` is untouched.
- [ ] Verify the JSON parses and that panel ids and `gridPos` do not collide.

**Verify:** `python3 monitoring/grafana/provisioning/alerting/validate_rules.py` and a
`json.load` of `weather.json`.

---

## Repo 5 — `prog-strength-docs`

### Task 18: Mark the SOW shipped

- [ ] `sows/activity-weather-conditions.md`: frontmatter `status: shipped`; body header
      `**Status**: Shipped` and `**Last updated**: 2026-08-09`.
- [ ] Commit `docs: mark activity-weather-conditions as shipped`.
- [ ] Commit this plan alongside it.

---

## Rollout order (for the docs PR body)

1. **`prog-strength-api`** — migration `057`, `Observation` + `Historical`, the
   repository, the detached capture, the DTO field, metrics, and `cmd/weather-backfill`.
   Everything downstream reads the `weather` key this PR adds, so nothing else can merge
   usefully before it.
2. **Run `weather-backfill --dry-run`** against production, read the count, then run it
   for real across as many days as the budget requires.
3. **`prog-strength-web`** — after the backfill, so the feature does not debut mostly
   empty.
4. **`prog-strength-mcp`** — `get_activity`; independent of web.
5. **`prog-strength-infra`** — last: the gauges must exist before panels can render them.
6. **`prog-strength-docs`** — flip to shipped.
