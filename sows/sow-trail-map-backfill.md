---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-docs
---

# Trail Map Route Backfill

**Status**: Shipped · **Last updated**: 2026-07-17

> **Prerequisite:** [`sows/sow-trail-map.md`](sow-trail-map.md) must be
> implemented and deployed first (schema columns, ingest-path RDP /
> `route_geojson`, detail `route` field, web map). This SOW only fills history.
> No web or design-system work — the map UI already renders when `route` is
> present.

## Introduction

[`sow-trail-map.md`](sow-trail-map.md) deliberately ships **without** backfill:
new outdoor TCX uploads get a route map; every run already in Prog Strength
stays map-less even though the GPS is sitting in the archived TCX on S3.

That gap is temporary and visible. Opening an old trail run still looks like a
treadmill session on the detail page. The raw files were stored exactly so we
could re-parse later — this SOW is that later.

Once it ships, a single API boot re-parses archived TCX for outdoor activities
that still lack `route_geojson`, writes the same simplified route (and
per-trackpoint lat/lon) the live ingest path would have written, and the
existing detail map lights up with no client change.

## Proposed Solution

Add a **boot-time, idempotent** backfill on `prog-strength-api`, matching the
existing best-efforts / window-bounds pattern in `internal/activity/backfill.go`
and the registration site in `internal/server/server.go`.

On each API start:

1. Select live outdoor activities with `route_geojson IS NULL` (and a TCX key).
2. For each: fetch archived TCX from S3 → `parseTCX` → run the **same**
   gap-split + RDP + serialize path used at ingest (shared helper — do not
   fork the algorithm) → update `activities.route_geojson` and the activity's
   downsampled `activity_trackpoints.latitude` / `longitude` to match what a
   fresh upload would store.
3. Missing / unparseable TCX: log with `activity_id`, skip, continue — never
   fail the boot (same operator-safety rule as best-efforts backfill).
4. Activities that parse but have no usable Position geometry leave
   `route_geojson` NULL (correct) and drop out of meaningful work after a
   logged skip, or remain selected until an operator fixes the archive — at
   single-user scale this is acceptable.

**Deprecation (loud comment only).** No completion marker, no calendar nag.
Register the call next to the other activity backfills with an unmistakable
comment block, e.g.:

```go
// TEMPORARY — route geometry backfill (sows/sow-trail-map-backfill.md).
// Remove this call and BackfillActivityRoutes once prod outdoor history has
// maps (or you no longer care about pre-trail-map uploads). Safe to delete:
// new ingest already writes route_geojson; this only repairs the past.
if err := activityRepo.(*activity.SQLiteRepository).BackfillActivityRoutes(context.Background()); err != nil {
	log.Printf("backfill activity routes: %v", err)
}
```

Mirror a short `TEMPORARY — delete after…` note on the backfill function itself.
The owner will remove it in a later `chore:` PR when ready; the comment is the
reminder, not a self-deleting mechanism.

## Goals and Non-Goals

### Goals

- On API boot, backfill `route_geojson` (and matching trackpoint lat/lon) for
  existing **outdoor** activities that still have `route_geojson IS NULL`, by
  re-parsing S3 TCX with the same geometry pipeline as live ingest.
- Idempotent: activities that already have `route_geojson` are never rewritten.
- Skip + log on S3 / parse failures; boot always continues.
- Log a summary (`processed`, `skipped`, `total`) like the best-efforts
  backfill.
- Loud `TEMPORARY` comments at the boot registration site and on the backfill
  function so removal is obvious in review.
- Cross-link from [`sow-trail-map.md`](sow-trail-map.md) Non-Goals / migration
  section to this SOW (docs-only edit when this lands or when trail-map is
  approved — either is fine).

### Non-Goals

- **Re-implementing RDP / gap rules** — reuse the trail-map ingest helpers;
  parameters stay as settled there (ε = `5×10⁻⁵`, MultiLineString, 50 m
  teleport split, 6-decimal truncation).
- **Web, mobile, MCP, agent** — no client work; detail already conditional on
  `route`.
- **Timeline / feed route projection** — still out of scope (same as trail-map
  SOW).
- **Backfilling indoor activities** — they correctly have no route; do not
  select `environment = 'indoor'`.
- **Completion markers, feature flags, or timed auto-removal** — deprecation
  is a loud comment + human `chore:` later.
