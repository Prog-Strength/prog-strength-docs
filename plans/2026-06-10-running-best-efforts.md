# Implementation Plan: Running Best Efforts (Running PRs)

**SOW**: `sows/running-best-efforts.md`
**Date**: 2026-06-10
**Repos**: prog-strength-api, prog-strength-mcp, prog-strength-web, prog-strength-docs

This plan turns the SOW into discrete, subagent-sized tasks. Each task is
implemented by a subagent, then spec-reviewed and code-quality-reviewed
before moving on. Build/test gates are noted per task.

## Reconciled decisions (where the SOW assumed state the repos don't have)

The SOW is broadly aligned with the codebase. Four reconciliations, all
additive and called out in the PRs:

1. **No `/running` route group on the API yet.** Running activities are
   served under `/activities` (the `internal/activity` handler). The SOW
   specifies `GET /running/best-efforts` and
   `GET /running/best-efforts/{distance_key}/history`. We add a small new
   `r.Route("/running", ...)` block inside `activity.Handler.Mount` rather
   than a new package — the data lives in the activity repository, so the
   handler that already owns it is the natural home.

2. **Per-exercise 1RM history reuses the existing repo method.**
   `GET /personal-records/{exercise_id}/history` lives in the
   `internal/workout` handler (which already mounts `GET /personal-records`).
   The workout repository already exposes `ListOneRepMaxHistory`; the new
   endpoint is a thin per-exercise slice over it, returning one point per
   workout (max estimated 1RM across that workout's sets).

3. **TanStack Query has no provider yet.** `@tanstack/react-query` is a
   dependency but no `QueryClientProvider` is mounted and `useQuery` is
   used nowhere. We add a single client `Providers` wrapper at the app
   root (alongside the existing `ToastProvider` / `DistanceUnitProvider`)
   so the personal-records page (and future pages) can use queries. All
   other pages keep their existing manual-fetch code untouched.

4. **MCP package path.** The MCP package lives at
   `src/prog_strength_mcp/`; `register(...)` calls in `server.py` are not
   strictly alphabetized. We insert `running.register(mcp, api)` in a
   sensible position (after `recipes`, before `nutrition`) matching the
   existing domain grouping, and add `tests/test_running_tools.py`.

Internally the API stays metric; pace is derived at request/render time.

## Tasks

### T1 — api: migration 016 + distance set + sweep (prog-strength-api)
The pure core. No handler/repository wiring yet.
- `internal/db/migrations/016_activity_best_efforts.sql`: create
  `activity_best_efforts(activity_id TEXT, distance_key TEXT, duration_seconds REAL)`
  with `PRIMARY KEY (activity_id, distance_key)`,
  `FOREIGN KEY(activity_id) REFERENCES activities(id) ON DELETE CASCADE`,
  `CHECK(distance_key IN ('1mi','2mi','5k','10k','half_marathon','marathon'))`,
  and `CREATE INDEX idx_activity_best_efforts_distance ON activity_best_efforts(distance_key, duration_seconds);`.
  Match the SQL style of `015_activities_generalize.sql`.
- `internal/activity/best_efforts.go`: `StandardDistance{Key,Meters,DisplayName}`,
  the `StandardDistances` slice (exact six entries/values from SOW §Standard
  Distance Set), `const bestEffortsVersion = 1`, the `ActivityBestEffort`
  struct (`DistanceKey string`, `DurationSeconds float64`), and
  `bestEfforts(tps []parsedTrackpoint, targets []StandardDistance) []ActivityBestEffort`
  implementing the two-pointer sweep with right-edge interpolation per
  §Algorithms. Omit a distance whose target exceeds total cumulative
  distance. Document the paused-time and left-edge-bias limitations inline.
- `internal/activity/best_efforts_test.go`: the six pure-function cases in
  SOW §Tests (constant-pace exact multiple; non-exact sampling within ±0.1s;
  fast 5K window inside a slow 10mi run; activity too short; monotonic
  non-strict / zero-distance segment no divide-by-zero; sweep correctness).
- `TestStandardDistances_MatchMigrationCheck` (in `best_efforts_test.go`):
  read `016_*.sql`, extract the `CHECK(distance_key IN (...))` token set,
  assert equality with `{d.Key for d in StandardDistances}`.
- `internal/db/migrate_test.go`: add coverage that 016 applies cleanly on
  an empty DB and on a DB pre-populated via 015.
- Gate: `go test ./internal/activity/... ./internal/db/...`.

### T2 — api: model + summarizer wiring (prog-strength-api)
- `internal/activity/model.go`: add `BestEfforts []ActivityBestEffort` to
  `Activity`.
- `internal/activity/tcx_summarizer.go`: inside the existing
  `if actType == ActivityRunning` block, set
  `a.BestEfforts = bestEfforts(p.Trackpoints, StandardDistances)`. Non-running
  activities leave it nil/empty.
- `internal/activity/tcx_summarizer_test.go`: add an `intervals`-style
  fixture (or reuse `testdata/intervals.tcx`) with a known embedded fast 5K
  and pin the expected `BestEfforts`; assert a walk/cycling activity yields
  no best-effort rows.
- Gate: `go test ./internal/activity/...`.

### T3 — api: repository write + read (prog-strength-api)
- `internal/activity/repository.go`: add
  `GetUserRunningBestEfforts(ctx, userID) ([]RunningBestEffort, error)` and
  `GetRunningBestEffortHistory(ctx, userID, distanceKey) ([]BestEffortPoint, error)`
  to the interface. Define the returned row structs (carry
  `DistanceKey`, `DurationSeconds`, `ActivityID`, `ActivityStartTime`).
- `internal/activity/sqlite_repository.go`: write `activity_best_efforts`
  rows inside the existing `Create` transaction (after trackpoints, before
  the S3 Put / commit). Implement the two read methods. The bests query uses
  `ROW_NUMBER() OVER (PARTITION BY distance_key ORDER BY duration_seconds ASC, start_time ASC)`
  joined to `activities` with `deleted_at IS NULL AND activity_type='running'`,
  filtered to the user. History query: all rows for one `distance_key` for
  that user (live running activities), ordered by `start_time ASC`.
- `internal/activity/memory_repository.go`: mirror the write + both reads so
  handler tests against the in-memory repo stay accurate.
- `internal/activity/sqlite_repository_test.go`: insert activity with bests,
  assert rows persist; assert `GetUserRunningBestEfforts` returns the
  per-distance minimum with correct activity ID; assert tie resolves to the
  earliest `start_time`; assert a walk activity's bests never appear.
- Gate: `go test ./internal/activity/...`.

### T4 — api: best-efforts endpoints + backfill (prog-strength-api)
- `internal/activity/handler.go`: in `Mount`, add
  `r.Route("/running", func(r chi.Router){ r.Get("/best-efforts", h.runningBestEfforts); r.Get("/best-efforts/{distance_key}/history", h.runningBestEffortHistory) })`.
  Implement both handlers reading `userID` from context. DTOs per SOW
  §API Surface: bests entry carries `distance_key`, `distance_label`,
  `distance_meters`, `duration_seconds`, `pace_sec_per_km` (derived
  `duration_seconds / (distance_meters/1000)`), `activity_id`,
  `activity_start_time`; sorted in `StandardDistances` order. History
  response carries `distance_key`, `distance_label`, `distance_meters`,
  `points[]`. `404` `code:"unknown_distance_key"` when the path param is not
  a standard key (use `httpresp.ErrorWithCode`). Empty bests →
  `{"best_efforts": []}`.
- `internal/activity/sqlite_repository.go` (or a `backfill.go`):
  `BackfillActivityBestEfforts(ctx)` — `existing > 0` row-count gate; for
  each live `activity_type='running'` row with zero best-effort rows: fetch
  `tcx_s3_key`, `archiver.Get` the bytes, `parseTCX` + `validate`, run
  `bestEfforts`, insert rows in one tx per activity. Log + skip on
  parse/missing-S3 failure with the activity ID (Open Question #4 lean (a)).
  Requires a `Get` method on the archiver — add to the `Archiver` interface,
  the S3 impl, and `MemoryArchiver` if not present.
- `internal/server/server.go`: call
  `activityRepo.BackfillActivityBestEfforts(context.Background())` in the
  boot sequence next to the existing `Backfill*` calls (sqlite path only).
- `internal/activity/handler_test.go`: happy-path `GET /running/best-efforts`
  for a user with multiple activities incl. `pace_sec_per_km` derivation;
  empty case → `{"best_efforts": []}`; history happy path + unknown key 404.
- Gate: `go build ./... && go test ./...`.

### T5 — api: per-exercise 1RM history endpoint (prog-strength-api)
- `internal/workout/handler.go`: add `r.Get("/personal-records/{exercise_id}/history", h.exerciseOneRMHistory)`.
  Response per SOW §API Surface: `exercise_id`, `exercise_name`, `unit`,
  `points[]` of `{workout_id, performed_at, estimated_1rm}` ascending by
  `performed_at`, one point per workout (max estimated 1RM across that
  workout's sets on the exercise — reuse `ListOneRepMaxHistory` /
  `max_estimated_1rm`). `404` `code:"unknown_exercise_id"` when the slug is
  not in the exercise catalog.
- `internal/workout/handler_test.go`: happy path + unknown-exercise 404.
- Gate: `go test ./internal/workout/...` then `go build ./... && go test ./...`.

### T6 — mcp: running tool (prog-strength-mcp)
- `src/prog_strength_mcp/running.py`: module modeled on `workouts.py` —
  copy the `_auth_header_or_raise()` helper and `register(mcp, api)` shape;
  one `@mcp.tool async def get_running_best_efforts()` with the SOW's
  docstring, forwarding to `api.list_running_best_efforts(auth)` and
  mapping `APIError` → `RuntimeError`.
- `src/prog_strength_mcp/api_client.py`: add
  `list_running_best_efforts(self, auth_header) -> dict[str, Any]` — GET
  `/running/best-efforts`, return the `data` dict verbatim (mirror the
  existing GET wrappers; the endpoint returns `{best_efforts: [...]}` under
  `data`).
- `src/prog_strength_mcp/server.py`: add `running.register(mcp, api)` after
  `recipes.register(...)` and import `running`.
- `tests/test_running_tools.py`: respx-mock the client method to assert the
  GET path/headers and verbatim forwarding; tool-boundary test that a
  missing Authorization header raises `RuntimeError` (sentinel API that must
  not be called), modeled on `test_nutrition_tools.py`.
- Gate: `uv sync && pytest` (or `python -m pytest`).

### T7 — web: api client + Providers + types (prog-strength-web)
- `lib/api.ts`: add `RunningBestEffort`, `RunningBestEffortHistory` /
  `BestEffortPoint`, `ExerciseOneRMHistory` types and
  `listRunningBestEfforts(token)`, `getRunningBestEffortHistory(token, distanceKey)`,
  `getExerciseOneRMHistory(token, exerciseId)` — `unwrap` + bearer header,
  matching house style and the documented response shapes.
- `app/providers.tsx` (new client component) wrapping children in
  `QueryClientProvider` with a module-level `QueryClient`; mount it in
  `app/layout.tsx` around the existing providers (innermost so existing
  providers stay outer, or wherever it composes cleanly).
- `lib/api.test.ts`: unit-test the three new client methods (mock `fetch`,
  consistent with the repo's existing approach — there is no msw setup, so
  follow the existing `vi.mock`/`fetch` test style); assert thrown errors
  carry the API `error` text.
- Gate: `npm run typecheck && npm test`.

### T8 — web: personal-records two-view redesign (prog-strength-web)
- Reshape `app/(app)/personal-records/page.tsx` into a shell + `_components/`
  per SOW §Web — Page Redesign:
  - `_components/ViewSwitcher.tsx` — `role="tablist"` segmented Lifts/Running
    control, URL-backed via `router.replace("/personal-records?view=...")`,
    arrow-key focus traversal, `aria-selected`.
  - `_components/LiftsView.tsx` + `_components/LiftPRCard.tsx` — existing
    `PRCard` body unchanged, wrapped in a card shell adding the expand
    chevron (`aria-expanded`, suppressed when `!hasPR`) and
    `<ProgressionChart kind="lifts" exerciseId=... />` on expand.
  - `_components/RunningView.tsx` + `_components/RunningPRCard.tsx` — mirror
    the lift grid; distance display name title, best time headline
    (`formatDuration`), pace secondary via `formatPace(secPerKm)` + `/unitLabel`,
    "Set on {date}", "View activity →" → `/running/{activity_id}`; empty-state
    cards ("NO RECORD YET") for distances not yet achieved (chevron
    suppressed); no status badge. Render all six standard distances, filling
    achieved ones from the API and the rest as empty states.
  - `_components/ProgressionChart.tsx` — one Recharts line chart for both
    kinds (model on `running/_components/PaceChart.tsx`: stroke `#60a5fa`,
    grid `#27272a`, ticks `#a1a1aa`); lift Y = est. 1RM (unit), running Y =
    duration tick-formatted `m:ss` with a "lower is faster" annotation;
    loading skeleton, error message, single-point empty state. Data
    lazy-fetched via `useQuery(["pr-history", view, id], …, { enabled: expanded })`.
  - `page.tsx` — reads `?view` (default `lifts`), header band with title +
    `ViewSwitcher` + Customize button (Customize hidden on running), routes
    to the active view. Per-view `useQuery({ queryKey: ["personal-records", "lifts"|"running"], enabled: view === ... })`.
    `HeadlineExercisesModal` and the lift card body are otherwise unchanged.
  - Reuse `DistanceUnitProvider` formatters for all running numbers; prefer
    the API's `pace_sec_per_km` when present.
- Tests per SOW §Tests (web), adapted to the repo's `vi.mock` style (no msw):
  `RunningPRCard.test.tsx` (mi/km pace switch, link target, empty-state
  chevron suppressed), `ViewSwitcher.test.tsx` (replace not push, arrow-key
  traversal), `page.test.tsx` (default lifts renders lift cards & running
  endpoint not called; `?view=running` renders running cards & lifts
  endpoint not called; expand fires exactly one history query and re-expand
  reuses cache; Customize hidden on running).
- Gate: `npm run typecheck && npm run lint && npm test && npm run build`.

### T9 — docs: status flip (prog-strength-docs)
- Frontmatter `status: shipped`; body `**Status**: Shipped`,
  `**Last updated**: 2026-06-10`. Commit on `feat/running-best-efforts`.

## Rollout / merge order
api → mcp → web (per SOW §Rollout). The API change is additive; the backfill
runs once on first boot post-deploy (row-count gated). mcp and web each pick
up the new endpoints once the API is live. The docs PR is the ship signal.
