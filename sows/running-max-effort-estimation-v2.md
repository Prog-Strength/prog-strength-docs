---
status: draft
repos:
  - prog-strength-api
  - prog-strength-mcp
  - prog-strength-web
  - prog-strength-mobile
  - prog-strength-docs
---

# Running Max-Effort Estimation Engine v2

**Status**: Draft · **Last updated**: 2026-07-05

## Introduction

The shipped max-effort estimator ([`running-max-effort-estimates.md`](running-max-effort-estimates.md))
answers a simple question: *if I raced this distance today, all-out, what time should I
expect?* That number should feel trustworthy. Right now it often does not.

The failure mode is concrete and reproducible. A runner with a logged 5K best of **23:06**
(race-like effort) sees a current max-effort estimate of **29:43** — nearly seven minutes
slower. A logged mile best of **7:20** yields an estimate of **8:34**. The estimate-over-time
chart trends *away* from the logged-best line even when the underlying best has not
regressed. The UI labels these estimates "High" confidence, which makes the mismatch
worse: the product is confidently wrong in the direction users notice first.

This violates a basic invariant users carry in their heads: **a max-effort potential at
distance D cannot be slower than the fastest time they have already run at D.** They have
already proved they can cover that distance at least that fast; an "all-out today" forecast
should be at or above that demonstrated capability (equal when the logged best was truly
maximal, faster when the logged best was likely sub-maximal — e.g. a fast window inside
an easy long run). The v1 engine frequently predicts the opposite because it treats every
historical best-effort row at every distance as equally valid regression evidence, lets
sub-maximal long-run windows and stale slower PRs pull the fitted curve upward (slower),
and never applies a per-distance floor from the user's current logged best.

v1 was the right first ship: a swappable pure engine, versioned output, honest bands, and
a co-located README. This SOW is the methodical second pass — refactor the evidence model,
restore the logged-best floor invariant, enrich effort-quality signals (pace ratio and,
where HR data exists, time-in-zone during the effort window), wire demographics the engine
already accepts, and expand the engine README into the living implementation guide for
future accuracy iterations. API response shapes stay stable where possible; behavioral
change is concentrated in `internal/running/estimate/` and the attempt assembly path.

## Proposed Solution

Ship **estimator v2** (`EstimatorVersion = "2.0.0"`) as a deliberate refactor inside the
existing `internal/running/estimate/` package — same `Estimator` interface, same read-time
computation model (no materialized estimate table), but a revised evidence set, weighting
policy, post-processing invariants, and documentation.

At a high level, v2 makes four moves:

1. **Evidence selection** — feed the engine one *current-best anchor* per standard distance
   (from `GetUserRunningBestEfforts`) plus optional *recent supporting* rows, instead of
   the full ascending history at every distance. Historical slower bests at the same
   distance are not equal peers to the current best; they are either excluded or
   down-weighted below a configurable gap threshold.
2. **Effort-quality signals** — extend the v1 distance-ratio heuristic with a **pace-ratio**
   signal (window pace vs. source activity average pace) and, when trackpoint HR is
   available, a **high-intensity fraction** during the winning window (zones 4–5 under the
   existing `hrzones` model). Sub-maximal windows earn less precision; race-like,
   high-HR windows earn full weight.
3. **Logged-best floor** — after the Bayesian prediction, clamp the point estimate and
   confidence band at each distance to be **no slower than** the user's logged best at
   that distance (when one exists). Expose a new result flag (`floored_at_logged_best`) so
   the UI and agent can explain when the raw model sat above the demonstrated capability.
4. **Demographics wiring** — pass profile fields into `EstimateInput.Demographics`:
   `height_cm` (already stored), plus new optional `birthdate` and `sex` on the user
   record so age and sex can anchor the level prior. Replace the placeholder standards
   table with a small, documented age/sex reference table (age-grading-style, weak prior)
   and expand the engine README to document every constant, signal, invariant, and
   iteration workflow.

The handler layer (`assembleAttempts`, max-effort endpoints) grows to enrich attempts with
pace/HR quality inputs and to pass demographics; clients receive the same endpoints with
bumped `estimator_version`, optional new metadata fields, and materially different (more
trustworthy) numbers. No LLM involvement; MCP remains a forwarder with updated tool
description noting v2 semantics.

## Goals and Non-Goals

### Goals

- **Restore the logged-best floor invariant.** For any standard distance where the user
  has a logged best, `estimate_seconds ≤ actual_best_seconds` (within a documented
  sub-second tolerance for floating-point rounding only).
