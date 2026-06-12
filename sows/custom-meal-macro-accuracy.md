---
status: draft
repos:
  - prog-strength-agent
  - prog-strength-mcp
  - prog-strength-infra
  - prog-strength-docs
---

# Custom Meal Macro Accuracy: Eval Harness, Nutrition Lookup, and Estimation Routing

**Status**: Draft · **Last updated**: 2026-06-11

## Introduction

When a user tells the **Prog Strength** agent "log 10 chicken minis from Chick-fil-A," the macros that land in their nutrition log are invented by the LLM from memory. The custom-meal path (`prompt.py:78-104`) instructs the model to "call log_custom_meal with a best-estimate of the macros" — and because `log_nutrition` turns classify to the simple tier, that estimate usually comes from Haiku, the weakest model in the stack for world knowledge about restaurant nutrition. There is no lookup tool, no external nutrition database, and no web access anywhere in the agent or MCP. The numbers are a guess, the guess is made by the cheapest model, and nothing measures how good or bad the guesses are.

This matters because one-off restaurant meals are exactly the case where users lean on the agent instead of the pantry. A pantry item has user-entered macros; a recipe derives from pantry items; a custom meal is whatever the model hallucinates. Chick-fil-A publishes that a chicken mini is ~91 calories — that number is retrievable, not guessable. Today the agent guesses anyway.

The deeper problem is that we can't even say *how wrong* the agent is, or whether any change helps. The existing test suites are structural (schema validation, auth guards, tool invocation); there is no accuracy evaluation at all. Improving estimation without measurement is guesswork stacked on guesswork — and worse, future prompt or model changes can silently regress estimation quality with no signal until a user notices their weekly calories are fiction.

After this work ships:

1. **A macro-accuracy eval harness exists** with a golden dataset of real meals and published ground-truth macros, runnable locally and in CI.
2. **Every pull request to `prog-strength-agent` and `prog-strength-mcp` gets a sticky PR comment** showing estimation accuracy for the PR's code versus the `main` baseline — an explicit, persistent track record of whether each change made the agent better or worse at this job.
3. **The agent grounds custom-meal macros in real data** via a new `lookup_food_nutrition` MCP tool backed by FatSecret (restaurant + branded foods, free tier) with USDA FoodData Central as the generic-foods fallback, falling back to LLM estimation only when lookup misses.
4. **True no-data estimation routes to the complex tier**, so when the agent must guess, the strongest available model does the guessing.

## Proposed Solution

The work lands in three phases, deliberately ordered so measurement exists before the things being measured change.

**Phase 1 — eval harness + CI/PR integration.** A new `evals/` package in `prog-strength-agent` holds a golden dataset (~75–100 meal descriptions with published ground-truth macros, spanning chain-restaurant, packaged, and generic/homemade categories) and a runner that drives the real agent pipeline — router, system prompt, harness, MCP tools — against a local fake Prog Strength API that records what the agent logs. Per-case scoring is absolute percentage error per macro; aggregates roll up per category. A reusable GitHub Actions workflow runs the eval on PRs in both repos, compares against the latest `main` baseline, and upserts a sticky PR comment with the results table and a regression/improvement verdict.

**Phase 2 — `lookup_food_nutrition` MCP tool.** A new tool in `prog-strength-mcp` that searches FatSecret Platform (Basic/free tier: US dataset, restaurant and branded foods, 5,000 calls/day) and falls back to USDA FoodData Central (free, authoritative for generic and homemade foods). The tool returns candidate matches with per-serving macros and a computed total for the requested quantity — multiplication happens in code, not in the model. The agent's custom-meal prompt section is rewritten: lookup first, copy the numbers, state the source and assumptions in the reply, estimate only when lookup returns nothing usable.

