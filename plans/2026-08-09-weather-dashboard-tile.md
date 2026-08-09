# Weather Dashboard Tile Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a `weather` dashboard tile (current conditions, today's high/low, hourly
strip, up to 5 saved locations paged in one tile) backed by a cache-first
`internal/weather` service with a hard, durable, SQLite-backed daily provider-call
budget, plus a Grafana dashboard and five Slack alerts.

**Architecture:** `internal/weather` mirrors `internal/nutritionlookup` (Provider seam,
degrade-don't-fail Service, global coordinate-keyed SQLite cache with per-endpoint
TTLs) and adds a reserve-before-call budget ledger keyed by UTC date. Prometheus only
*observes* the ledger through a gauge collector (the WHOOP postmortem pattern —
metrics never enforce). The web tile is self-fetching (deliberately NOT a
`/dashboard/summary` section). Grafana dashboard + alert rules land in
`prog-strength-infra` following `rules-whoop.yml` / `validate_rules.py` conventions.

**Tech Stack:** Go 1.25 / chi / go-sqlite3 / prometheus client (API) · Next.js 16 /
React 19 / @dnd-kit / vitest (web) · Grafana provisioning YAML + dashboard JSON (infra).

**Repos & branches:** `feat/weather-dashboard-tile` in `prog-strength-api`,
`prog-strength-web`, `prog-strength-infra`, `prog-strength-docs`.

**Design system:** conforms to `design-system.md` v0.4.4. Existing tokens only —
`--surface`, `--surface-2`, hairline `--border`, `--radius-card`, `--muted`/`--faint`,
`--foreground`, `--accent` (selection/focus chrome only), Manrope. **No new hues.**
Weather condition icons are hand-rolled `currentColor` stroke SVGs rendered in
`--foreground`/`--muted` neutrals — no sky-blue, no sun-yellow.

**Source spec:** `prog-strength-docs/sows/weather-dashboard-tile.md`. Where this plan
and the SOW disagree, the SOW wins.

---

## Global conventions (read before any task)

- **API repo** (`/workspace/prog-strength-api`): module
  `github.com/Prog-Strength/prog-strength-api`. Local gate before push:
  `go build ./... && go vet ./... && golangci-lint run && go mod tidy` (no
  `go.mod`/`go.sum` drift) `&& go test ./...`. DB tests use
  `internal/db/dbtest.New(t)` (temp-file SQLite + all migrations). Services take a
  `now func() time.Time` field for injected clocks. IDs come from `internal/id.New()`.
  Responses use `internal/httpresp` (`OK`, `Error`, `ErrorWithCode`, `ServerError`).
  Comments explain *why*, never *what*. No new third-party dependencies.
- **Web repo** (`/workspace/prog-strength-web`): npm. Local gate:
  `npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build`.
  All API calls go through typed wrappers in `lib/api.ts`. Components hand-roll SVG
  icons with `currentColor` (no icon library). Tests co-located `*.test.tsx`, vitest +
  Testing Library, `vi.hoisted` mock pattern (see
  `app/(app)/dashboard/_components/quote-tile.test.tsx`).
- **Infra repo** (`/workspace/prog-strength-infra`): local gate:
  `python3 monitoring/grafana/provisioning/alerting/validate_rules.py`, plus
  `for t in deploy/tests/*.test.sh; do bash "$t"; done`, plus
  `shellcheck -x deploy/*.sh deploy/lib/*.sh`. **No literal `$` anywhere in alerting
  YAML** (Grafana expands `$VAR` from the container env). Alert rule uids are globally
  unique; `__dashboardUid__`/`__panelId__` annotations are all-or-nothing per rule.
- **Commit messages:** conventional commits, lowercase subjects
  (`feat(weather): …`, `feat(dashboard): …`, `feat(monitoring): …`, `docs: …`).
- **The `weather` tile id** is appended at the END of both catalogs (after `quote`),
  giving 20 tiles. Position must be identical in Go and TS.
- **Canonical units:** the provider is always called in METRIC; the cache stores
  metric values (the cache is global — users with different unit prefs share rows).
  Conversion to °F / mph happens at serve time from `users.distance_unit`
  (`mi` → °F + mph, `km` → °C + km/h). Temperatures round to whole numbers, wind to
  whole numbers.
- **Config values** (from the SOW, `[weather]` in `config.toml`): `enabled=true`,
  `api_key="${OPENWEATHER_API_KEY}"`, `daily_call_budget=800`,
  `allow_paid_overage=false`, `daily_call_hard_ceiling=2000`,
  `count_geocoding_calls=true`, `current_ttl_minutes=15`, `hourly_ttl_minutes=60`,
  `daily_ttl_minutes=180`, `geocoding_ttl_days=30`, `max_locations=5`,
  `eager_load_all_locations=false`.
- **Cache keys:** readings `{lat_2dp}:{lon_2dp}:{endpoint}` with endpoint ∈
  `current|hourly|daily`, coordinates formatted `%.2f`; geocoding
  `geocode_direct:{normalized_query}` (lower-cased, whitespace collapsed — same
  normalization as nutritionlookup) and `geocode_reverse:{lat_2dp}:{lon_2dp}`.
- **Budget semantics:** `reserve(n)` happens BEFORE the HTTP call(s), atomically, in
  one transaction: upsert today's UTC row, then
  `UPDATE … SET calls_used = calls_used + n WHERE usage_date = ? AND calls_used + n <= ceiling`;
  zero rows affected ⇒ `ErrBudgetExhausted`. The active ceiling is
  `daily_call_hard_ceiling` when `allow_paid_overage` else `daily_call_budget`.
- **`GET /weather` `status` values:** `ok | stale | disabled | budget_exhausted | unavailable`.
  All five weather routes stay mounted when `enabled=false` and return 200 with
  `status:"disabled"` (search/reverse/locations return their own disabled shapes, see
  Task A9).
- **Metrics naming** exactly as the SOW's Observability table (12 metrics,
  `api_weather_*`). Every label is a small closed set.

---

# PART 1 — prog-strength-api

### Task A1: Migration 056 — three weather tables

**Files:**
- Create: `internal/db/migrations/056_weather.sql`

- [ ] **Step 1: Create the branch**

```bash
cd /workspace/prog-strength-api && git checkout -b feat/weather-dashboard-tile
```

- [ ] **Step 2: Check the current head migration** — run
  `ls internal/db/migrations/ | tail -3` and confirm 055 is the newest; if a later
  migration exists, renumber this one accordingly (and use that number everywhere
  this plan says 056).

- [ ] **Step 3: Write the migration**

```sql
-- migrations/056_weather.sql
-- Weather tile storage: saved locations, the daily provider-call budget
-- ledger, and the global reading/geocode cache.
-- See sows/weather-dashboard-tile.md.
--
-- user_weather_locations carries NO UNIQUE(user_id, position): writes are a
-- whole-list replace that rewrites positions in one transaction, and a unique
-- constraint would fight the intermediate states. The 5-location cap is
-- enforced in Go (migration 049's precedent: the Go layer is the source of
-- truth, SQL carries no CHECK).
CREATE TABLE user_weather_locations (
    id         TEXT PRIMARY KEY,
    user_id    TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    position   INTEGER NOT NULL,
    label      TEXT NOT NULL,
    country    TEXT NOT NULL,
    state      TEXT,
    lat        REAL NOT NULL,
    lon        REAL NOT NULL,
    created_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_user_weather_locations_user
    ON user_weather_locations(user_id, position);

-- One row per UTC day, account-level (Prog Strength is single-user by design;
-- the budget is a constraint against OpenWeather, not against a user). Rows
-- are never swept — the history is the audit trail for spend. Durable SQLite
-- rather than a Prometheus counter so a deploy can never hand the integration
-- a fresh allowance (the WHOOP counter postmortem).
CREATE TABLE weather_call_budget (
    usage_date TEXT PRIMARY KEY,           -- YYYY-MM-DD in UTC
    calls_used INTEGER NOT NULL DEFAULT 0, -- monotonically increasing reservations
    updated_at TIMESTAMP NOT NULL
);

-- Global reading/geocode cache, exactly like nutrition_lookup_cache: Denver
-- is Denver regardless of who asked. Eviction piggybacks on write, sweeping
-- rows unused for 90 days (longer than the 30-day geocoding TTL so a geocode
-- row is never evicted at the exact moment it goes stale).
CREATE TABLE weather_cache (
    cache_key    TEXT PRIMARY KEY,
    payload_json TEXT NOT NULL,   -- normalized reading, not the raw provider body
    fetched_at   DATETIME NOT NULL,
    last_used_at DATETIME NOT NULL
);

CREATE INDEX idx_weather_cache_last_used
    ON weather_cache(last_used_at);
```

