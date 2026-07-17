---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Trail Map Rendering

**Status**: Shipped · **Last updated**: 2026-07-17

> Frontend work on `/running/[id]` conforms to the design system
> ([`design-system.md`](../design-system.md)): dark soft near-black surfaces,
> periwinkle `--accent` for the route stroke / map chrome, Manrope, run
> discipline hue only where the page already uses it. No new palette tokens.

## Introduction

Outdoor runs already land in Prog Strength with pace, elevation, and heart rate
parsed from Garmin TCX — but the route itself is invisible. The activity detail
page (`/running/[id]`) can answer *"how fast?"* and *"how hard?"*; it cannot
answer *"where did I go?"* A trail run looks like a treadmill run with elevation
numbers bolted on.

The coordinates are already in the files we store. TCX `<Trackpoint>` elements
carry an optional `<Position>` with `<LatitudeDegrees>` / `<LongitudeDegrees>`
for any GPS-recorded activity. The API parser already detects Position
**presence** to classify running activities as outdoor vs indoor
(`environment`), then **discards the coordinates**. Raw TCX remains on S3; the
DB trackpoint rows have no lat/lon. No new Garmin artifact type is required, and
this work is not blocked on Garmin developer-program approval.

Once this ships, uploading an outdoor TCX shows a fitted route map on the
activity detail page. Indoor / non-GPS uploads keep today's page — no empty map,
no error.

## Proposed Solution

Capture GPS position at TCX ingest, persist it alongside the existing trackpoint
series, derive a simplified GeoJSON route for rendering, and show that route on
the web activity detail page with MapLibre GL JS + OpenFreeMap tiles.

**API (`prog-strength-api`)** remains the sole DB writer:

1. Extend the TCX parser to keep lat/lon when `<Position>` is present (still
   tolerate its absence).
2. Additive migration: nullable `latitude` / `longitude` on
   `activity_trackpoints`, plus nullable `route_geojson` on `activities`.
3. At ingest, from the **full raw** positioned series (before the existing
   ~300-point even-stride downsample used for pace charts): gap-split into
   segments, run Douglas–Peucker (RDP), serialize as GeoJSON
   `MultiLineString`, and persist on the activity row. Carry lat/lon onto
   downsampled trackpoint rows when the kept sample had a Position.
4. Expose `route` on `GET /activities/{id}` only. List and other callers stay
   unchanged.

**Web (`prog-strength-web`)**: on `/running/[id]`, when `route` is present,
render a MapLibre map fitted to the route bounds. When absent, omit the map
entirely — the page matches today.

### Relationship to `run-route-geometry-capture.md`

A prior draft, [`sows/run-route-geometry-capture.md`](run-route-geometry-capture.md),
targeted timeline-card SVG polylines plus a historical S3 backfill. **This SOW
supersedes that draft for ingest and schema.** It does **not** implement
timeline wiring, feed projection, or backfill — those remain follow-ups (the
timeline `RouteMap` placeholder and `TimelineRoute` type can stay as-is until a
separate SOW lights them up). Mark `run-route-geometry-capture.md` superseded
when this SOW is approved.

### Codebase contradiction (carried as a decision, not a silent workaround)

Settled intent says "full trackpoints remain the source of truth" and "nullable
lat/lon on the existing trackpoint table." In the current stack:

- DB `activity_trackpoints` are already an even-stride **downsample to ~300**
  (`maxTrackpoints` in `tcx_summarizer.go`) for chart payloads.
- Full fidelity lives only as archived TCX on S3 (`TCXS3Key`).

**Interpretation for this SOW:** S3 TCX remains the only full-fidelity store.
Lat/lon columns attach to the existing downsampled rows. The simplified
GeoJSON is computed from the **raw positioned series at ingest**, not from the
~300 downsampled points. FIT ingestion later will normalize to the same
trackpoint + route shape so the map layer stays source-agnostic; this SOW does
not introduce FIT.

## Goals and Non-Goals

### Goals

- Extract `<Position>` lat/lon from TCX on ingest; persist nullable columns on
  `activity_trackpoints`.
- Compute and persist a gap-aware, RDP-simplified GeoJSON `MultiLineString` on
  the activity row at ingest.
- Expose the simplified route on the activity detail endpoint; leave list and
  other endpoints' payloads unchanged for callers that do not request the route.
- Render the route on `/running/[id]` with MapLibre GL JS + OpenFreeMap when
  route data exists; omit the map entirely when it does not.
- Outdoor TCX upload → detail page shows the route, camera fitted to bounds.
- Indoor / no-GPS TCX upload → detail page identical to today (no map, no error).
- TCX with a mid-run GPS gap (NULL Position and/or teleport) → discontinuous
  segments, never a straight line across the gap.
