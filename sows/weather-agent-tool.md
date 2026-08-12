---
status: draft
repos:
  - prog-strength-api
  - prog-strength-mcp
  - prog-strength-agent
  - prog-strength-infra
  - prog-strength-docs
---

# Weather in the Agent

**Status**: Draft · **Last updated**: 2026-08-12

## Introduction

Outdoor training is weather-bound in a way that lifting is not. In Denver a
long run at 2pm in July is a different physiological event than the same run at
6am — hotter, drier, more dehydrating, and worse for both performance and
safety. Deciding *when* in the week to run is therefore a real training
decision, and it is one Prog Strength cannot currently help with.

The gap is not missing data. `prog-strength-api` already owns a full weather
integration — an OpenWeather provider, a durable daily call budget, a
coordinate-keyed cache, and saved locations — shipped for the dashboard tile in
[`weather-dashboard-tile.md`](weather-dashboard-tile.md). That data reaches
exactly one surface: a card on the web dashboard. The chat agent, which is the
surface a user actually asks planning questions in, cannot see any of it.

So today a user asking *"when should I schedule my long run this week?"* gets an
answer grounded in their training history and nothing else. After this work the
agent can read the same forecast the tile reads and answer with the week's
conditions in hand: which day is mildest, how hot the afternoon gets, when the
sun comes up, whether Saturday's 40% chance of storms argues for Sunday.

## Proposed Solution

One new MCP tool, `get_weather_forecast`, forwarding to the shipped
`GET /weather` endpoint. The tool returns the forecast as data; the judgment
about what makes a good running window lives in the agent's prompt, not in Go.

Three supporting changes make that one tool useful:

1. **Ad-hoc place resolution on `GET /weather`.** The endpoint resolves a
   location by `location_id` or falls back to the user's first saved location.
   A new `place` parameter accepts free text, so the agent can answer "what
   about Boulder on Saturday?" without the user having saved Boulder.

2. **Training-aware prompt rules** on the `plan_workout` intent, teaching the
   model how heat, humidity, wind, precipitation, and daylight bounds bear on an
   outdoor session — and what the forecast can and cannot tell it.

3. **A `source` label** on the existing request metric, so the shared provider
   budget can be attributed between the tile and the agent in Grafana.

There is no new table, no new migration, no new configuration, and no second
provider integration. The paid-integration machinery this feature depends on —
reserve-before-call, the hard daily ceiling, the cache — is reused untouched,
which is the entire reason this SOW is small.

### Reversing a prior non-goal

`weather-dashboard-tile.md` lists **"No MCP tool or agent capability"** among
its non-goals, on the grounds that it "was not asked for, and every agent
capability is a new MCP tool plus token cost on every conversation."

Both halves of that reasoning still hold and neither blocks this work. It is
being asked for now, by the use case in the Introduction. And the token cost is
real but bounded: this SOW adds exactly **one** tool definition, not a family of
them, which is why the design below repeatedly collapses would-be second tools
into parameters. That non-goal is amended in place alongside this document to
point here, so a future reader of the shipped SOW does not conclude the
decision still stands.

## Goals and Non-Goals

### Goals

- A single MCP tool, `get_weather_forecast`, returning current conditions, the
  hourly strip, and the multi-day forecast for the user's location.
- Free-text place resolution, so the agent can answer about somewhere the user
  has not saved.
- Every expected degradation — no saved locations, unknown place, feature
  disabled, budget exhausted, stale cache — reaches the model as structured
  data it can speak to, never as a tool error.
- The user's timezone is injected automatically; the model never supplies one.
- Prompt rules that let the model reason about outdoor training conditions and
  state its answer's confidence honestly given the forecast horizon.
- Provider spend attributable to tile vs. agent in Grafana.

### Non-Goals

- **No extended hourly forecast.** The provider's hourly timeline is kept at the
  20 buckets one call returns. Paging further is billed per page — see
  Algorithms § "The horizon asymmetry".
- **No server-side suitability score.** No Go code decides what "too hot" means.
  `weather-dashboard-tile.md` rejected a derived training verdict as opinionated
  and hard to test; that judgment is correct for the API and does not apply to a
  language model reasoning in context.
- **No prefetch.** Weather is not added to any intent's prefetch set. See
  Algorithms § "Why `plan_workout` does not prefetch weather".