- [ ] **Step 4: Run the migration tests**

Run: `go test ./internal/db/...`
Expected: PASS (the migrate test applies every migration against a fresh DB).

- [ ] **Step 5: Commit**

```bash
git add internal/db/migrations/056_weather.sql
git commit -m "feat(weather): add locations, call-budget ledger, and cache tables"
```

### Task A2: `[weather]` config block

**Files:**
- Modify: `config.toml`
- Modify: `internal/config/config.go`
- Test: `internal/config/config_test.go`

- [ ] **Step 1: Write failing config tests** — in `internal/config/config_test.go`,
  following the existing tests' style, assert that loading the embedded default
  config yields `Weather.Enabled == true`, `Weather.DailyCallBudget == 800`,
  `Weather.AllowPaidOverage == false`, `Weather.DailyCallHardCeiling == 2000`,
  `Weather.CountGeocodingCalls == true`, `Weather.CurrentTTLMinutes == 15`,
  `Weather.HourlyTTLMinutes == 60`, `Weather.DailyTTLMinutes == 180`,
  `Weather.GeocodingTTLDays == 30`, `Weather.MaxLocations == 5`,
  `Weather.EagerLoadAllLocations == false`, and that `api_key = "${OPENWEATHER_API_KEY}"`
  interpolates from env (set the env var with `t.Setenv`).

- [ ] **Step 2: Run to verify failure** — `go test ./internal/config/...` → FAIL
  (undefined `Weather`).

- [ ] **Step 3: Add the config block to `config.toml`** (verbatim from the SOW,
  including its comments):

```toml
[weather]
# Master kill switch, mirroring [calendar_sync].enabled. false ⇒ no provider
# calls, no geocoding, no budget consumption; every /weather route returns
# status "disabled" and the tile says so plainly. Flipping it back resumes
# immediately — nothing to reconcile, because nothing is persisted but cache.
enabled = true

# OpenWeather One Call 4.0 + Geocoding. The one real secret here → env.
# Empty ⇒ the integration behaves exactly as if enabled = false.
api_key = "${OPENWEATHER_API_KEY}"

# --- cost control -------------------------------------------------------
# The auto-shutoff point: reserving past this refuses the call.
#
# 800, not 1000. Two reasons. First, our count and OpenWeather's can drift —
# a transport-layer retry or a call that fails after leaving this process is
# billed by them and may not be counted by us — so the stop belongs on OUR
# side of the paid boundary. Second, lazy loading puts realistic usage near
# ~128 calls/day, so the remaining headroom costs nothing and buys real
# protection against a render loop or retry storm.
daily_call_budget = 800

# false ⇒ daily_call_budget is a HARD STOP and the free tier can never be
# exceeded. true ⇒ calls continue past it up to daily_call_hard_ceiling and
# OpenWeather bills the difference. Overage is never unbounded: there is no
# configuration in which this integration can spend without a ceiling.
allow_paid_overage = false
daily_call_hard_ceiling = 2000

# Geocoding calls count against the budget. OpenWeather does not publicly
# document the Geocoding API as exempt from the One Call quota, so the safe
# assumption is that it is not. If confirmed exempt, flip to false and the
# budget stretches further.
count_geocoding_calls = true

# --- cache TTLs: the real cost lever ------------------------------------
# One Call 4.0 bills each timeline endpoint separately, so each gets the
# longest TTL its data tolerates. current is 15 (the provider itself only
# refreshes every ~10); an hourly strip and a daily high/low move far more
# slowly than a thermometer does.
current_ttl_minutes = 15
hourly_ttl_minutes  = 60
daily_ttl_minutes   = 180
geocoding_ttl_days  = 30   # a city's coordinates do not move

# --- product knobs ------------------------------------------------------
max_locations = 5
# false ⇒ the tile fetches only the location being viewed (see the SOW's
# Algorithms section). true ⇒ it fans out one GET /weather per saved location
# on mount, trading ~5× the provider calls for instant swipes. Client fetch
# pattern only — the API contract and the budget accounting are identical
# either way.
eager_load_all_locations = false
```

- [ ] **Step 4: Add `WeatherConfig`** to `internal/config/config.go`, following the
  `CalendarSyncConfig` pattern exactly (public struct + doc comments, matching
  `fileConfig` block with toml tags, mapping in `Load`):

```go
// WeatherConfig groups the OpenWeather dashboard-tile integration knobs.
// APIKey is the one secret (env-interpolated); everything else is a public
// literal in config.toml. See sows/weather-dashboard-tile.md.
type WeatherConfig struct {
	Enabled               bool
	APIKey                string
	DailyCallBudget       int
	AllowPaidOverage      bool
	DailyCallHardCeiling  int
	CountGeocodingCalls   bool
	CurrentTTLMinutes     int
	HourlyTTLMinutes      int
	DailyTTLMinutes       int
	GeocodingTTLDays      int
	MaxLocations          int
	EagerLoadAllLocations bool
}
```

with the `fileConfig` mirror using tags `enabled`, `api_key`, `daily_call_budget`,
`allow_paid_overage`, `daily_call_hard_ceiling`, `count_geocoding_calls`,
`current_ttl_minutes`, `hourly_ttl_minutes`, `daily_ttl_minutes`,
`geocoding_ttl_days`, `max_locations`, `eager_load_all_locations` under
`` `toml:"weather"` ``, and the field-by-field copy in `Load` next to the
`CalendarSync` mapping. `api_key` goes through the same `${VAR}` interpolation as
the FatSecret credentials.

- [ ] **Step 5: Run tests** — `go test ./internal/config/...` → PASS.

- [ ] **Step 6: Commit**

```bash
git add config.toml internal/config/config.go internal/config/config_test.go
git commit -m "feat(weather): add [weather] config block"
```

### Task A3: `internal/weather` models, Provider seam, OpenWeather provider

**Files:**
- Create: `internal/weather/models.go`
- Create: `internal/weather/provider.go`
- Create: `internal/weather/openweather.go`
- Test: `internal/weather/openweather_test.go`

- [ ] **Step 1: Write `models.go`** — the normalized (metric, canonical) shapes that
  get cached and served:

```go
package weather

import "time"

// Endpoint names one metered provider surface. They double as cache-key
// suffixes and metric label values, so the set is closed and lowercase.
type Endpoint string

const (
	EndpointCurrent       Endpoint = "current"
	EndpointHourly        Endpoint = "hourly"
	EndpointDaily         Endpoint = "daily"
	EndpointGeocodeDirect Endpoint = "geocode_direct"
	EndpointGeocodeReverse Endpoint = "geocode_reverse"
)

// Current is the normalized current-conditions reading, metric units.
// This is what weather_cache stores for the "current" endpoint — never the
// raw provider body, so a vendor swap can't leak provider shapes into cache.
type Current struct {
	TempC     float64 `json:"temp_c"`
	FeelsLikeC float64 `json:"feels_like_c"`
	Humidity  int     `json:"humidity"`
	WindKMH   float64 `json:"wind_kmh"`
	Condition string  `json:"condition"`
	Icon      string  `json:"icon"`
}

// HourlyBucket is one hour of the forecast strip.
type HourlyBucket struct {
	At    time.Time `json:"at"`
	TempC float64   `json:"temp_c"`
	Icon  string    `json:"icon"`
}

// Daily is today's summary from the daily timeline.
type Daily struct {
	HighC   float64   `json:"high_c"`
	LowC    float64   `json:"low_c"`
	Sunrise time.Time `json:"sunrise"`
	Sunset  time.Time `json:"sunset"`
}

// GeoResult is one geocoding candidate (direct search or reverse).
type GeoResult struct {
	Name    string  `json:"name"`
	State   string  `json:"state,omitempty"`
	Country string  `json:"country"`
	Lat     float64 `json:"lat"`
	Lon     float64 `json:"lon"`
}

// Location is a user's saved place, ordered by Position.
type Location struct {
	ID        string
	UserID    string
	Position  int
	Label     string
	Country   string
	State     *string
	Lat       float64
	Lon       float64
	CreatedAt time.Time
}
```

