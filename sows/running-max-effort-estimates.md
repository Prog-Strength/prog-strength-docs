---
status: draft
repos:
  - prog-strength-api
  - prog-strength-mcp
  - prog-strength-web
  - prog-strength-mobile
  - prog-strength-docs
---

# Running Max-Effort Estimates

**Status**: Draft · **Last updated**: 2026-06-14

## Introduction

We already track running **best efforts** — the fastest actual window a user has run
at each of six standard distances (`1mi`, `2mi`, `5k`, `10k`, `half_marathon`,
`marathon`), surfaced on the Personal Records → Running view. A best effort answers
"what is the fastest I have *ever actually* covered this distance?"

This SOW adds the next question: **"what could I run *right now*, at max effort, for
this distance?"** — a *predicted* time, distinct from the historical best. A runner
who has raced a 5K in 22:14 and a 10K in 47:02 but never raced a mile still has a
knowable mile potential; a runner who set a 5K PR eight months ago and has trained
hard since is likely faster than that stale number suggests. The estimate fills both
gaps.

The estimate is the first piece of **proprietary running intelligence** on the
platform. It is designed from day one to be a single, swappable engine that we expect
to iterate on for accuracy over many releases. Accordingly, the central design
constraint is *legibility and replaceability of the formula*, not just shipping a
number.

The feature ships across four code repos: the engine and API (`prog-strength-api`),
an agent-facing tool (`prog-strength-mcp`), and a new detail view on web and mobile
(`prog-strength-web`, `prog-strength-mobile`).

## Proposed Solution

A new, self-contained **estimation engine** in the API computes a max-effort time for
any standard distance from the user's existing best efforts, optionally anchored
against age/sex running standards. The engine is a pure function behind one interface,
documented by a co-located `README.md`, and versioned so its output is traceable.

The estimate uses a **Bayesian power-law model** in log-space: it fits the user's
personal endurance curve `ln(T) = β₀ + β₁·ln(D)` with a prior that keeps predictions
conservative when data is thin and lets them follow the user's own data as evidence
accumulates. The model returns a point estimate *and* a confidence band, so the UI can
honestly communicate uncertainty.

Two new API endpoints expose a per-distance detail payload (current estimate, the
estimate's evolution over time, the underlying attempts, and stat-tile values) and a
cross-distance summary. The estimate-over-time series is **computed on read** — there
is no materialized derived table — so improving the formula never requires a data
migration or backfill; we redeploy and every graph reflects the new model.

A thin MCP tool forwards the estimate to the agent. New detail views on web and mobile
present stat tiles, a chart (estimate line + confidence band overlaid on the user's
actual attempts), a visually-paginated attempts table, and a switcher to move between
distances. The detail view is reachable from both the Personal Records → Running cards
and a new section on the Progress page.

## Goals and Non-Goals

### Goals

- **A single, swappable estimation engine.** All estimation logic lives behind one
  `Estimator` interface in one package (`internal/running/estimate`). Replacing or
  rewriting the model touches nothing outside that package. A co-located, concise
  `README.md` documents the formula, its assumptions, and how to iterate on it.
- **Accurate, conservative estimates.** Use *all* of a user's best efforts (not just
  one), shrink toward an age/sex standard prior when data is thin, and only let the
  estimate beat the user's actual best as fast as cross-distance evidence and effort
  consistency support. No "crazy extrapolation" from a single fast effort.
- **Honest uncertainty.** Every estimate carries a confidence band that is wide on
  thin/noisy data and narrows as the user logs consistent efforts, plus a `basis`
  label describing what produced the number.
- **Graceful, optional demographics.** The engine works from run attempts alone. Each
  demographic field that *does* exist (today: `height_cm`) sharpens the prior; missing
  fields (age, sex, weight) simply widen it. No profile schema changes in this SOW.
- **Estimate-over-time visualization.** A chart shows how the estimate has evolved
  (back-tested: at each past date, the estimate using only data up to that date),
  overlaid with the user's actual attempt data points as references.
- **A focused detail view.** Stat tiles (headline estimated max-effort time required),
  the chart, a paginated attempts table, and easy navigation between the six standard
  distances — reachable from both Personal Records and the Progress page.
