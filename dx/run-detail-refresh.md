---
type: dx
status: draft
surface: run-detail-refresh
idioms:
  - session-recap-parity
  - map-hero-garmin
  - strava-overview-split
  - traces-lab-stack
  - ledger-plus-voice
references:
  - Prog Strength Workout Detail
  - Garmin Connect
  - Strava
  - Whoop
  - Runalyze
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Run Detail Refresh

**Status**: Draft · **Last updated**: 2026-07-17

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.
>
> This is a **refinement DX**: `/running/[id]` already shipped
> [`dx/running-activity-detail.md`](./running-activity-detail.md)
> (`splits-ledger-spine`), [`dx/pace-trace.md`](./pace-trace.md)
> (`editorial-area-recap`), heart-rate zones, and the trail map from
> [`sows/sow-trail-map.md`](../sows/sow-trail-map.md). It is **not** a blank-slate
> redo. Variants keep the widgets the owner already likes and explore a better
> **overview + cohesion with workout detail**, plus missing time-series charts
> and a place for qualitative notes.

## Context

The **Running Activity Detail** page (`/running/[id]`) is where a runner lands
after a single outdoor or indoor run — *"how did that run go?"* Today's page is
functionally rich: editable title, outdoor/indoor toggle, an eight-cell metric
band, a MapLibre route map (when geometry exists), a mile splits ledger with
fastest/slowest pills, a hero **Pace recap** area chart, and **Heart rate zones**
ranked bars.

What's unsatisfying is the **opening story** and the **missing continuous
series**, not the existence of splits or pace:

- **There is no overview before the data dive.** The page jumps from title →
  equal metric tiles → map → splits. Lift detail
  ([`dx/workout-detail.md`](./workout-detail.md) → session-recap) opens as a
  *session*: date kicker, title, the athlete's note as prose, then a quiet
  metric strip, then **The Work**. Runs and lifts are both activities; they
  should feel like siblings. The run page still reads like a metrics dashboard
  stapled under a name.
- **There is nowhere to say how it felt.** Lifts have notes. Runs do not on the
  UI (API persistence is tracked separately in
  [`sows/running-session-notes.md`](../sows/running-session-notes.md)). Garmin
  and Strava both put qualitative reflection next to the overview. Without it,
  the page can't carry the human half of "how did it go?"
- **Pace has a recap; heart rate and elevation do not.** Zones answer *where
  time was spent*; they do not answer *when HR climbed on the climb* or *where
  the trail rolled*. Garmin stacks Elevation / Heart Rate (and more) as
  synchronized charts; the owner wants **HR over distance (or time)** and
  **elevation over distance** in the same family as Pace recap — not a return
  to the old undifferentiated Recharts trio.
- **Key statistics tiles are "alright," not loved.** Eight equal cells compete
  with the map and the ledger. The overview should promote a smaller set of
  headline numbers (or a quieter strip) so the map, notes, and splits can
  breathe.

**Keep across every variant (non-negotiable shared inventory):**

| Keep | Why |
| --- | --- |
| **Splits ledger** (per-mile / per-km table, fastest/slowest) | Owner likes it; it's the spine of the current page |
| **Pace recap** (editorial area chart, faster-is-higher, dropout honesty) | Owner likes it |
| **Heart rate zones** (ranked bars / zone scale) | Owner likes it |
| **Route map** when `route` exists; omit when absent | Already shipping; conditional UI |
| **Outdoor / Indoor** control | Existing product behavior |

**Add across every variant (shared additions, not the divergence axis alone):**

- Qualitative **notes** block (mocked — empty, filled, and editing affordance).
- **Heart-rate trace** over distance (or time — pick one x-axis per variant and
  stay consistent within that variant).
- **Elevation trace** over the same x-axis; **omit entirely** when no elevation
  samples (same rule as today — no empty chart frame).

`scope: in-system`: design-system **v0.4.2** (oura-calm) is fixed — soft
near-black ramp, periwinkle `--accent` for app chrome only, **run** discipline
hue (`--discipline-run-*`), Manrope, hairline panels, HR `--zone-1..5` scale,
desaturated success/danger for fastest/slowest. Variants diverge on **layout,
structure, density, and composition** — never palette, accent, or type family.
If the winning direction implies a reusable "activity session-recap" or "synced
metric stack" convention shared with workouts, note it for a **post-selection**
design-system amendment; do **not** invent new tokens inside the mockups.

## The surface

**Route:** `app/(app)/running/[id]/page.tsx` and its `_components/`
(`RunHeaderBand`, `SplitsSpine`, `PaceRecap`, `HeartRateZones`, route map,
environment toggle, calibrate for indoor).

**Comparison route for this DX:** `/design-explore/run-detail-refresh` (flag-gated
via `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE`).