- [ ] **Step 2: Write `provider.go`**:

```go
package weather

import "context"

// Provider is the vendor-swap seam, modelled on nutritionlookup.Provider.
// Implementations must be safe for concurrent use, always return METRIC
// values, and surface HTTP failures as errors — the Service decides how to
// degrade. One method per metered endpoint so the budget ledger can reserve
// exactly the calls a refresh will make.
type Provider interface {
	Configured() bool
	Current(ctx context.Context, lat, lon float64) (Current, error)
	Hourly(ctx context.Context, lat, lon float64) ([]HourlyBucket, error)
	Daily(ctx context.Context, lat, lon float64) (Daily, error)
	GeocodeDirect(ctx context.Context, query string, limit int) ([]GeoResult, error)
	GeocodeReverse(ctx context.Context, lat, lon float64) ([]GeoResult, error)
}
```

- [ ] **Step 3: Write the failing provider test** — `openweather_test.go` with an
  `httptest.Server` that serves canned OpenWeather JSON for each endpoint, asserting:
  correct URL paths + query params (`lat`, `lon`, `appid`, metric units), normalized
  output values, error on non-200, and `Configured() == false` when the key is empty.
  Follow the shape of `internal/nutritionlookup`'s provider tests.

- [ ] **Step 4: Run to verify failure** — `go test ./internal/weather/...` → FAIL.