- **No weather↔performance correlation.** "Your pace drops 8% above 80°F"
  remains a separate SOW with its own data model, as it was before.
- **No device geolocation.** Neither chat client starts sending coordinates;
  "current location" means the user's saved locations.
- **No web or mobile change.** The `source` parameter defaults such that the
  existing tile needs no edit, and neither client gains UI.
- **No proactive weather.** Nothing pushes "it will be 95°F Saturday, move your
  run." The agent answers when asked.
- **No new user preference.** Units continue to derive from `users.distance_unit`.
- **No weather eval case.** The agent's eval harness scores numeric macro
  accuracy against a tolerance; asserting tool-call behavior needs a mode it
  does not have. See Open Questions.

## Implementation Details

### Data Model

No changes. `user_weather_locations`, `weather_call_budget`, and `weather_cache`
are used exactly as shipped.

This is worth stating rather than omitting: the feature reads a metered third
party, and the only reason it needs no ledger of its own is that the shipped
ledger is account-level rather than per-surface. The agent's calls land in the
same daily bucket as the tile's and are refused by the same hard stop.

### API Surface

**`GET /weather?timezone=<IANA>&place=<free text>`** — the existing readings
endpoint gains two optional parameters.

| Parameter | Existing? | Description |
| --- | --- | --- |
| `timezone` | yes | Required IANA zone, validated as today. |
| `location_id` | yes | Optional; a saved location's id. |
| `place` | **new** | Optional free text — `Boulder`, `Moab, UT`. Mutually exclusive with `location_id`. |
| `source` | **new** | Optional; `tile` (default) or `agent`. Metrics attribution only. |

Behavior:

- `400` when both `place` and `location_id` are given. Two location selectors in
  one request is a caller bug, and silently preferring one hides it.
- `400` when `source` is present and outside the closed set. The value becomes a
  metric label; accepting arbitrary strings is an unbounded-cardinality hazard.
- `404 place_not_found` when geocoding returns no candidate for `place`.
- `404 no_locations` — unchanged — when no selector is given and the user has no
  saved locations.

The success payload is **unchanged in shape**. For an unsaved place the
`location` block carries the geocoded label, state, and country with an empty
`id` — an empty string rather than a null, so `locationRefPayload` needs no
pointer field and no existing consumer sees a type change:

```jsonc
{
  "status": "ok",
  "location": { "id": "", "label": "Boulder", "state": "CO", "country": "US" },
  "fetched_at": "2026-08-12T13:02:11Z",
  "units": { "temp": "F", "wind": "mph" },
  "current": { "temp": 71, "feels_like": 69, "humidity": 32,
               "wind_speed": 8, "condition": "Clear", "icon": "01d" },
  "today": { "high": 94, "low": 61, "sunrise": "…", "sunset": "…" },
  "hourly": [ { "at": "…", "temp": 74, "icon": "01d" } ],
  "daily":  [ { "at": "…", "high": 94, "low": 61, "condition": "Clear",
                "icon": "01d", "precip_chance": 10, "wind_speed": 9,
                "humidity": 28, "sunrise": "…", "sunset": "…" } ]
}
```

One shape for both saved and ad-hoc locations is deliberate: the MCP forwarder
below has no branch, and the model sees one payload format regardless of how the
location was named.

### Algorithms

#### Place resolution order

`place` resolves in two steps, and the first one is free:

1. **Case-insensitive match against the user's saved location labels.** A user
   who says "Denver" and has Denver saved should get *their* Denver — the exact
   coordinates they curated on the dashboard — not whatever the geocoder returns
   for the string. It also costs nothing: no provider call, no budget
   reservation, no cache lookup.
2. **`GeocodeDirect` with limit 1** otherwise, through the shipped 30-day
   geocoding cache and subject to `count_geocoding_calls`.

Without step 1, every "how's the weather in Denver?" from a Denver-based user
would spend a geocoding call to rediscover a coordinate already in their
account. The branch is three lines and removes an entire class of pointless
spend.

#### Why the API resolves the place, not the agent

The alternative is to give the model the shipped `GET /weather/search` as a
second tool and let it chain: search for "Boulder", read back coordinates, pass
them to a coordinate-taking weather call. Rejected on three counts.

