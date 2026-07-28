---
status: draft
repos:
  - prog-strength-api
  - prog-strength-mcp
  - prog-strength-web
  - prog-strength-mobile
  - prog-strength-docs
---

# Unified Activity Model

**Status**: Draft · **Last updated**: 2026-07-27

## Introduction

Prog Strength tracks two kinds of training: **lifting workouts** (`workouts` / `workout_exercises` / `sets`) and **endurance activities** (`activities` / `activity_trackpoints`, born as `running_sessions` and generalized in migration 015). Its user does many more kinds of training than that — hiking, kickboxing, and more to come — and the app's long-term shape is an activity tracker where sessions of many types share a common core (when it happened, how long it took, heart rate, calories) and diverge in type-specific detail (distance and pace for a run, exercises and sets for a lift, nothing extra for a kickboxing class).

Today that shape is blocked by an accident of history: the app has **two parallel session domains instead of one session domain with two types**. The `activities` table is endurance-shaped (`distance_meters NOT NULL`, `tcx_s3_key NOT NULL`, `ingest_source` limited to TCX/Garmin), so a kickboxing class literally cannot be stored in it. Lifts live entirely outside it, joined by an awkward half-bridge: migration 033 added a `strength_training` activity type plus a nullable `workouts.activity_id` so a lift can attach a TCX file for heart-rate enrichment — **two rows in two tables for one training session**. And because there is no single "sessions" table, every aggregate surface re-implements the merge in Go with its own discriminator: the timeline has `source_type IN ('workout','run',…)`, planned workouts have `completed_session_kind IN ('workout','activity')`, the dashboard and training snapshot each fetch both repos and union in memory, profile stats has per-domain source seams. Adding a third activity type today would mean threading it through every one of those seams by hand.

This SOW **fully unifies the model before any new type ships**: one `activities` base table holds every training session of every type, per-type detail tables hold the divergence, and a Go **type registry** makes each type a self-describing module. Lifting workouts migrate into the base table (keeping their ids), the dual-row TCX bridge collapses into a single record, and the unified `/activities` API surface replaces `/workouts`. After this ships, adding kickboxing is one Go descriptor and adding hiking is one descriptor plus one small detail-table migration — and both appear in the activity list, timeline, dashboard, and training snapshot automatically, because those surfaces read the unified base rather than enumerating domains.

This is deliberately a **foundation SOW**: it adds no new activity types itself. It ships ahead of them so every future type lands on a pattern instead of a pile of special cases.

## Proposed Solution

The schema follows **class-table inheritance with per-type detail tables**. A rebuilt `activities` base table carries only universal columns — identity, type, start time, duration, name, notes, heart rate, calories, ingest provenance, soft delete. Each activity type owns its detail table(s), 1:1 or 1:N on `activity_id` with `ON DELETE CASCADE`: endurance types get a details row (distance, pace, elevation, route), strength's details are its existing `workout_exercises`→`sets` children re-keyed to the base table, and a future base-only type like kickboxing needs no detail table at all. Two deliberate departures from current practice: `activity_type` and `ingest_source` lose their SQL `CHECK` enums (SQLite cannot alter a CHECK without rebuilding the table, which would make every new type pay the exact cost this SOW removes) — the Go registry becomes the single source of truth for valid values.

In Go, `internal/activity` becomes the base domain and each type registers a **descriptor** with it: type name, create-validation, a detail store, a summary renderer, and optional type-specific routes. The base repository owns everything universal (CRUD, unified list/range queries, dedup, soft delete); type modules own only their divergence. `internal/workout` dissolves into `internal/activity/strength`.

The API is restructured to match: `/activities` becomes the canonical type-uniform surface — one list across all types, one typed create/update/delete driven by registry validation — and `/workouts/*` is removed after a short shim period so web, mobile, and MCP migrate without a mid-flight break. Aggregate surfaces (dashboard, snapshot, timeline hydration, profile stats, plan matching) collapse their dual-domain merge code into reads over the unified base.

The data migration is the riskiest piece and is treated as the centerpiece: workouts move into `activities` **preserving their ids**, so PRs, 1RM history, timeline posts, and planned-workout completions survive as data rewrites of FK targets rather than id remapping. The 033 dual-row enrichment merges into single records. A migration-focused test fixture proves all of it.

## Goals and Non-Goals

### Goals

