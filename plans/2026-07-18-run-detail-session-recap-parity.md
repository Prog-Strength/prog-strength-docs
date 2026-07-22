# Plan: Run Detail — Session-Recap-Parity

Implements [`sows/run-detail-session-recap-parity.md`](../sows/run-detail-session-recap-parity.md).

Rebuild `/running/[id]` in `prog-strength-web` into the **session-recap-parity**
editorial composition (reads like workout detail): date kicker → large session
title → notes as prose → quiet metric strip → map → The Miles → Pace/HR/Elevation
sibling recaps → HR zones. Wire the `notes` field end-to-end, add HR + elevation
traces, and re-token the pace stroke onto the run hue. `scope: in-system` against
design-system v0.4.2 — tokens only, no new tokens, no re-tone. Docs flip the DX to
`selected` and add the activity session-recap grammar.

All web changes land in **one PR**. Docs land in the `prog-strength-docs` PR.

## Constraints held throughout

- **Render-only doctrine**: splits, intervals, strip-summary, best-pace, and zones
  stay server-derived and rendered verbatim. The only client mapping remains
  trackpoints → chart coordinates.
- **Preserve every behavior**: rename (optimistic), delete + confirm modal, plan
  pill / prescription / unlink, Outdoor/Indoor confirm flow, treadmill badge +
  calibrate + reset, unit-toggle refetch, auth redirect, not-found/error states.
- **Tokens only** (v0.4.2). Series colors: pace stroke + section cues
  `--discipline-run-dot`; HR trace `--zone-5`; elevation `--discipline-run-fg`;
  zone tokens stay inside the zones widget; **`--accent` only as edit/focus chrome**
  (notes affordance, name edit, links). No raw hex, no new tokens.
- **No backend / SDK change.** Web consumes the `notes` field + PATCH extension
  shipped by `running-session-notes.md`. HR/elevation come from trackpoints already
  on the detail response.

## Task 1 — Notes API seam (`lib/api.ts`)

- `RunningSession` gains `notes: string | null` (detail-only; comment it like
  `trackpoints`/`route` — present on the per-id GET, omitted from lists).
- Add `updateRunningSessionNotes(token, id, notes: string): Promise<RunningSession>`
  — `PATCH /activities/{id}` with body `{ notes }`, returns the updated detail DTO.
  Mirror `renameRunningSession` exactly (same headers, same `unwrap` + null guard).
  Empty string clears the note (server contract).
- Tests in `lib/api.test.ts`: PATCH hits `/activities/{id}`, method PATCH, body is
  exactly `{ notes }` (only `notes`, no `name`); returns the parsed session; throws
  on a null/absent body. Follow the existing `renameRunningSession` test.

## Task 2 — Trace derivation (`lib/running-traces.ts`, pure)

New pure module alongside `running-splits.ts`.

- `export type MetricStripPoint = { distanceUnit: number; value: number | null }`.
- `buildHeartRateStrip(trackpoints, unit): MetricStripPoint[]` — `value =
  heart_rate_bpm` (raw bpm; unit-independent), `null` where the sample lacks HR.
- `buildElevationStrip(trackpoints, unit): MetricStripPoint[]` — `value =
  elevation_meters`, `null` where absent.
- `distanceUnit = distance_meters / bucketMeters` — the exact same x-axis family as
  `buildPaceStrip` (reuse the meters-per-unit constants; factor a shared helper if
  clean, else mirror). **No smoothing, no interpolation** — a `null` sample stays
  `null` so the chart walk flushes a gap.
- `export function hasPlottableSeries(points: MetricStripPoint[]): boolean` — true
  iff at least two points have non-null `value`. Recaps use this to omit entirely.
- Tests: null samples pass through as gaps (not bridged/interpolated); a series with
  < 2 non-null points → `hasPlottableSeries` false; `distanceUnit` matches the
  buildPaceStrip mapping for the same trackpoints in both `mi` and `km`.

