---
status: draft
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-mcp
  - prog-strength-infra
  - prog-strength-docs
---

# Activity Weather Conditions

**Status**: Draft · **Last updated**: 2026-08-09

## Introduction

A run is not a number, it is an afternoon. Prog Strength already remembers the
distance, the splits, the heart rate, the route, and the photos — but not the
one variable that most often explains why an easy pace felt like a race effort.
A user opening a run from last July sees a 9:40 average and no reason for it.
The 88°F and 71°F dew point that produced it are gone.

The gap exists because weather has never been captured.
[`sows/weather-dashboard-tile.md`](weather-dashboard-tile.md) is **shipped** —
`internal/weather`, the OpenWeather provider, the durable call budget, the
dashboard tile, and a provisioned Grafana dashboard with five alerts are all
live. But that SOW is *forecast*-shaped — it answers "should I go out now?" —
and it lists weather history as an explicit non-goal. Nothing in the stack
records what the conditions actually *were* at the moment an activity happened.

This SOW is therefore almost entirely additive: the vendor, the key, the budget,
the kill switch, the metrics, the dashboard, and the icon set already exist and
are reused as-is.

This SOW closes that. Every outdoor activity that recorded GPS captures the
conditions at its start — temperature, feels-like, dew point, humidity, wind,
precipitation — permanently, at the moment it is imported. A shared
`WeatherRecap` widget renders that on the run and hike detail pages. And a
one-time backfill walks the existing history so the user can open a run from any
month and see the afternoon they actually ran in, not just the pace it produced.

## Proposed Solution

Five pieces, in dependency order:

1. **A durable per-activity record.** A new `activity_weather` table, 1:1 on
   `activities`. Historical weather is *immutable*: once captured it is correct
   forever, so it is stored, not cached, and never refreshed.

2. **One new provider method.** `Historical(ctx, lat, lon, at)` on the existing
   `weather.Provider` seam, plus one new `Endpoint` constant. Everything else —
   the budget ledger, the metrics, the kill switch — is reused unchanged.

3. **Capture at import.** After an activity commits, a detached, bounded
   goroutine fetches and stores the reading. Failure writes nothing, which makes
   the backfill command the single repair path for both history and
   import-time misses.

4. **A `cmd/weather-backfill` command.** Dry-runnable, paced, resumable, and
   reserving through the same ledger so it cannot exceed the daily cap. Manual by
   design — see Algorithms § "Why the backfill is a command and not a worker".

5. **A shared `WeatherRecap` widget** on `/running/[id]` and `/hiking/[id]`, plus
   a general activity-detail MCP tool so the chat agent can answer "what was it
   like on Tuesday's run?" — and every other question about a single activity it
   currently cannot answer at all.

The provider stays **OpenWeather One Call 4.0** under the existing key, budget,
and kill switch. This feature adds no new vendor and no new secret.

## Goals and Non-Goals

### Goals

- Capture and durably store the weather conditions at the start of every outdoor
  activity that has GPS coordinates, at import time, going forward.
- Backfill the existing outdoor activity history via a paced, resumable,
  dry-runnable command that cannot exceed the existing daily call budget.
- A shared `WeatherRecap` component rendering a conditions headline plus a metric
  strip (feels-like, humidity, dew point, wind, precipitation), wired to both
  `/running/[id]` and `/hiking/[id]`.
- A durable record of *attempted* captures, so neither the import path nor the
  backfill ever re-spends budget on an activity that has no answer.
- A general `get_activity` MCP tool exposing the full activity detail read —
  including the stored weather — to the chat agent.
- Metrics for activity-capture volume and remaining backfill work, emitted
  through the existing weather instrumentation and surfaced as a new row on the
  shipped `weather.json` Grafana dashboard.
- CI green in all three code repos.

### Non-Goals