- [ ] **Step 5: Implement `openweather.go`** — `OpenWeatherProvider` struct holding
  `client *http.Client`, `apiKey string`, `baseURL string` (defaulting to
  `https://api.openweathermap.org`, overridable for tests, exactly like the
  FatSecret provider does). One Call 4.0 endpoints: `Current` →
  `GET {base}/data/4.0/current?lat=&lon=&units=metric&appid=`; `Hourly` →
  `GET {base}/data/4.0/timeline/1h?...` (return the next 12 buckets, service
  truncates to 5 for the tile); `Daily` → `GET {base}/data/4.0/timeline/1day?...`
  (today's entry). Geocoding: `GET {base}/geo/1.0/direct?q=&limit=&appid=` and
  `GET {base}/geo/1.0/reverse?lat=&lon=&limit=1&appid=`. Parse into the normalized
  models; icons pass through the provider's icon code (e.g. `01d`).

- [ ] **Step 6: Run tests** — `go test ./internal/weather/...` → PASS.

- [ ] **Step 7: Commit**

```bash
git add internal/weather/
git commit -m "feat(weather): normalized models and openweather provider"
```

### Task A4: Budget ledger — reserve-before-call

**Files:**
- Create: `internal/weather/budget.go`
- Test: `internal/weather/budget_test.go`

- [ ] **Step 1: Write failing tests** (this is the highest-value test in the SOW —
  it is the only thing standing between a bug and a bill). Table tests over the
  boundary (`calls_used` at `budget-1`, `budget`, `budget+1` via successive
  reserves), `allow_paid_overage` on/off ceiling selection, UTC date rollover with
  an injected clock (reserve, advance the clock past midnight UTC, reserve again,
  assert a fresh row), restart durability (reserve, construct a NEW ledger against
  the same `*sql.DB`, assert `UsedToday` is preserved — this test exists
  specifically to prevent regressing to the WHOOP counter failure), and a
  concurrency test: with ceiling 50, run 100 goroutines each reserving 1 and assert
  exactly 50 succeed and the final `calls_used` is exactly 50.

- [ ] **Step 2: Run to verify failure** — `go test ./internal/weather/ -run Budget` → FAIL.

- [ ] **Step 3: Implement**:

```go
package weather

import (
	"context"
	"database/sql"
	"errors"
	"time"
)

// ErrBudgetExhausted means today's reservation would cross the active
// ceiling; the provider call must not happen.
var ErrBudgetExhausted = errors.New("weather: daily call budget exhausted")

// BudgetLedger is the durable spend ledger. Reservation happens BEFORE the
// HTTP request: a crash between reserving and calling over-counts by one,
// which is the safe direction for a spend cap.
type BudgetLedger struct {
	db  *sql.DB
	now func() time.Time
}

func NewBudgetLedger(db *sql.DB) *BudgetLedger {
	return &BudgetLedger{db: db, now: time.Now}
}

// Reserve atomically claims n calls against today's UTC row, creating it if
// absent. n is a parameter because a full refresh of one location is up to
// three calls and reserving them atomically avoids a half-refreshed location
// that consumed budget for a card it cannot draw.
func (l *BudgetLedger) Reserve(ctx context.Context, n, ceiling int) error {
	day := l.now().UTC().Format("2006-01-02")
	tx, err := l.db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer func() { _ = tx.Rollback() }()

	if _, err := tx.ExecContext(ctx, `
		INSERT INTO weather_call_budget (usage_date, calls_used, updated_at)
		VALUES (?, 0, ?)
		ON CONFLICT(usage_date) DO NOTHING
	`, day, l.now().UTC()); err != nil {
		return err
	}
	res, err := tx.ExecContext(ctx, `
		UPDATE weather_call_budget
		SET calls_used = calls_used + ?, updated_at = ?
		WHERE usage_date = ? AND calls_used + ? <= ?
	`, n, l.now().UTC(), day, n, ceiling)
	if err != nil {
		return err
	}
	affected, err := res.RowsAffected()
	if err != nil {
		return err
	}
	if affected == 0 {
		return ErrBudgetExhausted
	}
	return tx.Commit()
}

// UsedToday reads today's UTC row; 0 when the day has no reservations yet.
func (l *BudgetLedger) UsedToday(ctx context.Context) (int, error) {
	day := l.now().UTC().Format("2006-01-02")
	var used int
	err := l.db.QueryRowContext(ctx,
		`SELECT calls_used FROM weather_call_budget WHERE usage_date = ?`, day,
	).Scan(&used)
	if errors.Is(err, sql.ErrNoRows) {
		return 0, nil
	}
	return used, err
}
```

- [ ] **Step 4: Run tests** — `go test ./internal/weather/... -race` → PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/weather/budget.go internal/weather/budget_test.go
git commit -m "feat(weather): durable reserve-before-call budget ledger"
```

### Task A5: Cache + locations repositories

**Files:**
- Create: `internal/weather/repository.go` (interfaces + cache-key construction)
- Create: `internal/weather/sqlite_repository.go`
- Test: `internal/weather/sqlite_repository_test.go`

- [ ] **Step 1: Write failing tests**: cache get-miss returns `(nil, nil)`; put-then-get
  round-trips and bumps `last_used_at` on read; a put sweeps rows with
  `last_used_at` older than 90 days (injected clock); key construction rounds
  coordinates to 2dp (coordinates 0.004° apart share a key: `39.741` and `39.744`
  both key `39.74`); `geocode_direct` keys normalize the query (`"Denver  CO"` ==
  `"denver co"`). Locations: `ReplaceAll` rewrites the whole list transactionally
  (positions 0..n-1 in the given order), `List` returns ordered by position, a
  replace with a reordered input list preserves ids and coordinates, and
  `ReplaceAll` with an empty slice clears the list.

- [ ] **Step 2: Run to verify failure**, then **Step 3: Implement**.

`repository.go`:

```go
package weather

import (
	"context"
	"fmt"
	"strings"
	"time"
)

// CacheRow mirrors weather_cache. payload_json holds the normalized reading
// for the endpoint the key names, never the raw provider body.
type CacheRow struct {
	CacheKey    string
	PayloadJSON string
	FetchedAt   time.Time
	LastUsedAt  time.Time
}

type CacheRepository interface {
	Get(ctx context.Context, key string) (*CacheRow, error)
	Put(ctx context.Context, row CacheRow) error
	// LastSuccess is the newest fetched_at across all rows — the durable
	// liveness signal the metrics collector publishes. Zero time when the
	// cache is empty.
	LastSuccess(ctx context.Context) (time.Time, error)
}

type LocationsRepository interface {
	List(ctx context.Context, userID string) ([]Location, error)
	ReplaceAll(ctx context.Context, userID string, locations []Location) error
	Count(ctx context.Context) (int, error) // all users; feeds api_weather_saved_locations
}

// ReadingKey builds the coordinate cache key. 2 decimal places (~1.1 km):
// full precision would fragment the cache — two users, or one user re-running
// "use my location", would produce near-identical coordinates that miss each
// other's cached readings and burn budget for no benefit.
func ReadingKey(lat, lon float64, endpoint Endpoint) string {
	return fmt.Sprintf("%.2f:%.2f:%s", lat, lon, endpoint)
}

func GeocodeDirectKey(query string) string {
	return "geocode_direct:" + normalizeQuery(query)
}

func GeocodeReverseKey(lat, lon float64) string {
	return fmt.Sprintf("geocode_reverse:%.2f:%.2f", lat, lon)
}

// normalizeQuery matches nutritionlookup's normalization: lower-cased,
// whitespace collapsed, so "Denver  CO" and "denver co" share one row.
func normalizeQuery(q string) string {
	return strings.Join(strings.Fields(strings.ToLower(q)), " ")
}
```

`sqlite_repository.go`: `SQLiteCacheRepository` (fields `db *sql.DB`,
`now func() time.Time`; `const cacheEvictionAge = 90 * 24 * time.Hour`) with
`Get` bumping `last_used_at` on hit (miss ⇒ `(nil, nil)`), `Put` upserting via
`ON CONFLICT(cache_key) DO UPDATE` then sweeping
`DELETE FROM weather_cache WHERE last_used_at < ?` — copy the shapes from
`internal/nutritionlookup/sqlite_repository.go`. `LastSuccess` is
`SELECT MAX(fetched_at) FROM weather_cache` (NULL ⇒ zero time).
`SQLiteLocationsRepository` with `List` (`ORDER BY position`), `ReplaceAll`
(one transaction: `DELETE FROM user_weather_locations WHERE user_id = ?` then
insert each row with position = slice index, generating `id.New()` for rows
whose ID is empty and preserving existing ids otherwise), `Count`
(`SELECT COUNT(*)`).

- [ ] **Step 4: Run tests** — PASS. **Step 5: Commit**

```bash
git add internal/weather/repository.go internal/weather/sqlite_repository.go internal/weather/sqlite_repository_test.go
git commit -m "feat(weather): global reading cache and saved-locations repositories"
```

### Task A6: Metrics + durable-state collector

**Files:**
- Create: `internal/weather/metrics.go`
- Create: `internal/weather/collector.go`
- Test: `internal/weather/collector_test.go`

- [ ] **Step 1: Write `metrics.go`** following
  `internal/nutritionlookup/metrics.go` (package-level vars, `init()` MustRegister):

  - `api_weather_requests_total` counter, label `outcome` ∈
    `cache_hit|served|served_stale|budget_exhausted|disabled|failed`
  - `api_weather_provider_calls_total` counter, labels `endpoint`
    (`current|hourly|daily|geocode_direct|geocode_reverse`) × `result` (`ok|error`)
  - `api_weather_provider_latency_seconds` histogram, label `endpoint`
    (same buckets as nutritionlookup's provider duration histogram)
  - `api_weather_cache_events_total` counter, label `event` ∈
    `hit|miss|stale|corrupt|read_error`
  - `api_weather_cache_writes_total` counter, label `result` ∈ `ok|error`
  - Gauges (no labels): `api_weather_calls_used_today`, `api_weather_daily_budget`,
    `api_weather_budget_utilization_ratio`, `api_weather_shutoff_active`,
    `api_weather_last_success_timestamp_seconds`, `api_weather_enabled`,
    `api_weather_saved_locations`

- [ ] **Step 2: Write failing collector test** — seed the ledger (e.g. reserve 400
  of an 800 ceiling), a cache row, and two locations; call `refresh`; assert gauge
  values via `prometheus/client_golang/prometheus/testutil` (`ToFloat64`):
  used=400, budget=800, utilization=0.5, shutoff=0, last_success == the row's
  fetched_at unix, enabled=1, saved_locations=2. Second case: budget misconfigured
  to 0 ⇒ utilization 0 (divide-by-zero handled once, in Go, not in five alert rules).
  Third case: used == ceiling ⇒ shutoff 1.

- [ ] **Step 3: Implement `collector.go`** mirroring
  `internal/whoopadmin/connections_gauge.go`: an `Exporter` struct holding the
  ledger, cache repo, locations repo, and the resolved config (enabled, active
  ceiling); `Run(ctx)` refreshes immediately then every 5 minutes;
  `refresh(ctx)` reads durable state and Sets the seven gauges. The active ceiling
  is `hard_ceiling` when overage is allowed, else `daily_call_budget`. Publishing
  utilization as its own gauge keeps the alert expressions trivial. Zero
  `last_success` publishes 0 (the alert's `or vector(0)` handles absence too).

- [ ] **Step 4: Run tests** — PASS. **Step 5: Commit**

```bash
git add internal/weather/metrics.go internal/weather/collector.go internal/weather/collector_test.go
git commit -m "feat(weather): prometheus metrics and durable-state gauge collector"
```

### Task A7: Service — cache-first readings with degradation

**Files:**
- Create: `internal/weather/service.go`
- Test: `internal/weather/service_test.go`

- [ ] **Step 1: Write failing service tests** with a `fakeProvider` (per-endpoint
  canned values, error injection, call counters — the nutritionlookup fake pattern):

  - fresh hit: warm cache, zero provider calls, `Status == StatusOK`, outcome
    metric `cache_hit`
  - cold cache: three provider calls, three cache writes, `StatusOK`, ledger shows 3
  - stale serve on provider error: expired cache + erroring provider ⇒
    `StatusStale`, full reading, older `FetchedAt`
  - stale serve on budget exhaustion: ledger at ceiling ⇒ `StatusBudgetExhausted`,
    stale reading attached, zero provider calls
  - hard failure with no cache: `StatusUnavailable`
  - `enabled=false` ⇒ `StatusDisabled`, zero provider calls, zero reservations
  - empty api key behaves exactly like disabled
  - TTL boundaries with injected clock: current expires at 15m, hourly 60m,
    daily 180m; only expired endpoints are re-fetched and only that many calls
    reserved (e.g. current stale + hourly/daily fresh ⇒ reserve(1))
  - geocode search: cold ⇒ reserve(1) + provider call + 30-day cache write; warm ⇒
    no reservation; `count_geocoding_calls=false` ⇒ no reservation but still cached
  - `FetchedAt` on a mixed reading is the OLDEST served endpoint's fetched_at
    (an honest "updated 2h ago")

- [ ] **Step 2: Run to verify failure**, then **Step 3: Implement `service.go`**:

```go
package weather

// Status is the explicit reading disposition — the tile never has to guess
// why a payload is thin.
type Status string

const (
	StatusOK              Status = "ok"
	StatusStale           Status = "stale"
	StatusDisabled        Status = "disabled"
	StatusBudgetExhausted Status = "budget_exhausted"
	StatusUnavailable     Status = "unavailable"
)

// Reading is the assembled tile payload for one location, still metric;
// the handler converts units per user.
type Reading struct {
	Status    Status
	FetchedAt time.Time
	Current   *Current
	Hourly    []HourlyBucket
	Daily     *Daily
}

type Service struct {
	cfg       config.WeatherConfig
	cache     CacheRepository
	locations LocationsRepository
	budget    *BudgetLedger
	provider  Provider
	log       *slog.Logger
	now       func() time.Time
}
```

Core methods:

  - `Readings(ctx, lat, lon float64) Reading` — the algorithm:
    1. disabled or unconfigured provider ⇒ `StatusDisabled`.
    2. For each of the three endpoints read cache (recording
       hit/miss/stale/corrupt/read_error events), classifying fresh / stale /
       missing against its TTL.
    3. All three fresh ⇒ assemble, `StatusOK`, outcome `cache_hit`.
    4. Otherwise `n :=` count of non-fresh endpoints; `budget.Reserve(ctx, n, ceiling)`.
       `ErrBudgetExhausted` ⇒ assemble whatever cache exists (fresh + stale),
       `StatusBudgetExhausted`, outcome `budget_exhausted`, and set the shutoff
       gauge refresh on next collector tick (no direct action needed).
    5. Reserved ⇒ call the provider for each non-fresh endpoint, timing each call
       into the latency histogram and counting ok/error. Success ⇒ marshal the
       normalized payload, upsert cache (write errors are logged and swallowed —
       the cache is an optimization — but metered via
       `api_weather_cache_writes_total{result="error"}`). Failure ⇒ fall back to
       that endpoint's stale row if any.
    6. Disposition: every endpoint fresh-or-just-fetched ⇒ `StatusOK` (outcome
       `served`); at least one endpoint served stale ⇒ `StatusStale` (outcome
       `served_stale`); nothing at all for `current` ⇒ `StatusUnavailable`
       (outcome `failed`).
    7. `FetchedAt` = oldest fetched_at among served endpoints.
  - `Search(ctx, query string, limit int) ([]GeoResult, Status, error)` —
    disabled check ⇒ `StatusDisabled`; cache-first on `GeocodeDirectKey` with the
    30-day TTL; on miss, reserve 1 iff `CountGeocodingCalls`, call
    `GeocodeDirect`, cache. Budget exhausted with no cache ⇒
    `StatusBudgetExhausted` and empty results.
  - `Reverse(ctx, lat, lon float64) ([]GeoResult, Status, error)` — same shape on
    `GeocodeReverseKey`.

- [ ] **Step 4: Run tests** — `go test ./internal/weather/... -race` → PASS.
- [ ] **Step 5: Commit**

```bash
git add internal/weather/service.go internal/weather/service_test.go
git commit -m "feat(weather): cache-first service with budget gate and degradation"
```

### Task A8: Handler — the five `/weather` routes

**Files:**
- Create: `internal/weather/handler.go`
- Test: `internal/weather/handler_test.go`

- [ ] **Step 1: Write failing handler tests** (chi router + `httptest`, JWT context
  injected the way `internal/nutritionlookup/handler_test.go` does):

  - `GET /weather` without `timezone` ⇒ 400 "timezone is required"; junk timezone ⇒
    400 "invalid timezone …" (the `/dashboard/summary` convention verbatim)
  - `GET /weather` with no saved locations ⇒ 404, code `no_locations`
  - `GET /weather` resolves the first saved location when `location_id` absent;
    unknown `location_id` ⇒ 404
  - unit derivation: user with `distance_unit=mi` gets `units {"temp":"F","wind":"mph"}`
    and converted, rounded values; `km` ⇒ `{"temp":"C","wind":"km/h"}`
  - `enabled=false` ⇒ 200 with `status:"disabled"` on all five routes (kill switch
    must NOT change the route table)
  - `PUT /weather/locations`: 400 on >max_locations, out-of-range coordinates
    (|lat|>90, |lon|>180), or a blank label; success ⇒ list replaced in given order
  - round-trip: `GET /weather/locations` → reorder client-side → `PUT` → `GET`
    asserts coordinates survive unchanged (guards the read-modify-write hazard)
  - `GET /weather/locations` echoes `settings {enabled, max_locations,
    eager_load_all_locations}`
  - `GET /weather/search` requires `q`; `GET /weather/reverse` validates lat/lon

- [ ] **Step 2: Run to verify failure**, then **Step 3: Implement `handler.go`**.

Routes (all inside the JWT-gated router; `Mount(r chi.Router)`):

```go
r.Get("/weather", h.readings)
r.Get("/weather/locations", h.listLocations)
r.Put("/weather/locations", h.putLocations)
r.Get("/weather/search", h.search)
r.Get("/weather/reverse", h.reverse)
```

The handler depends on the Service, the LocationsRepository, the resolved
`config.WeatherConfig`, and a narrow user reader for the unit preference:

```go
// userReader is the slice of the user store the handler needs: the
// distance_unit preference that picks °F/mph vs °C/km/h.
type userReader interface {
	// use the existing user repository's getter — check internal/user for
	// the exact method (the /me handler wiring in server.go shows it).
}
```

`GET /weather` response `data` shape (inside the standard httpresp envelope):

```jsonc
{
  "status": "ok",              // ok | stale | disabled | budget_exhausted | unavailable
  "location": { "id": "…", "label": "Denver", "state": "CO", "country": "US" },
  "fetched_at": "2026-08-08T14:02:11Z",
  "units": { "temp": "F", "wind": "mph" },
  "current": { "temp": 38, "feels_like": 29, "humidity": 41,
               "wind_speed": 14, "condition": "Clear", "icon": "01d" },
  "today": { "high": 46, "low": 24, "sunrise": "2026-08-08T12:14:02Z", "sunset": "2026-08-09T01:52:40Z" },
  "hourly": [ { "at": "2026-08-08T15:00:00Z", "temp": 33, "icon": "01d" } ]   // next 5 buckets
}
```

`current`/`today`/`hourly` are omitted (`omitempty`) when the service has nothing
to serve. Temperatures/wind are rounded to integers after unit conversion
(°F = C×9/5+32; mph = km/h ÷ 1.609344). Hourly buckets are truncated to the next
5 whose `at` is >= now. `status:"disabled"` responses carry only `status`.

`GET /weather/locations` `data`:

```jsonc
{
  "locations": [ { "id": "…", "label": "Denver", "state": "CO", "country": "US",
                   "lat": 39.74, "lon": -104.98 } ],
  "settings": { "enabled": true, "max_locations": 5, "eager_load_all_locations": false }
}
```

Coordinates ARE returned (writes are a whole-list replace: the client sends back
what it read; omitting lat/lon would make a reorder silently destroy them).

`PUT /weather/locations` body: `{"locations": [{id?, label, state?, country, lat, lon}]}` —
400 (via `httpresp.Error`) on cap/coordinate/label violations; 200 with the saved
list on success. `GET /weather/search?q=&limit=` (limit default 5, max 5) and
`GET /weather/reverse?lat=&lon=` return
`{"results": [{name, state, country, lat, lon}], "status": "ok"}` (status
`disabled`/`budget_exhausted` with empty results when degraded).

- [ ] **Step 4: Run tests** — PASS. **Step 5: Commit**

```bash
git add internal/weather/handler.go internal/weather/handler_test.go
git commit -m "feat(weather): mount the five /weather routes"
```

### Task A9: Catalog — `weather` TileID

**Files:**
- Modify: `internal/dashboard/tiles.go`
- Test: `internal/dashboard/tiles_test.go`

- [ ] **Step 1: Update the catalog tests to expect `weather` appended after `quote`**
  (failing first). **Step 2: Run to verify failure.**
- [ ] **Step 3: Add** `TileWeather TileID = "weather"` and append `TileWeather` to
  `Catalog` (last, after `TileQuote`).
- [ ] **Step 4: Verify the summary path tolerates a sectionless tile** — read
  `internal/dashboard`'s summary handler/section resolution and confirm a layout
  containing `weather` (a) passes `PUT /dashboard/layout` validation, (b) does not
  break `GET /dashboard/summary` (the tile has no summary section by design — the
  SOW's "Why weather is not a summary section"). If the summary builder switches
  over tile ids exhaustively, add the explicit no-op case with a comment pointing
  at the SOW; add/extend a test asserting a layout with `weather` returns a
  summary without error.
- [ ] **Step 5: Run** `go test ./internal/dashboard/...` → PASS. **Step 6: Commit**

```bash
git add internal/dashboard/
git commit -m "feat(dashboard): add the weather tile id to the catalog"
```

### Task A10: Server wiring

**Files:**
- Modify: `internal/server/server.go`

- [ ] **Step 1: Wire it** next to the nutritionlookup wiring (~line 397) and the
  whoopadmin exporter (~line 536), following the repo's construction order
  (repo → providers → service → handler → mount):

  - construct `weather.NewSQLiteCacheRepository(database)`,
    `weather.NewSQLiteLocationsRepository(database)`,
    `weather.NewBudgetLedger(database)`,
    `weather.NewOpenWeatherProvider(httpClient, cfg.Weather.APIKey)`
  - `weatherSvc := weather.NewService(cfg.Weather, cacheRepo, locationsRepo, ledger, provider, logger)`
  - mount the handler in the JWT-gated group next to
    `nutritionlookup.NewHandler(...).Mount(r)` (~line 585)
  - start the collector on the background context next to the whoopadmin exporter:
    `go weather.NewExporter(cfg.Weather, ledger, cacheRepo, locationsRepo).Run(bgCtx)`
  - respect the repo's in-memory-mode handling (~line 214): if the server supports
    running without a database, gate the weather wiring the same way the
    nutritionlookup repo construction is gated, or provide in-memory equivalents —
    match whatever the surrounding code does for nutritionlookup.

- [ ] **Step 2: Build and test the whole repo** — `go build ./... && go test ./...` → PASS.
- [ ] **Step 3: Commit**

```bash
git add internal/server/server.go
git commit -m "feat(weather): wire service, routes, and metrics collector"
```

### Task A11: API local gate

- [ ] **Step 1: Run the full gate** (fix anything that fails — never `//nolint`,
  never skip a test):

