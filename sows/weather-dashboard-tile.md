---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-infra
  - prog-strength-docs
---

# Weather Dashboard Tile

**Status**: Shipped · **Last updated**: 2026-08-09

## Introduction

The dashboard is a command center for one question — *am I actually getting
stronger?* — and every tile on it today answers that from data Prog Strength
already owns: runs, lifts, steps, macros, recovery. Weather is the first thing
a user checks before an outdoor session that the app cannot tell them, so they
leave the dashboard to find it.

This SOW adds a **Weather** tile: a conventional forecast card — current
conditions, today's high/low, and a short hourly strip — for up to **five saved
locations** the user pages through inside the single tile. Locations are managed
in place, from a popover on the card itself.

It is also the first tile whose data comes from a **metered, paid third party**.
That single fact drives most of this document. A fitness dashboard that quietly
runs up an OpenWeather bill is a worse outcome than a dashboard with no weather
on it, so the cost-control and observability layer here is not an operational
afterthought bolted on at the end — it is a first-class deliverable, specified
before the tile itself.

## Proposed Solution

Four pieces, in dependency order:

1. **A hard, durable daily call budget.** A SQLite-backed ledger keyed by UTC
   date, incremented by a *reserve-before-call* gate. When the day's reservation
   would cross the ceiling, the provider call never happens and the tile serves
   stale cache. This is the auto-shutoff, and it is the load-bearing piece.

2. **A cache-first weather service** (`internal/weather`) modelled directly on
   `internal/nutritionlookup`: a `Provider` vendor-swap seam, a `Service` that
   degrades rather than fails, and a global coordinate-keyed cache with
   per-endpoint TTLs.

3. **A self-fetching tile** on the web dashboard. Deliberately *not* a section of
   `GET /dashboard/summary` — see Algorithms § "Why weather is not a summary
   section".

4. **A Grafana dashboard and five provisioned alerts** answering the two
   questions the owner actually asked: *is the integration working*, and *how
   close am I to the auto-shutoff*.

The provider is **OpenWeather One Call API 4.0** (subscribed, card on file, 1,000
calls/day free allowance), with city search and reverse geocoding from
OpenWeather's Geocoding API under the same key.

## Goals and Non-Goals

### Goals

- A `weather` dashboard tile showing current conditions, today's high/low, and an
  hourly strip for the selected location.
- Up to 5 saved locations per user, paged within the one tile (swipe on touch,
  arrows + keyboard on desktop, dot indicators).
- Location management in a popover on the tile: city search, reorder, delete,
  and a one-time "use my current location".
- A **hard** daily provider-call ceiling that cannot be exceeded, and that
  survives process restarts.
- A config-file kill switch for the whole integration, and a separate switch for
  whether paid overage past the free tier is permitted.
- A Grafana dashboard with sections, headers, per-panel descriptions, and warning
  and critical threshold lines on budget utilization.
- Slack alerts for budget warning, budget critical, auto-shutoff engaged,
  integration dead, and elevated provider errors.

### Non-Goals

- **No mobile surface.** `prog-strength-mobile` has no dashboard at all today;
  adding one is out of scope.
- **No MCP tool or agent capability.** The chat agent does not get weather. It
  was not asked for, and every agent capability is a new MCP tool plus token cost
  on every conversation. **Superseded** — the agent gets weather in
  [`weather-agent-tool.md`](weather-agent-tool.md), which was asked for and
  which adds exactly one tool.
- **No training verdict.** An earlier framing had the tile derive "good run
  window: now–4pm" from wind, precipitation, and daylight. Deliberately rejected
  in favour of a conventional forecast card — the derived-verdict version needs
  threshold logic that is opinionated, hard to test, and easy to get wrong.
- **No weather history, alerts/warnings feed, radar maps, or air quality.** All
  are additional One Call endpoints and therefore additional metered calls.
- **No weather↔activity correlation** ("your pace drops 8% above 80°F"). A real
  product idea, and a separate SOW with its own data model.
- **No background polling.** Nothing fetches weather on a timer. Every provider
  call is lazily triggered by a user actually looking at the tile.