- Simplified point count materially lower than raw (see Algorithms — observed
  ~94% reduction on a real trail file at the chosen ε).
- Additive-only migration; no backfill of existing activities in this scope.
- CI green in both code repos.

### Non-Goals

- **Mobile.**
- **FIT ingestion.**
- **Map matching** / snap-to-road.
- **Trail name lookup** or OSM / Overpass enrichment.
- **Backfill** of existing activities from S3 TCX (raw files remain recoverable
  for a follow-up).
- **Timeline / feed maps** (no change to `RouteMap` / `TimelineRoute` beyond
  documenting supersession of the prior geometry SOW).
- **Elevation overlay**, split markers, animated playback.
- **Basemap accounts / API keys** — OpenFreeMap requires neither.

## Implementation Details

### Data Model

**`activity_trackpoints`** — expand with nullable position columns (no separate
geometry table; it would only join back to the same rows). NULL is the natural
indoor / non-GPS signal on a per-point basis.

| Column | Type | Description |
| --- | --- | --- |
| `latitude` | `REAL` NULL | WGS84 latitude, truncated to 6 decimal places on write (~10 cm). NULL when the source trackpoint lacked `<Position>` or the kept downsample sample had none. |
| `longitude` | `REAL` NULL | WGS84 longitude, same precision rules. |

**`activities`** — nullable derived presentation artifact:

| Column | Type | Description |
| --- | --- | --- |
| `route_geojson` | `TEXT` NULL | Serialized GeoJSON `Feature` (or bare `MultiLineString`) for the simplified route. NULL when fewer than two positioned points remain after gap-splitting. |

**Why on `activities`, not a child table:** list paths already use an explicit
`activityColumns` constant in `sqlite_repository.go` — they do **not**
`SELECT *`. Omit `route_geojson` from list / range / metrics queries; select it
only in `Get` (detail). A real trail run's simplified GeoJSON is on the order of
~6 KB — cheap on a detail read, wrong to drag through every list scan if it were
accidentally added to the shared column list.

No new indexes required for v1 (route is loaded by primary key with the
activity).

### Write Path

- **TCX ingest (`IngestTCX` → parse → validate → summarize → `Create`)** —
  parser populates lat/lon on each `parsedTrackpoint` when Position is present;
  `HasPosition` presence bit behavior for `environment` is unchanged.
- **Summarize** — from the full raw series: (1) build gap-split segments of
  positioned points; (2) RDP each segment; (3) serialize `MultiLineString` at
  6-decimal precision into `route_geojson` (or leave NULL); (4) existing
  downsample path also copies lat/lon onto kept `Trackpoint` rows when present.
- **Delete** — existing `ON DELETE CASCADE` on trackpoints; `route_geojson`
  lives on the activity row and soft-deletes with it. No extra cleanup.
- **PATCH environment / calibrate** — do not recompute or clear `route_geojson`.
  Route is a function of ingested GPS, not of the user's outdoor/indoor label.

### Algorithms

#### Gap splitting → always `MultiLineString`

Garmin can omit Position in tunnels and canyons. **Never bridge gaps.**

Corpus check (`prog-strength-data/running/`, 20 GPS TCX files): **zero mid-run
NULL-Position gaps** — missing positions are leading GPS-warmup only. Large
**adjacent** jumps still occur at acquisition (up to ~162 m in 1 s while
Position is present). Normal consecutive step distance is ~2.5–3.5 m (p50) with
p95 ~4 m at 1 Hz.

**Split rules** (either triggers a new line segment):

1. A trackpoint with **NULL Position** between positioned neighbors ends the
   current segment (do not include the null point).
2. Two consecutive **positioned** points separated by **haversine distance >
   50 m** end the current segment (teleport / acquisition glitch). Time alone
   is a weak signal in this corpus (almost always 1 s sampling).

Emit GeoJSON type **`MultiLineString` for every activity that has a route**,
including the single-segment case — do not special-case `LineString`.

Leading warmup nulls simply delay the first segment start; they do not produce
an empty map if enough positioned points follow.

#### RDP epsilon

Hand-rolled Douglas–Peucker in Go (~40 lines). **No new dependency** —
`go.mod` has no geometry / simplification library, and the existing downsample
is even-stride, not RDP.

Work in **degrees** on (lat, lon) as a local planar approximation (acceptable
at city scale for trail rendering).

**ε = `5×10⁻⁵` degrees** (~5.6 m of latitude; ~4.3 m of longitude at lat 40°).

Observed on `W11 D2 - Trail Run.tcx` (gap-split on NULL only → one segment):