- **Agent access.** The agent can read a user's max-effort estimates via a new MCP
  tool, enabling coaching language ("you're on track to break 22:00 for the 5K").

### Non-Goals

- **No profile schema changes.** Adding `birthdate`/`sex`/profile-weight is explicitly
  deferred (see [`user-profile-and-preferences.md`](user-profile-and-preferences.md)).
  The engine is built to consume them the day they exist, but this SOW does not add
  them. Age and sex are the strongest demographic predictors, so the demographic prior
  is intentionally weak until that lands.
- **No new best-effort computation.** Best efforts are already computed and stored at
  upload time (see [`running-best-efforts.md`](running-best-efforts.md)). This feature
  reads them; it does not change how they are derived.
- **No materialized estimate table.** The estimate-over-time series is computed on
  read. We deliberately avoid a derived table (unlike `exercise_one_rep_max_history`)
  so that iterating on the formula requires no migration or backfill. If read cost ever
  becomes a problem, materialization is a clean later optimization.
- **No LLM in the estimate.** The engine is deterministic math — no model calls, no
  per-request token spend. "Intelligent" here means a good statistical model, not an
  LLM. The MCP tool is a plain forwarder.
- **No VDOT/Daniels table re-implementation.** We may validate against VDOT internally,
  but the shipped engine is our own Bayesian power-law model, not a port of published
  equivalence tables.
- **No new units infrastructure.** Reuse the existing `DistanceUnitProvider` and pace
  formatters on both clients.

## Implementation Details

### 1. The estimation engine (`prog-strength-api/internal/running/estimate/`)

A new, self-contained package. Nothing here imports the HTTP, DB, or activity layers;
it operates purely on plain input structs so it is trivially unit-testable.

```
internal/running/estimate/
  estimator.go      # Estimator interface, EstimateInput / EstimateResult, version const
  riegel_bayes.go   # v1 model: weighted Bayesian power-law, Riegel-seeded prior
  standards.go      # age/sex running-standard tables → demographic prior on the level
  weighting.go      # recency + effort-quality observation weights
  README.md         # the formula, assumptions, and how to iterate (see §2)
  estimator_test.go
  riegel_bayes_test.go
```

**Interface and types** (illustrative shapes):

```go
// EstimatorVersion is stamped onto every result so output is traceable and
// caches/labels can react to model changes. Bump on any behavioral change.
const EstimatorVersion = "1.0.0"

type Estimator interface {
    Estimate(in EstimateInput) EstimateResult
}

type Attempt struct {
    DistanceKey     string    // source distance of the effort ("5k", ...)
    DistanceMeters  float64
    DurationSeconds float64
    AchievedAt      time.Time
    // Quality signals (already available from the activity that produced the effort):
    ActivityDistanceMeters float64 // total distance of the run the effort came from
    // ...any additional signals the weighting heuristic uses
}

type Demographics struct { // every field optional
    Age      *int
    Sex      *string  // "male" | "female"
    WeightKg *float64
    HeightCm *float64
}

type EstimateInput struct {
    TargetDistanceKey    string
    TargetDistanceMeters float64
    Attempts             []Attempt     // ALL distances, not just the target
    Demographics         Demographics  // optional; missing fields widen the prior
    Now                  time.Time     // injected for deterministic recency + back-testing
}

type EstimateResult struct {
    Seconds      float64
    LowerSeconds float64  // confidence band (asymmetric — lognormal)
    UpperSeconds float64
    Basis        string   // "insufficient_data" | "demographic_prior" | "single_effort" | "fitted_curve"
    Confidence   string   // "low" | "medium" | "high" (derived from band width)
    NPoints      int
    NDistances   int
    Version      string   // = EstimatorVersion
}
```

- **Pure & stateless.** `Estimate` takes `Now` as an input rather than reading the
  clock, which makes the back-tested estimate-over-time series (§3) a matter of calling
  the same function with different `Now` and a filtered attempt set — no special path.
- **Versioned.** `EstimatorVersion` is returned on every result and surfaced by the API
  so the UI can note which model produced a number during iteration.

**The v1 model (`riegel_bayes.go`).** Weighted Bayesian linear regression in log-space.
Let `x = ln(D)`, `y = ln(T)`. Model: `y = β₀ + β₁·x + ε`.