## Implementation Details

### Data Model

Two new tables plus a cache table, all in the app database.

**`user_weather_locations`** — a user's saved places, ordered.

| Column | Type | Description |
| --- | --- | --- |
| `id` | text | Primary key. |
| `user_id` | text | `REFERENCES users(id) ON DELETE CASCADE`. |
| `position` | integer | 0-based display order. |
| `label` | text | Display name, e.g. `Denver`. User-editable. |
| `country` | text | ISO country code from geocoding, e.g. `US`. |
| `state` | text | Region from geocoding where present, e.g. `CO`. Nullable. |
| `lat` | real | Latitude, full precision as returned by geocoding. |
| `lon` | real | Longitude, full precision. |
| `created_at` | timestamp | |

Index: `UNIQUE (user_id, position)` is **not** used — a whole-list replace
rewrites positions in one transaction and a unique constraint would fight the
intermediate states. Instead `INDEX idx_user_weather_locations_user (user_id,
position)`. The 5-location cap is enforced in Go on write, matching how the tile
catalog and layout invariants are handled (migration 049's precedent: the Go
layer is the source of truth, SQL carries no `CHECK`).

**`weather_call_budget`** — the durable spend ledger. One row per UTC day.

| Column | Type | Description |
| --- | --- | --- |
| `usage_date` | text | `YYYY-MM-DD` in **UTC**. Primary key. |
| `calls_used` | integer | Monotonically increasing count of reserved provider calls. |
| `updated_at` | timestamp | |

Not per-user. The budget is an *account-level* constraint against OpenWeather,
and Prog Strength is single-user by design; a per-user ledger would model a
limit that does not exist.

**`weather_cache`** — the global reading cache.

| Column | Type | Description |
| --- | --- | --- |
| `cache_key` | text | Primary key. See key construction below. |
| `payload_json` | text | The normalized reading, not the raw provider body. |
| `fetched_at` | timestamp | Drives freshness against the endpoint's TTL. |
| `last_used_at` | timestamp | Drives opportunistic eviction. |

Global rather than per-user, exactly like `nutrition_lookup_cache`: Denver is
Denver regardless of who asked. Eviction piggybacks on write, sweeping rows
unused for **90 days** — no background job, so table growth is bounded by write
traffic. (90, matching `nutritionlookup`'s `evictionAge`, and deliberately longer
than the 30-day geocoding TTL so a geocode row is never evicted at the exact
moment it goes stale.)

The key has two forms, because the table serves two different lookups:

- **Readings** — `{lat_2dp}:{lon_2dp}:{endpoint}`, where `endpoint` is
  `current` | `hourly` | `daily`. Coordinates are rounded to **2 decimal places
  (~1.1 km)**. Full precision would fragment the cache: two users, or one user
  re-running "use my location", would produce near-identical coordinates that
  miss each other's cached readings and burn budget for no benefit.
- **Geocoding** — `geocode_direct:{normalized_query}` for city search, and
  `geocode_reverse:{lat_2dp}:{lon_2dp}` for coordinate resolution. The query is
  normalized the same way `nutritionlookup` does it (lower-cased, whitespace
  collapsed) so `"Denver  CO"` and `"denver co"` share one row.

A single table rather than two because the freshness/eviction machinery is
identical and the TTL is already a per-endpoint parameter.

### Write Path

- **Add a location** — client searches (`GET /weather/search`), picks a result,
  and `PUT /weather/locations` replaces the whole list. Rejected with 400 if the
  list exceeds `max_locations` or carries out-of-range coordinates.
- **Reorder / delete / rename** — the same whole-list `PUT`. Replace-whole-list
  rather than granular `POST`/`DELETE`/`PATCH` mirrors `PUT /dashboard/layout`:
  ordering is a property of the list, and a partial-update API makes two open
  tabs able to interleave into an order neither user asked for.
- **Reserve a provider call** — `UPDATE weather_call_budget SET calls_used =
  calls_used + 1 WHERE usage_date = ? AND calls_used < ?` inside a transaction,
  with an upsert creating the day's row. Zero rows affected means the ceiling was
  reached; the call is refused.