- **One base table** (`activities`) holding every training session — lifts included — with universal columns only.
- **Per-type detail tables** for type-specific data; strength's details are its exercise/set children.
- A **Go type registry** where each activity type is a self-contained descriptor (validation, detail store, summary, routes); adding a type touches the descriptor, its registration, and optionally one migration — nothing else.
- The **dual-row workout↔activity TCX bridge removed**: one session, one base row; TCX enrichment writes onto it.
- A **unified `/activities` API surface** (list, get, typed create/update, delete) replacing `/workouts`, with type-specific routes mounted by type modules.
- **Aggregate surfaces simplified**: dashboard, training snapshot, timeline hydration/backfill, profile stats, and plan matching read the unified base; new types appear in them automatically.
- **MCP updated** with existing tool contracts kept stable, plus new generic `log_activity` / `list_activities` tools.
- **Web and mobile migrated** to the new surface; the Activities tab becomes the unified all-types list.
- A **"adding an activity type" recipe doc** in `prog-strength-docs` capturing the exact steps.
- **Ids preserved end-to-end** through the migration; timeline posts, PRs, and planned-workout references survive untouched.

### Non-Goals

- **New activity types.** Hiking, kickboxing, etc. ship in follow-up SOWs using the recipe. This SOW's registered types are exactly today's set: `running`, `walking`, `cycling`, `other`, `strength_training`.
- **Whoop as an activity source.** `ingest_source='whoop'` is reserved as a value but no sync is built (Whoop remains recovery-only).
- **Garmin Connect API ingest** — still scaffolded, still unimplemented.
- **Agent repo changes.** MCP tool contracts stay stable precisely so `prog-strength-agent` needs nothing.
- **Visual redesigns.** Web/mobile changes are endpoint migration plus the unified list; existing card vocabulary and design system conventions apply.
- **Changing running analytics** — best efforts, max-effort estimation, running metrics keep their behavior and endpoints.

## Implementation Details

### Data Model

One migration (next free number in `internal/db/migrations/`, `042_unified_activity_model` at time of writing). Given the multi-step rebuild + data merge, the implementer may use the repo's registered-Go-migration mechanism (`internal/db/go_migrations.go`, precedent: 028/029) instead of pure SQL; either way it shares the `schema_migrations` ledger and runs in one transaction.

**`activities`** (rebuilt — the base table, one row per training session of any type):

- `id TEXT PRIMARY KEY`
- `user_id TEXT NOT NULL`
- `activity_type TEXT NOT NULL` — **no CHECK**; validated by the Go registry
- `start_time DATETIME NOT NULL`
- `duration_seconds INTEGER` — nullable (an in-progress lift has no duration yet)
- `name TEXT`, `notes TEXT`
- `avg_heart_rate_bpm INTEGER`, `max_heart_rate_bpm INTEGER`, `total_calories INTEGER` — nullable vitals; any type may have them, from any device
- `ingest_source TEXT NOT NULL` — **no CHECK**; values today: `manual` (app/agent-entered — all lifts), `manual_tcx`, `garmin_api`; `whoop` reserved
- `source_activity_id TEXT` — now nullable; manual entries have none
- `tcx_s3_key TEXT` — now nullable; ingest provenance for any TCX-derived session (including strength TCX enrichment)
- `created_at`, `updated_at DATETIME NOT NULL`, `deleted_at DATETIME` (soft delete)

Indexes:

- `idx_activities_dedup` — `UNIQUE (user_id, ingest_source, source_activity_id) WHERE deleted_at IS NULL AND source_activity_id IS NOT NULL` (partial-ized so manual entries don't collide)
- `idx_activities_user_start` — `(user_id, start_time DESC) WHERE deleted_at IS NULL`
- `idx_activities_user_type_start` — `(user_id, activity_type, start_time DESC) WHERE deleted_at IS NULL`

**Per-type detail tables** (each `activity_id TEXT PRIMARY KEY REFERENCES activities(id) ON DELETE CASCADE`):

- `activity_run_details` — `distance_meters REAL NOT NULL`, `avg_pace_sec_per_km REAL`, `best_pace_sec_per_km REAL`, `elevation_gain_meters REAL`, `environment TEXT NOT NULL DEFAULT 'outdoor'`, `raw_distance_meters REAL NOT NULL DEFAULT 0`, `route_geojson TEXT`
- `activity_walk_details`, `activity_cycle_details`, `activity_other_details` — same endurance shape (the accepted duplication of the per-type choice; in Go they share one detail-store implementation parameterized by table name)
- **Strength** — no 1:1 detail row. Its details are the children: `workout_exercises` → **`activity_exercises`** (`workout_id` → `activity_id`, FK to `activities`), `sets` unchanged beneath them. A type's "details" are whatever tables it needs; strength demonstrates the 1:N case.
- `activity_trackpoints`, `activity_best_efforts` — already keyed on `activity_id`; unchanged (best efforts remain running-scoped).

A future kickboxing type is a base row with no detail table; hiking adds one endurance-shaped detail table.

### Data Migration

Standard SQLite create-new + `INSERT…SELECT` + drop + rename sequence (pattern of migrations 012/014/015), `PRAGMA defer_foreign_keys` for the rekeying. Steps, in order:

1. **Assert id-space disjointness**: abort if any `workouts.id` already exists in `activities.id` (both come from `internal/id`; collision is practically impossible, but the migration must not silently merge strangers).
2. **Lift rows into the base**: every live-or-deleted `workouts` row becomes an `activities` row **reusing the workout's id** — `activity_type='strength_training'`, `start_time=performed_at`, `ingest_source='manual'`, `duration_seconds = ended_at − performed_at` when `ended_at` is set, `name`/`notes`/`created_at`/`updated_at`/`deleted_at` carried over.
3. **Collapse the 033 dual-row bridge**: for workouts with a linked `strength_training` enrichment row (`workouts.activity_id IS NOT NULL`), fold `avg/max_heart_rate_bpm`, `total_calories`, `tcx_s3_key`, `source_activity_id`, `ingest_source` from the enrichment row into the workout's new base row (enrichment duration wins over the computed one when present), re-key its `activity_trackpoints` to the workout's id, then delete the orphaned enrichment row. `workouts.activity_id` and the `AttachActivity`/`DetachActivity`/`GetByActivityID`/`SummariesByIDs` seam cease to exist.
4. **Split endurance columns off the base** into `activity_{run,walk,cycle,other}_details` by `activity_type`.
5. **Re-point strength children and FKs**: `workout_exercises` → `activity_exercises` (column rename `workout_id`→`activity_id`, FK to `activities`); rebuild `personal_records`, `personal_record_events`, `exercise_one_rep_max_history` so `workout_id` (renamed `activity_id`) references `activities`. Ids are unchanged, so these are table rebuilds, not data rewrites.
6. **Normalize discriminators**: `planned_workouts.completed_session_kind` collapses to the single value `activity` (column retained-then-dropped or dropped in place, implementer's call); `timeline_post` rows are **untouched** — `source_type='workout'` and `'run'` posts keep their ids and simply hydrate from the unified store.
7. **Drop `workouts`.**

### Domain Layout — the Type Registry

`internal/activity` is the base domain; each type is a subpackage registering a descriptor:

```go
type Descriptor struct {
    Type           string                       // "running", "strength_training", later "kickboxing"
    ValidateCreate func(ctx, CreateRequest) error // e.g. "runs require distance"
    Details        DetailStore                  // Load/Save/Delete the type's detail rows; nil for base-only types
    Summarize      func(Activity, Details) Summary // card/list summary: "12 sets · 8,400 lb", "5.0 mi · 41:12"
    MountRoutes    func(chi.Router)             // optional type-specific endpoints
}
```

- `internal/activity` — base model, registry, base `Repository` (CRUD, unified `List`/`ListInRange` with keyset + range paths as today, dedup, soft delete), unified handler.
- `internal/activity/run` (+ walk/cycle/other sharing the endurance detail store parameterized by table) — endurance descriptors, TCX ingest, calibration, environment, routes.
- `internal/activity/strength` — absorbs `internal/workout` wholesale: exercises/sets detail store, PR detection, 1RM history, headline exercises, TCX attach/detach. `/personal-records*` and headline-exercise endpoint paths do not change.
- Descriptors are registered in one place in `internal/server/server.go`; the registry rejects unknown `activity_type` values at the API boundary.

The recipe doc (`prog-strength-docs`) documents the exact steps: write a descriptor, (optionally) add a detail-table migration, register it — and enumerates what comes free (list, create, timeline, dashboard, snapshot, MCP `log_activity`).

### API Surface

`/activities` becomes the canonical type-uniform surface (auth, `httpresp`, and the timezone+local-date convention for date-windowed lists via `internal/daterange` all as today):

- **`GET /activities?type=&…`** — unified list across **all** types (strength included now; the current handler's exclusion of `strength_training` is removed). Existing cursor/range parameters preserved. Each item: base fields + `activity_type` + a type-keyed `details` summary from `Summarize`.
- **`GET /activities/{id}`** — base + full typed `details` (for strength: exercises and sets).
- **`POST /activities`** — one create endpoint for every type: base fields + type-specific `details` payload, validated by the registry (`ValidateCreate`). A lift posts exercises/sets here; a future kickboxing posts type + start + duration. Replaces `POST /workouts` and is the generic manual-log path.
- **`PUT /activities/{id}`**, **`DELETE /activities/{id}`** — same typed pattern; delete stays soft.
- **Type-mounted routes**: `POST /activities/tcx` (endurance ingest, unchanged), `POST|DELETE /activities/{id}/tcx` (strength TCX enrichment, was `/workouts/{id}/tcx`), `POST /activities/{id}/calibrate`, `PATCH /activities/{id}` rename/environment behaviors folded into `PUT` or kept, implementer's call with contract documented.
- **Unchanged**: `/running/*`, `/personal-records*`, headline exercises.
- **Moved**: `GET /workouts/progression` (strength analytics) relocates into the strength module under an `/activities` path, keeping its response shape.
- **`/workouts/*` compatibility shims** — kept during rollout as thin mounts over the unified store (cheap: ids preserved, shapes mapped), removed in the final cleanup stage.

**Aggregate surfaces simplify** (each loses its dual-domain merge):

- Dashboard: one `ListInRange` over the base replaces the separate activity+workout fetches; streaks/tiles compute over all sessions.
- Training snapshot: `countActiveDays` unions base sessions + steps only.
- Timeline: hydrator becomes registry-driven (`Summarize`) — `source_type='workout'|'run'` both resolve through the unified store; backfill queries collapse to reads over `activities`.
- Profile stats sources and plan matchers read the unified base with type filters instead of per-domain seams.

### MCP (`prog-strength-mcp`)

- **Contracts stable**: `create_workout`, `list_workouts` (`workouts.py`), `set_run_environment`, `calibrate_run_distance` (`running.py`), `complete_planned_workout` keep names and shapes; internals re-point to the new endpoints (`create_workout` → `POST /activities` with `type=strength_training`). `prog-strength-agent` requires zero changes.
- **New canonical tools** in a new `activities.py`: `log_activity(type, start_time, duration, name?, notes?, vitals?, details?)` and `list_activities(type?, window…)` following the timezone+local-date convention. These are what let the agent log a kickboxing class the day the type exists.
- MCP remains forwarder-only per house rule; no logic beyond request shaping.

### Web (`prog-strength-web`)

- Migrate the API client layer in `lib/` from `/workouts` to `/activities`; workout logging/detail flows keep their UI and write the typed create/update payloads.
- The **Activities tab becomes the unified all-types list**: one fetch, cards switching on `activity_type`, reusing the card vocabulary the timeline established. Lifts appear alongside runs.
- No visual redesign; design-system conventions (`prog-strength-docs/design-system.md`) apply to any new card variants.
- Per the repo's own warning: consult `node_modules/next/dist/docs/` and existing validated patterns before writing code — this is not stock Next.js.

### Mobile (`prog-strength-mobile`)

- Same client migration and unified Activities list, following the established research → plan → subagent parity workflow. Ships last (see Rollout) so TestFlight builds never point at removed endpoints.

### Testing

- **Migration tests are the centerpiece**: a fixture DB seeded with workouts (with and without 033 enrichment links), runs/walks, PRs, 1RM history, timeline posts, and planned-workout completions; assert ids survive, the dual-row merge folds vitals/trackpoints correctly, endurance columns land in the right detail tables, FKs re-point, `timeline_post` rows are byte-identical, and the migration is transactional (a forced mid-step failure leaves the old schema intact).
- **Repository**: base CRUD + list/range/keyset; each detail store round-trips; cascade deletes from base to details/children; dedup partial index (manual entries never collide; TCX re-upload still 409s).
- **Handler**: typed create/update validation per descriptor; unified list includes strength; type filter; shim endpoints return legacy shapes.
- **Registry contract test**: register a fake base-only type (kickboxing-shaped) in a test and assert it flows through create → list → timeline summary → dashboard → snapshot with **no changes outside its descriptor**. This test *is* the pattern's guarantee.
- **MCP**: existing tool tests keep passing unchanged (that's the stability proof); new `log_activity`/`list_activities` tests.
- **Web/mobile**: client-layer tests updated to new shapes; unified-list rendering tests per platform convention.

### Rollout

Staged inside this one SOW so nothing breaks mid-flight:

1. **API** — migration + unified domain + `/activities` surface, with `/workouts/*` shims kept. Deploy; existing web/mobile/MCP continue working through shims.
2. **MCP** — re-point internals, add `log_activity`/`list_activities`. Deploy.
3. **Web** — client migration + unified Activities tab. Deploy.
4. **Mobile** — client migration + unified list; TestFlight release.
5. **API cleanup** — remove `/workouts` shims and dead code once 2–4 are live.

Single-user beta makes the big-bang migration acceptable; the shims exist for sequencing safety, not long-term compatibility.

## Open Questions

- **`strength_training` naming** — kept as-is (033 precedent) to avoid a value rewrite; a cosmetic rename to `lift` could ride along in the migration if preferred. Default: keep.
- **Walk/cycle/other detail tables** — created in this migration even though row counts are tiny, for pattern uniformity. If the implementer finds zero walk/cycle rows in production data, creating the tables empty is still correct (the descriptors reference them).
- **`PATCH /activities/{id}`** — today's rename/environment PATCH vs. folding into the typed `PUT`. Either is fine; document whichever ships in the recipe doc.
- **Shim removal timing** — stage 5 should wait until the TestFlight build from stage 4 is the one in testers' hands; the beta allowlist makes stragglers unlikely but worth a glance at API logs for `/workouts` traffic before deleting.
