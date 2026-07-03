---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-mcp
  - prog-strength-agent
  - prog-strength-docs
---

# Treadmill Run Calibration

**Status**: Shipped · **Last updated**: 2026-07-03

> Cross-repo SOW. Adds indoor/treadmill tagging, post-ingest distance calibration,
> and a uniform-scale recompute of pace and trackpoints so the running detail page
> stays internally consistent. Treadmill runs are excluded from best-efforts/PRs.
> Mobile is deferred; per-activity-type detail-table normalization is a separate
> future architecture SOW.

## Introduction

When a Prog Strength user runs on a treadmill, they log the activity on their
Garmin watch and export it as a TCX file. The watch records distance from its
footpod, not from GPS — and that footpod distance is often wrong. The user
calibrates the distance on the watch before saving (to match the treadmill
console), but **that correction lives only on the watch**. The exported TCX still
carries the pre-calibration footpod distance, so Prog Strength ingests the wrong
number.

Today there is no way to fix this after import. The only mutation path on an
activity is rename (`PATCH /activities/{id}` with `{ name }`). The "workaround"
— delete and re-upload — does not help because the TCX file itself still has the
wrong distance. The running detail page (`/running/[id]`) renders distance and
pace from the API summary fields, but splits, the pace strip, and interval
detection are **derived client-side from the trackpoint series** — so any
calibration that updates only the top-level distance without rescaling
trackpoints would make the header and the splits table disagree.

Separately, treadmill runs are not distinguished from outdoor GPS runs anywhere
in the product. Both render identically (same discipline color, no badge), even
though they carry fundamentally different data (footpod distance + HR, no route,
no elevation) and different trust properties (distance is user-asserted after
calibration).

This SOW closes both gaps: **tag indoor runs**, **let the user calibrate
distance on tagged runs**, and **recompute every distance-derived field in one
transaction** so the detail page, list views, and dashboard mileage stay
internally consistent. Treadmill runs remain in weekly mileage and streak counts
but are **excluded from best-efforts, PRs, and max-effort estimates** — those
surfaces stay "real outdoor efforts only."

## Proposed Solution

Add two general columns to the existing `activities` table — `environment`
(`outdoor` | `indoor`) and `raw_distance_meters` (the distance as originally
ingested, never mutated by calibration) — plus a calibration endpoint and an
extended environment override on the existing PATCH.

**At ingest**, extend the TCX parser to detect whether any trackpoint carries a
`<Position>` (lat/lon). For `running` activities: no position anywhere → default
`environment = indoor`; otherwise `outdoor`. Validated against the owner's real
Garmin exports: 21 of 33 running TCX files in `prog-strength-data/running/` have
zero `<Position>` data (treadmill/indoor profile); outdoor runs (trail, long,
PR attempts) carry position and altitude. The user can override the tag via
PATCH at any time.

**Calibration** (`POST /activities/{id}/calibrate` with `{ distance_meters }`)
is permitted on **indoor runs only**. The server applies a uniform scale factor
`f = new_distance / current_distance` in one DB transaction: update summary
distance and paces, rescale every stored trackpoint's cumulative distance and
per-segment pace. Duration, HR, calories, elevation, and HR zones are
unchanged. The original ingest distance is always recoverable via
`raw_distance_meters` and the archived S3 TCX.

**Best-effort maintenance** follows `environment`:
- ingest indoor → skip best-effort computation (no rows written);
- outdoor → indoor → delete this activity's `activity_best_efforts` rows;
- indoor → outdoor → regenerate best-efforts by re-parsing the archived S3 TCX,
  applying the current effective scale factor (`distance_meters /
  raw_distance_meters`), and sweeping at full trackpoint resolution (matching
  ingest fidelity).

The web client adds a treadmill badge, a calibrate-distance modal on the run
detail page, and an environment override. MCP adds thin tools so the agent can
tag and calibrate runs in chat.

## Goals and Non-Goals

### Goals

- Add migration `036_*` (next free number at implementation time) extending
  `activities` with `environment TEXT NOT NULL DEFAULT 'outdoor' CHECK
  (environment IN ('outdoor','indoor'))` and `raw_distance_meters REAL NOT
  NULL`, backfilling `raw_distance_meters` from current `distance_meters` for
  all existing rows.
- Extend the TCX parser (`internal/activity/tcx_parser.go`) to record whether
  any trackpoint has a `<Position>` element. Do not store lat/lon — only a
  boolean `has_position` used at ingest for environment defaulting.