- **Refactor the evidence model** so current bests anchor the curve and historical slower
  rows cannot drag estimates above demonstrated capability at the target distance.
- **Enrich effort-quality weighting** with pace ratio and HR zone fraction (zones 4–5)
  during the effort window, reusing the existing `hrzones` engine where HR trackpoints
  exist.
- **Wire demographics into estimation** — pass `height_cm`, computed age from
  `birthdate`, and `sex` from the user profile; replace the placeholder standards table
  with a documented reference table keyed on age + sex.
- **Expand the engine README** (`internal/running/estimate/README.md`) into the
  authoritative implementation guide: model math, evidence policy, invariants, constants
  table, evaluation fixtures, and the rule for bumping `EstimatorVersion`.
- **Add a regression fixture suite** in the engine package with golden scenarios,
  including the owner-reported 5K/mile mismatch pattern, so future tuning cannot re-break
  the floor invariant silently.
- **Bump `EstimatorVersion` to `2.0.0`** and surface it on existing max-effort API
  responses; add optional metadata (`floored_at_logged_best`, enriched attempt `source`
  labels) without breaking existing clients.
- **Minimal profile schema extension** — add nullable `birthdate` and `sex` to `users`,
  editable from web Settings, so age/sex priors are real rather than permanently
  disabled.

### Non-Goals

- **Replacing the Riegel/Bayesian curve shape entirely.** v2 retunes evidence and
  invariants around the existing log-space power-law fit; a wholly different model (VDOT
  tables, critical-power, ML) is out of scope unless evaluation shows the v2 refactor
  cannot meet the floor invariant without it (file separately).
- **Materializing estimate history.** Estimates remain computed on read; no derived table
  or backfill (same rationale as v1).
- **Changing best-effort extraction.** The sweep algorithm and `activity_best_efforts`
  table semantics are unchanged except for optional new nullable columns to locate the
  winning window for HR/pace enrichment (see Implementation Details).
- **New standard distances.** Still the fixed six (`1mi`, `2mi`, `5k`, `10k`,
  `half_marathon`, `marathon`).
- **Mobile Settings for birthdate/sex.** Web Settings only for the new profile fields;
  mobile continues to consume estimates via existing endpoints (demographics improve when
  the user sets them on web).
- **Agent-side reasoning changes.** No new agent intent; MCP tool shape stays the same
  aside from description/version notes.
- **Guaranteeing estimates faster than logged bests.** The floor caps how *slow* an
  estimate may be; beating a logged best requires evidence (cross-distance projection or
  high-quality sub-maximal signal), not a forced uplift.

## Implementation Details

### Problem diagnosis (v1 → v2)

Document in the engine README and unit-test comments. The shipped v1 path
(`assembleAttempts` → `GetRunningBestEffortHistory` per distance) feeds **every**
historical best-effort row into the regression. For a runner with one true 5K PR (23:06)
and several earlier, slower 5K bests (27:59, 26:51, …), v1 treats those slower rows as
ongoing evidence at the same distance, pulling `β₀` toward slower times. Combined with:

- `qualityWeight` still giving material weight (≥ `qualityFloor` = 0.25) to long-run
  windows;
- recency decay that does not zero out stale slower same-distance rows;
- no post-fit clamp against `GetUserRunningBestEfforts`;

…the fitted curve at the target distance lands far above the logged-best line. The
estimate-over-time back-test re-runs this on each historical date, so the chart can
trend slower even as the user's best improves.

v2 fixes the *evidence set* and adds the *floor*, rather than only retuning constants.

### Data Model

#### User profile (`prog-strength-api` — `users` table)

| Column | Type | Description |
| --- | --- | --- |
| `birthdate` | `TEXT` (ISO date `YYYY-MM-DD`) | Optional. Used to compute integer age at `EstimateInput.Now` for standards lookup. Nullable — missing birthdate disables age anchoring (sex-only prior if sex present, else diffuse). |
| `sex` | `TEXT` | Optional. `"male"` or `"female"` (same enum as engine `Demographics.Sex`). Nullable. Validated on PATCH; invalid values rejected. |

`height_cm` already exists and is wired into PATCH/GET `/me`; no schema change.

Migration: one new SQL migration adding nullable columns with no backfill requirement.

#### Best-effort window bounds (optional but recommended for HR signal)

| Column | Type | Description |
| --- | --- | --- |
| `window_start_elapsed_seconds` | `REAL` | Nullable. Elapsed seconds into the source activity where the winning window starts. Populated at ingest by the existing sweep in `best_efforts.go`. |
| `window_end_elapsed_seconds` | `REAL` | Nullable. Elapsed seconds where the window ends (interpolated edge included). |