- **No mobile surface.** `prog-strength-mobile` has a run detail screen and will
  want this, but it is a separate read-only wiring task and is deliberately
  deferred to a follow-up rather than widening this SOW's blast radius.
- **No refresh, ever.** A stored reading is never re-fetched. The weather at
  14:07 on 12 July 2025 does not change, and a refresh path would be a
  budget-spending code path with no possible benefit.
- **No weather↔performance correlation.** "Your pace drops 8% above 80°F" is a
  real product idea and remains a separate SOW. This work stores the data such a
  feature would need, and stops there.
- **No conditions over the course of the activity.** A single observation at the
  start hour, not a start→end delta. See Algorithms § "One observation, at the
  start".
- **No weather on list, feed, or timeline surfaces.** Detail pages only. Adding
  it to a list projection is additive once the data exists.
- **No weather for indoor activities.** A treadmill run's outdoor temperature is
  a fact about a parking lot, not about the session.
- **No backfill of activities without GPS.** Position cannot be re-derived from
  distance. These are recorded as `no_coordinates` and never retried.
- **No new provider, secret, or budget.** Everything runs under the existing
  `[weather]` config block and the existing daily ledger.

## Implementation Details

### Data Model

One new table in migration **`057_activity_weather.sql`**.

**`activity_weather`** — one row per *attempted* activity.

| Column | Type | Description |
| --- | --- | --- |
| `activity_id` | text | Primary key. `REFERENCES activities(id) ON DELETE CASCADE`. |
| `status` | text | `ok` \| `no_coordinates` \| `unavailable`. |
| `lat` | real | The coordinate actually used. Nullable — absent for `no_coordinates`. |
| `lon` | real | As above. |
| `observed_at` | timestamp | The provider's observation hour, UTC. Nullable when not `ok`. |
| `temp_c` | real | Nullable when not `ok`; likewise every reading column below. |
| `feels_like_c` | real | |
| `dew_point_c` | real | |
| `humidity` | integer | Percent. |
| `wind_kmh` | real | Canonical km/h, converted from the provider's m/s in the provider layer. |
| `wind_deg` | integer | Meteorological degrees (direction the wind comes *from*). |
| `precip_mm` | real | Total precipitation for the observation hour. |
| `condition` | text | Provider condition tag, e.g. `Clear`. |
| `icon` | text | Provider icon code, e.g. `01d`. Passed through untranslated. |
| `fetched_at` | timestamp | When we asked, as distinct from `observed_at`. |

Index: none beyond the primary key. Every read is a point lookup by
`activity_id`, and the backfill's "what is left" query is an anti-join against
`activities` that a `activity_weather(activity_id)` PK already serves.

**Why a table and not columns on `activities`.** Migrations 036 and 038 both
added columns in place, so the precedent points the other way — but they added
two and three columns respectively, and this is fourteen. Fourteen columns that
are meaningful only for outdoor GPS activities would sit `NULL` on every lift,
every treadmill run, and every future non-geographic type, on a table that is
already wide and read on every list page. A 1:1 table also gives `status` a
natural home: a "we tried and there is no answer" marker inside a blob column
(the `route_geojson` alternative) is exactly the kind of thing that gets read
wrong once and then re-spends budget forever.

**Values stored — and served — metric.** `Provider` implementations already
contract to return metric, and storing display units would bake a mutable user
preference into immutable historical data.

The activity DTO carries *both* conventions today and weather has to pick one.
Raw measurements (`distance_meters`, `elevation_gain_meters`) travel metric and
are formatted by the client via `useDistanceUnit`; derived blocks (`splits`,
`strip_summary`, `best_pace_sec_per_unit`, `intervals`) are computed server-side
from the `?unit=` query param and the DTO echoes `unit` back. **Weather follows
the raw-measurement convention**, because that is what it is: a stored
observation, not a read-time derivation.