**Phase 3 — estimation tier routing.** The router prompt gains a rule: a meal-logging turn that references an external food source (chain name, "from <place>", ordering/buying language) classifies as `complex` tier. With Phase 2 landed this mostly matters for lookup misses — the model copying tool output doesn't need to be smart, but the model inventing numbers does. Phase 1's eval quantifies whether this routing change pays for its latency/cost before it ships as default.

The phases are separately shippable and the eval gates each one: Phase 1's first run records today's baseline; Phase 2 and 3 each have to beat it in their own PR comment to merge. That is the whole point of the ordering.

Community MCP servers for USDA data exist (asachs01/nutrition-mcp, neonwatty/food-tracker-mcp, FelipeAdachi/mcp-food-data-central) and were evaluated; none are deployed here. They are USDA-only — which misses the chain-restaurant case entirely — and several bundle their own meal-logging/SQLite state that would overlap confusingly with the existing `log_custom_meal` → Go API flow. One thin tool inside the existing MCP server keeps a single agent/data boundary, reuses the deployed process, and stays vendor-swappable behind env config.

## Goals and Non-Goals

### Goals

- Build a golden eval dataset of ~75–100 custom-meal cases with published ground-truth macros across three categories: `chain` (restaurant menu items), `packaged` (branded grocery items), `generic` (homemade/unbranded descriptions).
- Build an eval runner in `prog-strength-agent/evals/` that exercises the real router → harness → MCP → (fake) API pipeline, captures the macros the agent actually logs, and scores them against ground truth (absolute % error per macro, median over N trials per case).
- Run the eval automatically on pull requests in **both** `prog-strength-agent` and `prog-strength-mcp` via one reusable GitHub Actions workflow; post/update a single sticky PR comment per PR with: per-category accuracy, deltas versus the `main` baseline, a clear improved/regressed/neutral verdict, and a collapsible per-case detail section.
- Record a durable history of eval results on `main` so the accuracy track record outlives individual PRs.
- Add a `lookup_food_nutrition` MCP tool backed by FatSecret Platform (primary: restaurant + branded) and USDA FoodData Central (fallback: generic), with quantity math computed in code and a `source` field on every result.
- Update the agent system prompt's custom-meal section: lookup before estimating, cite the source and per-item assumption in the reply, estimate only on lookup miss, keep the existing "offer to save to pantry" follow-up.
- Add a macro plausibility check to the lookup tool results and the agent prompt (4·protein + 4·carbs + 9·fat should approximate calories) so internally-inconsistent numbers get flagged before logging.
- Route external-meal estimation turns to the complex tier via a router prompt rule, gated on the eval showing a measurable win.
- Keep the system fully functional with no external nutrition keys configured: the tool reports "lookup unavailable" and the agent falls back to today's estimate-from-knowledge behavior.

### Non-Goals

- **Nutritionix or other paid nutrition APIs.** Best-in-class restaurant coverage, but enterprise pricing (~$1,850/month) is absurd at this project's scale. The tool's interface (query in, candidates with source out) is vendor-shaped so a future swap is config + one client module, not a redesign.
- **Anthropic server-side web search as a lookup mechanism in this SOW.** It's the natural *next* fallback for long-tail misses (local restaurants, regional chains), but it adds per-search cost, latency, and nondeterminism. The eval harness this SOW ships is exactly the instrument that will tell us whether the residual miss rate justifies it. Revisit after Phase 2 data exists.
- **Improving photo-based meal estimation.** The receipt/plate-photo flow (`prompt.py:91-104`) has its own failure modes (portion estimation from pixels) that lookup doesn't address. The eval dataset is text-only in v1; a vision eval category is a natural follow-up.
- **Backfilling or correcting previously-logged custom meals.** Macros are frozen at log time by design; this SOW improves future logs only.
- **An eval that scores conversation quality, tool-call efficiency, or anything beyond macro accuracy.** The harness is built so new scorers can be added (the runner already captures full transcripts), but v1 scores macros. Expanding the eval as the agent gets smarter is the explicitly-intended follow-on work, not this SOW.
- **Web-app changes.** The `log_custom_meal` schema is unchanged; entries flow into the existing nutrition log UI untouched. (A "source: FatSecret" badge in the log UI is a possible later nicety.)
- **Caching nutrition lookups in the Go API / shared database.** v1 uses an in-process TTL cache in the MCP server, which is sufficient for one deployed instance and 5k calls/day headroom. A shared verified-foods table is premature.
- **Blocking merges on eval regressions.** The PR comment is informational, not a required check, in v1. LLM evals are noisy; making them merge-blocking before we understand run-to-run variance would manufacture false-positive friction. Revisit once variance data from a few weeks of PRs exists.