```bash
cd /workspace/prog-strength-api
go build ./... && go vet ./...
golangci-lint run
go mod tidy && git diff --exit-code go.mod go.sum
go test -race ./...
```

- [ ] **Step 2: Commit any gate fixes** (`fix(weather): …`).

---

# PART 2 — prog-strength-web

### Task W1: TS catalog + contract test

**Files:**
- Modify: `lib/dashboard-tiles.ts`
- Test: `lib/dashboard-tiles.test.ts`

- [ ] **Step 1: Create the branch**

```bash
cd /workspace/prog-strength-web && git checkout -b feat/weather-dashboard-tile
```

- [ ] **Step 2: Update the contract test first (failing)**: expected count 19→20,
  `"weather"` appended after `"quote"` in the order array, `PAGELESS_TILE_IDS`
  becomes `["quote", "weather"]`, `ALL_TILE_IDS` gains `weather: true`.
- [ ] **Step 3: Run** `npx vitest run lib/dashboard-tiles.test.ts` → FAIL.
- [ ] **Step 4: Add to the union and catalog** (append after `quote`; NO `href` —
  there is no weather page):

```typescript
{
  id: "weather",
  title: "Weather",
  description: "Forecast for your saved places — conditions, high/low, and the next few hours.",
},
```

- [ ] **Step 5: Run the test** → PASS. **Step 6: Commit**