These columns are written on new imports and backfilled for existing rows on next boot
(same pattern as the original best-efforts backfill). Nullable so old rows still work;
HR enrichment skips when null.

### Write Path

- **Profile** — `PATCH /me` accepts optional `birthdate` and `sex` with validation
  (birthdate parseable, not in the future, age within 10–100; sex ∈ {`male`,`female`}).
  Clearing: omit field (PATCH partial-update semantics); do not add explicit null-clear for
  v2 unless the existing profile PATCH pattern already supports it for other fields.
- **Best-effort ingest** — extend `bestEfforts` / summarizer to record
  `(window_start_elapsed_seconds, window_end_elapsed_seconds)` alongside
  `duration_seconds` when the sweep finds a window. Existing activities backfill window
  bounds by re-parsing archived TCX once (same boot-time backfill gate as v1 best efforts).

### The estimation engine v2 (`prog-strength-api/internal/running/estimate/`)

Keep the package layout; refactor in place. Target file responsibilities:

```
internal/running/estimate/
  estimator.go       # interface, types, EstimatorVersion = "2.0.0", new result fields
  riegel_bayes.go    # v2 fit + floor post-process (or riegel_bayes_v2.go if cleaner)
  evidence.go        # NEW: anchor selection, history filtering policy
  weighting.go       # recency + quality (distance ratio, pace ratio, HR fraction)
  standards.go       # replaced reference table + demographicLevelPrior
  fixtures_test.go   # NEW: golden scenarios including owner mismatch cases
  README.md          # expanded implementation guide (see § Engine README)
  *_test.go
```

#### Extended types

```go
type Attempt struct {
    DistanceKey            string
    DistanceMeters         float64
    DurationSeconds        float64
    AchievedAt             time.Time
    ActivityDistanceMeters float64
    ActivityAvgPaceSecPerKm *float64 // NEW: source activity average pace; nil = unknown
    WindowStartElapsed     *float64 // NEW: seconds; nil = unknown
    WindowEndElapsed       *float64 // NEW: seconds; nil = unknown
    IsCurrentBestAtDistance bool    // NEW: true for the per-distance anchor row
}

type EstimateInput struct {
    TargetDistanceKey    string
    TargetDistanceMeters float64
    Attempts             []Attempt
    Demographics         Demographics
    Now                  time.Time
    LoggedBestSeconds    *float64 // NEW: current best at target distance; drives floor
}

type EstimateResult struct {
    Seconds               float64
    LowerSeconds          float64
    UpperSeconds          float64
    RawSeconds            float64 // NEW: pre-floor model output (0 if insufficient)
    FlooredAtLoggedBest   bool    // NEW: true when floor clamp applied
    Basis                 string  // existing + optional "logged_best_floor"
    Confidence            string
    NPoints               int
    NDistances            int
    Version               string
}
```

`RawSeconds` and `FlooredAtLoggedBest` are exposed on the detail endpoint's `estimate`
block and optionally on summary rows so the UI can show "model said X, floored to your
logged best" when useful. Summary cards may omit `raw_seconds` for brevity.

#### Evidence policy (`evidence.go`)

Replace "all history rows" with:

1. **Anchors (required when present):** for each standard distance, include exactly one
   row — the current best from `GetUserRunningBestEfforts` — marked
   `IsCurrentBestAtDistance = true`. These always enter the fit with a configurable
   minimum weight floor (`anchorMinWeight`, default `1.0`).
2. **Supporting history (optional):** from `GetRunningBestEffortHistory`, include a row
   *only if* it is not the anchor *and* its duration is within `historyMaxGapPct` of the
   anchor at that distance (default **3%** slower than anchor — tune in fixtures). Rows
   beyond the gap are excluded entirely (they are superseded PRs, not current capability
   evidence).
3. **Cross-distance evidence:** anchors at *other* distances always participate (subject
   to quality weights). This preserves v1's ability to project an unrun distance from
   raced ones.

`assembleAttempts` implements this policy; the engine stays pure.

#### Weighting (`weighting.go`)

Combined precision weight per attempt:

```
w = w_recency · w_distance_ratio · w_pace_ratio · w_hr_intensity · w_anchor_boost
```

