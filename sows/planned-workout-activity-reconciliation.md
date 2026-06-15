---
status: ready_for_implementation
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Planned Workout ↔ Activity Reconciliation

**Status**: Ready for implementation · **Last updated**: 2026-06-15

## Introduction

Prog Strength lets a user plan lifts and runs ahead of time; those planned sessions sync to Google Calendar. But once the user actually trains, the planned session and the logged activity live side by side forever. If I plan an Easy Run for 1:30pm today and then upload the TCX from my Garmin after the run, my calendar shows **two** things on that day — the *planned* Easy Run (still `planned`) and the *actual* run Activity — when they describe one event.

The data model already anticipates the link: a `PlannedWorkout` has `Status` (`planned` / `completed` / `skipped`), `CompletedSessionID`, and `CompletedSessionKind` (`workout` for a lift, `activity` for a run), and there is a `POST /planned-workouts/{id}/complete` endpoint that sets those fields and rewrites the Google Calendar event with a completion marker. What's missing is **anything that decides which activity completes which plan**. Today the completion link can only be created by a client that already knows the session id; nothing creates it automatically, so in practice it never happens.

This SOW adds automatic reconciliation: when a session is logged, Prog Strength finds the planned session it fulfills and links them — collapsing "planned" and "actual" into one completed session that points at the real activity. It biases toward **auto-linking** (recall over precision) and gives the user a cheap **unlink** escape hatch for the rare false positive. It also backfills existing data via a new **Go-migration** capability in the API's migration runner.

### Goals

- A logged run/lift automatically completes and links to the matching planned session, with no user action.
- Matching is robust to the things that actually vary: import names are generic/empty, and users train off their scheduled time.
- A one-tap **unlink** reverses a wrong match.
- Deleting a linked session reverts its plan to `planned` (no dangling links).
- The activity/run/workout detail surfaces show the reverse relationship ("completes your planned Easy Run").
- Existing planned-but-unlinked history is reconciled once via a backfill that lives in git alongside its migration.

### Non-goals (deferred)

- **Manual linking** for cross-day / wrong-kind misses (pick a session from the plan page).
- **Auto-skip** of planned sessions that never got a matching activity.
- **Suggestion / confirm tier** — v1 is full-auto + unlink, per product decision.
- **Walks/cycling completing run plans** — v1 matches a `run` plan strictly to a `running` Activity.
- **Retroactive Google Calendar rewrites during backfill** — backfill is DB-only.

## Matching rule

Given a freshly logged session (a running `Activity` or a lifting `Workout`), find the plan it completes:

1. **Candidate set** — the session's user's plans where:
   - `Status == planned` (already-`completed`/`skipped` plans are never candidates), and
   - **kind matches**: a `run` plan pairs only with an `Activity` whose `ActivityType == "running"`; a `lift` plan pairs only with a `Workout`, and
   - not soft-deleted, and
   - **same local calendar day** as the session — computed in *the plan's own `Timezone`* (IANA). The plan's local day is `ScheduledStartUTC` rendered in `Timezone`; the session's local day is its start (`Activity.StartTime` / `Workout.PerformedAt`, both UTC) rendered in that same `Timezone`. Equal dates → same day.
2. **Disambiguation** — if multiple candidates remain (a two-a-day), pick the one whose `ScheduledStartUTC` is **closest in time** to the session start. Exact ties (effectively never) break by name-equality, then run-type match, then earliest scheduled start.
3. **Guards** — a session links to at most one plan; a plan that is already completed is not a candidate. No candidate → no-op (never fabricate a link).

The only signals are **time + kind**, both of which are final the instant the session is created and never change. Name, run type, distance, and duration are explicitly *not* gates (name is generic or empty at import time and mutates on rename); at most they break a tie. This is what makes matching a **single-trigger** operation — it runs once at ingest and never needs re-running when the user later renames a run.

The time comparison is pure UTC arithmetic; the timezone is used only to bucket into a local calendar day.

## Architecture: the live path

### A consumer-defined matcher port

The `activity` and `workout` packages must not depend on `planned_workout` (wrong layering, import-cycle risk). Each declares a small **port** it owns and depends on:

```go
// in activity (and an equivalent in workout)
type PlanMatcher interface {
    // OnSessionLogged best-effort links a freshly logged session to the
    // planned workout it completes, if any. Errors are logged, not returned
    // to the caller — a matching failure must not fail the ingest.
    OnSessionLogged(ctx context.Context, userID string, kind SessionRef)
}
```

`planned_workout` provides the implementation; it is wired into the activity/workout services at server construction (same seam style the codebase already uses). The hook fires at the two creation points — **running-activity ingest** (TCX upload / Garmin sync) and **workout (lift) creation**. Matching is **best-effort**: a failure logs and the session still saves, mirroring the existing best-effort Google-sync behavior.

### One link path

Extract today's `/complete` handler body (validate → `repo.SetCompletion` → rewrite Google event) into a `planned_workout` **service method**, e.g. `LinkCompletion(ctx, userID, planID, sessionID, kind)`. Both the HTTP complete handler and the auto-matcher call it, so manual and automatic completion are identical downstream (including the Google-event rewrite, which *is* wanted on the live path).

## Unlink + lifecycle integrity

- **Unlink endpoint** (new): `POST /planned-workouts/{id}/unlink`. Reverts `Status → planned`, clears `CompletedSessionID`/`CompletedSessionKind` (new repo method `ClearCompletion`), and re-renders the Google event back to its non-completed form (the inverse of the completion rewrite). This is the false-positive escape hatch. (Chosen over `DELETE …/completion` to match the existing action-verb endpoint style — `/complete`, `/skip`, `/resync`.)
- **Revert on session delete**: when a linked `Activity` or `Workout` is deleted, the plan that pointed at it reverts to `planned` with the link cleared. Implemented via the same `ClearCompletion` path, triggered from the delete flow (through a port, symmetric with `OnSessionLogged`, or a direct reverse-lookup + clear). Keeps the calendar honest when history changes.