- **Cache write** — after a successful provider call, upsert the normalized
  payload and sweep evictable rows. A cache write failure is logged and
  swallowed: the cache is an optimization, and failing a user-visible read
  because a write failed would be the wrong trade. It is metered
  (`api_weather_cache_writes_total{result="error"}`) precisely *because* nothing
  else surfaces a dying cache.
- **User deletion** — `ON DELETE CASCADE` drops saved locations, matching
  migration 049. The global cache and the budget ledger are not user-scoped and
  are untouched.

### Algorithms

#### Why the budget ledger is SQLite and not a Prometheus counter

This is the single most important decision in this document, and it comes
straight from this repo's own postmortem in
`monitoring/grafana/provisioning/alerting/rules-whoop.yml`.

A Prometheus counter is **per-process**. It resets to zero on every container
restart, and its labelled children do not exist until first increment. The WHOOP
liveness alert was built on `increase()` over such a counter and fired
continuously over completely healthy ingestion for months, because the API
container restarted about as often as the event it was counting occurred.

Applied here, the failure is worse than a noisy alert: if "calls used today"
lived in a counter, **every deploy would hand the integration a fresh 1,000-call
allowance.** The API deploys on every `feat:`/`fix:` merge. A cost cap with that
property is not a cost cap.

So the ledger is durable SQLite, and Prometheus *observes* it through a gauge
exporter — the same shape as
`api_whoop_last_window_sync_timestamp_seconds`, which exists for exactly this
reason. **Metrics never enforce; they only report.**

#### Reserve-before-call

```
reserve(n):
  BEGIN
    upsert today's row (usage_date = utc_today)
    ceiling = allow_paid_overage ? daily_call_hard_ceiling : daily_call_budget
    UPDATE ... SET calls_used = calls_used + n
      WHERE usage_date = utc_today AND calls_used + n <= ceiling
    if rows_affected == 0: ROLLBACK; return ErrBudgetExhausted
  COMMIT
  return ok
```

Reservation happens **before** the HTTP request, not after. A crash between
reserving and calling therefore *over*-counts by one — the safe direction for a
spend cap. Counting after the response returns would under-count on every
timeout and every crash, which is precisely the population of calls most likely
to occur during the incident where the cap matters.

`n` is a parameter because a full refresh of one location is three calls
(`/current`, `/timeline/1h`, `/timeline/1day`) and reserving them atomically
avoids a half-refreshed location that consumed budget for a card it cannot draw.

#### Why weather is not a `/dashboard/summary` section

Every existing section of the summary handler is a sub-millisecond local SQLite
read. The handler's defensive `defer1` wrapper is built for *recoverable
repository errors*, not for latency.

Folding weather in would put a third-party HTTP call on the critical path of the
request that renders every other tile. A cold cache means three sequential
provider calls before *any* tile paints, and a slow OpenWeather day turns a ~50 ms
dashboard into a multi-second one. `defer1` would degrade weather to nil on
failure — but only *after* the whole request had already waited for the timeout.

The precedent supports the split: `/nutrition/lookup`, the one other external
vendor call in this codebase, is deliberately its own endpoint rather than folded
into a hot aggregate. Weather follows it.

The cost is a second round trip and a tile that paints a beat after its
neighbours. Accepted: a card that arrives late is strictly better than a
dashboard that arrives late.

#### Lazy loading, and the call-budget arithmetic

The tile fetches **only the location currently being viewed**. Other saved
locations are fetched on first swipe and then served from cache for the TTL.

Worst case per location per day, assuming continuous all-day polling:

```
current      24h / 15min  =  96 calls
timeline/1h  24h / 60min  =  24 calls
timeline/1day 24h / 180min =   8 calls
                            ─────────
                             128 calls/location/day
```

| Strategy | Worst case/day | vs. 1,000 free |
| --- | --- | --- |
| Eager, all 5 locations | ~640 | 64% |
| **Lazy, visible only** | **~128** + occasional swipes | **~13%** |