| Factor | v2 behavior |
| --- | --- |
| `w_recency` | Same exponential decay (`tauDays`, default 180). Applied to supporting history; anchors at current best use `AchievedAt` of the record-setting activity. |
| `w_distance_ratio` | v1 ratio heuristic (unchanged constants unless fixtures dictate otherwise). |
| `w_pace_ratio` | NEW. `window_pace / activity_avg_pace`. Near 1.0 → full weight; >> 1.0 (window slower than run average) → down-weight toward a floor. Requires `ActivityAvgPaceSecPerKm`. |
| `w_hr_intensity` | NEW when window bounds + HR trackpoints available. Compute fraction of window time in zones 4–5 using the user's `hrzones` reference for that activity date. Low fraction → down-weight (easy-run window); high fraction → 1.0. When HR unavailable, factor = 1.0 (neutral). |
| `w_anchor_boost` | NEW. Multiplier `anchorBoost` (default 2.0) when `IsCurrentBestAtDistance`. |

Document all constants in the README constants table.

#### Logged-best floor (post-process)

After computing `(Seconds, Lower, Upper)` from the posterior prediction:

```
if LoggedBestSeconds != nil && *LoggedBestSeconds > 0 {
    RawSeconds = Seconds
    if Seconds > *LoggedBestSeconds {
        Seconds = *LoggedBestSeconds
        FlooredAtLoggedBest = true
    }
    if LowerSeconds > *LoggedBestSeconds {
        LowerSeconds = *LoggedBestSeconds
    }
    // Upper unchanged — uncertainty may still extend slower
}
```

Set `Basis` to append or map `"logged_best_floor"` when floored (document precedence in
README). Unit tests must assert the invariant for every golden fixture where a logged best
exists.

#### Demographics (`standards.go`)

- Replace placeholder constants with a small embedded table: reference 5K times by age band
  × sex (e.g. 5-year bands, 20–70), sourced from published age-grading / WMA-style factors
  (document provenance in README; values are weak priors, not medical claims).
- `demographicLevelPrior` returns `ok` when sex is present; age refines the band when
  birthdate is present; height applies the existing small multiplicative refinement.
- Handler computes `Age` at `Now` from `birthdate`.

### Engine README (`internal/running/estimate/README.md`)

Expand into the long-lived implementation guide. Required sections (update in this SOW's
implementation PR, not a separate doc task):

1. **What it does** — one paragraph; link to v1 SOW for product context.
2. **Version history** — table: v1.0.0 (shipped baseline) → v2.0.0 (this refactor).
3. **The model** — log-space Riegel Bayesian fit (keep v1 formulas; note what changed
   in v2 vs v1).
4. **Evidence policy** — anchors vs supporting history; exclusion rule; diagram optional.
5. **Weighting signals** — table of each factor, formula, defaults, and "what to tune
   when estimates feel too conservative/aggressive."
6. **Invariants** — **logged-best floor** stated as a hard rule; when it applies; what
   `FlooredAtLoggedBest` means for UX.
7. **Demographics** — which profile fields map to `Demographics`; standards table
   provenance; behavior when fields missing.
8. **Inputs & outputs** — full field glossary including new attempt/result fields.
9. **Assumptions & known limitations** — sub-maximal windows, missing HR, single-user
   tuning, etc.
10. **Evaluation fixtures** — how to run `go test ./internal/running/estimate/...`, list
    of named scenarios (`Owner5KMismatch`, `MileFloor`, …) and what each asserts.
11. **How to iterate** — constants table (all tunables), bump-version rule, checklist
    before merging formula changes.

This README is the document the owner will edit across future accuracy passes; keep it
skimmable (tables over prose) but complete enough that an agent can retune without
re-reading the SOW.

### Handler / API surface (`prog-strength-api`)

**Attempt assembly**

- Refactor `assembleAttempts` per evidence policy.
- Join activity average pace (and window bounds when columns populated) when building
  `estimate.Attempt`.
- Load user profile once per request; map to `estimate.Demographics`.
- Pass `LoggedBestSeconds` for the target distance into `EstimateInput` on detail; summary
  endpoint runs per-distance with each distance's logged best.

**HR enrichment (read path)**

For attempts where window bounds exist, load trackpoints for the source activity (reuse
repository read already used by activity detail — batch or cache within the request to
avoid N+1 explosion; typical user has ≪100 attempts). Run `hrzones.Engine.Compute` on
the window slice; derive zones 4–5 time fraction. If this proves too slow in practice,
limit HR enrichment to anchor rows only (document in README).

**Response additions (backward compatible)**

Detail `estimate` block gains optional fields:

```json
{
  "seconds": 1386,
  "lower_seconds": 1360,
  "upper_seconds": 1420,
  "raw_seconds": 1783,
  "floored_at_logged_best": true,
  "basis": "fitted_curve",
  "confidence": "high"
}
```