It doubles the tool count — the exact per-conversation token cost the prior SOW
cited. It puts latitude and longitude into model context as values the model
then hands back, and a transposed sign is a plausible failure that produces a
confident forecast for the wrong hemisphere. And it turns one round trip into
two inside a tool loop capped at eight iterations.

Resolving server-side keeps the model's vocabulary in place names, which is the
vocabulary the user speaks in anyway.

#### The horizon asymmetry

The provider returns roughly **20 hours** of hourly buckets and **8 days** of
daily forecasts in the calls the tile already makes. A long run planned for
Saturday is therefore outside hourly range on a Tuesday.

This is accepted rather than fixed. Paging the hourly timeline to cover a week
costs about eight additional billed calls per location per refresh — the single
largest cost change available in this design — to answer a question the daily
payload already answers well enough. Each daily entry carries high, low,
condition, precipitation chance, wind, humidity, sunrise, and sunset. That is
sufficient for the real recommendation:

> Saturday is your mildest day — high 78, low 52, 10% chance of rain, light
> wind. Start near sunrise at 6:12am and you will run the coolest hours of the
> week.

What the model must not do is invent hour-level precision beyond the hourly
window. The prompt rules below make that explicit, and the tool's field
documentation states the horizon so the model can see where its data ends.

#### Why `plan_workout` does not prefetch weather

The `plan_workout` intent could prefetch the forecast the way `log_nutrition`
prefetches the pantry, putting weather in context before the model asks. It
will not.

Most workout planning is indoor lifting. A prefetch would spend provider
budget and context tokens on the majority of planning turns that never mention
going outside, and it would do so on every turn rather than once per relevant
conversation. Prefetch is the right pattern when the data is needed almost
always — a pantry, for logging food. Weather is needed sometimes, which is what
a tool is for.

### MCP Tool Surface

A new domain module, `src/prog_strength_mcp/weather.py`, registered in
`server.py` alongside its siblings, plus one method on `APIClient`. It is a
transparent forwarder in the mold of `nutrition_lookup.py` — the Go API owns
the provider, the budget, and the cache, and this module is plumbing.

```python
get_weather_forecast(timezone: str, place: str | None = None) -> dict
```

`place` is optional; absent, the API resolves the user's primary saved location.
There is no `location_id` parameter — the model has never seen an id and has no
way to produce one, so exposing it would only invite a hallucinated value.

The tool forwards the inbound `Authorization` header like every sibling, and
sends `source=agent`.

**Error translation.** Expected degradation returns structured data; only
genuine faults raise:

| API result | Tool returns |
| --- | --- |
| `404 no_locations` | `{"error": "no_saved_locations"}` |
| `404 place_not_found` | `{"error": "place_not_found", "place": …}` |
| `status: disabled` / `budget_exhausted` / `stale` / `unavailable` | passed through as `status` |
| other non-2xx | `RuntimeError` |

The two 404s are the load-bearing ones. A user with no saved location is in a
perfectly ordinary state, and the useful response is "add a location on your
dashboard, or just tell me your city" — which the model can say only if it
receives a fact rather than a stack trace. This mirrors how `nutrition_lookup`
adapts the API's 503 into `{"error": …}` instead of failing the tool.

**Docstring scope.** The tool description carries *field semantics only*: the
20-hour hourly / 8-day daily horizon, that `units` states the temperature and
wind units and they follow the user's distance preference, that `precip_chance`
is a percentage, that `sunrise`/`sunset` bound daylight running, that a `stale`
status means the numbers may be dated, and that `request_id` is never read
aloud. Training judgment lives in the agent, not here — the MCP server is shared
plumbing and has no opinion about running.

### Agent Changes

**Timezone injection.** `get_weather_forecast` joins `_TZ_AWARE_TOOLS` in
`model_harness.py`, so `_maybe_inject_timezone` supplies the user's
`client_timezone` whenever the model omits it. This follows the project's
timezone convention exactly as the nutrition and snapshot tools do: the client
sends an IANA name, the server does the conversion, and the model never
constructs a window itself.

**Prompt rules.** The `plan_workout` intent's `rules` string in `intents.py`
gains guidance on outdoor conditions. In substance:

- Call `get_weather_forecast` when the session under discussion is outdoors —
  running, cycling, hiking, outdoor conditioning — and not for indoor work.