Eager loading is the intuitive design and was rejected on this arithmetic: at 64%
of the free tier in the steady state, a single bug — a render loop, a retry
storm, a TTL regression — crosses into paid territory. Lazy loading leaves a 7×
margin. `eager_load_all_locations` exists as a config knob so the decision is
reversible without a code change, but defaults to `false`.

Note these are ceilings imposed by the TTLs, not expected values. Because nothing
polls in the background, real usage is bounded by how often the dashboard is
actually opened.

#### Temperature units

Derived from the existing `users.distance_unit` preference: `mi` → °F and mph,
`km` → °C and km/h. No new user preference and no new settings toggle. A user who
wants miles with Celsius is not served; that is a deliberate simplification, and
adding `temperature_unit` later is a purely additive migration.

### API Surface

All routes sit behind `auth.RequireUser`.

**`GET /weather?timezone=<IANA>&location_id=<id>`** — readings for **one**
location.

`location_id` is optional; absent, it resolves the user's first saved location.
`timezone` is required and validated, consistent with `/dashboard/summary` and
the project's timezone convention (the client sends an IANA name and a local
date; it never constructs UTC windows itself).

One location per request, never a batch. This keeps the endpoint identical under
both loading strategies: with `eager_load_all_locations = false` the tile calls
it once for the visible location and again on each swipe to a cold city; with
`true` the tile fans out one call per saved location on mount. The server-side
behaviour — and the budget accounting — is the same either way, so the knob
changes only the client's fetch pattern and never the API contract.

```jsonc
{
  "status": "ok",              // ok | stale | disabled | budget_exhausted | unavailable
  "location": { "id": "…", "label": "Denver", "state": "CO", "country": "US" },
  "fetched_at": "2026-08-08T14:02:11Z",
  "units": { "temp": "F", "wind": "mph" },
  "current": {
    "temp": 38, "feels_like": 29, "humidity": 41,
    "wind_speed": 14, "condition": "Clear", "icon": "01d"
  },
  "today": { "high": 46, "low": 24, "sunrise": "…", "sunset": "…" },
  "hourly": [ { "at": "…", "temp": 33, "icon": "01d" } ]   // next 5 buckets
}
```

`status` is explicit rather than inferred from missing fields, so the tile never
has to guess why a payload is thin. `stale` carries a full reading plus an older
`fetched_at`; `budget_exhausted` and `unavailable` may carry a stale reading or
nothing at all.

**`GET /weather/locations`** → the saved list, plus the client-relevant slice of
server config:

```jsonc
{
  "locations": [
    { "id": "…", "label": "Denver", "state": "CO", "country": "US",
      "lat": 39.74, "lon": -104.98 }
  ],
  "settings": {
    "enabled": true,
    "max_locations": 5,
    "eager_load_all_locations": false
  }
}
```

`settings` is echoed because `max_locations` and `eager_load_all_locations` are
server-owned config the client must act on — the popover disables "add" at the
cap, and the tile chooses its fetch pattern from the eager flag. Without this the
web app would have to hardcode `5` and the loading strategy, and flipping either
knob in `config.toml` would silently desync the two surfaces.

Coordinates **are** returned, even though readings are always requested by
`location_id` and the client never plots them. Because writes are a whole-list
replace, reordering or renaming is a read-modify-write: the client sends back
what it read. Omitting `lat`/`lon` from the response would mean a reorder
round-trips locations with no coordinates and silently destroys them.

**`PUT /weather/locations`** → replace the whole list. 400 on >`max_locations`,
invalid coordinates, or a blank label.
**`GET /weather/search?q=<query>&limit=5`** → geocoding results
(`name`, `state`, `country`, `lat`, `lon`). Cached 30 days; counts against the
budget when `count_geocoding_calls = true`.
**`GET /weather/reverse?lat=&lon=`** → the "use my location" resolution, same
caching and budget treatment.

When `enabled = false`, all five routes are **still mounted** and return
`status: "disabled"` with 200 rather than 404. A kill switch that changes the
route table makes the client's failure mode indistinguishable from a bad deploy.

### Configuration

