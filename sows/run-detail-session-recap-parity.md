---
type: sow
status: draft
depends_on:
  - sows/running-session-notes.md
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Run Detail — Session-Recap-Parity

**Status**: Draft · **Last updated**: 2026-07-17

> Frontend SOW. It implements a chosen DX variant and therefore inherits that
> DX's `scope` — here **`in-system`**. The visual foundation (design-system
> **v0.4.2**, oura-calm) is already decided, so this SOW **conforms** to it: no
> re-tone, no new tokens, no accent/type change. The work is in
> `prog-strength-web`; `prog-strength-docs` flips the DX to `selected`, adds the
> **activity session-recap** convention the DX asked to document post-selection,
> and marks this SOW shipped.
>
> **Dependency:** the notes editor writes through the `notes` field shipped by
> [`sows/running-session-notes.md`](./running-session-notes.md) (rollout step 1
> of that SOW — the migration + `PATCH /running/{id}` extension). Only that
> write path is required here; the memory-source half of that SOW can land on
> its own schedule.

## Introduction

The **Running Activity Detail** page (`/running/[id]`) answers *"how did that
run go?"* — and today it answers only the numeric half. The page is
functionally rich (post web PR #114 it renders: editable name + delete in the
header, the eight-cell `RunHeaderBand`, the Outdoor/Indoor toggle with
treadmill calibrate/reset, the `RunRouteMap` when geometry exists, the
`SplitsSpine` with the miles↔intervals toggle and target-pace column, the hero
`PaceRecap`, and `HeartRateZones` ranked bars) — but it opens as a metrics
dashboard stapled under a name: title → eight equal tiles → map → splits. There
is no opening story, nowhere to say how the run *felt*, and only pace gets a
continuous recap — heart rate and elevation have zones and a single gain number
but no trace.