- Heat and humidity are the dominant limiters for endurance work; wind matters
  for pace and perceived effort; precipitation chance matters for safety and
  footing; `sunrise`/`sunset` bound when a run is possible in daylight.
- Hourly detail extends about 20 hours. Beyond that, recommend a **day** and
  infer time of day from the low, the high, and sunrise — and say so, rather
  than inventing an hourly temperature for Saturday morning.
- Name the date and how fresh the forecast is. A `stale` status gets said out
  loud.
- The forecast is one input. Recovery, planned distance, and heat acclimation
  are context the model already has, and a mild day is not automatically the
  right day.

This is the only place in the system that holds an opinion about running
weather, which is the point: it is the layer where opinions are cheap to change
and where the rest of the user's training context is already in scope.

### Configuration

No changes. The `[weather]` block in `prog-strength-api/config.toml` governs the
agent path exactly as it governs the tile — the same kill switch, the same
`daily_call_budget`, the same TTLs. When `enabled = false`, the tool returns
`status: "disabled"` and the model says weather is turned off.

### Observability

**Metrics.** `api_weather_requests_total` gains a `source` label
(`tile` | `agent`), a closed two-value set validated at the handler. Provider
call, latency, cache, and budget series are unchanged: they measure the
integration, which has one shared budget and one shared cache, and splitting
them by caller would model an isolation that does not exist.

Adding a label to a live metric changes its series. Existing alert rules in
`monitoring/grafana/provisioning/alerting/rules-weather.yml` must be reviewed
for any matcher that would silently stop matching; aggregate queries over
`outcome` are unaffected, but this must be checked rather than assumed.

**Dashboard.** One panel on `monitoring/grafana/dashboards/weather.json`
breaking request outcomes down by `source`, so the owner can answer "is chat
eating the tile's budget?" — the only new operational question this feature
creates.

**No new alerts.** The shipped budget-warning, budget-critical, auto-shutoff,
integration-dead, and provider-error rules already watch the shared ledger. That
they cover the agent path for free is the direct benefit of sharing the budget
rather than partitioning it.

### Testing

**`prog-strength-api`** — handler tests for each resolution path: `place`
matching a saved label (asserting no geocode call is made), `place` falling
through to geocoding, geocoding returning nothing → `404 place_not_found`,
`place` plus `location_id` → `400`, invalid `source` → `400`, and default
`source` labeling when the parameter is absent.

**`prog-strength-mcp`** — tool tests mirroring `nutrition_lookup`'s: happy path
forwarding, each translated 404, status passthrough for `disabled` and
`budget_exhausted`, a non-404 error raising, and a missing `Authorization`
header raising before any request is made.

**`prog-strength-agent`** — extend the existing timezone-injection test to cover
`get_weather_forecast`, and a prompt test asserting the `plan_workout` rules
carry the weather guidance.

### Backfill or Migration

None. No schema change, no derived data, nothing to populate. The feature is
inert until a user asks about weather in chat, and reverting it is deleting a
tool registration.

## Open Questions

1. **Should the agent be able to save a location the user names in chat?**
   Options: leave saving to the dashboard popover (today's design); add a
   `save: true` flag to the tool; add a separate write tool. Tentative lean:
   leave it. A write tool is a second tool definition, and the case for
   persisting "Moab" from a one-off question is weak. Revisit if users
   repeatedly ask about the same unsaved place — which the `place` value in
   request logs will show.

2. **How should a weather eval be scored?** The harness in `evals/` scores
   numeric macro accuracy against a percentage tolerance, which does not fit
   "did the model call the tool and ground its answer in real dates." Options:
   extend the harness with a tool-call-assertion mode; write a small standalone
   fixture; skip evals here and rely on unit tests. Tentative lean: extend the
   harness, in its own SOW — a tool-call assertion mode is useful well beyond
   weather, and bolting a one-off scorer onto this feature would produce
   something nobody reuses.

3. **Does the mobile app need this before the dashboard exists there?**
   `prog-strength-mobile` has no dashboard and therefore no weather tile, so
   chat would be its *only* weather surface — arguably making the agent path
   more valuable there than on web. Tentative lean: nothing to do. This SOW
   requires no client change, so mobile gets weather in chat the moment the
   agent ships it.