- **Prior `β ~ N(m₀, S₀)`:**
  - `β₁` (the fatigue/endurance exponent) is centered at **1.06** (the Riegel
    population value) with a *tight* prior variance. This is the primary conservatism
    knob: it takes several consistent, multi-distance efforts to move the exponent
    meaningfully, which prevents a single fast short effort from implying an absurd
    long-distance time.
  - `β₀` (overall speed level): when demographics are present, centered on an age/sex
    standard — compute a standard time `T_std` at a reference distance `D_ref` for the
    user's age+sex from `standards.go`, then `m₀[β₀] = ln(T_std) − 1.06·ln(D_ref)` with
    a *moderate* (weak) variance so the user's own data dominates quickly. When
    demographics are absent, `β₀` gets a diffuse prior so the level comes entirely from
    the data.
- **Likelihood weights `wᵢ = w_recency · w_quality`:**
  - `w_recency = exp(−Δt / τ)`, `τ ≈ 180 days` (configurable). Longer than the 1RM
    baseline's window because race potential is more persistent and running data is
    sparser.
  - `w_quality ∈ (0, 1]` down-weights efforts likely to be **sub-maximal windows**
    pulled from longer easy runs (a 5K window inside a 15K easy run is weaker evidence
    than a 5K race). v1 heuristic: full weight when the effort distance is a large
    fraction of its source activity's total distance, decaying for small windows of
    long runs. Explicitly a v1 heuristic and a prime iteration target — documented as
    such in the README.
- **Posterior (closed form, conjugate):**
  `S_N⁻¹ = S₀⁻¹ + XᵀWX`, `m_N = S_N(S₀⁻¹m₀ + XᵀWy)`, with `W = diag(wᵢ)/σ²`.
- **Prediction** at `x* = ln(D_target)`: `ŷ = m_Nᵀ[1, x*]`; parameter variance
  `v = [1,x*] S_N [1,x*]ᵀ`. Estimate `T̂ = exp(ŷ)`; band `exp(ŷ ± z·√v)` (asymmetric in
  seconds, which is correct for times). `σ` is fixed in v1 to a constant corresponding
  to a few-percent time error for predictable behavior; documented as tunable.
- **Basis & confidence:**
  - `insufficient_data`: no usable efforts and no demographics → API returns no
    estimate (UI shows an empty/"log a run" state).
  - `demographic_prior`: demographics only → estimate ≈ standard, wide band.
  - `single_effort`: one effort dominates → effectively a Riegel projection from that
    point (exponent stays at the prior).
  - `fitted_curve`: ≥2 efforts across ≥2 distances → exponent informed by the user.
  - `Confidence` is derived from band width relative to the estimate.

This yields exactly the requested behavior: thin data → conservative, prior-anchored,
wide band; rich and consistent data → follows the user's fitted curve with a narrow
band; never a wild extrapolation off one data point.

### 2. The engine README (`internal/running/estimate/README.md`)

A concise, well-formatted document so future contributors — human and agent — can
understand and safely iterate on the engine. Required sections:

- **What it does** — one paragraph: predicted max-effort time per standard distance.
- **The model** — the log-space power-law, with the prior/likelihood/posterior roles
  stated plainly and the key formulas shown.
- **Inputs & outputs** — `EstimateInput` / `EstimateResult` fields and what each means;
  the `basis` and `confidence` states.
- **Assumptions & known limitations** — e.g. best efforts may be sub-maximal windows;
  the v1 quality heuristic; demographics weak until age/sex exist; fixed `σ`.
- **How to iterate** — where the tunable constants live (prior means/variances, `τ`,
  `σ`, the quality heuristic), how to add a demographic factor, and the rule for
  bumping `EstimatorVersion`. This section is the point of the file.

Keep it tight and skimmable — prefer a short table of constants over prose.

### 3. Estimate-over-time — computed on read

The "how my estimate evolved" series is a back-test. For each date on which the user
logged a best effort relevant to the target distance, call `Estimate` with `Now` set
to that date and the attempt set filtered to efforts on or before it. The result is a
series of `{ asOf, seconds, lower, upper }` points showing the estimate stepping as new
evidence arrived. Data volumes per user are small, the estimator is cheap, so this is
computed per request — **no derived table, no backfill, no migration when the formula
changes.** Raw inputs come from the existing repository reads
(`GetUserRunningBestEfforts`, `GetRunningBestEffortHistory`).