- At ingest (`IngestTCX` / `summarize`): set `raw_distance_meters =
  distance_meters`; for `activity_type = running`, set `environment = indoor`
  when `has_position` is false, else `outdoor`. Skip `bestEfforts()` sweep for
  indoor runs.
- Add `POST /activities/{id}/calibrate` — body `{ distance_meters: number }`,
  auth required, indoor-only (400 if outdoor). Returns the full activity detail
  including rescaled `trackpoints`. Recompute in one transaction per §
  Algorithms.
- Extend `PATCH /activities/{id}` to accept `{ name?, environment? }` (either or
  both). Environment change triggers best-effort maintenance per § Write Path.
- Expose `environment` and `raw_distance_meters` on activity GET/list DTOs.
- Web (`prog-strength-web`): add `environment` and `raw_distance_meters` to
  `RunningSession` in `lib/api.ts`; add `calibrateRunningSession` and extend
  `renameRunningSession` (or add `patchRunningSession`) for environment;
  treadmill badge + calibrate modal + environment toggle on run detail; light
  treadmill indicator on activities run-list row.
- MCP (`prog-strength-mcp`): add `calibrate_run_distance` and
  `set_run_environment` tools; expose `environment` and `raw_distance_meters` on
  run-read tools.
- Agent (`prog-strength-agent`): document the new tools in the agent's tool
  guidance so chat can tag and calibrate treadmill runs.
- Tests per repo: calibration math, indoor-only guard, environment toggle +
  best-effort side effects, web modal + header/splits consistency with mocked
  rescaled trackpoints.

### Non-Goals

- **Mobile** (`prog-strength-mobile`). Follow-up once web + API settle.
- **Per-activity-type detail tables.** The `activities` table already carries
  running-specific nullable columns (`avg_pace_sec_per_km`, etc.); splitting
  those into `running_details` / `cycling_details` is a separate architecture
  SOW. The two new columns (`environment`, `raw_distance_meters`) are general
  cross-cutting attributes that belong on the base table even after that split.
- **Automated backfill** to re-tag existing runs. The owner will manually tag and
  calibrate existing treadmill runs. No one-time S3 re-parse job in this SOW.
- **Route map / GPS storage.** Position detection is boolean-only at ingest; no
  lat/lon columns. Route geometry remains
  [`sows/run-route-geometry-capture.md`](run-route-geometry-capture.md).
- **Editable duration, HR, calories, or elevation.** Calibration adjusts distance
  and distance-derived fields only.
- **Including treadmill runs in best-efforts / PRs / max-effort estimates.**
  Indoor runs are excluded by design.
- **Re-publishing or retracting timeline "new PR" posts** when a run is retagged
  indoor. Existing posts may become stale; matches today's behavior (no update
  path for timeline posts).
- **Cadence, grade-adjusted pace, lap entities.** Not in scope; unchanged.
- **Calibrating outdoor runs directly.** User must tag indoor first, then
  calibrate.
- **Batch calibration** across multiple runs.

## Implementation Details

### Data Model

One forward migration on `activities` (next number: `036_*` at implementation
time; current head is `035_multi_source_memory.sql`).

| Column | Type | Description |
| --- | --- | --- |
| `environment` | TEXT NOT NULL DEFAULT `'outdoor'` | `CHECK (environment IN ('outdoor','indoor'))`. Whether the activity was recorded without GPS (indoor/treadmill) or with GPS (outdoor). General — applies to any distance-based activity type, not running-only. |
| `raw_distance_meters` | REAL NOT NULL | Distance as originally ingested. Set at ingest equal to `distance_meters`; never updated by calibration. Enables provenance ("3.10 → 3.00 mi") and reset-to-original without an S3 read. |

Backfill: `UPDATE activities SET raw_distance_meters = distance_meters WHERE
raw_distance_meters IS NULL` (or set in the same `ALTER` default for SQLite).

Go model (`internal/activity/model.go`): add `Environment` enum
(`EnvironmentOutdoor`, `EnvironmentIndoor`) and `RawDistanceMeters float64` to
`Activity`. Web/API DTO: expose as `environment: "outdoor" | "indoor"` and
`raw_distance_meters: number`.

UI copy: for `activity_type = running` + `environment = indoor`, display label
**"Treadmill"** (not "Indoor") — the column is general, the label is contextual.