Attempt rows gain optional `pace_ratio` and `hr_z4_z5_pct` for transparency in the
detail attempts table (web/mobile may show in tooltips or footnotes).

Existing clients ignore unknown JSON fields. Web/mobile should update copy where the
estimate equals logged best because of floor ("At your proven best — model projected
slower before floor") — minimal UI tweak, not a redesign.

**Endpoints unchanged**

- `GET /running/max-effort`
- `GET /running/max-effort/{distance_key}`

### MCP (`prog-strength-mcp`)

Update `get_running_max_effort_estimate` tool description to note v2 semantics: estimates
are floored at logged bests; demographics improve with profile fields; `estimator_version`
=`2.0.0`. No new parameters.

### Web (`prog-strength-web`)

- Settings: optional **Birthdate** and **Sex** fields on the existing profile form; PATCH
  `/me` on save. Sex as a select (`Male` / `Female` / unset). Birthdate as date input.
- Max-effort detail: when `floored_at_logged_best`, adjust subtitle/helper copy (not the
  chart scale — logged-best dashed line remains). Optionally expose `raw_seconds` in an
  advanced footnote ("Model projection before floor: …") — lean toward footnote, not
  headline.
- No chart component rewrite required.

### Mobile (`prog-strength-mobile`)

- Consume new optional estimate fields if present; same floored copy tweak on max-effort
  detail screen.
- No Settings changes in this SOW.

### Tests

#### Engine (primary deliverable)

- Golden fixtures reproducing owner mismatch: logged 5K 23:06 → estimate ≤ 23:06; mile
  7:20 → estimate ≤ 7:20.
- Floor invariant property-style tests across randomized attempt sets.
- Evidence policy: superseded slower history excluded; anchor always included.
- Pace-ratio and HR-intensity monotonicity tests (better signals → higher weight → faster
  or equal prediction before floor).
- Demographics: age+sex narrows band vs empty profile on thin data.
- Version stamp `2.0.0` on all results.

#### API

- Handler tests: `assembleAttempts` shape with anchors; detail response includes new
  fields when floored; demographics passthrough from profile.

#### Profile

- PATCH validation for birthdate/sex; age computation edge cases (birthday not yet this
  year).

#### MCP

- Tool description snapshot or string assert (lightweight).

#### Web

- Settings form validation render test (optional, if pattern exists).

### Rollout

Forward-only. Merge order:

1. **`prog-strength-api`** — migration (profile + optional window columns), summarizer
   window bounds, engine v2 + README + fixtures, handler wiring, profile PATCH fields.
   Deploy first; estimates change immediately on read.
2. **`prog-strength-mcp`** — tool description.
3. **`prog-strength-web`** — Settings fields + max-effort copy.
4. **`prog-strength-mobile`** — copy parity.
5. **`prog-strength-docs`** — this SOW; mark shipped with "Resolved during
   implementation" when landed.

No client-side feature flags. `estimator_version` lets clients detect the bump.

Revert: revert API deploy restores v1 numbers; profile columns are harmless if unused.

## Open Questions

1. **History inclusion threshold.** Should supporting history be excluded only when slower
   than the anchor by >3%, or should *all* same-distance non-anchor rows be excluded
   (anchors-only model)? *Tentative lean:* exclude beyond 3% — keeps recent near-miss
   efforts as weak evidence without letting stale PRs dominate.

2. **Window bounds backfill scope.** Re-parse every archived TCX on boot to populate
   window columns, or populate lazily on first max-effort read? *Tentative lean:* boot
   backfill (consistent with original best-efforts backfill); HR signal is unreliable until
   done, but pace ratio works without window bounds.

3. **Standards table source.** Embed WMA/Alan Jones age-grading factors vs. a smaller
   curated internal table? *Tentative lean:* embed a minimal curated table with documented
   provenance links in README; expand later.

4. **Expose `raw_seconds` in UI.** Show pre-floor model output to power users, or keep
   server-only for debugging? *Tentative lean:* detail footnote only when
   `floored_at_logged_best`; hide on summary cards.

5. **Sex field labeling.** Store `sex` internally but label "Gender" in Settings? *
   Tentative lean:* Settings label "Sex (for running estimates)" to match engine field;
   avoid broader gender identity scope in this SOW.

6. **Upper confidence band when floored.** Should `upper_seconds` also clamp to logged
   best, or remain the model's slower tail? *Tentative lean:* clamp lower and point to
   logged best; leave upper as model upper (honest uncertainty that the demonstrated best
   might still be sub-maximal).