New `[weather]` block in `prog-strength-api/config.toml`. Every value is a public
literal except `api_key`.

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
# false ⇒ the tile fetches only the location being viewed (see Algorithms).
# true ⇒ it fans out one GET /weather per saved location on mount, trading ~5×
# the provider calls for instant swipes. Client fetch pattern only — the API
# contract and the budget accounting are identical either way.
eager_load_all_locations = false
```

`OPENWEATHER_API_KEY` follows the established secret path: GitHub Actions secret
→ `seed-secrets.yml` → Secrets Manager → `deploy/api.sh` renders `.env` → gated
in `REQUIRED_ENV_KEYS`. It must be added to that gate list, or a deploy will
succeed with a silently keyless integration.

### Observability

#### Metrics

Emitted from `internal/weather/metrics.go`, naming and cardinality following
`internal/nutritionlookup/metrics.go`. Every label is a small closed set.

| Metric | Type | Labels | Purpose |
| --- | --- | --- | --- |
| `api_weather_requests_total` | counter | `outcome` | `cache_hit`, `served`, `served_stale`, `budget_exhausted`, `disabled`, `failed` |
| `api_weather_provider_calls_total` | counter | `endpoint`, `result` | `current`/`hourly`/`daily`/`geocode_direct`/`geocode_reverse` × `ok`/`error` |
| `api_weather_provider_latency_seconds` | histogram | `endpoint` | p50/p95 per endpoint |
| `api_weather_cache_events_total` | counter | `event` | `hit`/`miss`/`stale`/`corrupt`/`read_error` |
| `api_weather_cache_writes_total` | counter | `result` | `ok`/`error` |
| `api_weather_calls_used_today` | **gauge** | — | Read from the SQLite ledger. Restart-proof. |
| `api_weather_daily_budget` | **gauge** | — | The active ceiling, so thresholds are drawn from config, not hardcoded in the dashboard. |
| `api_weather_budget_utilization_ratio` | **gauge** | — | `used / budget`, 0–1. The series every budget alert reads. |
| `api_weather_shutoff_active` | **gauge** | — | 1 when the ceiling has been reached today. |
| `api_weather_last_success_timestamp_seconds` | **gauge** | — | Durable. The liveness signal. |
| `api_weather_enabled` | **gauge** | — | 1/0, so alerts can gate on a deliberately-disabled integration. |
| `api_weather_saved_locations` | **gauge** | — | So liveness alerts don't fire for a user with no locations. |

The five gauges derived from durable state are published by a collector that
reads the ledger, mirroring `internal/whoopadmin`'s exporter. Publishing
utilization as its own gauge — rather than dividing two series in PromQL — keeps
the alert expressions trivial and means a divide-by-zero when the budget is
misconfigured to 0 is handled once, in Go, instead of in five alert rules.

#### Grafana dashboard — `monitoring/grafana/dashboards/weather.json`

Three rows, each a collapsible header with a description; every panel carries a
`description` explaining what it shows and what a bad value looks like.

**Row 1 — Budget & Cost Control** *(the "how close am I to auto-shutoff" row)*
- **Budget utilization** — gauge, 0–100%, threshold steps green → **yellow at
  75%** → **red at 90%**.
- **Calls used today vs. budget** — timeseries with horizontal threshold lines
  at the warning and critical levels and at `api_weather_daily_budget`.
- **Auto-shutoff status** — stat panel, green `ACTIVE`/red `SHUT OFF` from
  `api_weather_shutoff_active`.
- **Projected end-of-day calls** — current rate extrapolated to 24h, so the
  ceiling is visible before it is hit rather than after.
- **Calls by endpoint** — where budget is actually going.

**Row 2 — Integration Health** *(the "is it working" row)*
- **Time since last successful call** — stat, red past 6h.
- **Request outcomes** — stacked by `outcome`; a rising `served_stale` or
  `budget_exhausted` band is the visual signature of degradation.
- **Provider errors by endpoint** — rate.
- **Provider latency p50/p95**.

**Row 3 — Cache Efficiency** *(the cost lever)*
- **Cache hit rate** — `hit / (hit + miss + stale + corrupt)`, with a threshold
  line at the level below which the budget is at risk.
- **Cache events by type**, and **cache write errors**.

Cache hit rate sits in its own row rather than buried in a health panel because
it *is* the cost control mechanism — a collapsing hit rate is the leading
indicator of a budget breach, visible well before utilization moves.

#### Alerts — `monitoring/grafana/provisioning/alerting/rules-weather.yml`

Routed to the existing `slack-alerts` contact point. No changes to
`templates.yml`, `contact-points.yml`, `policies.yml`, or compose — the
provisioning directory is already mounted.

| uid | Severity | Condition | Rationale |
| --- | --- | --- | --- |
| `weather-budget-warning` | warning | `api_weather_budget_utilization_ratio > 0.75` for 10m | Early notice with room to react. |
| `weather-budget-critical` | critical | `> 0.90` for 5m | Shutoff is imminent. |
| `weather-shutoff-engaged` | critical | `api_weather_shutoff_active == 1` for 5m | The tile is now degraded for the rest of the UTC day. |
| `weather-integration-dead` | critical | no success in 6h, **and** enabled, **and** ≥1 saved location | Liveness. |
| `weather-provider-errors` | warning | `increase(...{result="error"}[1h]) > 5` | Distinguishes a flaky provider from a dead one. |

`weather-integration-dead` follows the WHOOP precedent deliberately and departs
from this directory's usual convention:

```promql
(time() - (max(api_weather_last_success_timestamp_seconds) or vector(0)) > 21600)
  and on() (max(api_weather_enabled) > 0)
  and on() (max(api_weather_saved_locations) > 0)