```bash
git add lib/dashboard-tiles.ts lib/dashboard-tiles.test.ts
git commit -m "feat(dashboard): add the weather tile to the catalog"
```

Note: the tile-renderer switch now fails `npm run typecheck` (non-exhaustive)
until Task W4 — the pre-commit hook runs `tsc --noEmit`, so Tasks W1–W4 land as
ONE commit if the hook rejects intermediate states. Preferred: do W1's catalog
edit and W4's renderer case in the same commit if `tsc` blocks; keep tests split
as written.

### Task W2: API wrappers + types

**Files:**
- Modify: `lib/api.ts`

- [ ] **Step 1: Add types + five wrappers** following the file's conventions
  (envelope `unwrap`, `encodeURIComponent` on params, Bearer token):

```typescript
export type WeatherStatus = "ok" | "stale" | "disabled" | "budget_exhausted" | "unavailable";

export type WeatherLocation = {
  id: string;
  label: string;
  state?: string;
  country: string;
  lat: number;
  lon: number;
};

export type WeatherSettings = {
  enabled: boolean;
  max_locations: number;
  eager_load_all_locations: boolean;
};

export type WeatherReading = {
  status: WeatherStatus;
  location?: { id: string; label: string; state?: string; country: string };
  fetched_at?: string;
  units?: { temp: "F" | "C"; wind: "mph" | "km/h" };
  current?: { temp: number; feels_like: number; humidity: number;
              wind_speed: number; condition: string; icon: string };
  today?: { high: number; low: number; sunrise: string; sunset: string };
  hourly?: { at: string; temp: number; icon: string }[];
};

export type WeatherGeoResult = {
  name: string; state?: string; country: string; lat: number; lon: number;
};

export async function getWeather(token, timezone, locationId?): Promise<WeatherReading | null>
export async function getWeatherLocations(token): Promise<{ locations: WeatherLocation[]; settings: WeatherSettings } | null>
export async function putWeatherLocations(token, locations: WeatherLocation[] /* id optional on new rows */): Promise<WeatherLocation[]>
export async function searchWeatherLocations(token, q): Promise<WeatherGeoResult[]>
export async function reverseWeatherLocation(token, lat, lon): Promise<WeatherGeoResult[]>
```

(`getWeather` hits `/weather?timezone=…&location_id=…`; put sends
`{locations}` with `PUT`; new locations may omit `id` — type the PUT input as
`(Omit<WeatherLocation, "id"> & { id?: string })[]`.)

- [ ] **Step 2: `npm run typecheck`** → PASS. **Step 3: Commit**

```bash
git add lib/api.ts
git commit -m "feat(weather): typed api wrappers for the weather endpoints"
```

### Task W3: Locations popover

**Files:**
- Create: `app/(app)/dashboard/_components/weather-locations-popover.tsx`
- Test: `app/(app)/dashboard/_components/weather-locations-popover.test.tsx`

- [ ] **Step 1: Write failing tests**: renders the saved list with an `n/5` counter;
  "Add" is disabled at the cap (5 rows ⇒ search result rows show no add
  affordance / disabled buttons); typing in the search box debounces (fake
  timers) and calls `searchWeatherLocations` once for the settled value;
  selecting a result calls `onChange` with the appended list; delete calls
  `onChange` without the row; "Use my current location" calls
  `navigator.geolocation.getCurrentPosition` (mock it on `global.navigator`) and
  on success calls `reverseWeatherLocation` then `onChange`; a denied permission
  (error callback) renders an inline muted note, not a thrown error.

- [ ] **Step 2: Run to verify failure**, then **Step 3: Implement**.

Props: `{ locations, settings, onChange(next: WeatherLocation-ish[]), onClose }` —
the popover is controlled; the tile owns the PUT round-trip. Composition:

  - a panel positioned under the gear: `bg-[var(--surface)]`,
    `border border-[var(--border)]`, `rounded-[var(--radius-card)]`,
    `shadow-[var(--shadow-raised)]`, ~w-72
  - header row: quiet uppercase label ("Locations") in `--muted` + `n/5` counter
    in `--faint`
  - search input styled per the design system's form-control spec: `--surface-2`
    fill, hairline `--border`, `--muted` placeholder, accent focus ring
    (`outline-none focus:border-[var(--accent)]`), 300 ms debounce, results as
    rounded rows (`name, state country` with the metadata in `--muted`)
  - saved list: `@dnd-kit` vertical sortable (copy the `useSortable` +
    `verticalListSortingStrategy` pattern from `section-list.tsx`, single-list
    case — no cross-container logic needed), drag handle in `--faint`, delete
    button (small ×) per row
  - "Use my current location" — a full-width secondary action (surface-2 pill,
    hairline border). Geolocation is requested ONLY on this press, never on
    load; resolved once via reverse geocoding and saved as a pinned coordinate.
    Denied ⇒ inline `--muted` note ("Location permission denied").
  - all colors from tokens; `--accent` only for focus/selection chrome

- [ ] **Step 4: Run tests** → PASS. **Step 5: Commit**

```bash
git add app/\(app\)/dashboard/_components/weather-locations-popover.*
git commit -m "feat(weather): locations popover with search, reorder, and geolocation"
```

### Task W4: Weather tile + renderer case

**Files:**
- Create: `app/(app)/dashboard/_components/weather-tile.tsx`
- Create: `app/(app)/dashboard/_components/weather-icons.tsx`
- Modify: `app/(app)/dashboard/_components/tile-renderer.tsx`
- Test: `app/(app)/dashboard/_components/weather-tile.test.tsx`