The Design Exploration
[`dx/run-detail-refresh.md`](../dx/run-detail-refresh.md) (PR
Prog-Strength/prog-strength-web#115) explored the surface across **five
`in-system` variants**, all keeping the non-negotiable inventory (splits
ledger, Pace recap, HR zones, conditional route map, Outdoor/Indoor) and all
adding the shared new elements (qualitative notes, HR trace, elevation trace),
diverging only on layout, density, type scale, and spacing rhythm. The owner
selected **`session-recap-parity`** (reference: **Prog Strength Workout
Detail**): make the run page **read like workout detail**, so a lifter and a
runner opening `/workouts/[id]` and `/running/[id]` back-to-back recognize the
same product. The page opens as a *session* — uppercase date kicker, large
session title, the athlete's note as first-class prose, then a **quiet inline
metric strip** (not eight equal tiles) — then map, then **"The Miles"** (the
splits, titled in the same register as workout detail's "The Work"), then Pace
/ Heart rate / Elevation as sibling editorial recaps, then zones. Editorial
vertical rhythm throughout: generous gaps, Whoop/Athletic calm, never Garmin
density.

This SOW reimplements that variant **production-quality** against the real page
and data. Because `scope: in-system` there is no palette, accent, or type
change — it conforms to v0.4.2. The only API surface it consumes beyond
today's detail GET is the `notes` field from `running-session-notes.md`; the
HR and elevation traces are drawn entirely from the **trackpoints already on
the detail response** (`heart_rate_bpm`, `elevation_meters` per sample) — no
new endpoints.

## Proposed Solution

Rebuild the body of `app/(app)/running/[id]/page.tsx` into the session-recap
composition, keeping every existing behavior: rename (optimistic), delete with
confirm, the linked-plan pill + prescription + unlink, the Outdoor/Indoor
toggle with its PR-membership confirm, treadmill badge + calibrate + reset, the
unit-aware refetch on mi↔km toggle, and the auth/not-found/error guards. The
mockup on the DX branch
(`app/design-explore/run-detail-refresh/_variants/SessionRecapParity.tsx` +
`_charts.tsx`) is the **visual spec, not code to promote**.

The page becomes, top to bottom:

1. **Editorial lead** — a tiny wide-tracked uppercase **date kicker** (day ·
   date · start time), the **session title at workout-parity scale** (large,
   ~4xl — still the click-to-edit `EditableName`, restyled up from today's
   `text-lg`), the **treadmill badge** beside it when indoor, then **notes as
   first-class prose**: the saved note rendered at reading size
   (`whitespace-pre-wrap`, generous leading), or — when empty — a dashed
   *"How did this run feel?"* affordance that opens an inline editor. Below,
   the **quiet inline strip**: a `border-y` definition list of
   Distance · Time · Avg pace (· Elev gain · Avg HR when present) — numbers
   support, they don't lead. The eight-cell `RunHeaderBand` is absorbed:
   best pace moves into the Pace recap's subtitle (it's already there as
   *fastest*), max HR into the Heart rate recap's subtitle.
2. **Route map** — the shipped `RunRouteMap`, unchanged, when `route` exists;
   **no map slot at all** when absent (indoor / no GPS).
3. **The Miles** — a section kicker in the workout-detail register over the
   existing `SplitsSpine`, keeping the miles↔intervals toggle (still gated on
   the linked plan's `run_type`) and the target-pace column.
4. **Sibling recaps** — **Pace recap** (kept, restyled into the recap grammar:
   large section title + muted subtitle carrying fastest/slowest/dropouts),
   then **Heart rate** (avg · max bpm in the subtitle) and **Elevation** (gain
   in the subtitle) as new editorial area charts **in the same visual language**
   — shared distance axis, area-under-curve at low opacity, and the same
   dropout honesty: a `null` sample window is a gap, never a bridged diagonal.
   HR recap is omitted when the run has no HR; Elevation is omitted entirely
   when there are no elevation samples (no empty chart frame).
5. **Time in heart-rate zones** — a section kicker over the existing
   `HeartRateZones` ranked bars, unchanged inside.

**Series color logic** (the DX's decided token application): the pace stroke
and section cues carry the **run discipline hue** (`--discipline-run-dot`),
the HR trace uses `--zone-5`, the elevation trace `--discipline-run-fg`; zone
tokens stay inside the zones widget; **`--accent` is reserved for edit/focus
chrome** (the notes affordance, name editing, links). This moves `PaceRecap`'s
stroke off `--accent` — the one deliberate re-application of existing tokens
the variant decided.

The real work beyond restyling is two seams: the **notes editor** (read/write
`notes` through the client, optimistic with rollback, matching the rename
idiom) and the **trace derivation** (trackpoints → HR/elevation chart series
with honest gaps, alongside the existing `buildPaceStrip` — the page's
render-only doctrine holds: server-derived blocks render verbatim, the only
client mapping is trackpoints → chart coordinates).

## Goals and Non-Goals

### Goals

- **Wire notes end-to-end in the web client** (the "downstream presentation
  SOW" that `running-session-notes.md` defers to):
  - `lib/api.ts`: `RunningSession` gains `notes: string | null` (detail-only,
    like trackpoints) and an update call sends `notes` through
    `PATCH /running/{id}` (alongside the existing rename; one partial-update
    body per that SOW's contract, ≤ 2000 chars, empty clears).
  - The editorial lead renders the filled note as prose; the empty state is the
    dashed *"How did this run feel?"* affordance; editing is inline (textarea
    in place, save on blur / ⌘-Enter, Escape reverts — the `EditableName`
    idiom at paragraph scale), optimistic with rollback + toast on failure.
- **Add Heart rate and Elevation recaps** as siblings of Pace:
  - A client mapping from `trackpoints` to HR and elevation series over
    distance (same x-axis family as `buildPaceStrip`), in `lib/` and pure:
    `null` samples produce gaps, never interpolation; fewer than two plottable
    samples ⇒ the recap is **omitted entirely** (no empty frame — the
    elevation rule today's page applies to the map slot).
  - Chart rendering shares `PaceRecap`'s editorial SVG language — extract its
    geometry (nice ticks, segment-flushing gap walk, area+line+extreme dot)
    into a shared internal rather than forking 300 lines twice; pace keeps its
    two load-bearing conventions (faster-is-higher inversion + dropout bands),
    HR and elevation plot uninverted.
  - Subtitles own the demoted header numbers: Pace — fastest · slowest ·
    dropouts (server `strip_summary`, as today); Heart rate — avg · max bpm;
    Elevation — gain.
- **Rebuild the page composition** into the editorial lead → map → The Miles →
  recaps → zones order described above, at the variant's rhythm (single
  editorial column, large vertical gaps between beats), conforming to v0.4.2 —
  and **preserve every existing behavior**: rename, delete + confirm modal,
  plan pill/prescription/unlink, Outdoor/Indoor confirm flow, treadmill badge
  + calibrate + reset, unit-toggle refetch, auth redirect, not-found/error
  states.
- **Apply the decided series tokens** — pace stroke + section cues
  `--discipline-run-dot`, HR trace `--zone-5`, elevation
  `--discipline-run-fg`, `--accent` only as edit/focus chrome. Tokens only; no
  raw hex, no new tokens.
- **Handle every state the DX enumerated**: outdoor with route + elevation +
  HR; outdoor with route but no elevation (trace omitted); indoor / no route
  (no map slot); notes empty vs filled; missing HR (HR recap and zones omitted
  honestly — pace and splits remain).
- **Tests** — notes editor (empty → add → rendered as prose; edit; 2000-char
  cap surfaced; optimistic rollback on failure; PATCH body sends only `notes`);
  trace mapping (gaps not bridged, omit-below-two-samples, distance axis);
  page states (indoor hides the map slot, no-HR hides the HR recap + zones,
  no-elevation hides the elevation recap — no empty frames); existing
  behaviors unregressed (rename, delete, environment confirm, calibrate/reset,
  unlink, unit refetch); `PaceRecap`'s inversion + dropout tests carried
  through the restyle. **CI green** (lint/format/typecheck/test/build).
- **Docs** (`prog-strength-docs`): flip `dx/run-detail-refresh.md` to
  `status: selected` (winning idiom `session-recap-parity`); add the
  **activity session-recap** convention to `design-system.md` (the shared
  header grammar for lift + run detail: kicker → title → notes as prose →
  quiet strip — the amendment the DX explicitly deferred to this SOW); mark
  this SOW `shipped` on merge.

### Non-Goals

- **Any backend / API / SDK change.** The `notes` column and PATCH extension
  ship in `running-session-notes.md` (`prog-strength-api`); this SOW only
  consumes them. The HR/elevation traces need nothing new — trackpoints
  already carry both. `prog-strength-api` and `prog-strength-sdk` are not in
  `repos:`.
- **Any design-system re-tone or token change.** `scope: in-system` against
  v0.4.2 — the `design-system.md` edit is additive documentation of the
  session-recap grammar, not a token or palette change.
- **The other four variants** (`map-hero-garmin`, `strava-overview-split`,
  `traces-lab-stack`, `ledger-plus-voice`) are not built. DX PR #115 is
  **closed, never merged**; the `design-explore/run-detail-refresh` route stays
  flag-gated and ships no production code — the mockup is a visual spec only.
- **Map changes.** `RunRouteMap`, MapLibre/OpenFreeMap, and route ingest are
  untouched; no pace-colored polyline (the DX marked it an optional flourish;
  this variant doesn't use it).
- **Workout detail.** `/workouts/[id]` is the cohesion *reference*, not a work
  item; it is unchanged.
- **Mobile.** Web only, per the DX; the mobile parity track picks this up
  separately if desired.
- **Re-deriving server blocks client-side.** Splits, intervals,
  strip-summary, zones, and best pace stay server-derived and rendered
  verbatim; the only client mapping remains trackpoints → chart coordinates.

## Implementation Details

### Notes seam (`lib/api.ts` + editorial lead)

`RunningSession` gains `notes: string | null` (detail GET only — the list
response omits it, mirroring trackpoints). The existing `renameRunningSession`
call generalizes to (or is joined by) an update helper posting the partial
`PATCH /running/{id}` body with `notes`; the response is the updated detail
DTO, spliced into state the way rename does today (preserving `trackpoints`,
which the PATCH summary omits). The editor follows the established inline-edit
idiom: click the prose (or the empty-state affordance) → textarea in place →
blur / ⌘-Enter commits, Escape reverts → optimistic update with rollback +
toast on error. Cap mirrors the API's 2000 chars.

### Trace derivation (`lib/`)

A pure module alongside `running-splits.ts` mapping `trackpoints` to plottable
HR and elevation series over the response's `unit` distance axis: each sample
→ `{ distanceUnit, value | null }`, `null` where the sample lacks the metric.
No smoothing, no interpolation — the chart walk flushes segments at every
`null` exactly as `PaceRecap` does for GPS dropouts. A series with fewer than
two non-null points reports "no data" and its recap is not rendered.

### Recap chart language (`app/(app)/running/[id]/_components/`)

Extract `PaceRecap`'s SVG internals (tick math, gap-flushing path builder,
area/line/extreme-dot rendering, axis type) into a shared recap-chart internal
parameterized by series color, y-inversion, and value formatter. `PaceRecap`
becomes the pace-configured instance (inverted, `--discipline-run-dot`,
`formatPaceClock`, dropout bands + server `strip_summary` header); new
`HeartRateRecap` (`--zone-5`, bpm) and `ElevationRecap`
(`--discipline-run-fg`, meters) are thin siblings. Each recap renders under
the variant's grammar: section title + muted subtitle, then the chart.

### Page rebuild (`app/(app)/running/[id]/page.tsx`)

Keep the data layer and handlers verbatim (fetches, optimistic rename, delete,
environment confirm, calibrate, unlink, unit refetch, guards). Replace the
render: the compact header row (back link + delete) stays as slim utility
chrome; the body becomes the editorial column — kicker, 4xl `EditableName`,
notes block, quiet `border-y` strip (absorbing `RunHeaderBand`, which is
deleted with its tests migrated), the environment/calibrate/plan row folded in
beside or beneath the strip as quiet chrome, then map → The Miles →
recaps → zones with the variant's generous gaps. `SplitsSpine`,
`RunRouteMap`, and `HeartRateZones` are reused as-is inside the new beats;
section kickers use the shared uppercase-tracked register.

### Docs (`prog-strength-docs`)

- `dx/run-detail-refresh.md` → `status: selected`, winning idiom noted.
- `design-system.md` — add the **activity session-recap** grammar (kicker /
  title / notes-as-prose / quiet strip; section kickers for the body beats)
  as the shared convention for activity detail surfaces, citing workout detail
  and this page as the two instances.
- This SOW → `shipped` on merge.

## Rollout

1. **`prog-strength-api`** (outside this SOW) —
   [`running-session-notes.md`](./running-session-notes.md) rollout step 1
   (migration + `PATCH` notes) lands first; this SOW's web PR is blocked on it.
   Its memory-source half is independent and not a blocker.
2. **`prog-strength-web`** — one PR: notes seam + trace derivation + recap
   extraction + page rebuild + tests. Verify on the Vercel preview against the
   DX's state matrix: the trail-run happy path, outdoor-no-elevation, indoor
   (no map slot), notes empty vs filled, missing HR, both mi and km units,
   and mobile/desktop breakpoints.
3. **`prog-strength-docs`** — flip the DX to `selected`, add the
   session-recap amendment, mark this SOW shipped; **close DX PR #115**
   (never merge).

### Verification after rollout

- `/running/[id]` opens as a **session**: kicker → large title → the note as
  prose (or an inviting empty affordance) → a quiet metric strip — not eight
  equal tiles; opened back-to-back with `/workouts/[id]`, the two pages read
  as siblings.
- A note written on the page survives reload (real PATCH persistence) and
  respects the cap; a failed save rolls back with a toast.
- Pace, Heart rate, and Elevation read as **one chart family** over a shared
  distance axis; GPS/HR dropouts stay gaps; the pace chart still reads
  faster-is-higher with its fastest/slowest/dropout subtitle intact.
- The strip and section cues carry the run hue; `--accent` appears only as
  edit/focus chrome; zones keep the `--zone-1..5` scale. No new tokens, no
  raw hex.
- Indoor runs show no map slot and keep calibrate/reset; a no-HR run shows no
  HR recap and no zones but full pace + splits; a no-elevation run shows no
  elevation recap — nothing renders an empty frame.
- Rename, delete, plan unlink, environment confirm, and the mi↔km refetch all
  still work; CI is green; the `design-explore` route still 404s in
  production; DX PR #115 is closed unmerged.

## Resolved Questions

Owner-approved decisions (2026-07-18); these bind the implementation:

1. **Operational chrome placement.** The variant's mock omits the environment
   toggle, calibrate/reset, and the plan pill (fixtures, not product); all
   survive. The plan-completion pill + prescription line sit directly under
   the quiet strip (they are context, like the strip), and the Outdoor/Indoor
   toggle + calibrate/reset row stays a quiet line below that — same order as
   today, restyled to the editorial register.
2. **Calories stays.** The variant's strip drops `Calories`; silently losing
   a number the page shows today is a regression, so include Calories in the
   quiet strip when non-null — one `dl` entry, keeping the strip honest to
   "what the watch recorded."
3. **Column width: `max-w-2xl`.** Adopt the variant's `max-w-2xl` for the
   editorial read; verify the intervals view of `SplitsSpine` (the widest
   content, with the target-pace column) at that width on the preview, and
   fall back to `3xl` only if the table genuinely can't breathe.
4. **HR/elevation x-axis: distance.** Distance for all three recaps — one
   shared axis family is the point. Time-basis only becomes interesting for a
   future workout-detail HR alignment, out of scope here.
5. **Pace stroke re-tokens to the run hue.** Moving `PaceRecap` off
   `--accent` onto `--discipline-run-dot` is decided — accent-as-chart-series
   is exactly what the design system's "activity ≠ selection" rule exists to
   prevent. The implementation PR description must call the change out so the
   restyle of a shipped hero is visibly intentional.