### 4. API surface (`prog-strength-api`)

Two new JWT-gated endpoints under the existing `/running` namespace, handled alongside
the current best-effort handlers. Each orchestrates repository reads → `estimate.Estimate(...)`
→ JSON; no new tables.

**`GET /running/max-effort`** — cross-distance summary (powers the PR cards and the
switcher):

```json
{
  "estimator_version": "1.0.0",
  "distances": [
    {
      "distance_key": "5k",
      "distance_label": "5K",
      "distance_meters": 5000,
      "estimate_seconds": 1322,
      "lower_seconds": 1298,
      "upper_seconds": 1351,
      "basis": "fitted_curve",
      "confidence": "high",
      "actual_best_seconds": 1334,
      "actual_best_activity_id": "act_...",
      "actual_best_achieved_at": "2026-02-10T15:04:05Z"
    }
  ]
}
```

**`GET /running/max-effort/{distance_key}`** — detail payload for the view:

```json
{
  "estimator_version": "1.0.0",
  "distance_key": "5k",
  "distance_label": "5K",
  "distance_meters": 5000,
  "estimate": {
    "seconds": 1322, "lower_seconds": 1298, "upper_seconds": 1351,
    "basis": "fitted_curve", "confidence": "high",
    "n_points": 7, "n_distances": 3
  },
  "actual_best": { "seconds": 1334, "activity_id": "act_...", "achieved_at": "2026-02-10T15:04:05Z" },
  "estimate_history": [
    { "as_of": "2025-11-02", "seconds": 1390, "lower_seconds": 1300, "upper_seconds": 1495 },
    { "as_of": "2026-02-10", "seconds": 1322, "lower_seconds": 1298, "upper_seconds": 1351 }
  ],
  "attempts": [
    { "activity_id": "act_...", "achieved_at": "2026-02-10T15:04:05Z",
      "duration_seconds": 1334, "pace_sec_per_km": 266.8, "source": "race_like" }
  ],
  "stats": {
    "estimated_max_effort_seconds": 1322,
    "current_best_seconds": 1334,
    "gap_seconds": -12,
    "confidence": "high",
    "data_summary": "7 efforts across 3 distances"
  }
}
```

Unknown `distance_key` returns the existing `unknown_distance_key` error code.
`insufficient_data` returns `200` with a null `estimate` and an explanatory `basis`,
so clients render an empty state rather than treating it as an error.

### 5. MCP tool (`prog-strength-mcp`)

Add one **forwarder** tool (no LLM spend) so the agent can reference estimates:

- `get_running_max_effort_estimate(distance_key: str | None = None)` — when
  `distance_key` is given, proxies `GET /running/max-effort/{distance_key}`; when
  omitted, proxies the `GET /running/max-effort` summary. Lives in the existing
  `running.py` module next to `get_running_best_efforts()`. JWT is forwarded the same
  way as the existing running tool.

### 6. Web detail view (`prog-strength-web`)

- **Route:** `app/(app)/progress/running/[distanceKey]/page.tsx` — the single shared
  detail view.
- **Entry points (both → this route):**
  - Personal Records → Running: each distance card in `RunningView.tsx` gains a
    `View estimate & progress →` link.
  - Progress page: a new "Running max-effort estimates" section linking in (one entry
    per distance, or a single link to the user's primary distance with the in-view
    switcher handling the rest).
- **View contents:**
  - **Cross-distance switcher** — pills/segmented control (1mi · 2mi · 5K · 10K · Half ·
    Marathon) that swap the active distance without leaving the view (URL-param driven,
    matching existing `?view=` conventions).
  - **Stat tiles** — reuse the `StatCards` pattern: headline **Estimated max-effort
    time** (required), Current best (+ gap), Confidence / data basis, and a
    Trend/projection tile.
  - **Chart** — Recharts: estimate line + shaded confidence band, with actual attempts
    overlaid as reference dots. Reuse `ProgressChart`/`CustomTooltip` patterns.
  - **Attempts table** — visually-paginated table of attempts at this distance (date,
    time, pace, source), reusing the existing table/pagination patterns.