- [ ] **Step 1: Write failing tile tests** (mock `@/lib/api` + `@/lib/auth` via
  `vi.hoisted`): shows the skeleton while loading; no saved locations ⇒ an
  inviting "Add a location" CTA (MiniCardEmpty-style, opens the popover — NOT an
  error banner); `ok` ⇒ label, temp, condition, high/low, 5 hourly buckets;
  `stale` ⇒ full reading plus a quiet "updated Nh ago" note; `budget_exhausted`
  / `unavailable` / `disabled` ⇒ a calm `--muted` one-liner (assert no
  alert/error styling); a REJECTED `getWeather` promise degrades to the
  unavailable line rather than throwing; paging: with 2+ locations, dots render,
  `‹ ›` buttons switch location and fetch the newly-visible one exactly once
  (second visit served from component cache), `ArrowLeft`/`ArrowRight` on the
  focused card page too; always opens on location 1.

- [ ] **Step 2: Run to verify failure**, then **Step 3: Implement `weather-tile.tsx`**:

  - `"use client"`; `WeatherCard` takes NO data props (self-fetching — weather is
    deliberately not a `/dashboard/summary` section)
  - on mount: `getWeatherLocations(token)`; then `getWeather(token, browserTz,
    firstLocation.id)`. Timezone from
    `Intl.DateTimeFormat().resolvedOptions().timeZone` at call time (the repo
    convention). If `settings.eager_load_all_locations`, fan out one `getWeather`
    per saved location on mount; otherwise fetch lazily on first visit of each
    location and keep a `Record<locationId, WeatherReading>` component cache
  - page state is ephemeral (`useState(0)`) — always opens on location 1
  - paging UI: `‹ ›` buttons in `--muted` (hidden when 1 location), dot
    indicators (active dot `--foreground`, inactive `--faint`), `onKeyDown`
    Arrow handlers on the focusable card body (`tabIndex={0}`), touch swipe via
    `onTouchStart`/`onTouchEnd` deltaX threshold (~40px)
  - layout inside `MiniCard title="Weather"` (no `href`, resolved like `quote`):
    location label + state/country metadata line (`--muted`); big current temp
    (tight numeric tracking, `--foreground`) with condition icon + feels-like /
    humidity / wind metadata row in `--muted`; high/low line; hourly strip of 5
    columns (hour label in `--faint`, small icon, temp) separated by hairline
    border-t; gear button (top-right, `--muted`, hover `--foreground`) toggles
    the popover
  - the popover's `onChange` does the whole-list `putWeatherLocations` (sending
    back the coordinates it read), refetches locations, clamps the page index,
    and invalidates the reading cache for removed/changed locations
  - number formatting: `Math.round` on temps; `fetched_at` age line for stale
    ("updated 2h ago") computed from `Date.now() - fetched_at`

- [ ] **Step 4: Implement `weather-icons.tsx`** — one `WeatherIcon({ icon, className })`
  mapping OpenWeather icon-code prefixes (`01`→clear, `02|03|04`→clouds,
  `09|10`→rain, `11`→storm, `13`→snow, `50`→fog) to hand-rolled 24-viewBox
  `stroke="currentColor"` SVGs (repo icon convention, see
  `components/chat/icons.tsx`). Rendered in `--foreground`/`--muted` neutrals
  only — the palette reserves saturated color for activity disciplines, so no
  sky-blue and no sun-yellow.

- [ ] **Step 5: Add the renderer case** in `tile-renderer.tsx` — weather has no
  `href` and no summary section, so it resolves BEFORE the href assertion,
  alongside `quote`:

```tsx
if (id === "quote") {
  return data.quote.present ? <QuoteCard quote={data.quote} /> : null;
}
// Weather self-fetches — deliberately not a /dashboard/summary section
// (see sows/weather-dashboard-tile.md), and like quote it has no page
// behind it, so it also resolves before the href assertion.
if (id === "weather") {
  return <WeatherCard />;
}
const href = tileEntry(id).href as string;
```

- [ ] **Step 6: Run** `npx vitest run app` → PASS. **Step 7: Commit**

```bash
git add app/\(app\)/dashboard/_components/weather-tile.* app/\(app\)/dashboard/_components/weather-icons.tsx app/\(app\)/dashboard/_components/tile-renderer.tsx
git commit -m "feat(weather): self-fetching weather tile with paging and tile states"
```

### Task W5: Web local gate