```

- `or vector(0)` makes a **missing** series fire rather than silently pass. If
  the exporter is not publishing, we genuinely do not know whether the
  integration is alive, and fail-loud is the right direction for liveness.
- The two `and on()` gates prevent paging for an integration that is off on
  purpose, or that has no locations to fetch.
- `noDataState: Alerting` and `execErrState: Alerting`, because a broken query on
  a liveness monitor must surface loudly.

The other four are threshold/error rules and keep `noDataState: OK`, per the
directory README: counter series often do not exist until first increment, and
absence must not page.

Per the README's gotchas: **no literal `$`** in the YAML (Grafana expands
`$VAR` from the container environment), dot-context in templates, and
counter-rate thresholds set at the midpoint below the target integer because
`increase()` extrapolates. `validate_rules.py` must pass.

### Frontend

Conforms to [`design-system.md`](../design-system.md): existing tokens only —
`--surface`, `--surface-2`, hairline `--border`, `--radius-card` (14px),
`--muted`/`--faint` for metadata, Manrope throughout. **No new hues.** Weather
condition icons render in `--foreground`/`--muted` neutrals rather than
introducing sky-blue or sun-yellow: the palette reserves saturated colour for
activity disciplines and `--accent` for selection and focus, and a yellow sun
would read as a discipline hue that means nothing.

**Catalog registration** — `weather` is added to `TileID`/`Catalog` in
`internal/dashboard/tiles.go` and to `TileId`/`TILE_CATALOG` in
`lib/dashboard-tiles.ts`, in the same position in both. The contract test pins
that they stay identical. No `href` (there is no weather page), so `TileCard`
resolves it before the `href` assertion, alongside `quote`. The exhaustive
`switch` makes omitting the card a compile error.

**`weather-tile.tsx`** — a `MiniCard` with no `href`, self-fetching on mount.

- **Paging**: touch swipe, `‹ ›` buttons, and `ArrowLeft`/`ArrowRight` when the
  card has focus. Dot indicators show position and count. Page position is
  ephemeral client state and always opens on location 1 — unlike the quote
  reroll, there is nothing here worth a server round trip and a table to persist.
- **Gear → locations popover**: debounced city search, drag-reorder (`@dnd-kit`,
  already a dependency), delete, "use my current location", and an `n/5` counter
  that disables adding at the cap.
- **Geolocation**: requested only on an explicit button press inside the popover,
  never on page load. Resolved once via reverse geocoding and saved as a pinned
  coordinate. A denied permission shows an inline note, not a thrown error.

Tile states: `loading` → the existing skeleton · `no locations` → an
inviting `MiniCardEmpty`-style "Add a location" CTA · `ok` · `stale` → the full
reading plus a quiet "updated 2h ago" · `budget_exhausted` / `unavailable` /
`disabled` → a calm muted line. None of these render as an error banner; a
weather card is not worth alarming the user over.

### Backfill or Migration

**Mechanism** — three `CREATE TABLE` statements in one new migration. Nothing to
backfill: no user has saved locations, the budget ledger creates today's row
lazily on first reserve, and the cache starts cold.

**Recoverability** — the migration is additive and creates only new tables, so it
is safe to re-run against a partially-migrated database and safe to roll back by
dropping them. No existing table is altered, and no existing read path changes.

**Scale boundary** — every table is bounded: locations at 5 per user on a
single-user product, the ledger at one row per day (~365/year, never swept
because the history is the audit trail for spend), and the cache by the 30-day
eviction sweep. Nothing here needs a strategy change short of multi-tenancy.

### Testing

- **Budget gate** — table tests over the boundary (`calls_used` at
  `budget - 1`, `budget`, `budget + 1`), `allow_paid_overage` on and off, the
  hard ceiling, UTC date rollover with an injected clock, and **concurrent
  reservations** asserting the ceiling is never exceeded under parallel calls.
  This is the highest-value test in the SOW: it is the only thing standing
  between a bug and a bill.
- **Restart durability** — reserve, discard and rebuild the service against the
  same database, and assert `calls_used` is preserved. This test exists
  specifically to prevent regressing to the WHOOP counter failure.
- **Service degradation** — a fake `Provider` covering: fresh hit, stale serve on
  provider error, stale serve on budget exhaustion, hard failure with no cache,
  and `enabled = false`.
- **TTLs** — injected clock, per-endpoint expiry, and cache-key rounding
  (coordinates 0.004° apart share a key).
- **Handlers** — the location cap, coordinate validation, whole-list replace
  reordering, and the required-`timezone` contract.
- **Location round-trip** — `GET /weather/locations` → reorder → `PUT` → `GET`
  asserts coordinates survive unchanged. Guards the read-modify-write hazard the
  whole-list replace creates.
- **Catalog contract** — the existing Go↔TS test extended to `weather`.
- **Alert rules** — `validate_rules.py` over `rules-weather.yml`, plus a check
  that the file contains no literal `$`.
- **Web** — paging (touch, buttons, keyboard), the 5-location cap, each tile
  state, and that a failed fetch degrades rather than throwing.

## Open Questions

1. **Does OpenWeather's daily quota reset at 00:00 UTC, or on an account-local
   boundary?** The ledger keys by UTC date. If the reset is account-local, the
   ledger's key and the provider's accounting drift by up to a day, and the
   budget could be consumed against the wrong window. **Tentative lean**: UTC,
   which is the industry norm and matches how OpenWeather documents its other
   rate limits. Verify on the account page before implementation; the fix is a
   one-line change to the date key.

2. **Are Geocoding API calls exempt from the One Call quota?** OpenWeather's
   docs do not say, and Geocoding is presented as a separate free product.
   **Tentative lean**: assume they count (`count_geocoding_calls = true`). The
   safe assumption costs a handful of calls a month — geocoding is cached 30 days
   and only fires when adding a location — and the unsafe assumption silently
   uncaps a metered call.

3. **Should the tile show a manual refresh button?** It would let the user force
   a fetch past the TTL. **Tentative lean**: no, for v1. It is a user-facing
   button whose only function is to spend budget, and the TTLs are already short
   enough that a stale reading corrects itself within 15 minutes. Revisit if
   `served_stale` turns out to be common in practice.

4. **Should `weather` appear in the add-tile tray for users with no saved
   locations?** It will render an "Add a location" CTA, which is a reasonable
   empty state but is the only tile whose empty state cannot be resolved by
   logging training data. **Tentative lean**: yes, show it — the CTA is
   self-explanatory and hiding a tile until it is configured makes it
   undiscoverable.