## Implementation Details

### Phase 1: Eval harness

#### Dataset

`prog-strength-agent/evals/dataset/custom_meals.json` — an array of cases:

```json
{
  "id": "cfa-chicken-minis-10",
  "category": "chain",
  "message": "log 10 chicken minis from chick fil a for breakfast",
  "expect": { "calories": 910, "protein_g": 50, "fat_g": 41, "carbs_g": 84 },
  "tolerance_pct": 15,
  "source": "https://www.chick-fil-a.com/nutrition (per mini: 91 cal)",
  "notes": "quantity-scaling case: per-item macros x 10"
}
```

Composition targets: ~40 `chain` cases across the majors (Chick-fil-A, Chipotle, McDonald's, Subway, Starbucks, Taco Bell, Domino's…), weighted toward quantity-scaling and modifier phrasing ("no rice", "double meat") since that's where pure estimation falls apart; ~20 `packaged` cases (branded bars, frozen meals, drinks with label data); ~25 `generic` cases ("two scrambled eggs and a slice of buttered toast") with USDA FNDDS-derived ground truth. Every case records its source URL so ground truth is auditable and refreshable when chains reformulate.

Ground truth is curated by hand once, with published-source values, and checked in. Chains reformulate rarely; a `verified_at` date per case plus an open question below covers staleness.

#### Runner

`prog-strength-agent/evals/run_eval.py`, invoked as `uv run python -m evals.run_eval --dataset evals/dataset/custom_meals.json --trials 3 --out eval-results.json`.

Per case, the runner drives the **real pipeline**: `ModelRouter.route()` on the case message, then the selected harness's tool-use loop with a live MCP session — the same code path production `/chat` takes, minus the FastAPI layer. Two test doubles make this hermetic:

- **Fake Prog Strength API** — a small in-process FastAPI app (pattern already exists in the MCP test suite's `respx` fixtures, promoted here to a live uvicorn-on-localhost stub) that serves an empty pantry and recipes list (forcing the custom-meal path) and records every `POST /nutrition/log/custom` payload. The recorded payload IS the eval observation.
- **Real MCP server** — launched as a subprocess from a configurable checkout path (`--mcp-path`), pointed at the fake API. This is what lets `prog-strength-mcp` PRs run the same eval against *their* changed tool code, including the Phase 2 lookup tool. External nutrition APIs are hit live when keys are configured (see Open Questions for the recorded-fixture alternative).

Scoring per case: absolute percentage error per macro between the logged values and `expect`, median across `--trials` runs (default 3) to damp LLM nondeterminism. A case **passes** when calories land within `tolerance_pct` (default ±15%). Aggregates per category and overall: median APE per macro, pass rate, and a single composite score (mean of per-category pass rates, so the small `packaged` category isn't drowned out). Cases where the agent never called `log_custom_meal` (asked a question instead, refused, errored) score as failures with a distinct `no_log` flag — an agent that stops logging is a regression even if its numbers would have been accurate.

Output: `eval-results.json` (full per-case, per-trial detail) plus a rendered `eval-summary.md` used verbatim as the PR comment body.

Estimated cost per full run: ~100 cases × 3 trials × (1 Haiku router call + 1–3 harness calls) — single-digit dollars on the current model mix. Cheap enough to run on every PR; the path filters below keep it from running on README typos.

#### CI / PR integration

One reusable workflow owns the logic; both repos invoke it.

**`prog-strength-agent/.github/workflows/eval-reusable.yml`** (`workflow_call`):

```yaml
inputs:
  agent_ref: { type: string, default: main }
  mcp_ref:   { type: string, default: main }
secrets:
  ANTHROPIC_API_KEY: { required: true }
  FATSECRET_CLIENT_ID: { required: false }
  FATSECRET_CLIENT_SECRET: { required: false }
  USDA_FDC_API_KEY: { required: false }
```

Steps: check out `prog-strength-agent@agent_ref` and `prog-strength-mcp@mcp_ref` side by side; `uv sync` both; run the eval; download the most recent successful `main` baseline artifact (`gh api /repos/Prog-Strength/prog-strength-agent/actions/artifacts?name=eval-baseline`); render the comparison; upload `eval-results.json` as an artifact. If no baseline artifact exists (first run, or >90-day artifact expiry), run the eval a second time at `main` refs inline and use that as the baseline — slower but self-healing.

**`prog-strength-agent/.github/workflows/eval.yml`**: `pull_request` trigger with path filters (`src/**`, `evals/**`, `pyproject.toml`), calls the reusable workflow with `agent_ref: ${{ github.event.pull_request.head.sha }}`; plus a `push: branches [main]` trigger that runs the eval and uploads the `eval-baseline` artifact and appends one summary line to the history record (below).

**`prog-strength-mcp/.github/workflows/eval.yml`**: `pull_request` trigger with path filters (`src/**`, `pyproject.toml`), calls `Prog-Strength/prog-strength-agent/.github/workflows/eval-reusable.yml@main` with `mcp_ref: ${{ github.event.pull_request.head.sha }}`. Requires the org setting allowing private-repo reusable workflows across the org, and `ANTHROPIC_API_KEY` (plus optional nutrition keys) added to the MCP repo's secrets.

Concurrency group per PR with `cancel-in-progress: true` so rapid pushes don't stack paid eval runs. An `eval:skip` PR label short-circuits the job for changes the author knows are irrelevant.

**PR comment.** Upserted sticky comment keyed on a `<!-- macro-eval -->` marker (find-and-update via `gh api`, same pattern as popular sticky-comment actions, vendored as a ~30-line script step so there's no third-party action dependency):

```
## 🧪 Macro estimation eval

**Verdict: ✅ improved** (composite 71 → 78, threshold ±3)

| Category | Cases | Pass rate | Δ vs main | Cal MAPE | Δ | P/F/C MAPE |
|----------|------:|----------:|----------:|---------:|--:|-----------:|
| chain    | 40    | 75%       | +10pp     | 11%      | −6pp | 14% / 19% / 16% |
| packaged | 20    | 85%       | +5pp      | 8%       | −2pp | 10% / 12% / 11% |
| generic  | 25    | 72%       | ±0pp      | 13%      | ±0pp | 15% / 21% / 18% |

<details><summary>12 failing cases</summary> … per-case table … </details>

Baseline: main@a1b2c3 (run 2026-06-12) · 3 trials/case · details in workflow artifacts
```

The verdict thresholds (±3 composite points, ±5pp category pass rate) start as informed guesses and get tuned against observed trial variance — the first few weeks of runs ARE the variance study. Verdict is one of ✅ improved / ❌ regressed / ➖ no significant change, with ❌ never blocking merge in v1 (see Non-Goals).

**History.** The `main`-push job appends one JSON line (date, sha, composite, per-category pass rates) to `evals/history.jsonl` and commits it back with a `chore(eval): record results for <sha>` message — conventional-commit `chore` so semantic-release ignores it. This file is the long-lived track record the PR comments individually can't provide, and it's greppable/plottable when the eval inevitably grows new categories.

### Phase 2: `lookup_food_nutrition` MCP tool

#### Tool surface

New tool in `prog-strength-mcp` alongside the existing nutrition tools:

```python
@mcp.tool
async def lookup_food_nutrition(
    query: str,          # "Chick-fil-A chicken minis", "Fage 2% greek yogurt"
    quantity: float = 1, # items/servings the user consumed
    max_results: int = 5,
) -> list[FoodMatch]
```

`FoodMatch`: `{name, brand, serving_description, per_serving: {calories, protein_g, fat_g, carbs_g}, total_for_quantity: {…}, source: "fatsecret" | "usda", source_id}`. `total_for_quantity` is computed in the tool — the ×10 multiplication for chicken minis happens in Python, not in a model that flubs arithmetic. A `plausibility_warning` field is set when 4·P + 4·C + 9·F diverges from stated calories by >25% (some sources have entry errors; the agent is told to prefer candidates without warnings).

#### Providers

- **FatSecret Platform, Basic edition** (primary): free, 5,000 calls/day, US dataset including restaurant and branded foods, OAuth2 client-credentials flow (`foods.search` → `food.get.v4`). Attribution required on the Basic tier — see Open Questions.
- **USDA FoodData Central** (fallback, and primary for generic-food queries): free API key, `/v1/foods/search` across FNDDS/Foundation/Branded data types. Authoritative for "two scrambled eggs" but nearly useless for "chicken minis."

Provider modules live behind a common `NutritionProvider` protocol in `mcp/nutrition_lookup/` so the FatSecret→Nutritionix-someday swap is one module. Results merge: FatSecret first, USDA appended when FatSecret returns < `max_results` or the query looks generic (no brand-shaped token). An in-process TTL cache (24h, keyed on normalized query) keeps repeat lookups off the daily quota.

Config (`mcp/config.py`): `FATSECRET_CLIENT_ID`, `FATSECRET_CLIENT_SECRET`, `USDA_FDC_API_KEY` — all optional. With neither configured the tool returns a structured `{"error": "lookup_unavailable"}` and the agent prompt covers the fallback. Deployment: secrets added to the MCP repo's deploy workflow and the `compose/mcp` env in `prog-strength-infra`, mirroring how `ANTHROPIC_API_KEY` flows to the agent today.

#### Prompt changes (`prog-strength-agent/src/prog_strength_agent/prompt.py`)

The custom-meal paragraph is rewritten. New behavior, in order: (1) pantry/recipe match as today; (2) on miss with external-meal wording, call `lookup_food_nutrition` with the food noun and quantity; (3) pick the best candidate (prefer exact brand match, no plausibility warning), call `log_custom_meal` with `total_for_quantity`, and say in the reply where the numbers came from — *"Logged 10 chicken minis — 910 cal (Chick-fil-A via FatSecret, 91 cal each)"*; (4) only when lookup returns nothing usable, estimate from knowledge as today, conservative-high, and *say it's an estimate*; (5) keep the existing "save to pantry?" follow-up, now seeded with looked-up (not hallucinated) per-serving macros. The `log_nutrition` intent rules block in `intents.py` gets a matching one-line nudge.

`tests/test_prompt.py` gains assertions that the lookup-first instruction and the cite-your-source instruction survive prompt composition, mirroring the existing custom-meal prompt tests.

### Phase 3: Estimation tier routing

`ROUTER_SYSTEM_PROMPT` (`model_router.py:48-66`) gains one rule: meal-logging messages that reference an external source (chain/restaurant name, "from <place>", "ordered/bought/picked up") classify as `(log_nutrition, complex)`. Rationale: when lookup misses, the macros come from model knowledge, and Sonnet's knowledge measurably beats Haiku's; when lookup hits, the complex model costs a little more to copy numbers — acceptable for the minority of turns this rule catches.

This ships **only if** the eval shows it: Phase 3 is a one-line PR whose own sticky comment must show the win. If Phase 2's lookup coverage turns out high enough that routing adds cost without moving the composite, the PR gets closed with the data attached — which is precisely the workflow this SOW exists to enable.

### Testing

- **Eval harness unit tests** (`prog-strength-agent/tests/test_evals.py`): scorer math (APE, medians, pass thresholds, `no_log` handling), dataset schema validation (every case has positive macros, valid category, source URL), comparison/verdict rendering against fixture result files.
- **MCP lookup tool tests** (`prog-strength-mcp/tests/test_nutrition_lookup.py`): `respx`-mocked FatSecret and USDA responses; quantity scaling; provider fallback order; TTL cache behavior; plausibility-warning math; unconfigured-keys degradation; OAuth token refresh.
- **Prompt tests** as above.
- The eval itself is the integration test for the whole feature — that's the point of building it first.

### Rollout

1. **Phase 1 PR** (agent repo): dataset + runner + workflows. Its own PR can't compare against a baseline yet; the first `main` run after merge records baseline #1 — today's pure-Haiku-estimation accuracy, which immediately becomes the number to beat. Add `ANTHROPIC_API_KEY` secret to the MCP repo and flip the org reusable-workflow setting; land the MCP repo's thin `eval.yml`.
2. **Phase 2 PRs**: MCP tool first (agent prompt unchanged — tool is inert until the prompt points at it, so the MCP PR's eval comment should read "no significant change," itself a useful null-test of comment noise), then the agent prompt PR, whose comment should show the chain-category jump. Register FatSecret + USDA keys; add secrets to both repos and infra compose.
3. **Phase 3 PR**: router rule, merged or closed on its eval numbers.

No feature flags: lookup degrades to today's behavior when unconfigured, and each phase is a revertable PR with its accuracy impact documented in the PR thread.

## Open Questions

1. **Live external APIs in CI, or recorded fixtures?** Live FatSecret/USDA calls in eval runs test the true end-to-end path but add flake risk, quota consumption, and a secrets surface in CI. Recorded responses (cassette-style, refreshed by a scheduled weekly job) are hermetic but can mask provider drift. Recommendation: live in v1 with the 24h cache wired into the runner (one quota hit per food per day across all PRs), recorded fixtures only if flake shows up in practice.
2. **FatSecret Basic-tier attribution.** Basic requires "Powered by fatsecret" attribution. Where does it live — agent reply text (noisy), web app footer on the nutrition page, or both? Needs a read of the actual license terms during implementation; if attribution placement is unacceptable, USDA-only still covers the `generic`/`packaged` categories and the eval will quantify exactly what the chain category loses.
3. **Verdict noise thresholds.** ±3 composite points is a guess. After ~10 main-branch runs exist in `history.jsonl`, compute the observed run-to-run standard deviation and set thresholds at ~2σ. Until then, expect some ➖ verdicts that are really small wins.
4. **Dataset staleness.** Chains reformulate. Proposal: `verified_at` per case plus a quarterly scheduled workflow that flags cases older than a year for re-verification. Acceptable, or overkill for v1?
5. **Should `log_custom_meal` itself enforce the plausibility check?** A server-side reject (or warn-and-log) on 4P+4C+9F vs calories divergence would catch hallucinated macro combos from any client, not just lookup-path entries. Leaning warn-and-log (telemetry counter) in v1 — hard rejects on a heuristic risk blocking legitimate edge foods (fiber-heavy, sugar alcohols, alcohol itself at 7 cal/g).
6. **Mobile/web Quick Add custom tab.** The web app's custom-meal form takes user-typed macros and is unaffected here, but once `lookup_food_nutrition` exists in the MCP, a "search nutrition database" affordance in the web Quick Add custom tab becomes cheap to add (new API pass-through endpoint). Separate SOW if wanted.