This deliberately diverges from the shipped `GET /weather`, which converts
server-side and returns a `units: {temp, wind}` block. That was the right call
there — a standalone tile endpoint with no `unit` param and no other fields to
be consistent with. Here the surrounding DTO already sets the convention, and
converting server-side would mean transforming stored metric on every read to
produce a field sitting next to `distance_meters`, which is not transformed.
The cost of the divergence is that the two surfaces format temperature in
different layers; it is called out here so a reviewer reads it as a decision
rather than drift.

### Coordinate and Observation Time

**Coordinate** — the first positioned trackpoint, i.e. where the activity
started. Full stored precision (migration 038 already truncates to 6 decimals on
write); no rounding is applied for the provider call.

A 10 km loop sits comfortably inside a single weather grid cell, so the start
point is not a lossy approximation of the run — it *is* the run's weather.
Point-to-point activities are the exception and are recorded as Open Question 3
rather than guessed at.

**Observation time** — `activities.start_time`, rounded to the nearest hour,
which is the resolution historical data is published at. `observed_at` records
the hour the provider actually answered for, so a reading is never silently
attributed to a timestamp it does not describe.

### Write Path

- **On import** — after the activity row commits and only when
  `environment = 'outdoor'` and a positioned trackpoint exists, a detached
  goroutine reserves 1 call, fetches, and inserts a `status = 'ok'` row. See
  Algorithms § "Detached, bounded, and non-fatal".
- **Transient failure** (timeout, 5xx, `ErrBudgetExhausted`, weather disabled) —
  **no row is written.** The activity is simply still eligible, and the backfill
  command picks it up on its next run.
- **Definitive no-data** — a `status = 'unavailable'` row, written *only* by the
  backfill command and only when the provider gives a definitive negative (the
  timestamp predates its history window). This is the one terminal failure state,
  and `--retry-unavailable` exists to clear it if the provider's coverage
  changes.
- **No coordinates** — a `status = 'no_coordinates'` row, written by the backfill
  command with **no provider call**. Terminal: position cannot be re-derived.
- **Indoor activities** — no row at all, and no call. An activity later retagged
  outdoor therefore becomes eligible automatically and is captured on the next
  backfill run. This is a second reason the command is re-runnable rather than
  one-shot.
- **Environment retag `outdoor → indoor`** — the row is **retained**, the widget
  hides. Retagging back re-shows the stored reading with no refetch. Deleting on
  retag would turn a reversible UI toggle into a budget-spending operation.
- **Activity deletion** — `ON DELETE CASCADE`.

### Algorithms

#### One observation, at the start

The alternative — a start→end delta ("52° → 58°, wind picked up") — was
considered and rejected. It costs a second metered call on every activity,
doubling both the steady-state spend and the backfill's total, and it produces
*no signal at all* on sub-hour activities, which are the majority. The runs where
it would say something interesting are the long ones, which are the minority, and
even there "it warmed up over three hours" is closer to trivia than to insight.
The single start observation answers the actual question — what did I step out
into — at half the cost.

#### Why historical readings do not use `weather_cache`

The dashboard tile routes every reading through the global, TTL'd
`weather_cache`, and this feature deliberately does not.

The cache exists to stop repeated lookups of the *same coordinate at the present
moment* from each burning a call. Historical readings have no such repetition:
the cache key would be `{lat}:{lon}:{hour}`, and two activities colliding on it
requires two separate GPS activities starting in the same ~1 km cell within the
same hour. That does not happen, so the hit rate would be approximately zero
while the code carried TTL and eviction machinery that is actively wrong here —
a historical reading has no TTL, and a 90-day eviction sweep would delete data
that is still perfectly correct.

`activity_weather` is the durable store. It has better keys (an activity ID, not
a rounded coordinate), better semantics (permanent, not expiring), and it is
exactly as many rows.

#### Detached, bounded, and non-fatal