## Reverse lookup + display

To show "✓ Completes your planned *Easy Run*" on a session's detail page, add a repo method `GetByCompletedSession(ctx, userID, sessionID, kind) → *PlannedWorkout`. The web run-detail and workout-detail pages call it (via a new `GET` field or a dedicated lookup endpoint) and render the relationship with an **Unlink** action — the natural place to undo a bad match is on the activity you're looking at.

On the plan side, the planned-workout detail page and the calendar `PlannedBanner` already render a completed plan with a "View logged run/workout →" link; this SOW adds the **Unlink** affordance next to it.

Auto-linking is **silent** — no confirmation dialog. The calendar collapsing the planned + actual into one completed banner *is* the feedback. (An optional "Linked to your planned Easy Run" toast on upload is out of scope for v1.)

## Go-migration runner support

The app-db runner (`internal/db/migrate.go`) currently embeds `migrations/*.sql`, applies each file in numeric (`NNN_`) order inside a transaction, and records the version in `schema_migrations`. Extend it to also run **Go migrations** sharing that same version ledger and per-migration transaction:

- Introduce a registered Go-migration type, e.g. `type goMigration struct { Version int; Name string; Run func(ctx context.Context, tx *sql.Tx) error }`, collected in a registry within the `db` package.
- The unified runner merges SQL files and registered Go migrations into one version-ordered list (each version is *either* a SQL file *or* a Go migration), applies pending ones in order, each in its own transaction, each recorded in `schema_migrations`. Existing SQL migrations are unaffected.

This is reusable infrastructure for **every** future migration that needs a data backfill, not a one-off — backfills now version and ship in git history alongside the schema change that necessitates them.

### Constraints on Go migrations (important)

- **Self-contained and frozen.** A Go migration operates directly on its `*sql.Tx` via raw SQL; it must **not** call service-layer code. Migrations must produce the same result forever — calling the evolving matcher service would mean a rebuilt DB reconciles differently after a future code change. The backfill therefore carries its **own copy** of the (small) same-day + kind + nearest-start rule. The live path keeps the shared service matcher; the migration is a frozen snapshot.
- **No network I/O.** Migrations touch only the database.

## Backfill (the first Go migration)

A Go migration (next free version, e.g. `028_backfill_planned_workout_links`) reconciles existing history by **replaying the matcher over already-logged sessions**:

1. Select every running `Activity` and `Workout` that is not already some plan's `CompletedSession`, ordered by `created_at` ascending.
2. For each, run the frozen same-day + kind + nearest-start rule (timezone bucketing via `time.LoadLocation`) against the user's `planned`-status plans, all within the migration transaction.
3. On a match, `UPDATE planned_workouts SET status='completed', completed_session_id=?, completed_session_kind=?`.

Processing oldest-first reproduces what would have happened had the feature been live (the already-completed and nearest-start guards resolve two-a-days identically). It is **idempotent** (already-linked sessions skipped; only `planned` plans are candidates) and **DB-only** (no Google rewrites, no service calls). On a fresh database it is a harmless no-op (no historical data to reconcile).

## Edge cases

| Case | Behavior |
|------|----------|
| No planned session that day/kind | No-op; session stays unlinked. |
| Two plans same day + kind (two-a-day) | Nearest scheduled start wins; the other stays `planned`. |
| Two activities same day, one plan | First logged links; the plan is then `completed`, so the second stays unlinked. |
| Plan already `completed` or `skipped` | Not a candidate; never overwritten. |
| Run plan, user logged a walk | No match (v1 is strictly `running`). |
| Logged-then-renamed run | No re-match needed; name was never load-bearing. |
| Linked session deleted | Plan reverts to `planned`, link cleared. |
| User manually completed already | Existing `/complete` path still works; matcher skips (plan no longer `planned`). |
| Timezone drift | Day bucket computed in the plan's own IANA `Timezone`. |

## Testing

- **Matcher unit tests** (table-driven): same-day match, off-schedule same-day match, two-a-day nearest-start, wrong-kind no-match, already-completed guard, timezone-boundary day bucketing, no-candidate no-op.
- **Ingest integration**: TCX upload and workout create each auto-link the right plan and rewrite the Google event (mocked); ingest still succeeds when matching errors (best-effort).
- **Unlink**: reverts status + clears link + re-renders Google event; revert-on-delete reverts a linked plan.
- **Reverse lookup**: `GetByCompletedSession` returns the linking plan; web detail pages render the relationship + unlink.
- **Go-migration runner**: a Go migration applies once, records its version, is skipped on re-run, and runs in version order interleaved with SQL migrations.
- **Backfill migration**: seeds plans + sessions, asserts correct links, asserts idempotent re-run and fresh-DB no-op.

## Rollout / phasing

The implementation plan should sequence as:

1. **API engine** — matcher + `LinkCompletion` refactor + `PlanMatcher` ports + ingest hooks (live auto-link working).
2. **API integrity** — unlink endpoint + `ClearCompletion` + revert-on-delete + `GetByCompletedSession`.
3. **Runner + backfill** — Go-migration support, then the backfill migration.
4. **Web** — unlink affordances on the plan page / `PlannedBanner`; reverse "completes planned …" display + unlink on run/workout detail.

Phases 1–2 ship the forward behavior; phase 3 cleans up history; phase 4 exposes the controls. The backfill (phase 3) should land only after the matcher rule is finalized in phase 1, since it freezes a copy of that rule.
