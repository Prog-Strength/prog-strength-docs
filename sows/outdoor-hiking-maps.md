---
status: draft
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Outdoor Hiking Maps

**Status**: Draft · **Last updated**: 2026-07-30

> Frontend work conforms to the design system
> ([`design-system.md`](../design-system.md)). Two of its rules do real work
> here and are called out where they bind: **dark theme is a Fixed Point**
> (so a basemap that cannot be re-toned dark is disqualified, and the satellite
> style is a deliberate, scoped exception), and the **"activity ≠ selection"**
> rule under Activity session-recap (`--accent` is edit/focus chrome only,
> never a series hue) — which today's route stroke violates. This SOW fixes
> that rather than propagating it. One new token family is proposed
> (`--map-*` chrome), justified in § Design System.

## Introduction

**Prog Strength** can draw where you hiked. It cannot show you the mountain.

`/hiking/[id]` shipped with the hiking activity type
([`hiking-activity-type.md`](hiking-activity-type.md), web PR #124) and renders
a real GPS route on a real map — `RunRouteMap` over MapLibre GL JS. But the
basemap under that route is
`https://tiles.openfreemap.org/styles/dark`: a **generic dark street map**.
It knows about roads, and it does not know about terrain. Load a Quandary Peak
hike onto it and you get a periwinkle squiggle floating on near-black
emptiness, because above treeline in Summit County there are no streets to
draw. The single most information-dense fact about that hike — that it climbs
3,450 feet up the side of a fourteener — is invisible on the surface whose
entire job is to show you where you went.

The gap is not the renderer. MapLibre is the right engine and stays. The gap is
that every layer *above* the renderer is the one a city street map needs: no
contours, no hillshade, no summits, no water, no trails, one basemap with no
alternative, and a route stroke with no legibility treatment because a
near-black canvas never required one.

There is a second, subtler gap. The elevation profile
(`ElevationRecap` → `RecapChart`) sits directly beneath the map on the same
page, plotting the same hike over the same distance axis — and the two
components have never been introduced. The profile shows a 700-foot wall at
mile 2.8 and the map cannot tell you where mile 2.8 *is*. Every serious outdoor
app (AllTrails, Gaia GPS, CalTopo) treats map-and-profile as one linked
instrument. Ours are two pictures of the same hike stacked vertically.

When this ships, opening a hike shows shaded terrain with contour lines, named
peaks, and water under a bordered, direction-marked route; a style switcher for
Standard / Topographic / Satellite / Satellite + Trails / Dark; and an elevation
profile that drives a marker along the route as you scrub it, reporting grade,
remaining climb, and remaining distance at the point under your finger.

## Proposed Solution

Keep MapLibre. Replace the single hardcoded style URL with a **style registry**,
put a **composable overlay stack** on top of whichever basemap is active, and
**wire the elevation profile and the map to one shared scrub index**.

Four moving parts:

**1. Provider is a per-layer decision, not a vendor.** MapLibre composes sources
and layers, so "which map provider" is the wrong unit of choice. The basemap,
the DEM that drives hillshade, and the satellite imagery are three independent
procurements, and the cheapest correct answer picks a different supplier for
each: **MapTiler Outdoor** (vector, restylable, contours and peaks in-style) for
topographic; **Mapterhorn** (free, open, global Terrain-RGB) for the DEM behind
hillshade; **Esri World Imagery** for satellite; **OpenFreeMap** retained for
Standard and Dark at zero cost. Full evaluation in § Third-Party Evaluation.
The registry means swapping any one of them later is a one-entry edit.

**2. A style registry replaces the constant.** `lib/map-styles.ts` exports a
keyed record of style definitions — id, label, style source, attribution,
whether it needs a key, and which overlays it supports. `MapView` reads the
registry. Adding a sixth style is adding a record entry; it touches no component.

**3. Overlays are owned separately from the basemap.** MapLibre's `setStyle()`
discards every source and layer the caller added — so a naive style switcher
silently drops the route the moment the user changes basemap. The route,
hillshade, direction arrows, mileage markers, and scrub marker live in one
`installOverlays(map)` function re-invoked on every `styledata` event, inserted
with explicit `beforeId` anchors so the route sits above terrain and below
labels. This is the single load-bearing abstraction in the SOW: it is what makes
"add another overlay later" free, and it is where the bug lives if it isn't.

**4. One scrub index links profile and map.** `RecapChart` is a hand-rolled SVG
chart (`components/activity-detail/RecapChart.tsx`) whose points are built by
`buildElevationStrip`, a straight `.map` over the served trackpoints — so strip
index *i* **is** trackpoint *i*. Lift that index into page state, and the map
marker, the ahead/behind route split, and the grade/remaining readouts all fall
out of one number. The only thing missing is coordinates: `trackpointDTO`
carries `distance_meters` and `elevation_meters` but no lat/lon, deliberately
deferred in [`sow-trail-map.md`](sow-trail-map.md) Open Question 2. **This SOW
answers that question yes** — three nullable fields on an existing DTO, no new
endpoint, no schema change (the columns already exist and are already
backfilled).

**No new charting library.** See § Elevation Profile Integration for why
extending `RecapChart` beats adopting one.

### Scope correction: this is web, not mobile

The feature request that prompted this SOW described the target as the mobile
application. It isn't, and the correction changes the work materially:

- `prog-strength-web` has `maplibre-gl@^5.24.0`, `RunRouteMap`, and shipped
  `/hiking/[id]` and `/running/[id]` pages that mount it. This is the
  implementation to evolve.
- `prog-strength-mobile` has **no map dependency and no activity detail screen
  at all** — `app/(tabs)/activities/` is Overview / Workouts / Running / Steps
  list views. There is nothing there to improve.

A mobile outdoor map is a real and wanted thing, but it is a different SOW:
`@maplibre/maplibre-react-native` is a native module, so it needs a
`runtimeVersion` bump, an EAS native rebuild, and a TestFlight submission (OTA
cannot deliver it), *and* it needs a hike detail screen built from scratch
first. It belongs in the mobile-parity phase sequence
([`mobile-feature-parity-and-testflight.md`](mobile-feature-parity-and-testflight.md)),
not here. Explicit non-goal below.

### What this inherits and what it is blocked on

Nothing. The data is already in the database and already on the wire:
`activity_trackpoints.latitude/longitude` and `activities.route_geojson` shipped
in [`sow-trail-map.md`](sow-trail-map.md), historical activities were backfilled
from S3 TCX in [`sow-trail-map-backfill.md`](sow-trail-map-backfill.md)
(`internal/activity/backfill.go`), and the elevation quadruple
(gain / loss / high / low) shipped in migration `044_activity_hike_elevation.sql`.
This SOW adds **no migration**. It is presentation and one additive DTO change.

## Goals and Non-Goals

### Goals

- `/hiking/[id]` and `/running/[id]` render an outdoor basemap with **hillshaded
  terrain and contour lines** by default for hikes.
- A **style switcher** offering Standard, Topographic, Satellite,
  Satellite + Trails, and Dark, with the chosen style persisted per user and
  restored on the next visit.
- Style changes **never drop overlays** — the route, markers, and hillshade
  survive every `setStyle()` call.
- The route renders with a **dark casing under a discipline-hue stroke**,
  start/finish markers, direction arrows, and mileage markers — legible against
  both a pale topo canvas and satellite imagery.
- `GET /activities/{id}` returns `latitude`, `longitude`, and `grade_percent`
  on each element of `trackpoints[]`, nullable, additive.
- **Scrubbing the elevation profile** moves a marker along the route, splits the
  route into travelled/remaining strokes, and shows elevation, grade, remaining
  climb, and remaining distance at that point.
- **Hovering or tapping the route** highlights the corresponding position on the
  elevation profile (the reverse binding).
- The profile header reports **total ascent, total descent, high point, low
  point, and distance** from the values the API already stores.
- Peaks, water features, and forest/landcover render as part of the topographic
  basemap, with **summit names visible** at hike zoom levels.
- Attribution for every active source renders on the map, per each provider's
  terms.
- Provider keys are **non-secret client config** in `lib/config.ts` alongside
  `apiUrl`, origin-restricted at the provider — not repository secrets.
- The map falls back to today's OpenFreeMap dark style, with the route intact,
  if a keyed provider is unconfigured or failing.
- CI green in both code repos.

### Non-Goals

- **Mobile.** See § Scope correction. Separate SOW, gated on a hike detail
  screen existing at all.
- **Offline maps and downloadable regions.** Architected for
  (§ Offline Readiness names the seam and the licensing constraint) but **not
  built**. No download UI, no storage quota, no Service Worker tile cache.
- **Live GPS navigation, turn-by-turn, or voice guidance.** The app ingests
  completed TCX files; it does not record activities. Navigation presupposes a
  recorder.
- **POI enrichment — trailheads, campsites, water sources, viewpoints,
  user waypoints.** These are not in our data and are not reliably in a
  commercial basemap. They need an OSM/Overpass ingestion pipeline with its own
  storage, refresh, and licensing story. Named peaks are the exception: they
  come free with the topographic basemap as label layers. See § Route Rendering
  for the phase split.
- **Public land / National Forest / wilderness boundaries.** Same reason —
  PAD-US and USFS boundary data is a separate dataset with a separate pipeline.
- **Weather, snow depth, avalanche forecast overlays.** Future; the overlay
  architecture is what makes them cheap later.
- **Heatmaps, public trail discovery, user-generated routes.** These are social
  and multi-user features; the app is pre-launch and single-user.
- **3D terrain (`map.setTerrain`) / pitched camera.** Hillshade gives terrain
  legibility in 2D at a fraction of the DEM request volume and none of the
  interaction cost. Revisit after the 2D map is right.
- **Map matching / snap-to-trail**, route editing, or GPX *export*.
- **Any database migration.** The columns exist and are backfilled.
- **Re-simplifying `route_geojson` or changing the RDP epsilon.** ε = 5×10⁻⁵
  is the committed default from `sow-trail-map.md` and stays.
- **Backfilling `design-system.md` to v0.4.4** for the hike hue that shipped in
  `globals.css` but never reached the doc. Real drift, pre-existing, tracked as
  Open Question 5 — not silently absorbed here.

## Implementation Details

### Third-Party Evaluation

The request framed this as choosing one provider. MapLibre's layer model means
that framing is a trap: the basemap style, the DEM behind hillshade, and the
satellite imagery are independently sourced, and buying all three from one
vendor is strictly more expensive than picking each. The evaluation below is
therefore per-role, with the basemap decision first because it is the one that
carries the licensing risk.

#### Basemap candidates

| | Tiles | MapLibre fit | Dark re-tone | Contours + hillshade in style | Free tier | Commercial | Paid entry |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **MapTiler Outdoor** | Vector | Native (MapTiler is a principal MapLibre sponsor) | **Yes** — style JSON is editable | **Yes**, both | 100k requests / 5k sessions per month | **Free tier is non-commercial** | **Flex $30/mo** — 500k req / 25k sessions |
| **Thunderforest Outdoors** | Raster + vector (`outdoors-v2`) | Good, documented MapLibre vector-style path | Poor — the look is a light OS-style topo; a dark re-tone fights the design | Contours yes; hillshade baked into raster | Hobby: 150k tiles/mo | Hobby is a trial tier | **Solo Developer $125/mo** — 4× MapTiler |
| **OpenTopoMap** | Raster only | Works as a `raster` source | **No** — raster, unrestylable | Both, baked in | Community server, ~400k/mo soft cap, 5k tiles/hr per-IP block | **Non-commercial only** | None — no paid tier exists |
| **Mapbox Outdoors** | Vector | **Disqualified** | — | Yes | — | — | — |
| **Esri Topographic** | Raster (vector basemaps exist behind an ArcGIS account) | Works as a `raster` source | No — light cartography | Yes | Free with a developer account, <1M tiles/mo, non-revenue app | Revenue triggers paid | Usage-based |

**Mapbox is disqualified on licensing, not quality.** Its Outdoors style is
arguably the best-looking candidate, but Mapbox's terms tie its tiles to its own
SDK; consuming them from MapLibre is a terms violation. Since "preserve the
MapLibre rendering stack" is a stated constraint of this work, Mapbox cannot be
the answer regardless of how the style looks.

**OpenTopoMap is disqualified as a primary.** Raster-only means it cannot be
re-toned, and dark theme is a design-system Fixed Point — a bright cream-and-
brown 1:25k topo raster is not a style choice inside this product, it is a
different product. The per-IP hourly block and the non-commercial term also make
it structurally unfit for anything but a fallback.

**Thunderforest loses on cost and on tone.** It is a legitimate outdoors
cartographer with a real MapLibre vector path, and its 150k free tiles beat
MapTiler's 100k. But the first commercial tier is **$125/mo against MapTiler's
$30/mo**, and offline bulk-download — the capability we are explicitly
architecting toward — is gated at the **$255/mo** Small Business tier. For a
pre-launch solo product that is a 4× premium to buy a style that then has to be
fought into a dark theme.

**Recommendation: MapTiler Outdoor.** It is the only candidate that is
simultaneously (a) vector, therefore re-tonable to the dark theme the design
system fixes as inviolable; (b) MapLibre-native rather than MapLibre-tolerant;
(c) shipping contours, hillshade, peak labels, water, and landcover *inside the
style*, which collapses most of the Terrain Visualization goal into picking the
right style URL; (d) able to serve the satellite layer from the same key and the
same account when we outgrow the free imagery; and (e) an order of magnitude
cheaper than the runner-up at the first commercial tier. Its free tier's
non-commercial restriction is the one real objection, and it is a **launch-date
billing decision, not an architectural one** — see § Cost Analysis and Risk R1.

#### DEM (hillshade) candidates

Hillshade needs a Terrain-RGB or Terrarium-encoded elevation raster. MapTiler
sells one, but hillshade tiles are requested at the same rate as basemap tiles
and would roughly **double** the metered request count against a plan whose
binding limit is sessions and requests. Two free global alternatives exist:

- **Mapterhorn** (`https://tiles.mapterhorn.com/tilejson.json`) — free, global,
  open (BSD-3 code), distributed as static PMTiles, and specifically maintained
  as the default terrain source for MapLibre. Terrarium-encoded, orthometric
  heights, maxzoom 12.
- **AWS Terrain Tiles** (`s3.amazonaws.com/elevation-tiles-prod/terrarium/…`) —
  the Mapzen-derived Registry of Open Data set. Also free and Terrarium-encoded,
  but less actively maintained.

**Recommendation: Mapterhorn**, with AWS Terrain Tiles as the drop-in fallback
(same encoding, so it is a URL swap in the registry). This keeps every hillshade
tile off the metered plan.

⚠️ **Encoding is load-bearing.** MapLibre's `raster-dem` source defaults to
`"mapbox"` encoding; Mapterhorn and AWS are **`"terrarium"`**. Declaring the
wrong one does not error — it renders a plausible-looking but numerically wrong
hillshade. `encoding: "terrarium"` is not optional and is asserted in test.

#### Satellite candidates

- **Esri World Imagery** — 0.3 m over the continental US (which is where the
  hikes are), free under an ArcGIS developer account while the app is
  non-revenue and under 1M tiles/month, attribution required
  ("Esri, Maxar, Earthstar Geographics, and the GIS User Community").
- **MapTiler Satellite** — same key and account as the basemap, metered against
  the same plan.

**Recommendation: Esri World Imagery now, MapTiler Satellite as the paid
upgrade path.** Both are `raster` sources behind one registry entry, so the swap
is a URL and an attribution string. Esri's free terms expire at the same
event MapTiler's do — first revenue — which usefully makes it one decision
rather than two.

#### Summary of the procurement

| Role | Source | Cost today | Cost at launch |
| --- | --- | --- | --- |
| Standard / Dark basemap | OpenFreeMap (unchanged) | $0 | $0 |
| Topographic basemap | MapTiler Outdoor | $0 (free tier) | $30/mo (Flex) |
| Hillshade DEM | Mapterhorn | $0 | $0 |
| Satellite imagery | Esri World Imagery | $0 | $0 → MapTiler Satellite if revenue |
| Contours | In the MapTiler Outdoor style | $0 | included |

### Technical Architecture

#### Vector vs raster

**Vector wherever there is a choice.** Vector tiles are restylable at runtime —
which is what lets a topographic style be re-toned to the dark ramp rather than
imported as someone else's light cartography — and a vector map view costs
roughly 4–8 tile requests against MapTiler's meter versus 10–16 for 256px
raster. Raster is used only where the medium requires it: satellite imagery and
the DEM.

This has a design consequence worth stating plainly: **Satellite is the one
style where the dark-theme Fixed Point does not apply**, because photographic
imagery has no theme. That is a scoped, intentional exception (map chrome, route
stroke, and controls stay in system tokens over it), not a precedent for light
surfaces elsewhere in the app.

#### The style registry

`lib/map-styles.ts` — pure, no React, unit-testable:

```ts
export type MapStyleId =
  | "standard" | "topo" | "satellite" | "satellite-trails" | "dark";

export type MapStyleDef = {
  id: MapStyleId;
  label: string;                       // switcher label
  /** Resolved at call time so a missing key degrades instead of throwing. */
  resolve: (keys: MapKeys) => StyleSpecification | string | null;
  attribution: string[];               // rendered in the attribution control
  requiresKey: null | "maptiler" | "esri";
  /** Hillshade is skipped on styles that already bake it into the imagery. */
  supportsHillshade: boolean;
  /** Preferred stroke treatment — imagery needs a heavier casing. */
  canvas: "dark" | "light" | "imagery";
};

export const MAP_STYLES: Record<MapStyleId, MapStyleDef> = { /* … */ };
export const DEFAULT_STYLE: Record<"hike" | "run", MapStyleId> =
  { hike: "topo", run: "standard" };
```

Adding a sixth style is adding one record entry. Nothing in `MapView`,
the switcher, or the overlay installer enumerates styles — the switcher maps
over `Object.values(MAP_STYLES)`, and `canvas` (not a style-id check) is what
drives stroke and chrome treatment, so a new style declares its own needs
instead of being special-cased at each consumer.

`resolve()` returning `null` when its key is absent is the fallback contract:
`MapView` filters unresolvable styles out of the switcher and falls back to
`"dark"` (OpenFreeMap, keyless) if the persisted choice is one of them. A
missing `NEXT_PUBLIC_MAPTILER_KEY` therefore degrades to exactly today's map —
it never renders a broken tile grid or throws.

**`satellite-trails` is a composed style, not a vendor product.** No provider
ships "imagery with trail lines and labels." It is built by fetching the
MapTiler Outdoor style JSON, replacing its background and landcover layers with
the Esri imagery `raster` source, and retaining its path, waterway, and symbol
layers on top. This is genuine work and is the one style deferred to Phase 3.

#### Overlay architecture

The single most important behavior in this SOW:

> `map.setStyle()` replaces the entire style document. **Every source and layer
> added by application code is destroyed.** A style switcher written naively
> loses the route on first use.

So overlays are not added once at load; they are declared once and *installed*
idempotently:

```ts
// Ordered, id-stable. Re-run after every style load.
installOverlays(map, {
  hillshade,        // raster-dem + hillshade layer, skipped when !supportsHillshade
  routeCasing,      // dark 7px under-stroke — legibility on pale canvases
  routeTravelled,   // discipline hue, filtered to scrubIndex and earlier
  routeRemaining,   // muted, filtered to after scrubIndex
  routeArrows,      // symbol layer, symbol-placement: "line"
  mileMarkers,      // symbol layer from client-computed points
  endpoints,        // start / finish
  scrubMarker,      // single point, follows the profile cursor
});

map.on("styledata", () => installOverlays(map, spec));
```

`styledata` (not `load`, and not `style.load`) is the event that fires on both
initial load and every subsequent `setStyle`, so one subscription covers both
paths and the overlay set has exactly one installation site. Each layer is
inserted with an explicit `beforeId` resolved from the active style's first
symbol layer, so the ordering invariant — **terrain below route, route below
labels** — holds across providers whose layer names differ.

The overlay list is data. Adding a weather raster or a public-land boundary
later is appending an entry and its paint spec, with no change to lifecycle
code. That is the extensibility the request asked for, and it is the whole
reason the installer is a separate function rather than inline `useEffect` work.

#### Caching

Three tiers, none of which we build:

1. **HTTP cache.** Tile responses carry long `Cache-Control` from every provider
   above; the browser cache absorbs pan/zoom re-requests within a session and
   across page views. This is why the *session* limit binds before the *request*
   limit on MapTiler.
2. **MapLibre's in-memory tile cache**, per map instance.
3. **Our own request shaping.** `maxzoom` on the DEM source is capped at 12
   (Mapterhorn's native max) so MapLibre overzooms rather than requesting
   nonexistent tiles, and the map does not mount until the route is present —
   the existing `if (!route) return null` contract in `RunRouteMap`, preserved.

Anything beyond this is offline territory, below.

#### Offline readiness (architected, not built)

Two seams are placed now so that a future offline SOW is additive:

- **Every tile source is declared through the registry**, so a future
  PMTiles-over-`file://` or Cache-API-backed source is a fourth `resolve()`
  variant, not a rewrite. Mapterhorn already ships as PMTiles, which is the
  format an offline DEM would use unchanged.
- **`MapStyleDef.requiresKey` doubles as the offline-eligibility flag.** Offline
  is a *licensing* problem before it is a technical one: bulk pre-fetching tiles
  for later viewing is prohibited or separately priced by most providers
  (Thunderforest gates it at the $255/mo tier; MapTiler's caching terms are
  plan-dependent). A registry that already records which sources are keyed and
  under whose terms is the thing that keeps a future offline feature from
  quietly violating a contract.

No download UI, no region picker, no quota accounting in this SOW.

### API Surface

One changed contract. **No new endpoint, no migration.**

**`GET /activities/{id}`** (and `POST /activities/tcx`, which returns the same
detail-shaped DTO) — `trackpointDTO` in `internal/activity/handler.go` gains
three nullable fields:

| Field | Type | Description |
| --- | --- | --- |
| `latitude` | `*float64` | WGS84 latitude of this sample, already stored on `activity_trackpoints`. NULL when the sample had no `<Position>`. |
| `longitude` | `*float64` | WGS84 longitude, same. |
| `grade_percent` | `*float64` | Signed grade at this sample, in percent. NULL when it cannot be computed. |

Per the DTO convention already documented in `handler.go` — *"nullable numerics
are pointers WITHOUT omitempty so the key is always present"* — all three keys
are always present and render `null` when absent. `trackpoints` itself keeps its
`omitempty` and stays off list responses; this change costs nothing on
`GET /activities`.

This answers [`sow-trail-map.md`](sow-trail-map.md) Open Question 2 — *"whether
detail `trackpoints[]` should also expose lat/lon on the wire"* — in the
affirmative, with the reason that SOW anticipated: pace↔map brushing now needs
it. That SOW should be updated to record the answer.

**Payload cost.** Trackpoints are capped at ~300 by `maxTrackpoints` in
`tcx_summarizer.go`. Three float64s at six decimal places is ~30 bytes per
point, so ~9 KB uncompressed and well under 2 KB gzipped on a detail read that
already carries the pace, HR, and elevation series. Acceptable.

**Why the server computes grade.** It could be a client one-liner, and that is
exactly why it shouldn't be: the moment grade exists on both the profile tooltip
and (later) a splits table or the agent's session summary, two implementations
disagree and a user does the arithmetic. This is the lesson recorded in
[`running-detail-metric-alignment.md`](running-detail-metric-alignment.md), and
the same reason `clean_pace` is a server-owned boolean on this exact DTO rather
than a client threshold.

**Grade is derived at read time, not stored.** It is a pure function of the
trackpoint series that a detail read has already loaded, so computing it in the
handler's DTO projection keeps this SOW free of a migration *and* means every
historical activity gets grade on the next request rather than only on
re-ingest. It also cannot go stale against a recalibrated distance. The cost is
one O(n) pass over ~300 points per detail read, which is noise beside the query
that fetched them.

**Grade derivation** (in the `trackpointDTO` projection,
`internal/activity/handler.go`):

```
grade[i] = 100 × (elev[i] − elev[i−w]) / (dist[i] − dist[i−w])
```

over a **smoothing window `w`** rather than adjacent samples. Adjacent-sample
grade on a ~300-point downsample of a 6-hour hike is dominated by barometric
noise — a 1 m altimeter jitter over a 4 m step reads as 25% grade. `w` is
chosen as the smallest window spanning **≥ 30 horizontal metres**, which is
roughly a switchback's worth of trail and empirically stable. `grade_percent` is
NULL for the first `w` samples, when either endpoint lacks elevation, or when
the horizontal denominator is zero (a stationary sample). Sign is preserved:
descent is negative.

Because it is projected rather than stored, every existing activity carries
grade from the moment this deploys — no backfill, no re-ingest. The elevation
*aggregates* the profile header displays — ascent, descent, high, low — are
already stored per-activity from `044_activity_hike_elevation.sql` and need
nothing.

### Terrain Visualization

| Feature | Source | Phase |
| --- | --- | --- |
| Contour lines | MapTiler Outdoor style (built in) | 1 |
| Hillshading | MapLibre `hillshade` layer over Mapterhorn `raster-dem` | 1 |
| Elevation labels | Contour labels in the MapTiler style | 1 |
| Peak names | `natural=peak` symbol layers in the MapTiler style | 1 |
| Water features | Waterway / water layers in the style | 1 |
| Forest & landcover | Landcover layers in the style | 1 |
| National park / public land boundaries | **Not available** in a general basemap — needs PAD-US or NPS ingestion | Non-goal |

Most of this table is "pick the right style URL," which is the strongest
argument for the MapTiler recommendation and the reason Terrain Visualization is
Phase 1 rather than a phase of its own.

**Hillshade is ours, not the style's,** even though MapTiler ships one. Running
it as our own layer over Mapterhorn keeps every DEM tile off the metered plan
(§ DEM candidates), and — because it is our layer — lets `hillshade-exaggeration`
and `hillshade-shadow-color` be tuned to the dark ramp instead of to MapTiler's
defaults. On the Satellite style `supportsHillshade` is `false`: imagery already
carries its own shading and a hillshade over it reads as mud.

**Dark re-tone of the topographic style.** MapTiler Outdoor ships light. The
style JSON is fetched, and a small pure transform re-maps its background,
landcover, and water fills onto the design system's near-black ramp, leaving
contours, paths, and labels at raised luminance so they carry. This transform
lives in `lib/map-styles.ts` as a data operation over the style document — it is
tested against a checked-in style fixture, not against the network.

⚠️ **Contour density at phone width.** A 1:24k contour interval that reads well
at 1200px is illegible at 390px. The re-tone pass sets a `minzoom` on the
contour layers so they appear at hike-relevant zooms only; the exact value is
tuned on-device (Open Question 4).

### Route Rendering

**Phase 1 — derivable from data we already have:**

- **Casing.** A dark 7px under-stroke beneath the 3px coloured line. Today's
  single stroke on near-black needs no casing; on a pale topo canvas or over
  imagery it disappears. This is the single largest legibility win in the SOW
  and it is pure paint.
- **Discipline-hue stroke, replacing `--accent`.** Today `RunRouteMap` resolves
  `--accent` (`#9aa6d6`) for the line. The design system's Activity
  session-recap section is explicit that *"`--accent` stays edit/focus chrome
  only … and is never a section or series hue (the 'activity ≠ selection'
  rule)"* — and the page's own elevation chart already obeys this, drawing in
  `--discipline-hike-fg`. The map is the outlier. It moves to
  `--discipline-hike-fg` (`#c9a690`) on hikes and `--discipline-run-fg`
  (`#9cc7b8`) on runs, matching the chart it is now linked to. That the route
  and the profile share a colour is not decoration once they share a scrub
  index; it is what tells the user they are one instrument.
- **Start and finish markers.**
- **Direction arrows** — a `symbol` layer with `symbol-placement: "line"` and a
  spacing that thins with zoom. No new data.
- **Mileage markers** — computed client-side from `trackpoints[]`, walking the
  now-available lat/lon and picking the sample that crosses each whole
  mile/kilometre boundary in the user's active unit. No new data, and it
  respects the existing `DistanceUnit` context.
- **Travelled vs remaining split** — see below.

**Later phases — need data we do not have:**

Trailheads, campsites, water sources, viewpoints, and user waypoints are POIs.
They are not in `activity_trackpoints` and are not dependably in a commercial
basemap's symbol layers. Delivering them honestly means an Overpass/OSM
ingestion pipeline, a table to hold the results, a refresh policy, and an ODbL
attribution story — that is a SOW, not a bullet. **Named peaks are the
exception** and land in Phase 1 because the topographic basemap already labels
them.

**On "completed vs remaining route colouring."** For a *completed* hike that has
already been uploaded, there is no remaining — the request describes a live-
navigation concept, and live navigation is a non-goal. But the same treatment
has a real meaning here: with a scrub index, the route splits into **travelled**
(full-strength discipline hue, up to the cursor) and **ahead** (muted, after
it). That is implemented, in Phase 2, as two filtered line layers over one
source — and it is the thing that makes scrubbing feel like playback rather than
like moving a dot.

### Elevation Profile Integration

#### Why no new library

`components/activity-detail/RecapChart.tsx` is a hand-rolled SVG chart
(`W = 860`, `H = 300`, explicit margins, `niceStep` tick selection, caller-chosen
y-inversion, and a path builder that **never bridges a `null`**). It is shared
by the pace, heart-rate, and elevation recaps.

`recharts@^3.8.1` *is* already a dependency — but it is used by the older
`app/(app)/running/_components/` charts, not by these. Adopting it here would
mean re-implementing, in a library's idiom, the gap-flushing and editorial-axis
conventions `RecapChart` exists to encode, across three sibling recaps, to
obtain hover handling that is roughly forty lines of pointer math on an SVG we
already own. Specialist elevation-profile libraries have the same problem in
sharper form: they bring their own canvas and their own visual language, and the
design system has opinions about both.

**Recommendation: extend `RecapChart`.** Add an optional
`onScrub?: (index: number | null) => void` and `scrubIndex?: number | null`.
When `onScrub` is absent the component is byte-for-byte what it is today, so
pace and heart-rate recaps are untouched by this SOW.

#### The shared index

`buildElevationStrip` is a `.map` over the served trackpoints
(`lib/running-traces.ts`), so `MetricStripPoint[]` is **index-aligned with
`trackpoints[]`** — position *i* in the chart is trackpoint *i*. That alignment
is what makes one integer sufficient to link the two components, and it is the
reason the route's own `route_geojson` cannot serve this purpose: it is
RDP-simplified to ~245 points from ~4,217 raw, deliberately not index-aligned
with anything.

`/hiking/[id]` holds the state:

```ts
const [scrubIndex, setScrubIndex] = useState<number | null>(null);
```

- **Profile → map.** `RecapChart` maps pointer x → nearest plottable index,
  calls `onScrub`. The map moves `scrubMarker` to
  `trackpoints[i].latitude/longitude` and re-filters the travelled/ahead layers.
- **Map → profile.** `MapView` binds `mousemove`/`touchmove` on the route layer,
  finds the nearest trackpoint by haversine, calls the same setter.
  ⚠️ Guard against a feedback loop: the setter is idempotent on equal values,
  and neither direction re-emits on a programmatic update.
- **Readouts.** A caption strip under the profile reports, at the cursor:
  elevation, `grade_percent`, **remaining climb** (a suffix sum of positive
  elevation deltas from *i* to the end, memoised once per trackpoint array), and
  **remaining distance** (`total − distance_meters[i]`). With no cursor it shows
  the whole-hike aggregates — total ascent, descent, high, low, distance — from
  the values the API already stores.
- **Touch.** Scrub is drag-to-track with a hit target padded to 44px vertically;
  a tap outside the plot clears the cursor. Touch-scrubbing calls
  `preventDefault` only while a drag is active, so vertical page scroll over the
  chart still works.

Both components render exactly as today when `scrubIndex` is `null`, which is
also the state on a device with no pointer.

### Design System

This work conforms to `design-system.md` v0.4.3 in three ways and extends it in
one.

**Conforms:** the route stroke moves from `--accent` onto the discipline hue,
correcting an existing violation of the "activity ≠ selection" rule rather than
propagating it. The style switcher is a pill row in the existing timeframe-pill
idiom, not a `<select>`. The map card keeps `--radius-card`, `--border`, and
`--surface`, and beats with no data are omitted whole — `RunRouteMap`'s existing
`if (!route) return null` contract is preserved verbatim.

**Extends:** map chrome cannot be expressed in the existing token set. Controls
sit over provider-supplied canvases whose luminance we do not control — a pale
topo, an aerial photograph — and `--surface` over imagery is invisible. One new
token family, proposed for **v0.4.5**:

| Token | Purpose |
| --- | --- |
| `--map-chrome-bg` | Scrim behind switcher/attribution, tuned to read on imagery and on pale topo |
| `--map-chrome-fg` | Chrome label colour |
| `--map-route-casing` | The dark under-stroke; deliberately not `--background` (it must darken imagery, not match a panel) |

Three tokens, one surface, an explicit amendment with provenance — not an
open-coded hex in a component. `design-system.md` bumps to v0.4.5 with a
changelog entry; the **Satellite dark-theme exception is written into that entry
as a scoped carve-out**, so a future reader finds the reasoning rather than an
apparent contradiction of a Fixed Point.

### Configuration

Two client keys, added to `lib/config.ts` beside `apiUrl` and `agentUrl`:

```ts
maptilerKey: process.env.NEXT_PUBLIC_MAPTILER_KEY ?? null,
esriImageryUrl: process.env.NEXT_PUBLIC_ESRI_IMAGERY_URL ?? null,
```

**These are configuration, not secrets, and must not be treated as secrets.** A
browser-side tile key is transmitted to the client by construction — it is in
the network tab of every user who opens the map, and no amount of repository
hygiene changes that. What actually protects it is the provider-side control:
**an HTTP-referrer/origin allowlist on the MapTiler key**, restricted to the
production domain and Vercel preview origins. Set in the MapTiler console at
provisioning time, documented in `.env.example`, and listed in the runbook.

Both default to `null`, and a `null` key removes its styles from the switcher
(§ style registry) rather than breaking the map — so local dev and CI need no
key at all, and a forgotten Vercel env var degrades to today's behaviour instead
of a blank page.

### Verification

- **`lib/map-styles.ts` unit tests** — every registry entry resolves or returns
  `null` under a keyless config; the dark re-tone transform against a
  checked-in MapTiler style fixture (no network); `DEFAULT_STYLE` fallback when
  a persisted style id is unknown or unresolvable.
- **Overlay-reinstall test — the regression that matters.** Mount `MapView`
  against the existing `maplibre-gl` stub (the pattern in
  `RunRouteMap.test.tsx` and `app/(app)/running/[id]/page.test.tsx`), fire a
  style change, and assert the route source and every overlay layer are
  re-added. This test is the reason the feature does not ship broken; write it
  before the switcher.
- **DEM encoding test** — the `raster-dem` source spec asserts
  `encoding === "terrarium"`. Cheap, and it catches a failure mode that is
  otherwise invisible (§ DEM candidates).
- **Scrub linkage tests** — profile pointer at a known x sets the expected
  index; a null-elevation sample is skipped rather than selected; the
  map→profile direction does not re-enter the profile→map handler;
  remaining-climb suffix sums are correct including at index 0 and the last
  index.
- **Go tests** — grade derivation over a normal climb (positive), a descent
  (negative), a flat section (~0), a stationary sample (NULL, no divide-by-
  zero), a gap in the elevation stream (NULL, no phantom wall), and the first
  `w` samples (NULL). A detail-read round-trip asserting all three new keys are
  present-and-null for an activity whose trackpoints carry neither position nor
  elevation, and present-and-populated for one that carries both — including a
  row written before this change, since read-time projection means there is no
  such thing as a pre-existing row without grade.
- **Manual on-device** — open a real Denver hike on an iPhone-width viewport in
  each of the five styles: contours legible, summit labels present, route
  visible against imagery, switcher tappable, scrub tracking under a thumb, page
  still scrollable past the chart.
- **Quota check** — after a day of real use, read the MapTiler console's session
  and request counters against the projection in § Cost Analysis. A projection
  nobody checks is a guess.

## Cost Analysis

**Today (pre-launch, one user): $0/month.** MapTiler's free tier covers it many
times over, Mapterhorn and OpenFreeMap are free unconditionally, and Esri's
developer terms are satisfied while the app earns nothing.

**Request model.** A vector map view is ~4–8 tile requests; hillshade is free
(Mapterhorn); imagery is free (Esri). So the only metered traffic is the
topographic basemap. MapTiler bills two counters and **sessions bind first**:

| Plan | Sessions | Requests | Implied map opens/month |
| --- | --- | --- | --- |
| Free | 5,000 | 100,000 | ~5,000 (session-bound) |
| Flex — $30/mo | 25,000 | 500,000 | ~25,000 (session-bound) |

At ~6 requests per view, 25,000 sessions consume ~150k of the 500k request
allowance — so the request counter has ~3× headroom and sessions are the number
to watch. 25,000 map opens/month is roughly 800/day; for a hiking-detail surface
in a training app that is a substantial user base, not an early-launch one.

**The cost event is revenue, not traffic.** MapTiler's free tier prohibits
commercial use and Esri's free imagery terms require a non-revenue app. Both
expire the day the product charges anyone — not the day it gets busy. The
migration is a plan upgrade in a console and, if imagery is also switched,
one registry entry. **No code change, no schema change, no redeploy.**

**Counterfactual.** Thunderforest at the same commitment is $125/mo
(4.2×), or $255/mo (8.5×) if offline pre-fetch is ever wanted. Sourcing the DEM
from MapTiler instead of Mapterhorn would roughly double metered requests and
pull the request counter to parity with sessions as the binding limit.

## Risks

| | Risk | Mitigation |
| --- | --- | --- |
| **R1** | **MapTiler's free tier is non-commercial.** Continuing on it past first revenue is a licence breach. | Ship on the free tier pre-launch; the upgrade is a $30/mo billing action with no code change. Recorded as Open Question 1 with a named trigger (first paying user) rather than left implicit. |
| **R2** | **`setStyle()` silently destroys overlays** — the route vanishes on style switch. Highest-probability defect in the SOW. | One idempotent `installOverlays` bound to `styledata`; the reinstall regression test is written before the switcher exists. |
| **R3** | **Terrain-RGB encoding mismatch** renders a wrong-but-plausible hillshade. | `encoding: "terrarium"` asserted in the source spec test; the constant lives in the registry, not at a call site. |
| **R4** | **Contour and label density is unreadable at phone width.** | `minzoom` gating in the re-tone pass, tuned on-device before merge (Open Question 4). |
| **R5** | **Provider outage or key misconfiguration blanks the map.** | `resolve()` → `null` removes a style from the switcher; the map falls back to keyless OpenFreeMap with overlays intact. Degradation is to today's product. |
| **R6** | **Style-switch churn inflates session count** — a user cycling styles may open several sessions. | Sessions are the binding meter; the quota check after real use exists to catch this. If it bites, persist-and-restore (already scoped) plus not remounting the map on switch keeps it to one session per visit. |
| **R7** | **Satellite imagery contradicts the dark-theme Fixed Point.** | Written into the v0.4.5 changelog as an explicit scoped exception with reasoning; chrome and route stay on system tokens over imagery. |
| **R8** | **Scrub feedback loop** between map and profile causes a render storm. | Idempotent setter on equal values; neither direction re-emits on programmatic update; covered by test. |
| **R9** | **Grade noise** makes the readout untrustworthy on a downsampled series. | ≥30 m smoothing window, server-owned so one number exists product-wide; NULL rather than a fabricated value at the boundaries. |
| **R10** | **Esri's free imagery terms are account-scoped**, not anonymous — using the public endpoint without a developer account is out of terms. | Provision the ArcGIS developer account during Phase 3, not at merge; `esriImageryUrl` defaults to `null`, so Satellite is simply absent from the switcher until it is. |

## Phased Implementation Plan and Effort

Phases are sized to fit a single `dispatch-sow.yml` worker run (6-hour cap) and
are independently shippable — each leaves the product better and none depends on
a later one. **Dispatch one phase per run**, per the phase-sizing convention in
[`mobile-feature-parity-and-testflight.md`](mobile-feature-parity-and-testflight.md).

| Phase | Scope | Repos | Effort |
| --- | --- | --- | --- |
| **1 — Terrain foundation** | Style registry with Standard / Topo / Dark; dark re-tone of MapTiler Outdoor; Mapterhorn hillshade; `installOverlays` + `styledata` reinstall; route casing; discipline-hue stroke; endpoints and direction arrows; attribution control; `lib/config.ts` keys; the reinstall regression test. | web, docs | **2–3 days** · 1 dispatch |
| **2 — Linked profile** | API: `latitude`, `longitude`, `grade_percent` on `trackpointDTO` + smoothed grade derivation + tests. Web: `RecapChart` scrub props; shared `scrubIndex` on `/hiking/[id]`; scrub marker; travelled/ahead route split; caption strip with grade, remaining climb, remaining distance; map→profile reverse binding; mileage markers. | api, web | **3–4 days** · 1 dispatch (API and web can be one run; the API PR must merge first) |
| **3 — Imagery** | Esri developer account provisioned (owner, interactive); Satellite style; composed Satellite + Trails style; `canvas: "imagery"` stroke treatment; per-style attribution. | web, docs | **2–3 days** · 1 dispatch |
| **4 — Polish** | Per-user style persistence; on-device contour/label tuning; `design-system.md` v0.4.5 with the `--map-*` tokens and the satellite carve-out; `sow-trail-map.md` OQ2 marked answered; quota check against projection. | web, docs | **1–2 days** · 1 dispatch |

**Total: roughly 8–12 engineering days across 4 dispatches.** Phase 1 alone
closes the gap the Introduction describes — the map stops being a street map —
and is the phase to ship if only one ships.

Phase 3 has an **interactive prerequisite the autonomous worker cannot satisfy**:
the ArcGIS developer account and the MapTiler key/origin-restriction must be
provisioned by the owner in a browser first, exactly as Phase 0 of the mobile
parity SOW handles Apple credentials. Dispatching Phase 3 before that yields a
Satellite entry that correctly resolves to `null` and never appears — working
code, invisible feature.

## Success Metrics

The success criterion given was "comparable to modern outdoor navigation
applications," which is not measurable as stated. These are:

1. **Terrain legibility.** Opening a Quandary Peak hike, a viewer who has never
   seen the route can identify the summit, the drainage, and the direction of
   travel from the map alone, without reading a number. *(Qualitative, judged
   on-device. It is the actual goal; everything below is a proxy.)*
2. **Linkage.** Scrubbing the profile to any point puts the map marker within
   one trackpoint of the true position, and the reverse binding agrees.
   *(Asserted in test.)*
3. **Style integrity.** Cycling all five styles leaves the route, markers, and
   hillshade rendered, in correct z-order, every time. *(Asserted in the
   reinstall test; verified manually across providers.)*
4. **No regression for the no-route case.** Indoor and non-GPS activity pages
   are byte-for-byte unchanged — no map frame, no placeholder, no console error.
5. **Extensibility, measured concretely.** Adding a sixth basemap style is one
   record in `MAP_STYLES` and zero component edits; adding a seventh overlay is
   one entry in the overlay spec and zero lifecycle edits. *(The stated
   architectural goal, stated as a diff size so it can be checked at review.)*
6. **Cost stays at $0 pre-launch**, and one day of real use projects under
   MapTiler's free session ceiling with ≥5× headroom.
7. **Performance.** Time-to-first-route on the hike detail page does not regress
   measurably against today's OpenFreeMap map on a mid-tier phone; the map still
   does not mount at all when there is no route.

## Open Questions

1. **When does the MapTiler plan change, and who decides?** Options: (a) stay on
   the free tier and treat the upgrade as a launch-checklist item; (b) upgrade
   to Flex ($30/mo) at merge to remove the licensing question entirely.
   **Lean: (a)**, with the trigger written down as "first paying user, alongside
   the Esri account's identical trigger." The product is pre-launch and
   single-user; $30/mo now buys nothing but tidiness. The risk is that a launch
   checklist nobody reads is how licence breaches happen — so the trigger goes
   in the launch runbook, not only in this SOW.

2. **Should the route stroke really move off `--accent`?** Options: (a) yes,
   discipline hue, matching the linked chart and the "activity ≠ selection"
   rule; (b) keep `--accent`, on the grounds that the route is the page's
   focus object. **Lean: (a) strongly.** The design system is unambiguous that
   `--accent` is not a series hue, the elevation chart on the same page already
   obeys it, and once the two are one instrument sharing a scrub index, sharing
   a colour is functional rather than cosmetic. Flagged because it is a visible
   change to a shipped surface, not because it is close.

3. **Where does the style preference persist?** Options: (a) `localStorage`,
   client-only; (b) `user_preferences` on the API, syncing across devices.
   **Lean: (a) for Phase 4.** It is a display preference on one surface, it has
   no cross-device consequence today, and (b) is a schema change plus an
   endpoint for something the user re-picks in one tap. Revisit if and when
   mobile gets a map, since that is the first moment sync means anything.

4. **Contour `minzoom` and hillshade exaggeration values.** These are
   perceptual, and near-black neutrals shift warm terrain hues considerably.
   **Lean: tune on-device against a real hike before the Phase 1 PR merges**,
   the same way `hiking-activity-type.md` deferred its clay hex triple to
   eye-check on a real card. Do not guess them from the desktop viewport.

5. **`design-system.md` is still at v0.4.3 and has never recorded the hike
   discipline hue** that shipped in `globals.css` (`--discipline-hike-bg/fg/dot`
   = `#2a201c` / `#c9a690` / `#b08e77`) — the exact drift
   `hiking-activity-type.md` Open Question 4 predicted. Options: (a) fold the
   hike-hue backfill into this SOW's v0.4.5 edit, since it is already opening
   that file; (b) leave it and let the hiking SOW's own follow-up close it.
   **Lean: (a).** A decided-conventions doc that disagrees with the stylesheet
   is worse than no doc, and this SOW cites `--discipline-hike-fg` as the route
   stroke — citing a token the design system does not document would compound
   the drift rather than inherit it. The unrelated lift-dot/accent collision
   that same question raised stays out of scope.

6. **Should `run-detail` default to Topographic too?** Options: (a) Standard for
   runs, Topographic for hikes, per `DEFAULT_STYLE`; (b) Topographic for both.
   **Lean: (a).** Most runs are on streets, where a topo basemap is worse, not
   better — but trail runs are exactly the case where it is better, and the app
   cannot currently distinguish a trail run from a road run. If a trail-run
   signal ever exists (a surface field, or an elevation-gain threshold), this
   becomes a one-line change in `DEFAULT_STYLE`. Until then the user's persisted
   choice covers it.
