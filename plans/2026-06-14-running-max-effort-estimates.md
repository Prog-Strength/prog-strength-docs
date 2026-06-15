# Implementation Plan: Running Max-Effort Estimates

**SOW**: `sows/running-max-effort-estimates.md`
**Date**: 2026-06-14
**Repos**: prog-strength-api, prog-strength-mcp, prog-strength-web, prog-strength-mobile, prog-strength-docs

This plan turns the SOW into discrete, subagent-sized tasks. Each task is
implemented by a subagent, then spec-reviewed and code-quality-reviewed before
moving on. Build/test gates are noted per task.

## Reconciled decisions (where the SOW assumed state the repos don't have)

The SOW is well aligned with the codebase. Reconciliations, all additive and
called out in the PRs / in the docs PR's "Resolved during implementation":

1. **No `internal/running` package yet.** The running domain currently lives
   in `internal/activity` (best-effort sweep, repo reads, `/running/best-efforts`
   handlers mounted by `activity.Handler.Mount`). The new pure engine lands at
   `internal/running/estimate/` exactly as the SOW specifies; the two new HTTP
   handlers are added to the existing `activity.Handler` next to the
   best-effort handlers (the data lives in the activity repository, so the
   handler that owns it hosts the `/running/max-effort` surface).

2. **Quality signal needs the source activity's total distance.** The engine's
   v1 quality weight uses `effort_distance / source_activity_total_distance`.
   The existing read `GetRunningBestEffortHistory` returns only
   `{ActivityID, ActivityStartTime, DurationSeconds}` — no activity total
   distance. We extend `BestEffortPoint` with one additive field
   `ActivityDistanceMeters float64`, populated in both repos (the memory repo
   already scans the full `Activity`; the sqlite query adds `a.distance_meters`
   to its SELECT). NOTE: the existing best-efforts history handler does a
   struct *conversion* `bestEffortHistoryPointDTO(p)` — adding a field to
   `BestEffortPoint` breaks that conversion, so that handler switches to
   explicit field assignment (the DTO keeps its current three fields).

3. **All-distances attempt assembly.** The estimate uses ALL of a user's best
   efforts across all six distances, not just the target. The handler assembles
   the full attempt set by calling the existing `GetRunningBestEffortHistory`
   once per standard distance (data volumes per user are tiny — at most six
   short series). `GetUserRunningBestEfforts` still supplies each distance's
   `actual_best`. No new repository method is introduced.