## Task 3 — Recap chart extraction + HR/Elevation recaps (`_components/`)

Extract `PaceRecap`'s SVG geometry into a shared internal so pace/HR/elevation are
one chart family; do **not** fork it three times.

- New `_components/RecapChart.tsx` — the parameterized internal. Inputs: chart
  points (`{ distanceUnit, value|null }`), series `color` (a token var string),
  `inverted: boolean` (pace = true → faster-is-higher; HR/elev = false), a
  `formatValue(v: number) => string` for y labels + annotation, the axis `unit`
  suffix, an optional `showDropoutBands` + dropout window derivation, and the
  extreme-dot annotation config (which extreme to mark + its label). Owns: nice
  x-tick step (`niceStep`), y-domain padding, the **gap-flushing** segment/area path
  builder (flush at every `null` — never bridge), area+line render, y/x tick labels,
  extreme dot. Gradient id must be unique per instance (pass an `id` prefix) so
  multiple charts on one page don't collide on `<defs>` ids.
- `PaceRecap.tsx` becomes the pace-configured instance: `inverted`,
  `color="var(--discipline-run-dot)"`, `formatPaceClock`, dropout bands on, header
  numbers from server `strip_summary` (fastest/slowest/dropouts) with the plotted
  fallback, the "Faster is higher" pill, the "No pace data" card for < 2 points.
  **Re-token off `--accent` → `--discipline-run-dot`** for stroke, area gradient, and
  the fastest dot. `--accent` no longer appears in this component.
- `HeartRateRecap.tsx` — thin sibling: `color="var(--zone-5)"`, not inverted, value
  formatter `${bpm}` (integer). Subtitle: `Avg {avg} · max {max} bpm` from the
  session's `avg_heart_rate_bpm`/`max_heart_rate_bpm`. Marks the **max** sample dot.
  Omit-below-two → render nothing (page gates on `hasPlottableSeries`; component may
  also guard defensively). No dropout bands.
- `ElevationRecap.tsx` — sibling: `color="var(--discipline-run-fg)"`, not inverted,
  value formatter `${m.toFixed(0)} m`. Subtitle: `{gain} m gain` from
  `elevation_gain_meters`. No dropout bands.
- Each recap renders under the recap grammar: a large section title + muted subtitle,
  then the chart card (reuse the existing card chrome).
- Tests: update `PaceRecap.test.tsx` — the stroke-path selector and gradient/dot
  color assertions move from `var(--accent)` to `var(--discipline-run-dot)`; the
  inversion + dropout-not-bridged + server-summary-header + no-pace-data tests are
  carried through unchanged in intent. Add `HeartRateRecap.test.tsx` /
  `ElevationRecap.test.tsx`: uninverted (higher value → smaller y at top? no — for
  uninverted, larger value → higher on screen → smaller y), gap not bridged,
  subtitle carries avg/max (HR) and gain (elev).

## Task 4 — Page rebuild (`page.tsx`) + notes editor + section kickers

Keep the entire data layer + every handler **verbatim** (fetches, optimistic rename,
delete, environment confirm, calibrate, reset, unlink, unit refetch, guards). Add a
notes handler and replace the render.

- **Notes handler** `handleSaveNotes(next: string)`: optimistic splice
  (`setSession({ ...session, notes })`), call `updateRunningSessionNotes`, splice the
  returned DTO preserving `trackpoints` (like rename), roll back + toast on error,
  auth-error passthrough. No-op when unchanged.
- **`_components/NotesEditor.tsx`** — inline prose editor (the `EditableName` idiom at
  paragraph scale). Filled note renders as prose (`whitespace-pre-wrap`, generous
  leading). Empty → a dashed *"How did this run feel?"* affordance
  (`--accent`/`--accent-line` edit chrome). Click → `<textarea>` in place, `maxLength`
  2000; commit on blur / ⌘-Enter, revert on Escape. Persistence is the page's
  `onSave` prop. Component test covers empty→edit UI, ⌘-Enter/Escape, cap.