**Data already on the detail GET** (mock with fixtures; do not invent API
fields except `notes` as a UI mock):

```ts
type RunningSession = {
  id: string;
  name: string | null;                 // "W11 D2 - Trail Run"
  start_time: string;                  // RFC3339
  environment: "outdoor" | "indoor";
  distance_meters: number;
  duration_seconds: number;
  avg_pace_sec_per_km: number | null;
  best_pace_sec_per_km: number | null;
  avg_heart_rate_bpm: number | null;
  max_heart_rate_bpm: number | null;
  total_calories: number | null;
  elevation_gain_meters: number | null;
  trackpoints?: RunningTrackpoint[];   // pace, HR, elev per sample
  heart_rate_zones?: HeartRateZones;
  splits?: RunningSplit[];
  strip_summary?: RunningStripSummary;
  route?: {                            // GeoJSON MultiLineString + bounds
    type: "Feature";
    geometry: { type: "MultiLineString"; coordinates: number[][][] };
    properties: { bounds: { min_lat: number; min_lng: number; max_lat: number; max_lng: number } };
  } | null;
  // MOCK ONLY for this DX — persist via running-session-notes SOW later
  notes?: string | null;
};
```

**Representative fixture** (match the owner's screenshots —
`dx/_references/run-detail-refresh/`):

- **Activity:** `W11 D2 - Trail Run`, Wed Jul 15 2026 · 6:46 PM, **1:10:48**,
  **7.0 mi**, outdoor, Frisco loop on the map.
- **Stats:** avg pace `10:06 /mi`, best `8:59 /mi`, avg HR `167`, max HR `184`,
  calories `992`, elev gain `243 m` (Garmin shows ~558 ft — same ballpark;
  use metric elev gain consistently with the app).
- **Splits:** seven miles; Mi 1 fastest (~`9:23`), Mi 6 slowest (~`11:05`);
  elev Δ varies (descents and climbs); HR mid-150s–170s.
- **Pace recap:** fastest sample ~`7:38 /mi`, slowest ~`10:56 /mi`, GPS
  dropouts called out in the subtitle (honest gaps — do not bridge).
- **HR zones:** Threshold-heavy (e.g. ~62% threshold, ~33% VO2max) — keep the
  ranked-bar treatment recognizable.
- **Notes (mock filled state):** something trail-real, e.g. *"Legs heavy on the
  climb after mile 4; backed off on purpose. Cool evening in Frisco — felt
  better once I stopped chasing pace."*
- **Notes empty state:** placeholder *"How did this run feel?"* with a clear
  affordance to add (Strava/Garmin-adjacent, but on-brand).

**States every variant must show** (tabs or stacked examples on the comparison
route):

- Outdoor with route + elevation + HR (the trail-run happy path).
- Outdoor with route but **no elevation** samples → elev trace omitted.
- Indoor / no route → **no map slot** (not a placeholder).
- Notes empty vs notes filled.
- Missing HR → zones + HR trace omit or empty honestly; pace + splits remain.

**Owner reference shots** (checked into this repo for the worker):

| File | What it shows |
| --- | --- |
| `dx/_references/run-detail-refresh/prog-strength-run-detail-top.png` | Current PS run detail — tiles, map, splits |
| `dx/_references/run-detail-refresh/prog-strength-run-detail-charts.png` | Current Pace recap + HR zones |
| `dx/_references/run-detail-refresh/prog-strength-workout-detail.png` | Workout session-recap cohesion target |
| `dx/_references/run-detail-refresh/garmin-run-overview.png` | Garmin overview + map + elev chart |
| `dx/_references/run-detail-refresh/garmin-run-charts.png` | Garmin stacked HR / biometrics charts |
| `dx/_references/run-detail-refresh/strava-run-overview.png` | Strava overview, notes CTA, splits + map |

## Idioms

Each idiom must be distinguishable on **type scale**, **color logic** (how
existing tokens are *applied*, not new hues), and **spacing rhythm**.

- **session-recap-parity** — Make the run page **read like workout detail**:
  uppercase date kicker, large session title, notes as first-class prose under
  the title, then a **quiet inline metric strip** (Distance · Time · Avg pace ·
  Elev — not eight equal tiles), then map, then a section labeled in the same
  register as **The Work** but for miles (splits), then Pace / HR / Elev recaps,
  then zones. **Type:** workout-parity title scale (large name, small kicker).
  **Color:** run-discipline on pace stroke and section cues; zone tokens only
  inside the zones widget; accent reserved for edit/focus chrome. **Spacing:**
  editorial vertical rhythm — generous gaps between overview → map → work →
  traces (Whoop/Athletic calm, not Garmin density).