4. **Demographics: build the hook, defer the data (Open Question #1, lean b).**
   The age/sex standard table in `standards.go` is keyed on age + sex, neither
   of which is stored yet (`user-profile-and-preferences.md` defers them). The
   engine, `standards.go`, and their unit tests fully support `Demographics`,
   but the v1 *wiring* passes empty `Demographics{}` from the handler, so the
   `β₀` prior is diffuse and the level comes entirely from the user's data —
   exactly the SOW's intended "prior effectively disabled until age/sex land."
   We deliberately do NOT couple the activity handler to the user repository in
   v1 (height alone cannot anchor an age/sex standard, so wiring it changes no
   output). The hook is proven by engine unit tests that pass synthetic
   demographics.

5. **`source` label on attempts.** The detail payload's per-attempt `source`
   field is a qualitative classification derived from the same effort/activity
   distance ratio the quality weight uses: `race_like` when the effort is a
   large fraction of its source activity, else `long_run_window`. Documented in
   the engine README as a v1 heuristic.

6. **Confidence band width (Open Question #4, lean):** displayed band uses
   `z ≈ 1.0` (≈68%, one σ); `Confidence` label carries the qualitative read.
   `EstimatorVersion = "1.0.0"`.

Internally the API stays metric; pace/distance are derived at request/render
time. Mobile has no test runner (typecheck + lint only); web/api/mcp gate on
their suites.

## Engine math (authoritative spec for T1)

Weighted Bayesian linear regression in log-space. For each attempt `i`:
`xᵢ = ln(Dᵢ_meters)`, `yᵢ = ln(Tᵢ_seconds)`. Design row `φᵢ = [1, xᵢ]`.
Model `y = β₀ + β₁·x + ε`.

- **Prior** `β ~ N(m₀, S₀)` with `m₀ = [m_β0, 1.06]` and **diagonal** `S₀`:
  - `β₁` centered at `1.06` (Riegel) with **tight** variance `σ²_β1 = 0.0025`
    (sd 0.05) — the conservatism knob.
  - `β₀`: when a demographic standard is available (age+sex present),
    `m_β0 = ln(T_std) − 1.06·ln(D_ref)` with **moderate** variance
    `σ²_β0 = 0.25`. When absent (v1), **diffuse** `m_β0 = ln(T_seed)`,
    `σ²_β0 = 100` (so data dominates). `T_seed` is a Riegel projection from the
    single fastest available effort to `D_ref` (used only to center a diffuse
    prior; with large variance it barely matters but keeps the math stable when
    there are zero efforts + zero demographics → `insufficient_data`).
- **Likelihood weights** `wᵢ = w_recency · w_quality`, observation noise `σ²`:
  - `w_recency = exp(−Δt_days / τ)`, `τ = 180` (const, documented tunable).
    `Δt_days = (Now − AchievedAt) / 24h`, floored at 0.
  - `w_quality`: let `ratio = clamp(EffortDistance / ActivityDistance, 0, 1)`
    when `ActivityDistance > 0`, else `1.0` (unknown → full weight). v1:
    `w_quality = ratioᵖ`-style ramp — full weight (`1.0`) when `ratio ≥ 0.9`,
    linearly decaying to a floor `0.25` as `ratio → 0`. Exact: 
    `w_quality = 0.25 + 0.75 · clamp((ratio − 0.0)/(0.9 − 0.0), 0, 1)`.
  - `σ²` fixed const `0.0009` (sd 0.03 in log-time ≈ ~3% time error).
- **Posterior** (conjugate, closed form):
  `S_N⁻¹ = S₀⁻¹ + Σᵢ (wᵢ/σ²) φᵢφᵢᵀ`, 
  `m_N = S_N (S₀⁻¹ m₀ + Σᵢ (wᵢ/σ²) φᵢ yᵢ)`. (2×2 — invert by hand.)
- **Prediction** at `x* = ln(D_target)`, `φ* = [1, x*]`:
  `ŷ = m_Nᵀ φ*`; parameter variance `v_param = φ*ᵀ S_N φ*`;
  predictive variance `v = v_param + σ²`. Point `T̂ = exp(ŷ)`;
  band `[exp(ŷ − z·√v), exp(ŷ + z·√v)]` with `z = 1.0` (asymmetric in seconds).
- **Basis** (priority order):
  - `insufficient_data`: 0 usable efforts AND no demographic standard → result
    with zero `Seconds` and `Basis="insufficient_data"`; API renders null
    estimate (200).
  - `demographic_prior`: 0 efforts, demographics present → estimate ≈ standard,
    wide band.
  - `single_effort`: efforts span exactly one distinct distance (exponent stays
    at prior → effectively a Riegel projection).
  - `fitted_curve`: ≥2 efforts across ≥2 distinct distances.
- **Confidence** from relative band half-width `h = (Upper − Lower) / (2·T̂)`:
  `high` if `h ≤ 0.04`, `medium` if `h ≤ 0.10`, else `low`.
- **NPoints** = usable efforts count; **NDistances** = distinct distance count.
- **Determinism**: no clock reads; `Now` is an input. Pure function.

## Tasks

### T1 — api: estimation engine package + README (prog-strength-api)
The pure IP core. Nothing imports HTTP/DB/activity layers.
- `internal/running/estimate/estimator.go`: `const EstimatorVersion = "1.0.0"`;
  `Estimator` interface (`Estimate(EstimateInput) EstimateResult`); the
  `Attempt`, `Demographics`, `EstimateInput`, `EstimateResult` structs exactly
  per SOW §1 (Attempt carries `DistanceKey, DistanceMeters, DurationSeconds,
  AchievedAt, ActivityDistanceMeters`). Add a `NewEstimator() Estimator`
  constructor returning the v1 model.
- `internal/running/estimate/riegel_bayes.go`: the v1 model implementing the
  **Engine math** section above (2×2 conjugate posterior, basis/confidence
  derivation, asymmetric band). Constants (`priorBeta1Mean=1.06`,
  `priorBeta1Var=0.0025`, `priorBeta0Var=0.25`, `diffuseBeta0Var=100`,
  `tauDays=180`, `obsVar=0.0009`, `bandZ=1.0`, quality floor `0.25`, ratio knee
  `0.9`) as named package consts grouped at top, each commented as tunable.
- `internal/running/estimate/standards.go`: `Demographics → (m_β0, σ²_β0, ok)`.
  v1 returns `ok=false` unless `Age != nil && Sex != nil` (table keyed on
  age/sex). Include a tiny, clearly-marked placeholder standards lookup used
  ONLY when age+sex present (so tests can exercise the `demographic_prior`
  path); height refines weakly when present. Documented as deferred data.
- `internal/running/estimate/weighting.go`: `recencyWeight(now, achievedAt)`
  and `qualityWeight(effortMeters, activityMeters)` pure helpers + the
  `source` classifier `classifySource(ratio) string` (`race_like` /
  `long_run_window`).
- `internal/running/estimate/README.md`: the required sections per SOW §2 —
  *What it does*, *The model* (formulas), *Inputs & outputs* (field-by-field,
  basis/confidence states), *Assumptions & known limitations* (sub-maximal
  windows; v1 quality heuristic; demographics weak until age/sex; fixed σ),
  *How to iterate* (a short **table of constants** with file/const/role, how to
  add a demographic factor, the `EstimatorVersion` bump rule). Tight, skimmable.
- `internal/running/estimate/estimator_test.go` + `riegel_bayes_test.go`: the
  SOW §8 (Go engine) cases — zero efforts + demographics → `demographic_prior`
  wide band; one effort → `single_effort` ≈ Riegel projection (assert close to
  `T1·(D2/D1)^1.06`); multi-distance consistent efforts → exponent moves toward
  data and band narrows; thin/noisy data stays conservative (no wild
  extrapolation off one fast short effort); recency down-weights stale efforts;
  quality heuristic down-weights small windows of long runs; band monotonically
  narrows as consistent data is added; determinism given fixed `Now`; zero
  efforts + no demographics → `insufficient_data`.
- Gate: `gofmt -l`, `go vet ./internal/running/...`,
  `go test ./internal/running/...`.

### T2 — api: repo read extension + two endpoints + handler tests (prog-strength-api)
Wire the engine to HTTP. Orchestration only; no new tables.
- `internal/activity/repository.go`: add `ActivityDistanceMeters float64` to
  `BestEffortPoint` (doc-comment it as the source activity's total distance,
  for the estimator quality weight).
- `internal/activity/memory_repository.go`: in `GetRunningBestEffortHistory`
  populate `ActivityDistanceMeters: a.DistanceMeters`.
- `internal/activity/sqlite_repository.go`: add `a.distance_meters` to the
  `GetRunningBestEffortHistory` SELECT and scan into the new field.
- `internal/activity/handler.go`:
  - Fix the now-broken conversion: replace `bestEffortHistoryPointDTO(p)` with
    explicit field assignment (DTO unchanged — keeps its 3 fields).
  - In `Mount`'s `/running` group add
    `r.Get("/max-effort", h.runningMaxEffort)` and
    `r.Get("/max-effort/{distance_key}", h.runningMaxEffortDetail)`.
  - Implement a private helper `assembleAttempts(ctx, userID) ([]estimate.Attempt, error)`
    that loops `StandardDistances`, calls `GetRunningBestEffortHistory` per
    distance, and builds `estimate.Attempt{DistanceKey, DistanceMeters=d.Meters,
    DurationSeconds, AchievedAt, ActivityDistanceMeters}`.
  - `runningMaxEffort` (summary): assemble attempts once; for each
    `StandardDistance` call the engine with `Now = time.Now().UTC()` (inject via
    a handler field `now func() time.Time` defaulting to `time.Now`, so tests
    are deterministic — mirror MemoryRepository's `nowFunc` style); also fetch
    `GetUserRunningBestEfforts` for `actual_best_*`. Emit the SOW §4 summary
    JSON (`estimator_version`, `distances[]`). When basis is
    `insufficient_data`, still emit the row with null estimate fields
    (pointers) but populate `actual_best_*` if present.
  - `runningMaxEffortDetail`: validate `distance_key` via
    `standardDistanceByKey` → 404 `unknown_distance_key` (reuse existing code).
    Assemble attempts; compute current estimate; compute `estimate_history` by
    back-testing: the sorted distinct `AchievedAt` dates of attempts AT the
    target distance, each → `Estimate(Now=date, attempts filtered ≤ date across
    ALL distances)` → `{as_of(YYYY-MM-DD), seconds, lower_seconds, upper_seconds}`;
    build `attempts[]` (this distance only: activity_id, achieved_at,
    duration_seconds, pace_sec_per_km = dur/(meters/1000), source via
    `classifySource`); `actual_best` from `GetUserRunningBestEfforts`; `stats`
    block (estimated_max_effort_seconds, current_best_seconds, gap_seconds =
    estimate−best, confidence, data_summary "N efforts across M distances").
    `insufficient_data` → 200 with `estimate: null` and explanatory `basis` in
    stats; clients render empty state.
- Use `httpresp.OK` / `httpresp.ErrorWithCode`. DTO numerics that can be absent
  are pointers (rendered `null`).
- `internal/activity/handler_test.go`: both endpoints against the memory repo —
  summary payload shape incl. a `fitted_curve` distance and the
  `actual_best`/estimate coexistence; detail payload shape incl.
  `estimate_history` step count and `attempts` ordering; `unknown_distance_key`
  404; `insufficient_data` 200 with null estimate. Add a `seedRunWithEfforts`-
  style helper variant if needed for multi-distance seeding.
- Gate: `gofmt -l`, `go build ./...`, `go vet ./...`,
  `go test ./internal/activity/... ./internal/running/...` then `go test ./...`.

### T3 — mcp: forwarder tool + client + tests (prog-strength-mcp)
- `src/prog_strength_mcp/api_client.py`: add
  `get_running_max_effort_estimate(self, auth_header, *, distance_key=None) -> dict[str, Any]`
  — GET `/running/max-effort` when `distance_key` is None, else
  `/running/max-effort/{distance_key}` (path-segment, `quote` the key);
  forward the `Authorization` header; `_raise_for_status`; return
  `resp.json().get("data")` (dict, else `{}`). Mirror `list_running_best_efforts`.
- `src/prog_strength_mcp/running.py`: add inside `register()` a
  `@mcp.tool async def get_running_max_effort_estimate(distance_key: str | None = None)`
  with a clear docstring (predicted max-effort time per standard distance;
  distinct from best efforts; omit `distance_key` for the cross-distance
  summary, pass one of `1mi/2mi/5k/10k/half_marathon/marathon` for the detail).
  `_auth_header_or_raise()`, forward to the client, map `APIError` → `RuntimeError`.
- `tests/test_running_tools.py`: add respx client tests (summary path with no
  key; detail path with key; auth header forwarded; verbatim `data` forwarding;
  APIError surfacing) and a tool-boundary test (missing auth → `RuntimeError`
  before any HTTP via an `_ExplodingAPI` sentinel; APIError → `RuntimeError`
  with status), modeled on the existing best-efforts tests.
- Gate: `uv sync` then `uv run pytest`; `uv run ruff check`.

### T4 — web: api client + detail view + entry points + tests (prog-strength-web)
- `lib/api.ts`: add types `RunningMaxEffortSummary` /
  `RunningMaxEffortDistanceSummary`, `RunningMaxEffortDetail` (+ `estimate`
  nullable, `actual_best` nullable, `estimate_history[]`, `attempts[]`, `stats`)
  and wrappers `getRunningMaxEffortSummary(token)` (GET `/running/max-effort`)
  and `getRunningMaxEffort(token, distanceKey)` (GET
  `/running/max-effort/{distanceKey}`), using `unwrap` + bearer header, matching
  the existing running wrappers' house style. `basis`/`confidence` are strings.
- Route `app/(app)/progress/running/[distanceKey]/page.tsx` (client) +
  `_components/`:
  - `DistanceSwitcher.tsx` — segmented pills (1mi·2mi·5K·10K·Half·Marathon),
    URL-param driven (`router.replace` matching `?view=` conventions), routing
    between `[distanceKey]` segments.
  - `EstimateStatTiles.tsx` — reuse the `StatTile` pattern: headline **Estimated
    max-effort time** (required), Current best (+ gap), Confidence / data basis,
    Trend/projection tile.
  - `EstimateChart.tsx` — Recharts `ComposedChart`/`AreaChart`: shaded
    confidence band (gradient `Area` between lower/upper, model on
    `ElevationChart` gradient), estimate `Line`, actual attempts as overlaid
    reference dots (`Scatter` or `Line` with `dot` + `stroke="none"`); reuse the
    `#60a5fa`/grid `#27272a`/tick `#a1a1aa` palette and a `CustomTooltip`. Y axis
    formats `m:ss`, "lower is faster" annotation; loading/empty/single-point
    states.
  - `AttemptsTable.tsx` — visually-paginated table (date, time, pace, source)
    reusing the `PaginationFooter` pattern (PAGE_SIZE ≈ 10).
  - `page.tsx` — reads `[distanceKey]` param, `useQuery(["running-max-effort",
    distanceKey], ...)` for detail + the switcher; `insufficient_data`/null
    estimate → empty "log a run" state; units via `useDistanceUnit`.
- Entry points:
  - `app/(app)/personal-records/_components/RunningPRCard.tsx`: add a
    `View estimate & progress →` `Link` to `/progress/running/${distanceKey}`
    (shown for all six cards, including empty-state distances).
  - `app/(app)/progress/page.tsx`: add a "Running max-effort estimates" section
    — single entry → the user's most-run distance with the in-view switcher
    handling the rest (Open Question #3 lean). Default to `5k` if no runs.
- `lib/api.test.ts`: wrapper tests (summary + detail; bearer header; null
  estimate unwrap) in the repo's `vi.stubGlobal("fetch", …)` style. A render
  test for the `insufficient_data`/empty state (Recharts stubbed via `vi.mock`).
- Gate: `npm run typecheck && npm run lint && npm test && npm run build`.

### T5 — mobile: api client + detail screen + entry points (prog-strength-mobile)
- `lib/api.ts`: add the same `RunningMaxEffort*` types and
  `getRunningMaxEffortSummary(token)` / `getRunningMaxEffort(token, distanceKey)`
  wrappers (mirror the existing running wrappers + `unwrap`). `basis`/`confidence`
  are strings.
- New screen `app/(tabs)/activities/running-estimate/[distanceKey].tsx`
  mirroring the web view: `SegmentedControl` distance switcher; `StatTile` grid
  (estimated max-effort time, current best + gap, confidence/basis); a chart
  reusing/extending `components/charts/time-series-chart.tsx` to draw the
  estimate line + confidence band (SVG `Polygon`/`Path` for the band) + attempt
  dots; a paginated attempts list (`FlatList` or sliced view). Units via
  `lib/units.ts` (`formatRunDuration`, `formatPace`, `formatDistance`) and the
  user's `distance_unit` from `useProfile()`. `insufficient_data`/null →
  empty state.
- Register the route in `app/(tabs)/activities/_layout.tsx`
  (`<Stack.Screen name="running-estimate/[distanceKey]" options={{ title: "Max-Effort Estimate" }} />`).
- Entry points: `components/progress/running-prs-view.tsx` — add a
  `View estimate →` `Pressable` → `router.push('/activities/running-estimate/${distance_key}')`
  on each distance row; and a link/section from the Progress tab to the
  most-run distance.
- Gate: `npm run typecheck && npm run lint` (no test runner in this repo).

### T6 — docs: status flip (prog-strength-docs)
- Frontmatter `status: shipped`; body `**Status**: Shipped`,
  `**Last updated**: 2026-06-14`. Add a short "Resolved during implementation"
  section capturing reconciliations 1–6. Commit on `feat/running-max-effort-estimates`.

## Rollout / merge order
api → mcp → (web ∥ mobile) per SOW §9. The API change is purely additive
(no migration, engine is pure/read-time). mcp depends on the API endpoints;
web and mobile are independent of each other and both depend on the API. The
docs PR is the ship signal.