- **Composition** (single editorial column, `max-w-2xl`, generous vertical gaps):
  1. Slim utility header row: back link + Delete (keep as-is).
  2. **Date kicker** — tiny uppercase wide-tracked metadata (`--faint`/`--muted`):
     weekday · date · start time (derive from `start_time`).
  3. **Session title** — `EditableName` at ~`text-4xl` (add a size/class option to the
     local `EditableName`; keep click-to-edit + optimistic rename). Treadmill badge
     beside it when indoor.
  4. **Notes block** — `NotesEditor`.
  5. **Quiet strip** — a `border-y` definition list (`dl`) absorbing `RunHeaderBand`:
     Distance · Time · Avg pace, plus Elev gain · Avg HR · Calories **when non-null**
     (Calories stays per Resolved Q2). Numbers support, not lead. Delete
     `RunHeaderBand.tsx` (no test file to migrate).
  6. **Plan context** — plan-completion ✓ pill + prescription line + Unlink, directly
     under the strip (Resolved Q1), restyled to the run-hue register.
  7. **Operational chrome** — Outdoor/Indoor toggle + calibrate/reset as a quiet line
     below (Resolved Q1), same order as today.
  8. **Route map** — `RunRouteMap` when `route` exists; **no slot at all** when absent.
  9. **The Miles** — a section kicker (uppercase tracked, `--discipline-run-dot`) over
     the existing `SplitsSpine` (miles↔intervals toggle + target-pace column intact).
  10. **Recaps** — `PaceRecap`, then `HeartRateRecap` **only when**
      `hasPlottableSeries(buildHeartRateStrip(...))`, then `ElevationRecap` **only
      when** `hasPlottableSeries(buildElevationStrip(...))`. No empty frames.
  11. **Time in heart-rate zones** — a section kicker over the existing
      `HeartRateZones` (unchanged inside; it self-hides when no HR).
- **`_components/SectionKicker.tsx`** (or inline) — uppercase, wide-tracked,
  `--discipline-run-dot`. Used for The Miles + zones.
- **Verify Resolved Questions**: Q1 chrome placement, Q2 calories kept, Q3
  `max-w-2xl`, Q4 distance x-axis, Q5 pace re-tokened (call out in PR body).
- Tests (`page.test.tsx`, extend): notes empty→add renders prose; optimistic rollback
  + toast on failed save; PATCH body sends only `notes`; indoor hides map slot; no-HR
  hides HR recap + zones (pace + splits remain); no-elevation hides elevation recap;
  existing behaviors unregressed (rename, delete, environment confirm, calibrate/reset,
  unlink, unit refetch).

## Task 5 — Docs (`prog-strength-docs`)

- `dx/run-detail-refresh.md` → `status: selected`; note the winning idiom
  `session-recap-parity`; update the Status header line + Last updated 2026-07-18.
- `design-system.md` → add an **activity session-recap** grammar under Decided
  (kicker → title → notes-as-prose → quiet strip; section kickers for body beats) as
  the shared convention for activity detail surfaces, citing workout detail + this
  run page as the two instances. Additive documentation only — no token change.
  Bump changelog (v0.4.3, 2026-07-18) and Last updated. Provenance entry pointing to
  this SOW.
- `sows/run-detail-session-recap-parity.md` → `status: shipped`, header Status
  `Shipped`, Last updated 2026-07-18 (done in workflow step 4).

## Gate (before any push)

`npm run lint && npm run format:check && npm run typecheck && npm run test && npm run
build` all green in `prog-strength-web`. Fix code, never bypass hooks or silence rules.

## Rollout / PR order

1. `prog-strength-web` — the single feature PR (blocked in prod on the API's `notes`
   PATCH from `running-session-notes.md`, but the web code is independently mergeable
   and CI-green; notes are simply inert until the API deploys).
2. `prog-strength-docs` — DX selected + design-system amendment + SOW shipped.
