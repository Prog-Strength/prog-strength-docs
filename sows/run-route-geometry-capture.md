---
type: sow
status: draft
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Run Route Geometry Capture

**Status**: Draft · **Last updated**: 2026-06-16

## Introduction

The timeline redesign ([`sows/timeline-redesign.md`](timeline-redesign.md)) selected the `strava-social-dashboard` direction, whose signature element is a **run route map** on each run card. That SOW was de-scoped to ship without the map (a graceful placeholder) once it surfaced that **no geographic position is captured anywhere in the stack** — the route can't be drawn because the data doesn't exist. This SOW is the prerequisite that makes route maps possible: capture, store, and expose run GPS geometry.

The gap, concretely:

- **TCX parser** `internal/activity/tcx_parser.go` decodes time / distance / HR / altitude only; the TCX `<Position>` element (`<LatitudeDegrees>` / `<LongitudeDegrees>`) is never read and is dropped on import.
- **DB** `internal/db/migrations/015_activities_generalize.sql` (`activity_trackpoints`) has no `latitude` / `longitude` columns.
- **Domain model** `internal/activity/model.go` `Trackpoint` carries no coordinates.

Raw TCX files persist in S3 (`TCXS3Key`), so historical GPS data is recoverable for runs that recorded it — but it must be re-parsed and backfilled, since the stored trackpoints have no position. Treadmill / indoor / non-GPS runs have no `<Position>` at all and must degrade gracefully (no route, which is the placeholder the timeline already renders).

## Proposed Solution

Capture position on import, store it, backfill history, and expose a compact route in the feed projection.

**API (`prog-strength-api`)**

- **Parser** — extend `tcx_parser.go` to read `<Position><LatitudeDegrees>/<LongitudeDegrees>` into the parsed trackpoint when present; tolerate its absence.
- **Schema** — a forward migration adding nullable `latitude` / `longitude` to `activity_trackpoints`; the domain `Trackpoint` model and repository read/write paths carry the coordinates.
- **Backfill** — a one-off (idempotent, resumable) job that re-parses each existing activity's raw TCX from S3 and populates coordinates where the source has them. Runs whose TCX lacks `<Position>` are left null.
- **Feed projection** — extend the run post's `content` with the **compact route summary** the timeline originally specified: a simplified (Douglas–Peucker-reduced, small point budget) polyline + bounds, present only when the run has coordinates, sized so a feed page doesn't bloat or re-introduce an N+1. The full-fidelity path stays on the detail surface, not the feed.

**Web (`prog-strength-web`)**

- Wire the new `content.route` into the timeline `RouteMap` component that `sows/timeline-redesign.md` left in place — render the static SVG polyline within bounds when present, keep the placeholder when absent. This is a localized prop change, not a card rebuild.

## Goals and Non-Goals

### Goals

- Capture `<Position>` from TCX on import; store nullable `latitude` / `longitude` on `activity_trackpoints`.
- Backfill coordinates for existing activities from raw S3 TCX (idempotent, resumable); null where the source has none.
- Expose a compact, simplified run **route** (polyline + bounds) in the timeline feed projection, only when coordinates exist, sized for a feed page.
- Flip the timeline `RouteMap` from placeholder to real map via the new projection field.
- Graceful no-route handling end to end (treadmill / indoor / non-GPS).
- CI green in both repos.

### Non-Goals

- **A tile basemap.** A lightweight static SVG polyline (the timeline's existing approach) is the target; a real map basemap is a later additive option.
- **Full-fidelity routes in the feed.** The feed carries a thumbnail-grade simplified path; full geometry stays on the activity detail surface.
- **Re-deriving position from distance.** Not possible; runs without stored `<Position>` simply have no route.
- **Re-opening the timeline visual design.** That's settled in `sows/timeline-redesign.md`; this only supplies data + the prop wiring.

## Implementation Details

> To be expanded at implementation: the exact migration shape and index strategy on `activity_trackpoints`, the backfill job's batching / checkpointing and how it's invoked, the simplification point budget and bounds encoding, and the import-path and projection test coverage. Deploy order: parser + migration first (additive, nullable), then backfill, then the feed projection + web wiring.

## Rollout

1. **`prog-strength-api`** — parser + migration (additive, nullable) deploy first; then run the backfill; then add the feed-projection route field.
2. **`prog-strength-web`** — wire `content.route` into the existing `RouteMap`.
3. **`prog-strength-docs`** — mark shipped on merge.

## Open Questions

- **Backfill invocation** — one-off script vs. an admin-triggered job vs. a migration step; how progress is checkpointed and resumed.
- **Point budget / encoding** — the smallest simplified path that still reads as the route's shape in a feed thumbnail.
- **Coverage of historical data** — how many existing runs actually carry `<Position>` in their TCX (drives how impactful the backfill is); worth a quick audit before building.