- [ ] **Step 1: Run the full gate** (fix failures; don't disable rules):

```bash
cd /workspace/prog-strength-web
npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build
```

- [ ] **Step 2: Commit any fixes.**

---

# PART 3 — prog-strength-infra

### Task I1: Secrets plumbing — `OPENWEATHER_API_KEY`

**Files:**
- Modify: `deploy/api.sh` (REQUIRED_ENV_KEYS)
- Modify: `.github/workflows/seed-secrets.yml`
- Modify: `compose/api/docker-compose.yml`

- [ ] **Step 1: Create the branch**

```bash
cd /workspace/prog-strength-infra && git checkout -b feat/weather-dashboard-tile
```

- [ ] **Step 2: `deploy/api.sh`** — add `OPENWEATHER_API_KEY` to the
  `REQUIRED_ENV_KEYS` array. The SOW is explicit: it must be gated, or a deploy
  succeeds with a silently keyless integration.
- [ ] **Step 3: `seed-secrets.yml`** — add `OPENWEATHER_API_KEY` to the api job's
  `env:` block, the `jq --arg` list, and the payload object (alongside the other
  provider keys). Also add it to the `required_secrets` gate array — it is in the
  deploy's REQUIRED_ENV_KEYS, so seeding without it would wedge every subsequent
  deploy; failing the seed loudly is the kinder failure.
- [ ] **Step 4: `compose/api/docker-compose.yml`** — forward it into the api
  service's `environment:` block with a why-comment following the file's style
  (note that config.toml `[weather]` references it as a `${VAR}` label, that an
  empty value disables the integration exactly like `enabled=false`, and the
  standing trap: the deploy gate validates `.env` on the HOST, so this forward
  line is load-bearing):

```yaml
      # OpenWeather One Call + Geocoding key (config.toml [weather] references
      # it as a ${VAR} label). Deploy-gated in REQUIRED_ENV_KEYS: an empty key
      # silently disables the weather tile, which is exactly the failure the
      # gate exists to catch. The require_env_keys gate validates .env on the
      # HOST, not the container, so this forward line is load-bearing.
      - OPENWEATHER_API_KEY=${OPENWEATHER_API_KEY}
```

- [ ] **Step 5: Run the deploy tests + shellcheck**

```bash
for t in deploy/tests/*.test.sh; do bash "$t"; done
shellcheck -x deploy/*.sh deploy/lib/*.sh
```

Expected: PASS (the compose-forwards test is exactly what Step 4 satisfies).

- [ ] **Step 6: Commit**

```bash
git add deploy/api.sh .github/workflows/seed-secrets.yml compose/api/docker-compose.yml
git commit -m "feat(weather): plumb OPENWEATHER_API_KEY through secrets, deploy gate, and compose"
```

### Task I2: Grafana dashboard — `monitoring/grafana/dashboards/weather.json`

**Files:**
- Create: `monitoring/grafana/dashboards/weather.json`

- [ ] **Step 1: Author the dashboard** following the house shape (see
  `nutrition-lookup.json` for structure): top-level
  `{"title": "Weather", "uid": "ps-weather", "tags": ["prog-strength"],
  "schemaVersion": 39, "refresh": "30s", "time": {"from": "now-24h", "to": "now"},
  "timezone": "browser", "panels": [...]}`; every panel uses datasource
  `{"type": "prometheus", "uid": "prometheus"}` and carries a `description`
  explaining what it shows and what a bad value looks like. Three rows
  (`type: "row"`, `collapsed: false`), each with a description. Panel ids are
  pinned below because the alert rules link to them.

**Row 1 — "Budget & Cost Control"** (row id 100) — *how close am I to auto-shutoff*:

| id | type | title | expr | notes |
|---|---|---|---|---|
| 1 | gauge | Budget utilization | `api_weather_budget_utilization_ratio * 100` | unit `percent`, min 0 max 100, absolute thresholds green@null / yellow@75 / red@90 |
| 2 | timeseries | Calls used today vs. budget | A: `api_weather_calls_used_today`, B: `api_weather_daily_budget`, C: `api_weather_daily_budget * 0.75`, D: `api_weather_daily_budget * 0.9` | legend: used / budget / warning / critical; style B–D as dashed threshold lines via overrides |
| 3 | stat | Auto-shutoff status | `api_weather_shutoff_active` | value mappings 0→`ACTIVE` (green), 1→`SHUT OFF` (red) |
| 4 | stat | Projected end-of-day calls | `api_weather_calls_used_today / clamp_min(time() % 86400, 1) * 86400` | description: current rate extrapolated to 24h so the ceiling is visible before it is hit; thresholds green@null / yellow@600 / red@800 |
| 5 | timeseries | Calls by endpoint | `sum by (endpoint) (increase(api_weather_provider_calls_total[1h]))` | where budget is actually going |

**Row 2 — "Integration Health"** (row id 101) — *is it working*:

| id | type | title | expr | notes |
|---|---|---|---|---|
| 6 | stat | Time since last successful call | `time() - (max(api_weather_last_success_timestamp_seconds) or vector(0))` | unit `s`, thresholds green@null / red@21600 (6h, mirrors the liveness alert) |
| 7 | timeseries | Request outcomes | `sum by (outcome) (rate(api_weather_requests_total[5m]))` | stacked (fillOpacity 50, stacking normal); description: a rising served_stale or budget_exhausted band is the visual signature of degradation |
| 8 | timeseries | Provider errors by endpoint | `sum by (endpoint) (rate(api_weather_provider_calls_total{result="error"}[5m]))` | |
| 9 | timeseries | Provider latency p50/p95 | `histogram_quantile(0.5, sum by (le) (rate(api_weather_provider_latency_seconds_bucket[5m])))` and p95 variant | unit `s` |

**Row 3 — "Cache Efficiency"** (row id 102) — *the cost lever; a collapsing hit
rate is the leading indicator of a budget breach, visible before utilization moves*:

| id | type | title | expr | notes |
|---|---|---|---|---|
| 10 | stat | Cache hit rate (24h) | `100 * sum(increase(api_weather_cache_events_total{event="hit"}[24h])) / clamp_min(sum(increase(api_weather_cache_events_total{event=~"hit\|miss\|stale\|corrupt"}[24h])), 1)` | unit percent, thresholds red@null / yellow@40 / green@70 |
| 11 | timeseries | Cache events by type | `sum by (event) (rate(api_weather_cache_events_total[5m]))` | |
| 12 | timeseries | Cache write errors | `sum(rate(api_weather_cache_writes_total{result="error"}[5m]))` | description: metered precisely because nothing else surfaces a dying cache |

- [ ] **Step 2: Validate JSON** — `python3 -m json.tool monitoring/grafana/dashboards/weather.json > /dev/null`.
- [ ] **Step 3: Commit**

```bash
git add monitoring/grafana/dashboards/weather.json
git commit -m "feat(monitoring): weather integration dashboard"
```

### Task I3: Alert rules — `rules-weather.yml`

**Files:**
- Create: `monitoring/grafana/provisioning/alerting/rules-weather.yml`

- [ ] **Step 1: Author the five rules**, copying the file scaffolding
  (`apiVersion: 1`, group/orgId/folder/interval) and the per-rule `data` block
  shape from `rules-whoop.yml` / `rules-api.yml`. Routed to the existing
  `slack-alerts` contact point via the root policy — NO changes to
  `templates.yml`, `contact-points.yml`, or `policies.yml`. No literal `$`
  anywhere. Rules:

| uid | severity | condition (threshold on refId A) | noData/execErr | annotations |
|---|---|---|---|---|
| `weather-budget-warning` | warning | `max(api_weather_budget_utilization_ratio)` > 0.75, `for: 10m` | OK / OK | summary: early notice with room to react; link `__dashboardUid__: ps-weather` + `__panelId__: "1"` |
| `weather-budget-critical` | critical | same expr > 0.90, `for: 5m` | OK / OK | shutoff is imminent; link panel `"1"` |
| `weather-shutoff-engaged` | critical | `max(api_weather_shutoff_active)` > 0.5, `for: 5m` | OK / OK | the tile is now degraded for the rest of the UTC day; link panel `"3"` |
| `weather-integration-dead` | critical | expression below > 21600, `for: 0s` | **Alerting / Alerting** | liveness; link panel `"6"` |
| `weather-provider-errors` | warning | `sum(increase(api_weather_provider_calls_total{result="error"}[1h]))` > 5.5, `for: 0s` | OK / OK | distinguishes a flaky provider from a dead one; link panel `"8"` |

The liveness expression (the WHOOP precedent — `or vector(0)` makes a MISSING
series fire rather than silently pass, and the two `and on()` gates prevent
paging for an integration that is off on purpose or has nothing to fetch):

```promql
(time() - (max(api_weather_last_success_timestamp_seconds) or vector(0)))
and on() (max(api_weather_enabled) > 0)
and on() (max(api_weather_saved_locations) > 0)
```

with the threshold condition `> 21600` (6h). Carry a why-comment block at the top
of the rule, in the style of `rules-whoop.yml`, explaining the fail-loud choices.
`weather-provider-errors` uses 5.5 rather than 5 or 6: per the directory README,
`increase()` extrapolates, so thresholds sit at the midpoint below the target
integer — this fires reliably on the 6th error in an hour. The gauge-based rules
keep `noDataState: OK` per the README (the exporter publishes them continuously;
a scrape gap must not page — the liveness rule is the one that owns
"metrics are missing").

- [ ] **Step 2: Validate**

```bash
python3 monitoring/grafana/provisioning/alerting/validate_rules.py
grep -n '\$' monitoring/grafana/provisioning/alerting/rules-weather.yml && echo "FAIL: literal \$ found" || echo OK
```

Expected: `validate_rules: OK`, and no `$` matches.

- [ ] **Step 3: Commit**

```bash
git add monitoring/grafana/provisioning/alerting/rules-weather.yml
git commit -m "feat(monitoring): weather budget, liveness, and provider-error alerts"
```

### Task I4: Infra local gate

- [ ] **Step 1: Run the repo's checks** (terraform untouched, but run what CI runs
  for the touched surfaces):

```bash
cd /workspace/prog-strength-infra
python3 monitoring/grafana/provisioning/alerting/validate_rules.py
for t in deploy/tests/*.test.sh; do bash "$t"; done
shellcheck -x deploy/*.sh deploy/lib/*.sh
python3 -m json.tool monitoring/grafana/dashboards/weather.json > /dev/null
```

- [ ] **Step 2: Commit any fixes.**

---

# PART 4 — prog-strength-docs (orchestrator, not a subagent)

- [ ] Commit this plan file on `feat/weather-dashboard-tile`.
- [ ] After all implementation PRs are open: flip
  `sows/weather-dashboard-tile.md` frontmatter `status: draft` → `status: shipped`,
  body header `**Status**: Draft` → `**Status**: Shipped`, `**Last updated**` →
  `2026-08-09`. Commit as `docs: mark weather-dashboard-tile as shipped`.
- [ ] Open the docs PR with the operator-facing template (implementation PR links,
  deployment order, verification steps).

---

## Self-review checklist

- [ ] Non-goals respected: no mobile surface, no MCP tool, no training verdict, no
  history/alerts-feed/radar/AQI endpoints, no weather↔activity correlation, **no
  background polling** (the collector reads SQLite only — it never calls the
  provider; nothing fetches weather on a timer).
- [ ] The ledger is SQLite; Prometheus only observes (no enforcement in metrics).
- [ ] Reservation precedes every provider HTTP call, including geocoding when
  `count_geocoding_calls`.
- [ ] Kill switch: all five routes stay mounted when disabled; 200 not 404.
- [ ] Cache is global, metric-canonical, 2dp keys, 90-day sweep on write, write
  errors swallowed-but-metered.
- [ ] `GET /weather` is one location per request under both loading strategies;
  `settings` echoed from `GET /weather/locations`; coordinates round-trip through
  the whole-list PUT.
- [ ] Units derived from `users.distance_unit`; no new preference.
- [ ] Catalog: `weather` appended last in BOTH repos; pageless like `quote`;
  contract tests pinned on both sides.
- [ ] Design system: tokens only, neutrals for icons, `--accent` only as
  focus/selection chrome; no error banners on the tile.
- [ ] Alerts: unique uids, paired panel links, no literal `$`, `validate_rules.py`
  green, liveness fails loud (`or vector(0)`, noData Alerting), others noData OK.