| ε (degrees) | ~m N–S | Raw positioned | Simplified | Retained |
| --- | --- | --- | --- | --- |
| `1×10⁻⁵` | ~1.1 | 4217 | 784 | 18.6% |
| `2×10⁻⁵` | ~2.2 | 4217 | 498 | 11.8% |
| **`5×10⁻⁵`** | **~5.6** | **4217** | **245** | **5.8% (~94% drop)** |
| `1×10⁻⁴` | ~11 | 4217 | 146 | 3.5% |

Same ε on `April 30 - 5k PR Attempt.tcx`: 1391 → 74 (~95% drop). This matches
the settled "RDP typically drops 80–90%+" expectation; ε = `5×10⁻⁵` is the
committed default.

#### Coordinate precision

Truncate to **six decimal places on write** (trackpoint columns and GeoJSON
coordinates). Six decimals ≈ 10 cm — enough for map zoom; raw TCX float64 on
disk in S3 retains full precision for future re-parse.

GeoJSON coordinate order is **`[longitude, latitude]`** per RFC 7946.

### API Surface

**`GET /activities/{id}`** (existing, JWT-authed) — extend the detail DTO:

```json
{
  "route": {
    "type": "Feature",
    "geometry": {
      "type": "MultiLineString",
      "coordinates": [[[lng, lat], ...], ...]
    },
    "properties": {
      "bounds": {
        "min_lat": 0,
        "min_lng": 0,
        "max_lat": 0,
        "max_lng": 0
      }
    }
  }
}
```

- Omit `route` (or send `null`) when `route_geojson` is NULL.
- Do **not** add lat/lon to each `trackpoints[]` element in the wire DTO for
  this SOW unless needed for tests — the map consumes `route` only. (DB columns
  still store per-trackpoint coords for future correlation / FIT parity.)
- **`GET /activities/`** and other list/summary endpoints: no `route` field;
  do not select `route_geojson`.

`POST /activities/tcx` already returns the detail-shaped DTO; include `route`
there the same way so a just-uploaded outdoor run can navigate to a mapped
detail page without a second fetch shape.

### Web (`/running/[id]`)

- Add `maplibre-gl` (and types). Tile source: **OpenFreeMap** style URL (no API
  key, no account) — document the exact style endpoint chosen at
  implementation in the PR.
- New presentational component (e.g. `RunRouteMap`) mounted on the detail page
  when `session.route` is present with at least one line of ≥ 2 positions.
- Fit camera to `bounds` on load; route stroke uses design-system `--accent`
  (or a clearly documented discipline token if accent-on-basemap fails contrast
  — call out any deliberate extension in the PR).
- **Conditional UI:** if no route, render nothing in that slot — no placeholder,
  no "map coming soon", no empty frame. Indoor pages stay byte-for-byte the
  same layout as today aside from unrelated content.
- Placement: above or below the existing pace recap is an implementation
  choice; prefer **below the header band / above or beneath the splits spine**
  so the ledger composition from `running-activity-detail` stays intact. Do not
  invent a card-heavy hero.

### Backfill or Migration

1. **Mechanism** — next free migration under `internal/db/migrations/`:
   `ALTER TABLE activity_trackpoints ADD COLUMN latitude REAL;`
   `ALTER TABLE activity_trackpoints ADD COLUMN longitude REAL;`
   `ALTER TABLE activities ADD COLUMN route_geojson TEXT;`
   Expand-only (same pattern as `036_activity_environment.sql`). No table
   rebuild required.
2. **Recoverability** — additive nullables; failure mid-deploy leaves old
   rows with NULL route (correct for "no map").
3. **Scale boundary** — **no backfill in this SOW.** Existing activities keep
   NULL coords / NULL route until a follow-up re-parses S3 TCX. New uploads
   after deploy populate both.

### Indoor vs GPS-never-acquired

Both produce NULL lat/lon and NULL `route_geojson`. For running, the existing
`environment` field already disambiguates (`HasPosition` anywhere → outdoor;
else indoor). The map does **not** key off `environment` or sport type — it
keys off **route presence**. A user who toggles outdoor→indoor on a GPS run
still has a route (and still sees the map); that is intentional given
Non-Goals around recomputing on PATCH.

## Open Questions

1. **Exact OpenFreeMap style URL / dark basemap variant.** Options: default
   OpenFreeMap liberty/positron-style vs a darker style that matches the app
   chrome. *Tentative lean:* pick the darkest first-party OpenFreeMap style that
   stays readable with an accent route stroke; document URL in the web PR.
2. **Whether detail `trackpoints[]` should also expose lat/lon on the wire.**
   DB stores them; map uses GeoJSON. *Tentative lean:* omit from DTO in this
   SOW to keep payloads stable for chart consumers; add later if pace↔map
   brushing needs it.
3. **Supersession edit to `run-route-geometry-capture.md`.** *Tentative lean:*
   on approval of this SOW, set that draft's status to superseded with a pointer
   here (docs-only commit, can land with this file).
)