The import-time fetch runs in a goroutine with **`context.WithTimeout` over
`context.Background()`, not over the request context.** The request context is
cancelled the instant the upload response is written, so a fetch inheriting it
would be cancelled essentially always — the failure mode would be a feature that
silently never works, which is precisely the shape of the WHOOP webhook outage
recorded in [`sows/whoop-integration-diagnostics.md`](whoop-integration-diagnostics.md).
This deserves an explicit test.

The upload path must not wait on OpenWeather. TCX import is already the slowest
user-facing write in the product (parse, summarize, S3 archive), and a
third-party HTTP call on that critical path would make a slow OpenWeather day
look like a broken uploader. A weather failure is never surfaced to the uploading
user and never fails the import.

The corollary is the useful part: because a failed capture leaves *no row*, the
activity remains indistinguishable from one that was never attempted — so the
backfill command is the single repair path for both a five-year history and
yesterday's timeout. There is one recovery mechanism to build, test, and trust,
not two.

#### Why the backfill is a command and not a worker

An automatic paced background drain was considered and rejected in favour of
`cmd/weather-backfill`.

The tradeoff was explicit: a worker needs no manual step, which is generally the
preferred shape in this project. But this is the one operation here that spends
money in bulk, against a metered vendor, over a history whose size is not known
until it is counted. A command puts a human between the count and the spend:
`--dry-run` prints exactly how many calls the run will make *before* any of them
happen, and the operator decides. A worker would make that decision at boot, and
the first time anyone learned the real number would be from the ledger.

It is also re-runnable rather than one-shot, which turns out to matter for
reasons unrelated to cost — retagged activities and cleared `unavailable` rows
both make new work appear long after the "one-time" migration is done.

The accepted cost is a manual ops step that has to be remembered and possibly
re-run across days when the history is larger than one day's budget. The command
mitigates it by exiting cleanly on budget exhaustion with an explicit "resume
tomorrow" message and a count of what remains.

#### Budget interaction

Every call — import-time and backfill alike — goes through the existing
`BudgetLedger.Reserve(ctx, 1, activeCeiling(cfg))`. There is no separate
allowance and no new ceiling: the ledger is an account-level constraint against
OpenWeather, and inventing a second budget would model a limit that does not
exist.

Steady-state cost is **one call per outdoor activity uploaded** — a handful per
week against a 800/day budget, which is noise. The backfill is the only material
consumer, it is bounded by the same ceiling, and its `--rate` flag exists so it
degrades the dashboard tile gradually rather than draining the day's allowance in
the first ninety seconds.

### Provider

One method on the existing seam in `internal/weather/provider.go`:

```go
Historical(ctx context.Context, lat, lon float64, at time.Time) (Observation, error)
```

and one endpoint constant in `models.go`, which the existing metrics labels pick
up with no further change:

```go
EndpointHistorical Endpoint = "historical"
```

`Observation` is a new normalized struct — `TempC`, `FeelsLikeC`, `DewPointC`,
`Humidity`, `WindKMH`, `WindDeg`, `PrecipMM`, `Condition`, `Icon`, `ObservedAt`.
It is deliberately *not* `Current`: `Current` is the tile's shape, carries no dew
point, direction, or precipitation, and has no observation timestamp because for
a forecast "now" is implicit.

**The response shape must be captured from a live call, not assumed.** The
comment block at the top of `openweather.go` records why: three independently
inlined copies of the path prefix all omitted `/onecall`, and every reading
404'd in production. A second trap is documented there too — One Call 4.0 wraps
readings in a top-level `data` array *including* `/current`, and parsing the root
decodes into a zero-value struct **without error**, yielding a plausible 0°C
reading rather than a failure. A historical endpoint returning an empty or
differently-wrapped array must therefore fail loudly, exactly as `Daily` does
today. Fixtures go in `openweather_test.go` captured from live responses.

### API Surface

**No new endpoint.** The reading rides the existing activity detail DTO in
`internal/activity/handler.go`, alongside `Route`:

```jsonc
"weather": {
  "status": "ok",
  "observed_at": "2025-07-12T18:00:00Z",
  "temp_c": 31.1,
  "feels_like_c": 36.4,
  "dew_point_c": 21.7,
  "humidity": 58,
  "wind_kmh": 14.5,
  "wind_deg": 210,
  "precip_mm": 0,
  "condition": "Clear",
  "icon": "01d"
}
```

`omitempty`, so the key is absent when the activity has no row — which is the
same "omit the beat entirely" contract `route` already has and the design
system's editorial rhythm requires.

Non-`ok` rows serialize with `status` and the null reading fields dropped. The
client renders nothing for them; `status` exists so a future surface (or the
backfill's own reporting) can tell "never attempted" from "attempted, no answer"
without a second query.

### MCP Surface

One new tool in `src/prog_strength_mcp/activities.py`, beside `list_activities`:

```
get_activity(activity_id) -> dict
```

backed by the **existing** `APIClient.get_activity`, which is already
implemented and currently called by no tool at all. The agent can today *list*
activities but cannot read one, so this closes a gap that is wider than weather:
splits, intervals, heart-rate zones, elevation, plan linkage, and now conditions
are all unreachable in chat.

**The payload is trimmed before it reaches the model.** The detail DTO carries
`trackpoints` — hundreds of per-second samples — plus `route`, a serialized
GeoJSON geometry. Both are rendering data: they are meaningless to a language
model, they would dominate the response, and on a long run they would crowd out
the conversation. The tool drops both keys and returns everything else. The
summary series the agent *can* reason about (`splits`, `strip_summary`,
`heart_rate_zones`, `intervals`) are already aggregates and are kept.

Weather is returned with **both unit systems** — `temp_c`/`temp_f`,
`wind_kmh`/`wind_mph`. The conversion is two multiplications in Python and it
removes an entire class of quiet arithmetic error from the model's answer. The
rest of the DTO passes through in its existing units, unchanged, consistent with
every other tool.

The token cost is honest and worth naming: one more tool definition on every
conversation, and a response an order of magnitude larger than
`list_activities` when it is invoked. The trim is what keeps the second number
bounded — without it this tool would be unusable on exactly the long runs it is
most interesting for.

### Configuration

One addition to the existing `[weather]` block in
`prog-strength-api/config.toml`:

```toml
# Capture historical conditions on activity import. false ⇒ no import-time
# provider calls and no new activity_weather rows; existing rows still render.
# The master `enabled` switch above gates this too — false there disables
# capture regardless of this value.
capture_activity_weather = true
```

One field on `config.WeatherConfig`. No new secret, no new env var, and
therefore no change to `REQUIRED_ENV_KEYS` or the Secrets Manager path.

The knob is separate from `enabled` because the two failure modes are different:
`enabled = false` is "the vendor integration is off", while
`capture_activity_weather = false` is "stop capturing but keep the tile working",
which is what an operator wants during a backfill that needs the whole day's
budget.

### Observability

`monitoring/grafana/dashboards/weather.json` and
`provisioning/alerting/rules-weather.yml` are both **live** in
`prog-strength-infra` (three rows — Budget & Cost Control, Integration Health,
Cache Efficiency — and five provisioned alerts). This SOW extends them rather
than creating anything new.

Three additions to the existing weather instrumentation, all following
`internal/weather/metrics.go`:

| Metric | Type | Labels | Purpose |
| --- | --- | --- | --- |
| `api_weather_activity_captures_total` | counter | `result` | `ok` \| `no_coordinates` \| `unavailable` \| `failed` |
| `api_weather_activities_with_weather` | gauge | — | Rows in `activity_weather` with `status = 'ok'`. |
| `api_weather_activities_pending_capture` | gauge | — | Eligible outdoor GPS activities with **no** row. Backfill progress, and the number `--dry-run` reports. |

Both gauges are published by the existing `Exporter`, which already reads
durable SQLite on a 5-minute tick and — as its doc comment states, load-bearing —
never calls the provider. They are gauges over stored state rather than counters
for the reason the tile SOW records at length: a counter resets on every deploy,
and "how much history is left to backfill" must survive a restart to mean
anything.

`EndpointHistorical` flows into the existing
`api_weather_provider_calls_total{endpoint,result}` and
`api_weather_provider_latency_seconds` with no new metric, so historical spend
appears in the shipped **"Calls by endpoint (1h increase)"** panel on day one
with no dashboard change at all.

**A fourth row on `weather.json` — "Activity Capture"**, collapsible with a
description like the other three:

- **Activities with weather** and **Pending capture** — the two new gauges as
  stat panels. Pending trending to zero *is* backfill progress.
- **Captures by result (24h)** — `api_weather_activity_captures_total` stacked by
  `result`. A rising `failed` band during a backfill is the signal to stop and
  look.

**One caveat worth writing down, because it will otherwise look like a bug.**
`api_weather_last_success_timestamp_seconds` is published from the cache
repository's `LastSuccess` — the newest `fetched_at` across `weather_cache`.
Historical readings deliberately do not touch that table (see Algorithms § "Why
historical readings do not use `weather_cache`"), so **activity captures do not
advance the liveness signal.** A backfill can run all day while "Time since last
successful call" climbs. That is correct — the gauge measures *tile* liveness,
and the `weather-integration-dead` alert is already gated on
`api_weather_saved_locations > 0` for the same reason — but it is surprising
enough to belong in the new row's description.

**No new alerts.** The existing budget-utilization and shutoff alerts cover the
only genuinely bad outcome (spend), and a stalled backfill is a thing an operator
started and can look at, not a thing that should page. `--dry-run` is the
operator's real instrument here, and it runs before the spend rather than
alerting after it.

### Frontend

Conforms to [`design-system.md`](../design-system.md) § *Activity session-recap*.
Existing tokens only. **No new hues** — condition icons render in
`--foreground`/`--muted` neutrals, for the reason the tile SOW gives: the palette
reserves saturated colour for activity disciplines and `--accent` for selection
and focus, and a sun rendered in yellow would read as a discipline hue that means
nothing.

**Reuse the shipped icon set, and promote it.** `WeatherIcon` already exists at
`app/(app)/dashboard/_components/weather-icons.tsx`: hand-rolled `currentColor`
stroke SVGs keyed by OpenWeather icon code, already neutral-tinted for exactly
the reason above, already collapsing day/night variants onto one glyph. Drawing a
second set would be duplicated work that then drifts.

It sits in a **route-private** `_components` directory, though, and a second
consumer on a different route cannot import it there without reaching across
routes. So this SOW **moves it to `components/weather/icons.tsx`** and updates
the tile's import. That is the whole of the refactor — no behaviour change, no
API change, one file moved and one import rewritten — and it is the difference
between "common widget" being true and being aspirational.

**`components/activity-detail/WeatherRecap.tsx`** — a new shared component
beside `MapView`, `HeartRateRecap`, and `ElevationRecap`.

- **Section kicker** — `<SectionKicker>Conditions</SectionKicker>`, in the
  discipline hue like every other beat.
- **Headline** — `<WeatherIcon>`, temperature at a large numeric scale, and the
  condition word. `31°C · Clear`.
- **Metric strip** — the same `border-y` definition-list idiom the detail page
  already uses for distance/time/pace: feels-like, dew point, humidity, wind
  (speed + a compass direction derived from `wind_deg`), precipitation.
  Precipitation is omitted when zero rather than rendered as `0 mm`, matching
  the "beats with no data are omitted whole" rule.
- **Units** — formatted client-side from `useDistanceUnit()`, the same hook the
  detail page already uses for distance, pace, and elevation: `mi` → °F and mph,
  `km` → °C and km/h. New `formatTemperature` / `formatWindSpeed` helpers land in
  `lib/distance-unit-context.tsx` beside `formatElevationValue`. No new user
  preference and no settings toggle, exactly as the tile SOW decided; a
  standalone `temperature_unit` later remains purely additive.
- **Absent by omission** — the component renders `null` when `weather` is absent
  or `status !== "ok"`, and when `environment !== "outdoor"`. No skeleton, no
  empty frame, no "weather unavailable" line. A run from 2019 with no GPS simply
  has no conditions beat, the same way it has no map.

**Placement** — immediately after `MapView` and before *The Miles* on
`/running/[id]`, and in the equivalent slot on `/hiking/[id]`. Both are answers
to *where and what was it like*, and they belong in the same beat of the page,
before the work begins.

**Wiring** — `app/(app)/running/[id]/page.tsx` and
`app/(app)/hiking/[id]/page.tsx`. Both already import from
`@/components/activity-detail/`, so each is a one-import, one-element change.
Neither refetches: the reading arrives on the detail response both pages already
load, so there is no second round trip and no new loading state.

### Backfill or Migration

**Mechanism** — `cmd/weather-backfill`, on the `cmd/memory-backfill` precedent
(a one-time, manual-run tool with its own `main.go` and injectable dependencies
for testing).

```
weather-backfill --dry-run            # count and cost, zero provider calls
weather-backfill --limit 200          # cap this run's provider calls
weather-backfill --rate 2             # calls per second, default 1
weather-backfill --retry-unavailable  # clear terminal 'unavailable' rows and retry
```

Selection is an anti-join: live activities with `environment = 'outdoor'` and no
`activity_weather` row. For each, in `start_time` order (newest first — recent
runs are the ones the user is most likely to open):

1. No positioned trackpoint → write `no_coordinates`, **no call**, continue.
2. Otherwise reserve 1 call through the ledger. `ErrBudgetExhausted` → stop
   cleanly, print what remains and "resume tomorrow", exit 0.
3. Fetch. Definitive no-data → write `unavailable`. Transient error → log with
   the activity ID, write **nothing**, continue.
4. Success → write the `ok` row.

`--dry-run` performs steps 1 and the selection query only, and prints the split:
*N activities eligible (N provider calls), M with no GPS (free), K already
captured*. It is the number the operator approves before spending.

**Recoverability** — resumable by construction, because the presence of a row
*is* the checkpoint; there is no separate progress state to corrupt. A crash,
a `SIGINT`, or an exhausted budget all leave a consistent database, and re-running
resumes exactly where it stopped. It never re-spends on an activity it has
already resolved.

**Scale boundary** — the design assumes a history in the low thousands. Above
roughly 800 eligible activities the backfill spans more than one UTC day and
becomes a multi-day operation; above ~10,000 it would want batched writes and a
resume cursor rather than a per-activity anti-join. Neither is close for a
single-user product, and both are additive changes to a command that nothing else
depends on.

**Migration safety** — `057` creates one new table and alters nothing. Safe to
re-run against a partially-migrated database, safe to roll back by dropping the
table, and no existing read path changes.

### Testing

- **Detached-context capture** — the highest-value test in this SOW. Assert the
  import-time fetch **completes after the request context is cancelled**. Wiring
  it to the request context is the one mistake here that produces a feature which
  silently never works, and is the exact shape of the WHOOP webhook outage.
- **Import path is non-fatal** — a provider that errors, times out, and returns
  `ErrBudgetExhausted`: the upload succeeds in all three, and **no row is
  written** in all three (so the backfill remains the repair path).
- **Provider** — `Historical` against captured fixtures; an empty `data` array
  fails loudly rather than decoding a 0°C reading; wind converts m/s → km/h.
- **Status terminality** — `no_coordinates` and `unavailable` rows are never
  re-attempted; `--retry-unavailable` clears the latter and not the former.
- **Backfill** — `--dry-run` makes zero provider calls; `--limit` is respected;
  budget exhaustion exits cleanly mid-run leaving a consistent database;
  re-running resumes without re-spending; indoor activities are skipped and
  become eligible after a retag.
- **Environment retag** — `outdoor → indoor` retains the row and hides the
  widget; retagging back re-renders it with **no provider call**.
- **Cascade** — deleting an activity deletes its weather row.
- **DTO** — `weather` is absent (key omitted) with no row; non-`ok` rows
  serialize with null reading fields dropped.
- **MCP** — `get_activity` strips `trackpoints` and `route` from the response
  (asserted against a fixture that contains both), returns weather in both unit
  systems, degrades to a clear "no weather recorded" answer rather than an error
  when the key is absent, and surfaces a not-found activity as a clean message.
- **Web** — `WeatherRecap` renders `null` for every non-`ok` status, for a
  missing key, and for indoor activities; unit formatting flips with
  `useDistanceUnit`; zero precipitation is omitted; both detail routes render it
  in the right slot. The existing `weather-tile` and `weather-icons` tests must
  stay green across the icon move — they are the regression guard on the
  refactor.
- **Grafana** — `validate_rules.py` still passes and `weather.json` still parses
  after the new row, per the monitoring directory's README.

## Rollout

1. **`prog-strength-api`** — migration `057`, `Observation` + `Historical`,
   `activity_weather` repository, the detached import-time capture, the DTO
   field, metrics, and `cmd/weather-backfill`. All additive; the DTO key is
   simply absent until rows exist.
2. **Run `weather-backfill --dry-run`** against production and read the count
   before spending anything. Then run it for real, across as many days as the
   budget requires.
3. **`prog-strength-web`** — promote `weather-icons.tsx` to `components/weather/`
   and rewrite the tile's import (a standalone, zero-behaviour-change PR that can
   land first), then `WeatherRecap`, the unit formatters, and the two route
   wirings. Ships after the backfill so the feature does not debut mostly empty.
4. **`prog-strength-mcp`** — `get_activity`.
5. **`prog-strength-infra`** — the "Activity Capture" row on `weather.json`.
   Last, because the gauges must exist before panels can render them; `--dry-run`
   covers the backfill's own visibility in the meantime.
6. **`prog-strength-docs`** — mark shipped on merge.

## Open Questions

1. **What is the One Call 4.0 historical endpoint's path and response shape?**
   The 4.0 surfaces in `openweather.go` are `/onecall/current`,
   `/onecall/timeline/1h`, and `/onecall/timeline/1day`; the historical surface
   is presumably a sibling taking a timestamp, but that is an inference.
   **Tentative lean**: none — this must be captured from a live call before
   implementation, not assumed. The `/onecall` prefix omission that 404'd every
   reading in production is the precedent for why.

2. **How far back does the provider's history extend, and are historical calls
   billed at the same rate as forecast calls?** This bounds what the backfill can
   recover and how much it costs. **Tentative lean**: assume the same rate (the
   conservative direction for a spend cap) and treat pre-window activities as
   terminal `unavailable`. Verify on the account page; if history is billed at a
   premium, `--limit` and `--rate` already provide the control needed and only
   the dry-run's cost estimate needs adjusting.

3. **Start point or route centroid for point-to-point activities?** For loops
   they are the same cell and the question is moot. For a long point-to-point run
   the finish can be a genuinely different microclimate. **Tentative lean**:
   start point. It is what the user stepped out into, it needs no geometry math,
   and `activity_weather.lat`/`lon` record which coordinate was used, so changing
   the rule later is a re-backfill and not a schema change.

4. **Should `precip_mm` be hidden or shown as `0` when dry?** The design system
   says omit empty beats, which argues for hiding. **Tentative lean**: hide.
   "0 mm" is a cell that says nothing, and the absence of a precipitation entry
   reads as "it was dry" without spending a slot on it.