- **map-hero-garmin** — **Map is the first hero** under a compact header
  (title + one-line meta + notes affordance inline or just below). Headline
  stats sit as a **thin strip on or immediately under the map** (Garmin-like
  overview compression). Splits follow; then a **tight synchronized stack** of
  Pace, HR, Elevation sharing one x-axis. **Type:** smaller title, denser
  labels, more chrome-in-map. **Color:** route in `--accent` or run-dot on the
  basemap; chart fills use run-fg / zone-5 / muted elev green-teal from the run
  ramp — still tokens, applied as chart series. **Spacing:** map-dominant first
  viewport; charts packed with shared gutters (Garmin Connect density).

- **strava-overview-split** — **Two-column overview on wide viewports**: left =
  title, "Add notes" / filled notes, compact 2×3 key stats; right = map.
  Below full-bleed: splits table (primary mid-page), then analysis traces +
  zones. **Type:** medium social-feed hierarchy — title strong, stats
  card-grid but fewer cells than today. **Color:** minimal decoration; run-dot
  only on fastest split / map stroke; zones keep their scale. **Spacing:**
  Strava card gaps — overview band, then clear break into splits, then
  analysis.

- **traces-lab-stack** — The **analytical stack is the hero**: Pace + HR + Elev
  as three equal editorial area charts with shared distance axis and crosshair
  (Garmin charts column energy, Prog Strength recap styling). Overview is
  minimal (title + notes + 4-number strip). Map is secondary (below overview or
  beside traces on xl). Splits stay but sit under the lab. **Type:** tabular /
  tight tracking on axis labels; title not oversized. **Color:** series
  encoding is the story — pace run-fg, HR zone-warm, elev muted teal; no
  competing tile chrome. **Spacing:** dense stacked charts with hairline
  rules; overview compressed.

- **ledger-plus-voice** — **Least disruptive**: keep today's **splits-ledger
  spine** and Pace recap + zones as they are. Insert a new **overview band**
  above the map (notes + quieter stats), add HR and Elev as **sibling recaps**
  to Pace (same editorial-area-recap language), leave map where it is. **Type:**
  current ledger fine tabular type unchanged. **Color:** status pills for
  fastest/slowest only; new traces borrow Pace recap accent treatment.
  **Spacing:** current page rhythm with one inserted overview block — the
  "evolution, not revolution" pole.

## References

- **Prog Strength Workout Detail** (`session-recap`) — date kicker, title, note
  as prose, quiet metric strip, then the structured body of work. The cohesion
  target for activity-detail *voice*.
- **Garmin Connect** — overview compression, map as primary visual, stacked
  metric charts (Elevation, Heart Rate) with a shared time/distance axis.
  Take the *stacking and overview*, not the white theme or gear sidebar.
- **Strava** — "Add private notes" / description next to identity; compact
  key-stats grid; splits + map as a paired mid-page beat. Take the *notes +
  overview pairing*, not the social chrome.
- **Whoop** — calm single-accent-on-charcoal density; avoid chartjunk when
  stacking series.
- **Runalyze** — honest splits ledger and pace-as-data respect (already in the
  shipped spine); keep dropout honesty when adding HR/elev traces.

## Selection criteria

When picking at the gate, weigh:

1. **Does the first screen feel like a session** (story + map or story +
   quiet numbers) before the splits dive — closer to workout detail than to a
   tile dashboard?
2. **Are notes obvious and inviting** without stealing the run's structure?
3. **Do Pace + HR + Elev read as one family** (shared axis honesty, same dropout
   rules) without turning the page into a chart wall?
4. **Do splits stay scannable and loved** — not demoted into noise?
5. **Would a lifter and a runner recognize the same product** if they opened
   `/workouts/[id]` and `/running/[id]` back-to-back?
6. **Indoor / no-route / no-elev** still look intentional — no empty frames.

## Design-system follow-up (after selection, not in mockups)

If the chosen idiom establishes a reusable pattern, amend
[`design-system.md`](../design-system.md) in the **downstream SOW** (or a tiny
docs PR), for example:

- **Activity session-recap** — shared header grammar for lift + run (kicker,
  title, notes, quiet strip).
- **Synced metric stack** — conventions for multi-series run charts (shared
  x-axis, gap handling, series token mapping).

Do not block the DX on those edits; mockups stay on existing tokens.

## Out of scope for this DX

- Implementing or dispatching [`sows/running-session-notes.md`](../sows/running-session-notes.md)
  (API + memory source) — mock the UI only; the production SOW wires persistence.
- Pace-coloring the map polyline (Garmin heatmap) — optional flourish only if a
  variant needs it; not required.
- Mobile layout exploration (web DX only).
- Replacing MapLibre / OpenFreeMap or changing trail-map ingest.
- Merging any `design-explore` code to production.
)