- **API client:** add typed wrappers `getRunningMaxEffortSummary(token)` and
  `getRunningMaxEffort(token, distanceKey)` plus their response types to `lib/api.ts`,
  matching the existing duplicated-client pattern. Units via `DistanceUnitProvider`.

### 7. Mobile detail view (`prog-strength-mobile`)

- **Screen:** new Expo Router screen mirroring the web detail view, linked from the
  running PR cards (`running-prs-view.tsx`) and from the Progress tab.
- **Contents:** same structure — distance switcher (`SegmentedControl`), stat tiles,
  chart (reuse the hand-rolled SVG `TimeSeriesChart` / `progression-chart` to draw the
  estimate line + band + attempt dots), and a paginated attempts list.
- **API client:** add the same `getRunningMaxEffortSummary` / `getRunningMaxEffort`
  wrappers and types to mobile `lib/api.ts`; reuse `lib/units.ts` for pace/distance.

### 8. Tests

- **Go (engine):** unit tests are the core deliverable since the engine is the IP.
  Cover — zero efforts + demographics → `demographic_prior`, wide band; one effort →
  `single_effort` ≈ Riegel projection; multi-distance consistent efforts → exponent
  moves toward data, band narrows; thin/noisy data stays conservative (no wild
  extrapolation); recency down-weights stale efforts; quality heuristic down-weights
  small windows of long runs; band monotonically narrows as consistent data is added;
  determinism given fixed `Now`.
- **Go (API):** handler tests for both endpoints against the memory repository —
  payload shape, `unknown_distance_key`, and the `insufficient_data` 200 case.
- **MCP (Python):** tool test asserting it forwards to the right endpoint with/without
  `distance_key` and passes the JWT.
- **Web/Mobile (TypeScript):** client wrapper + envelope-unwrap tests; a render test
  for the empty/`insufficient_data` state.

### 9. Rollout

Forward-only, no migrations, no backfill (the engine is pure and read-time). Merge
order:

1. **`prog-strength-api`** — engine package + README + two endpoints. Backward
   compatible (purely additive). Deploy first.
2. **`prog-strength-mcp`** — forwarder tool. Depends on the API endpoints existing.
3. **`prog-strength-web`** and **`prog-strength-mobile`** — detail views + entry points
   + client wrappers. Independent of each other; both depend on the API.
4. **`prog-strength-docs`** — this SOW; mark `status: shipped` and add a "Resolved
   during implementation" section once landed.

Revert behavior: removing the clients/endpoints leaves best efforts and Personal
Records untouched, since this feature only reads existing data.

## Open Questions

1. **Demographic standard tables.** `standards.go` needs an age/sex standard at a
   reference distance. Options: (a) embed a small curated table (e.g. age-grading-style
   factors) now, used weakly; (b) ship with the demographic prior effectively disabled
   (diffuse `β₀`) until age/sex are stored, and add the table when the profile fields
   land. *Tentative lean: (b)* — age/sex aren't stored yet, so a table would be dead
   weight; build the hook, defer the data. Revisit when
   [`user-profile-and-preferences.md`](user-profile-and-preferences.md) adds the fields.

2. **Effort-quality signal fidelity.** The v1 quality heuristic uses the
   effort-distance-to-activity-distance ratio. Is that signal sufficient, or should v1
   also factor effort pace vs. the activity's average pace? *Tentative lean:* ship the
   ratio-only heuristic, document it as v1 in the README, and treat pace-ratio as a
   fast follow once we can eyeball real user data.

3. **Progress-page entry granularity.** Should the Progress page link expose all six
   distances, or a single "Running max-effort estimates →" entry that lands on the
   user's most-run distance and relies on the in-view switcher? *Tentative lean:* single
   entry → most-run distance, to keep the strength-focused Progress page uncluttered.

4. **Confidence-band interval width.** What z (≈68% vs ≈90%) should the displayed band
   use? *Tentative lean:* ≈68% (one σ) for a tighter, less alarming visual, with the
   `confidence` label carrying the qualitative read. Easy to tune later.