### Write Path

**Ingest (`POST /activities/tcx`)** — unchanged flow except:
- Parser sets `parsed.HasPosition bool`.
- `summarize` sets `raw_distance_meters = distance_meters`.
- Running + `!has_position` → `environment = indoor`; else `outdoor`.
- If `environment = indoor`, skip `bestEfforts()` (no `activity_best_efforts`
  rows for this activity).

**Calibrate (`POST /activities/{id}/calibrate`)** — body `{ distance_meters }`:
- Auth: existing `auth.RequireUser`; activity must belong to caller.
- Guard: `environment = indoor` (400 `outdoor_run_not_calibratable` or similar
  slug if outdoor).
- Validate: `distance_meters > 0`; reject absurd scale factors (e.g. `f` outside
  `[0.5, 2.0]` — exact bounds at implementer's discretion, document in handler).
- One transaction (see § Algorithms): update `activities` summary fields +
  all `activity_trackpoints` for this activity.
- Return: full activity GET shape including `trackpoints` (same as detail GET).
- Idempotent-safe: re-calibration applies factor relative to **current**
  `distance_meters`. Reset = calibrate back to `raw_distance_meters`.

**Environment override (`PATCH /activities/{id}`)** — extend existing rename
handler to accept optional `{ name?, environment? }`:
- `outdoor → indoor`: delete all `activity_best_efforts` rows for this activity.
  Keep current `distance_meters` (including any prior calibration).
- `indoor → outdoor`: regenerate best-efforts — fetch archived TCX from S3,
  parse raw trackpoints, multiply cumulative distances by effective factor
  `distance_meters / raw_distance_meters`, run `bestEfforts()` sweep, insert
  rows. This is the **only** path that touches S3 at calibration/tag time.
- No-op if environment unchanged.

**Unchanged by any write above:** `duration_seconds`, HR fields, calories,
`elevation_gain_meters`, HR zones (computed on read from elapsed + HR).

### Algorithms

**Uniform scale calibration (Approach A — recommended and specified).**

Treadmill footpod error is a constant ratio (belt speed is steady; the footpod
miscounts by a fixed factor). Given corrected total distance `D_new` and current
effective distance `D_cur` (`activities.distance_meters`):

```
f = D_new / D_cur
```

In one transaction:

```
activities.distance_meters     = D_new
activities.avg_pace_sec_per_km = duration_seconds / (D_new / 1000)   // if running
activities.best_pace_sec_per_km = best_pace / f                     // if running

for each activity_trackpoints row:
  distance_meters *= f
  pace_sec_per_km   /= f   // when non-null
```

`raw_distance_meters` is **not** updated. "Calibrated" is derivable:
`raw_distance_meters != distance_meters`.

**Why not re-parse S3 on every calibration (Approach B)?** Mathematically
identical outcome for uniform scale; S3 fetch + full re-parse adds latency and
moving parts for no gain on the common path. S3 re-parse is reserved for indoor →
outdoor best-effort regeneration where full-resolution raw trackpoints matter.

**Best-effort exclusion.** Indoor runs never receive `activity_best_efforts`
rows at ingest. User-facing "current best per distance"
(`GetUserRunningBestEfforts`) is a read-time `ROW_NUMBER()` query — deleting
rows on outdoor → indoor automatically removes the run from PR surfaces. Max-
effort estimates (`internal/running/estimate/`) read from best-effort history, so
indoor exclusion propagates without separate logic.

**Indoor → outdoor regeneration.** Re-use the existing backfill/ingest path:
fetch TCX from `tcx_s3_key`, parse to raw trackpoints, apply scale
`distance_meters / raw_distance_meters` to each trackpoint's cumulative
distance, run `bestEfforts()` on the scaled series, insert rows.

### API Surface

All endpoints live on the existing activity handler (`internal/activity/handler.go`).
Auth: JWT bearer, user-scoped.

**`POST /activities/{id}/calibrate`**

Request:
```json
{ "distance_meters": 4828.03 }
```

Response: `200` + full activity object (same shape as `GET /activities/{id}`,
including `trackpoints`, `heart_rate_zones` if applicable).

Errors:
- `400` — outdoor run, invalid distance, absurd scale factor
- `404` — activity not found / not owned
- `401` — unauthenticated

**`PATCH /activities/{id}`** (extended)

Request (either or both fields):
```json
{ "name": "Morning treadmill", "environment": "indoor" }
```

Response: `200` + updated activity summary (trackpoints optional — match existing
rename behavior; detail page should refetch GET if trackpoints needed).

**Activity GET/list DTO** — add fields:
```json
{
  "environment": "indoor",
  "raw_distance_meters": 5050.0,
  "distance_meters": 4828.03,
  ...
}
```

Use `httpresp` envelope; machine-readable `code` slugs for client branching
(`outdoor_run_not_calibratable`, `invalid_calibration_distance`, etc.).

### Web (`prog-strength-web`)

Conforms to design-system **v0.4** ([`design-system.md`](../design-system.md)):
near-black field, periwinkle accent as app-chrome, run discipline hue for activity
color. The treadmill badge uses **shape + label** (status encoding), not a new
activity hue — per the "activity type owns color; status is shape + badge" rule.

**`lib/api.ts`:**
- Extend `RunningSession` with `environment: "outdoor" | "indoor"` and
  `raw_distance_meters: number`.
- Add `calibrateRunningSession(token, id, distanceMeters)` → POST calibrate.
- Extend PATCH wrapper to accept `environment` (rename existing function or add
  `patchRunningSession`).

**Run detail (`app/(app)/running/[id]/`):**
- `RunHeaderBand`: **"Treadmill" badge** when `environment === "indoor"` and
  `activity_type === "running"`.
- **"Calibrate distance"** action (indoor only) → `CalibrateDistanceModal`:
  pre-fill current distance in user's unit; validate > 0; show live pace
  preview; on success replace `session` with full API response **including
  trackpoints** (unlike rename, do not preserve stale trackpoints).
- Provenance line when calibrated: "Calibrated from X · Reset" (Reset = calibrate
  back to `raw_distance_meters`).
- Environment toggle (outdoor ↔ indoor) near badge; confirm if switching would
  affect PR membership.

**Activities run list:** small treadmill glyph on indoor running rows.

**Tests:** modal validation; post-calibration header distance matches sum of split
distances from rescaled trackpoints; badge rendering; environment toggle calls
correct API.

### MCP (`prog-strength-mcp`)

Transparent forwarder — no signing key, no business logic. Add:

- `calibrate_run_distance(activity_id, distance_meters)` → POST
  `/activities/{id}/calibrate`
- `set_run_environment(activity_id, environment)` → PATCH
  `/activities/{id}`

Ensure existing run-read tools return `environment` and `raw_distance_meters`.

### Agent (`prog-strength-agent`)

Add tool guidance (system prompt or tool descriptions): indoor/treadmill runs can
be distance-calibrated; outdoor runs must be tagged indoor first; pace and
splits recompute server-side; treadmill runs do not count toward running PRs.
No new endpoints — consumes MCP tools.

### Backfill or Migration

1. **Mechanism:** forward migration only. `raw_distance_meters` backfilled from
   current `distance_meters`; `environment` defaults to `outdoor` for all
   existing rows.
2. **Recoverability:** migration is additive columns with defaults — safe to
   re-run on fresh DBs via normal migration pipeline.
3. **Existing treadmill runs:** owner manually sets `environment = indoor` and
   calibrates via UI or agent. No automated S3 re-parse backfill in this SOW.
4. **Scale:** single-row calibration is O(trackpoints) for one activity (~300
   downsampled points) — trivial for single-user workload.

## Open Questions

1. **Indoor → outdoor on a previously calibrated run: keep calibrated distance
   or revert to raw?** Options: (a) keep current `distance_meters` and generate
   best-efforts from scaled trackpoints — my lean; (b) revert to
   `raw_distance_meters` on tag change. Tentative lean: **(a) keep calibrated**
   — the user calibrated for a reason; retagging outdoor shouldn't undo that.

2. **Timeline stale PR posts** when a run is retagged indoor. Options: (a) accept
   staleness; (b) add retraction/republish. Tentative lean: **(a) accept** —
   matches existing behavior; retraction is its own SOW.

3. **Calendar treadmill glyph.** Options: (a) run-list only in v1; (b) calendar
   banners too. Tentative lean: **(a) run-list only** — calendar can follow if
   the badge pattern proves useful.

4. **Scale factor bounds.** Tentative: reject `f` outside `[0.5, 2.0]` to catch
   unit-entry mistakes (miles vs meters). Tune if real calibrations exceed this.