- **Re-uploading or changing user-visible summary stats** — geometry only;
  do not recompute distance, pace, best efforts, or environment from the
  re-parse (environment and summaries already exist on the row).

## Implementation Details

### Selection query

```sql
SELECT id, tcx_s3_key, user_id
FROM activities
WHERE deleted_at IS NULL
  AND environment = 'outdoor'
  AND route_geojson IS NULL
  AND tcx_s3_key IS NOT NULL
  AND tcx_s3_key != ''
```

**Why outdoor-only:** indoor runs keep `route_geojson` NULL forever. Selecting
all NULL routes would re-fetch every treadmill TCX on every boot. Outdoor +
NULL is the historical gap the trail-map SOW left behind (plus any outdoor row
whose TCX truly had no Position — rare; re-parse is cheap and stays NULL).

Materialize the target list up front (same pattern as
`BackfillActivityBestEfforts`) so per-activity writes are not inside an open
read cursor.

### Write path

For each target:

1. `archiver.Get` → `parseTCX` (tolerate validate failures the same way best
   efforts does, or require validate — match whatever the trail-map ingest
   helpers expect; prefer calling a shared `BuildRouteGeometry(parsed) (geojson string, pointCoords []…, ok)` used by summarize + backfill).
2. Update `activities.route_geojson` when a non-empty MultiLineString was
   produced.
3. Update existing `activity_trackpoints` rows for that `activity_id`: set
   `latitude` / `longitude` on each sequence by re-applying the **same**
   even-stride downsample index mapping as `downsample` in
   `tcx_summarizer.go`, copying Position from the corresponding raw
   trackpoint (NULL when that raw point lacked Position). Do not insert or
   delete trackpoint rows; sequence count stays as stored.
4. If geometry is empty (no GPS in file): leave `route_geojson` NULL; optionally
   still write null lat/lon (no-op). Log at info/debug that there was nothing
   to map so the skip is visible in the summary counts.

Transaction per activity (or one UPDATE activities + one batch UPDATE
trackpoints) so a mid-loop crash leaves already-processed rows done and the
rest still selected next boot — natural resume.

### Boot registration

In `internal/server/server.go`, adjacent to the existing:

```go
BackfillActivityBestEfforts(...)
BackfillBestEffortWindowBounds(...)
```

add `BackfillActivityRoutes` with the **TEMPORARY** comment block quoted in
Proposed Solution. Soft-log errors; do not `Fatal` the process.

### Tests

- Fixture outdoor TCX with Position → after backfill, `route_geojson` non-NULL
  MultiLineString; trackpoint lat/lon populated on kept samples.
- Fixture treadmill / no-Position TCX on an outdoor-tagged row → remains NULL
  route; no panic.
- Activity that already has `route_geojson` → not in selection / unchanged
  (idempotent).
- Missing S3 object → skipped, function returns nil.
- Shared helper: backfill output matches summarize/ingest for the same raw
  bytes (golden or call both paths in one test).

### Backfill or Migration

1. **Mechanism** — boot-time Go backfill only. **No new SQL migration** if
   trail-map columns already exist; this SOW assumes they do.
2. **Recoverability** — per-activity skip + next-boot retry for failures;
   successful rows leave the `route_geojson IS NULL` set and are not touched
   again.
3. **Scale boundary** — single-user, tens–low hundreds of outdoor runs; full
   serial re-parse on one boot is fine. If this ever ran multi-tenant at
   volume, move to a background worker — out of scope here.

### Rollout

1. Confirm trail-map API (+ web) is live in the target environment.
2. Deploy API build that includes `BackfillActivityRoutes`.
3. On first boot, watch logs for `backfill activity routes: complete
   processed=… skipped=… total=…`.
4. Spot-check a few pre-feature outdoor runs on `/running/[id]` — map should
   appear.
5. Later (human): delete the boot call + function under a `chore:` PR when the
   TEMPORARY comment has earned its keep.

## Open Questions

1. **Shared helper extraction shape.** Options: export `BuildRouteFromParsed`
   from the activity package used by summarize + backfill, vs duplicate briefly
   and consolidate later. *Tentative lean:* one shared helper from day one so
   ε / gap rules cannot drift.
2. **Outdoor rows with no GPS in TCX.** Remain NULL forever and stay in the
   selection set. *Tentative lean:* accept re-fetch each boot at current scale;
   if log noise bothers, a follow-up can add a "examined, empty" sentinel —
   explicitly out of scope unless it hurts in practice.
)
