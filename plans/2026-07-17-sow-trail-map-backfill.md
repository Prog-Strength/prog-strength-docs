# Trail Map Route Backfill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: implement this plan
> task-by-task with a subagent per task, each spec-reviewed and
> code-quality-reviewed before moving on. Steps use checkbox
> (`- [ ]`) syntax for tracking.

**Goal:** Add a boot-time, idempotent backfill on `prog-strength-api`
that re-parses archived S3 TCX for existing **outdoor** activities still
missing `route_geojson`, writing the same simplified route and matching
downsampled trackpoint lat/lon the live ingest path would have produced.
No client changes — the detail map already renders when `route` is
present.

**Architecture:** One new method
`(*SQLiteRepository).BackfillActivityRoutes(ctx)` in
`internal/activity/backfill.go`, mirroring the existing
`BackfillActivityBestEfforts` / `BackfillBestEffortWindowBounds` shape
(materialize targets up front, per-activity transaction, skip+log on
failure, summary log line). It reuses the settled geometry primitives
`buildRoute` (route.go) and `downsample` (tcx_summarizer.go) verbatim —
the RDP/gap algorithm is NOT forked. Registered in
`internal/server/server.go` next to the other activity backfills with a
loud `TEMPORARY` comment block; errors are soft-logged, never fatal.

**Tech Stack:** Go 1.25 (chi + database/sql + mattn/go-sqlite3).
golangci-lint v2.12.2 is the CI-pinned linter.

**Repo path:** `/workspace/prog-strength-api`

**Source spec:** `prog-strength-docs/sows/sow-trail-map-backfill.md`

---

## Key reuse facts (verified against the codebase)

- `buildRoute(tps []parsedTrackpoint) *string` (route.go) is the shared
  route serializer used by `summarize`. Returns nil when fewer than two
  positioned points survive splitting → caller stores NULL. Reuse as-is.
- `downsample(raw []parsedTrackpoint, first parsedTrackpoint, actType ActivityType) []Trackpoint`
  (tcx_summarizer.go) produces the exact even-stride kept points, with
  `Latitude`/`Longitude` truncated to 6 decimals and set only when the
  raw point carried a Position. lat/lon assignment is **independent** of
  `actType`. Since the stored trackpoint rows came from this same
  function on the same raw bytes, re-running it yields identical
  sequences → update lat/lon by `sequence` with no insert/delete.
- `parseTCX` / `validate` are the same pre-checks best-efforts uses.
  `validate` guarantees ≥1 trackpoint, so `parsed.Trackpoints[0]` is safe.
- Selection: outdoor + `route_geojson IS NULL` + non-empty `tcx_s3_key`,
  `deleted_at IS NULL`. Idempotent because processed rows leave the set.
- `r.archiver.Get(ctx, key)` returns `ErrNotFound` on a missing object.
- `MemoryArchiver` (memory_archiver.go) backs tests; `newRepo(t)` returns
  `(*SQLiteRepository, *MemoryArchiver)`.

---

## Phase 1 — Backfill implementation

### Task 1: `BackfillActivityRoutes` + boot registration (TDD)

**Files:**
- Modify: `prog-strength-api/internal/activity/backfill.go`
- Create: `prog-strength-api/internal/activity/backfill_routes_test.go`
- Modify: `prog-strength-api/internal/server/server.go`

- [ ] **Step 1: Write tests first** in `backfill_routes_test.go`, using
  the `newRepo(t)` harness. Seed activities by calling `repo.Create`
  with a fixture TCX (so the raw bytes land in the MemoryArchiver and
  trackpoint rows exist), then simulate the pre-feature state with direct
  `UPDATE activities SET route_geojson = NULL` and
  `UPDATE activity_trackpoints SET latitude = NULL, longitude = NULL`.
  Cases:
  - **Outdoor with Position** (`typical_5k.tcx`): after backfill,
    `route_geojson` is a non-NULL MultiLineString feature and kept
    trackpoints have lat/lon populated. Assert the route string and the
    per-sequence lat/lon **equal** what a fresh `summarize` of the same
    bytes produces (shared-helper parity — no drift).
  - **No-Position on an outdoor-tagged row** (`treadmill_5k.tcx`, forced
    to `environment = 'outdoor'` via UPDATE): `route_geojson` stays NULL,
    no panic, counted as skipped.
  - **Already has route** (unmodified `typical_5k.tcx` Create): not in
    the selection set; row unchanged after backfill (idempotent).
  - **Missing S3 object** (UPDATE `tcx_s3_key` to a bogus key): skipped,
    `BackfillActivityRoutes` returns nil, route stays NULL.
  - **Indoor row** (`environment = 'indoor'`) with NULL route: never
    selected, untouched.

- [ ] **Step 2: Implement `BackfillActivityRoutes(ctx)`** in
  backfill.go. Shape (mirror `BackfillActivityBestEfforts`):
  1. Materialize targets up front from the SOW selection query
     (`id, tcx_s3_key, user_id, activity_type`), slice built inside a
     closed read cursor before any writes.
  2. Per target: `archiver.Get` → `parseTCX` → `validate`; skip+log
     (`activity_id`, `user_id`) and continue on any failure. Never fail
     the boot.
  3. `route := buildRoute(parsed.Trackpoints)`. If nil (no usable
     geometry), log at info that there was nothing to map, count as
     skipped, leave `route_geojson` NULL, continue.
  4. `pts := downsample(parsed.Trackpoints, parsed.Trackpoints[0], t.actType)`.
  5. One transaction per activity:
     `UPDATE activities SET route_geojson = ? WHERE id = ?` then, for
     each kept point, `UPDATE activity_trackpoints SET latitude = ?,
     longitude = ? WHERE activity_id = ? AND sequence = ?`. Commit.
     A mid-loop crash leaves done rows done and the rest still selected
     next boot (natural resume).
  6. `log.Printf("backfill activity routes: complete processed=%d skipped=%d total=%d", ...)`.
  Add a `// TEMPORARY — delete after prod outdoor history has maps ...`
  comment on the function, matching the SOW's deprecation instruction.

- [ ] **Step 3: Register the boot call** in server.go, adjacent to the
  `BackfillActivityBestEfforts` / `BackfillBestEffortWindowBounds` calls,
  using the loud `TEMPORARY` comment block quoted verbatim in the SOW's
  Proposed Solution. Soft-log the error
  (`log.Printf("backfill activity routes: %v", err)`) — do **not**
  `return nil, err` / `Fatal`, per the SOW ("boot always continues").

- [ ] **Step 4: Green the gate.** `go test ./...`, `go vet ./...`,
  `golangci-lint run` (v2.12.2), `go mod tidy` (no drift). Fix any
  finding in the code — never with `//nolint` or a skipped test.

---

## Non-Goals (from the SOW — do not do these)

- No new SQL migration (trail-map columns already exist).
- No re-implementation of RDP / gap rules — reuse `buildRoute` /
  `downsample`; ε, MultiLineString, 50 m split, 6-decimal truncation
  stay as settled.
- No web/mobile/MCP/agent work.
- No indoor selection; no recompute of distance/pace/best-efforts/
  environment/summary stats — geometry only.
- No completion marker, feature flag, or timed auto-removal — deprecation
  is the loud comment + a human `chore:` later.

## Verification

- `go test ./...` green (new backfill tests + existing suite).
- `golangci-lint run`, `go vet ./...`, `go mod tidy` drift-free.
- Manual (post-deploy, operator): watch boot logs for
  `backfill activity routes: complete processed=… skipped=… total=…`;
  open a pre-feature outdoor run on `/running/[id]` and confirm the map
  renders.
