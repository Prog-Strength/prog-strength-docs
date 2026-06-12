---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-agent
  - prog-strength-mcp
  - prog-strength-infra
  - prog-strength-docs
---

# Custom Meal Macro Accuracy: Eval Harness, Nutrition Lookup, and Estimation Routing

**Status**: Shipped · **Last updated**: 2026-06-12

> **Revision (2026-06-12):** eval cost policy tightened after the first live
> CI run tripped Anthropic rate limits and burned tokens on a pay-per-token
> budget. Eval runs are now **strictly opt-in** (an `eval:run` PR label or a
> manual `workflow_dispatch` in `prog-strength-agent` only), default to a
> curated 24-case **smoke subset × 1 trial** (~95% fewer LLM trials), and the
> mcp/api per-PR caller workflows and the inline-baseline re-run were removed.
> The full 77-case × 3-trial run is reserved for explicitly-requested
> baseline publishes. The cross-repo reusable-workflow shape can return if
> per-ref evals ever become worth their spend.
>
> **Revision (2026-06-11):** the nutrition lookup implementation moved from
> `prog-strength-mcp` to `prog-strength-api` after first-draft review. The MCP
> server stays a transparent forwarder (its design contract); the Go API owns
> all external-API integration and gains a durable `nutrition_lookup_cache`
> SQLite table with a freshness/eviction policy. The eval harness was upgraded
> from a Python fake API to running the **real Go API** so lookup + cache code
> is exercised end-to-end. Tool surface, prompts, dataset, and scoring are
> unchanged from the first draft.

## Implementation record (shipped 2026-06-12)

Implemented over one day across six repos. PRs, in merge order:

| PR | Contents | Released as |
|---|---|---|
| docs#36 / #37 / #38 | SOW draft → architecture revision → cost-policy revision | — |
| api#22 | `internal/nutritionlookup` (FatSecret + USDA providers, `GET /nutrition/lookup`), `nutrition_lookup_cache` migration 018, config, deploy secrets | api v0.45.0 (deployed) |
| api#23 | removed the per-PR eval workflow that #22 carried (superseded by the cost policy; it referenced a deleted reusable workflow) | no release (chore) |
| agent#8 | eval harness (dataset, real-API runner, scoring, opt-in workflow), lookup-first prompts, complex-tier routing rule | agent v0.20.0 — see finding 2 |
| agent#9 | conventional-PR-title check + the repo's first AGENTS.md | no release (ci) |
| mcp#4 | `lookup_food_nutrition` thin forwarder (the activation step) | mcp release on merge |
| infra#29 | provider env vars on the api compose stack | applied at next api deploy |

**Findings and deviations beyond the two pre-merge revisions above:**

1. **api#22 failed its first CI run** on the conventional-title check and golangci-lint: gosec G101 false-positived on the FatSecret URL constant *names* (inline `nolint` with reasons), gosec G704 (SSRF taint) was added to the repo's documented taint-false-positive excludes alongside G701/G706/G710, and one govet `err`-shadow in a test was a genuine fix. Root cause of the misses: pre-commit hooks are per-clone opt-in and the authoring clone had none armed, plus a stale golangci-lint v1 locally. The clone's hooks are now installed, golangci-lint upgraded to CI's pinned v2.12.2, and the api AGENTS.md gained an explicit run-CI's-checks-before-authoring rule.
2. **agent#8's squash merge silently skipped its release**: the PR title wasn't a conventional commit, semantic-release published nothing, and the prompt/routing changes sat on main undeployed. Remedied with an empty `feat(agent):` commit (→ v0.20.0); prevented by agent#9's title check (mirroring the api repo's) and AGENTS.md documentation. mcp#4 had the same latent problem and was retitled before merge.
3. **Secrets footprint shrank with the cost policy**: only `prog-strength-agent` runs evals and it already held `ANTHROPIC_API_KEY` — no new required secrets anywhere. The FatSecret/USDA provider keys remain optional: agent repo (live lookups in evals) and api repo (production lookup via the deploy workflow).

**Outstanding at ship time:** FatSecret + USDA keys not yet registered (lookup serves 503 `lookup_unavailable` in prod; the agent estimates, as designed); no eval baseline published yet — the first `publish_baseline` dispatch should run once provider keys exist so the baseline captures the lookup-enabled stack; open questions 2 (FatSecret attribution placement), 3 (verdict noise thresholds), and 4 (dataset staleness policy) remain open.

## Post-ship session (same day, 2026-06-12): operationalization and hardening

The hours after shipping turned the lookup path from "merged" into a verified, observable production service — and each instrumentation layer caught a real defect the same night it landed. PRs in landing order:

| PR | Contents | Released as |
|---|---|---|
| agent#10 | **Prompt caching**: two `cache_control` breakpoints in the harness (tools array; composed system prompt) — ~60–85% input-token cost reduction for chat and evals; router/title prompts deliberately uncached (below minimum cacheable length) | agent v0.21.0 |
| infra#30, #32 | Daily AI usage cap raised 0.17 → 0.33 → **0.67/day (~$20/user/month)** after the owner hit the cap twice under normal use; to be revisited against `cache_read_tokens` telemetry now that caching is live | compose tunable |
| api#24, infra#31 | **Request-correlated structured logging** for the lookup path: the repo's first `log/slog` adopter (owner-approved exception to the deferred list) — JSON records auto-stamped with the `requestid` middleware's correlation id, new `LOG_LEVEL` config (prod ships debug during development) | api v0.46.0 |
| api#25 | **Deploy parity fix**: `manual-deploy.yml` rewrites the host `.env` wholesale but had never been given the provider secrets — manual deploys clobbered what `release.yml` set, taking the providers dark ten minutes after they first went live. Lockstep warning comments added on both workflows | no release (ci) |
| mcp#5, agent#11 | **End-to-end request tracing**: the API's `X-Request-ID` rides the lookup tool result (success and error shapes) and surfaces on the `tool_result` SSE event — browser DevTools → CloudWatch `filter request_id` for agent-driven lookups | mcp + agent releases |
| api#26, infra#33 | **Metrics + dashboard**: five `api_nutrition_lookup_*` Prometheus series (request outcomes, cache events, cache writes, provider calls by source/result, provider latency histogram) feeding the dedicated `ps-nutrition-lookup` Grafana dashboard with warning/critical thresholds (cache hit rate 60/25%, provider failure 5/25%, lookup failure 1/10%, latency 2s/5s) | api v0.47.x |
| docs#40 | Architecture diagram: FatSecret + USDA join External Services, `nutrition_lookup_cache` named on the data layer, API box gains its lookup role and the "sole caller of external data APIs" boundary rule; README now points at the `.drawio` source of truth | — |

**Debugging story worth keeping** (the observability stack validated itself the night it shipped): the first live lookup returned `lookup_unavailable` despite org secrets existing. The frontend `request_id` → CloudWatch trace showed `provider skipped: not configured` — *never tried*, not *tried and rejected* — which eliminated both the FatSecret IP-allowlist theory and an org-secret-visibility theory in one read, and the deploy timeline pinned the real cause: the manual-deploy `.env` clobbering (api#25 above). Lookups were then verified live end-to-end: providers serving, cache hitting on repeat queries, dashboard rendering.

**Process lessons, encoded as guardrails:**
- agent#8's squash merge silently skipped its release (non-conventional PR title) → conventional-title checks + the agent repo's first AGENTS.md (agent#9), remedied via an empty `feat:` commit (→ v0.20.0). mcp#4 had the same latent problem and was retitled pre-merge.
- api#26's own title then tripped the lowercase-subject rule on a leading proper noun ("Prometheus") — and exposed that the api repo's title check doesn't re-run on `edited` events (manual rerun required; the agent repo's newer check listens for `edited`).
- The dashboard didn't appear after infra#33 merged: monitoring-only infra merges have **no deploy path of their own** — the host's infra checkout only pulls as a side effect of a service deploy (a four-minute race in this case). Fixed manually; a `sync-monitoring.yml` (push to main touching `monitoring/**` → SSH `git pull`) is the proposed permanent fix, not yet built.

**Status after the session:** provider keys live (org secrets, public-repo visibility), lookup verified in production, tracing and dashboard operational. Still open: the first `publish_baseline` eval dispatch (deliberate full-cost run, now that it would capture the lookup-enabled stack), `sync-monitoring.yml`, FatSecret attribution placement, verdict-noise thresholds, dataset staleness policy.

## Introduction

When a user tells the **Prog Strength** agent "log 10 chicken minis from Chick-fil-A," the macros that land in their nutrition log are invented by the LLM from memory. The custom-meal path (`prompt.py:78-104`) instructs the model to "call log_custom_meal with a best-estimate of the macros" — and because `log_nutrition` turns classify to the simple tier, that estimate usually comes from Haiku, the weakest model in the stack for world knowledge about restaurant nutrition. There is no lookup tool, no external nutrition database, and no web access anywhere in the agent or MCP. The numbers are a guess, the guess is made by the cheapest model, and nothing measures how good or bad the guesses are.

This matters because one-off restaurant meals are exactly the case where users lean on the agent instead of the pantry. A pantry item has user-entered macros; a recipe derives from pantry items; a custom meal is whatever the model hallucinates. Chick-fil-A publishes that a chicken mini is ~91 calories — that number is retrievable, not guessable. Today the agent guesses anyway.

The deeper problem is that we can't even say *how wrong* the agent is, or whether any change helps. The existing test suites are structural (schema validation, auth guards, tool invocation); there is no accuracy evaluation at all. Improving estimation without measurement is guesswork stacked on guesswork — and worse, future prompt or model changes can silently regress estimation quality with no signal until a user notices their weekly calories are fiction.

After this work ships:

1. **A macro-accuracy eval harness exists** with a golden dataset of real meals and published ground-truth macros, runnable locally and in CI against the real API + MCP + agent pipeline.
2. **Every pull request to `prog-strength-agent`, `prog-strength-mcp`, and `prog-strength-api` gets a sticky PR comment** showing estimation accuracy for the PR's code versus the `main` baseline — an explicit, persistent track record of whether each change made the agent better or worse at this job.
3. **The agent grounds custom-meal macros in real data**: the Go API owns a nutrition lookup service backed by FatSecret (restaurant + branded foods, free tier) and USDA FoodData Central (generic foods), fronted by a durable SQLite cache; the MCP server exposes it as a `lookup_food_nutrition` tool that is — like every other tool — a transparent forwarder. The agent falls back to LLM estimation only when lookup misses.
4. **True no-data estimation routes to the complex tier**, so when the agent must guess, the strongest available model does the guessing.

## Proposed Solution

The work lands in three phases, deliberately ordered so measurement exists before the things being measured change.

**Phase 1 — eval harness + CI/PR integration.** A new `evals/` package in `prog-strength-agent` holds a golden dataset (~75–100 meal descriptions with published ground-truth macros, spanning chain-restaurant, packaged, and generic/homemade categories) and a runner that drives the real production pipeline: the **actual Go API binary** (hermetic: temp SQLite file + a test `JWT_SIGNING_KEY` — verified that the API requires nothing else to boot), the real MCP server pointed at it, and the real router + harness. Each (case, trial) mints its own JWT (the API trusts the `sub` claim statelessly), so trials run concurrently and the runner reads what the agent actually logged straight out of the temp SQLite database. Per-case scoring is absolute percentage error per macro; aggregates roll up per category. A single opt-in GitHub Actions workflow (label- or dispatch-triggered; smoke subset by default) compares against the latest published baseline and upserts a sticky PR comment with the results table and a regression/improvement verdict — every run spends real tokens, so nothing triggers automatically.

**Phase 2 — nutrition lookup in the Go API, forwarded by MCP.** The architecture rule this phase establishes: *LLM-provider SDKs (Anthropic, OpenAI) live in the agent; every other external integration lives in the Go API; the MCP server holds no business logic.* Concretely:

- `prog-strength-api` gains an `internal/nutritionlookup` domain: FatSecret Platform provider (Basic/free tier: US restaurant + branded foods, OAuth2 client-credentials), USDA FoodData Central provider (free, generic/homemade foods), a merge/scale service, and an auth-gated `GET /nutrition/lookup` endpoint. Quantity math happens in Go — the model copies totals, it never multiplies.
- A **durable `nutrition_lookup_cache` table** fronts the providers: cache hits skip the external round trip entirely (fast lookups, quota protection), a freshness TTL forces periodic re-pull so reformulated foods don't serve stale macros forever, and an opportunistic eviction sweep bounds table growth. Stale data is still served (flagged) when providers are down — resilience over purity.
- `prog-strength-mcp`'s `lookup_food_nutrition` tool becomes a thin forwarder — pull the Authorization header, call the API, adapt errors — exactly the shape of every sibling tool.
- The agent's custom-meal prompt section is rewritten: lookup first, copy the numbers, state the source and assumptions in the reply, estimate only when lookup returns nothing usable.

This placement also means the web app and mobile app can call `GET /nutrition/lookup` directly later (a "search nutrition database" affordance in Quick Add becomes a frontend-only change), and FatSecret's 5,000 calls/day quota is guarded by one shared cache instead of one per process.

**Phase 3 — estimation tier routing.** The router prompt gains a rule: a meal-logging turn that references an external food source (chain name, "from <place>", ordering/buying language) classifies as `complex` tier. With Phase 2 landed this mostly matters for lookup misses — the model copying tool output doesn't need to be smart, but the model inventing numbers does. Phase 1's eval quantifies whether this routing change pays for its latency/cost before it ships as default.

The phases are separately shippable and the eval gates each one: Phase 1's first run records today's baseline; Phase 2 and 3 each have to beat it in their own PR comment to merge.

Community MCP servers for USDA data exist (asachs01/nutrition-mcp, neonwatty/food-tracker-mcp, FelipeAdachi/mcp-food-data-central) and were evaluated; none are deployed here. They are USDA-only — which misses the chain-restaurant case entirely — and several bundle their own meal-logging/SQLite state that would overlap confusingly with the existing `log_custom_meal` → Go API flow. Owning the integration in the Go API keeps a single boundary, one durable cache, and vendor-swappable providers behind env config.

## Goals and Non-Goals

### Goals

- Build a golden eval dataset of ~75–100 custom-meal cases with published ground-truth macros across three categories: `chain` (restaurant menu items), `packaged` (branded grocery items), `generic` (homemade/unbranded descriptions).
- Build an eval runner in `prog-strength-agent/evals/` that exercises the real router → harness → MCP → **real Go API** pipeline (temp SQLite, per-trial minted JWTs), captures the macros the agent actually logs by reading the database, and scores them against ground truth (absolute % error per macro, median over N trials per case).
- Make eval runs available on demand with **zero automatic token spend**: a single workflow in `prog-strength-agent` triggered by an `eval:run` PR label or manual `workflow_dispatch`; PR runs default to the 24-case smoke subset × 1 trial and post/update one sticky comment (per-category accuracy, deltas vs the published baseline restricted to the same cases, improved/regressed/neutral verdict, collapsible failures).
- Record a durable history of eval results on `main` so the accuracy track record outlives individual PRs.
- Add an `internal/nutritionlookup` domain to `prog-strength-api`: FatSecret + USDA FDC providers behind a provider interface, merge/quantity-scale service, macro plausibility warnings (4·protein + 4·carbs + 9·fat vs stated calories), and an auth-gated `GET /nutrition/lookup` endpoint.
- Add a **durable `nutrition_lookup_cache` SQLite table** with: normalized-query keying shared across all users, a freshness TTL (serve from cache when fresh; re-pull from providers when stale), stale-fallback when providers fail, `last_used_at` tracking, and opportunistic eviction of long-unused rows so the table stays bounded without a background job.
- Reduce `prog-strength-mcp`'s `lookup_food_nutrition` to a transparent forwarder over the new endpoint — identical tool name, parameters, and result shape, so agent prompts and the eval dataset are unaffected by the relocation.
- Update the agent system prompt's custom-meal section: lookup before estimating, cite the source and per-item assumption in the reply, estimate only on lookup miss, keep the existing "offer to save to pantry" follow-up.
- Route external-meal estimation turns to the complex tier via a router prompt rule, gated on the eval showing a measurable win.
- Keep the system fully functional with no external nutrition keys configured: the endpoint reports lookup-unavailable, the tool forwards that, and the agent falls back to today's estimate-from-knowledge behavior.

### Non-Goals

- **Nutritionix or other paid nutrition APIs.** Best-in-class restaurant coverage, but enterprise pricing (~$1,850/month) is absurd at this project's scale. The provider interface in `internal/nutritionlookup` is the swap seam if that calculus ever changes.
- **Anthropic server-side web search as a lookup mechanism in this SOW.** It's the natural *next* fallback for long-tail misses (local restaurants, regional chains), but it adds per-search cost, latency, and nondeterminism. The eval harness this SOW ships is exactly the instrument that will tell us whether the residual miss rate justifies it. Revisit after Phase 2 data exists.
- **A user-facing lookup UI in web/mobile.** The endpoint is deliberately client-agnostic and auth-gated so a Quick Add "search nutrition database" affordance becomes a small frontend-only follow-up — but that UI is not this SOW.
- **Improving photo-based meal estimation.** The receipt/plate-photo flow has its own failure modes (portion estimation from pixels) that lookup doesn't address. The eval dataset is text-only in v1; a vision eval category is a natural follow-up.
- **Backfilling or correcting previously-logged custom meals.** Macros are frozen at log time by design; this SOW improves future logs only.
- **An eval that scores conversation quality, tool-call efficiency, or anything beyond macro accuracy.** The harness captures full transcripts so new scorers can be added later; v1 scores macros.
- **A "verified foods" curation workflow on top of the cache.** The cache table is raw provider output keyed by query. Promoting cache rows into a curated, user-visible verified-foods catalog (with admin review) is a plausible future; the schema doesn't preclude it and this SOW doesn't build it.
- **Blocking merges on eval regressions.** The PR comment is informational, not a required check, in v1. LLM evals are noisy; thresholds get tuned against observed variance from `history.jsonl` first.
- **A background cache-eviction job.** Eviction is opportunistic (piggybacked on lookups). At this project's scale a scheduled sweeper is machinery without a payoff; revisit if the table ever measurably matters.

## Implementation Details

### Phase 2: Nutrition lookup (described first — the API surface the eval exercises)

#### `prog-strength-api`: `internal/nutritionlookup` domain

Follows the standard domain layout (`handler.go`, `repository.go`, `sqlite_repository.go`, `memory_repository.go`, models, tests):

- **Providers** (`fatsecret.go`, `usda.go`) behind a small interface:

  ```go
  type Provider interface {
      Source() string
      Configured() bool
      Search(ctx context.Context, query string, limit int) ([]Candidate, error)
  }
  ```

  *FatSecret*: OAuth2 client-credentials against `oauth.fatsecret.com/connect/token` (token cached in-process until shortly before expiry), then `foods.search` via `platform.fatsecret.com/rest/server.api`. Macros are parsed from the documented `food_description` format ("Per 1 sandwich - Calories: 440kcal | Fat: 19.00g | …") — one HTTP round trip per lookup instead of N+1 `food.get` calls; candidates with unparseable descriptions are skipped, never guessed. Handles the single-result dict-not-list JSON quirk.

  *USDA FDC*: `GET api.nal.usda.gov/fdc/v1/foods/search` across all data types. Search-response nutrients are per-100g; Branded rows with a gram/ml label serving are scaled to that serving (with `householdServingFullText` as the serving description); rows missing protein/fat/carbs are dropped.

  Both use a shared `http.Client` with an ~8s timeout — a slow third party must not stall the agent's tool loop indefinitely.

- **Candidate shape** (JSON, the contract the MCP tool and agent prompt already use — unchanged from the first draft):

  ```json
  {
    "name": "Chick-n-Minis (4 Count)", "brand": "Chick-fil-A",
    "serving_description": "4 minis",
    "per_serving": {"calories": 360, "protein_g": 19, "fat_g": 13, "carbs_g": 41},
    "total_for_quantity": {"calories": 900, "...": "..."},
    "source": "fatsecret", "source_id": "12345",
    "plausibility_warning": "<set when 4P+4C+9F diverges >25% from stated calories>",
    "stale": false
  }
  ```

  `per_serving` keeps two decimals (it gets multiplied); `total_for_quantity` is computed in Go and rounded to one.

- **Service** (`service.go`): cache-first lookup → provider merge (FatSecret first; USDA appended when FatSecret returns fewer than `max_results`) → quantity scaling → plausibility flags. Provider errors degrade: one provider down → results from the other; all down with no cache → structured failure; all down with a stale cache row → serve stale, flagged.

#### Durable cache: `nutrition_lookup_cache`

Migration `018_nutrition_lookup_cache.sql`:

```sql
-- Durable cache of external nutrition lookups (FatSecret / USDA).
-- Global, NOT per-user: food data is public, the FatSecret quota is
-- shared, and a global key maximizes hit rate. Rows store per-serving
-- candidates as JSON; quantity scaling happens at read time.
CREATE TABLE IF NOT EXISTS nutrition_lookup_cache (
    query_normalized TEXT PRIMARY KEY,   -- lower-cased, whitespace-collapsed
    candidates_json  TEXT NOT NULL,      -- []Candidate, per-serving values
    fetched_at       DATETIME NOT NULL,  -- last successful provider pull
    last_used_at     DATETIME NOT NULL   -- last cache read (eviction signal)
);

CREATE INDEX idx_nutrition_lookup_cache_last_used
    ON nutrition_lookup_cache(last_used_at);
```

**Freshness policy:** a row younger than the freshness TTL (`fetched_at` within **7 days**) serves directly — no external call, sub-millisecond lookups for repeat foods. Older rows trigger a provider re-pull; success overwrites the row (`fetched_at` resets), failure serves the stale row with `"stale": true` on each candidate so the agent can mention it. The 7-day constant lives in code, not env — same philosophy as `auth.JWTLifetime` (changing data-freshness semantics should be a reviewable code change).

**Eviction policy:** every cache write piggybacks a sweep: `DELETE FROM nutrition_lookup_cache WHERE last_used_at < now - 90 days`. Foods the user actually eats stay hot forever (each hit bumps `last_used_at` and each TTL re-pull refreshes the data); one-off lookups age out. No background job, no unbounded growth, and the 90-day constant is likewise code-pinned.

Repository interface:

```go
type Repository interface {
    Get(ctx context.Context, queryNormalized string) (*CacheRow, error) // bumps last_used_at
    Put(ctx context.Context, row CacheRow) error                        // upsert + eviction sweep
}
```

`memory_repository.go` mirrors it for in-memory mode (`DATABASE_URL` empty), where the cache is simply non-durable.

#### Endpoint

`GET /nutrition/lookup?query=<food>&quantity=<n>&max_results=<m>` mounted inside the existing `auth.RequireUser` group (public food data, but the endpoint spends shared provider quota — gating it keeps anonymous internet traffic off the FatSecret budget). Params: `query` required (≤200 chars), `quantity` optional float > 0 (default 1), `max_results` optional 1–10 (default 5).

Responses use the API's standard envelope:

- `200` — `data: {"matches": [...], "quantity": n}` (matches may be empty).
- `503` — `error: "lookup_unavailable: no nutrition data providers configured"` (no keys) or `error: "lookup_failed: <provider detail>"` (all providers down, no cache). 503 is honest REST for "dependency down"; the MCP forwarder adapts it (below).
- `400` — malformed params.

Config additions (`internal/config`): `FATSECRET_CLIENT_ID`, `FATSECRET_CLIENT_SECRET`, `USDA_FDC_API_KEY` — all optional, mirroring `AvatarBucketName`'s "absent = feature degrades with a clear message" pattern.

#### `prog-strength-mcp`: thin forwarder

The `lookup_food_nutrition` tool keeps its exact name, parameters, and docstring (the agent prompt is written against them) but its body becomes the standard forwarder shape: `_auth_header_or_raise()` → `api.lookup_food_nutrition(auth, query=…, quantity=…, max_results=…)` → return `data`. A 503 from the API is adapted into the structured `{"error": "lookup_unavailable" | "lookup_failed", "detail": …}` dict the agent prompt already handles — tool contract identical to the first draft, so nothing downstream of the MCP boundary changes. No provider code, no cache, no new config or secrets in the MCP server.

#### Deployment (`prog-strength-infra` + api deploy workflow)

The three provider env vars thread through the **api** repo's deploy workflow into `compose/api`'s `.env` (not the mcp stack). All optional; empty values deploy fine.

#### Prompt changes (`prog-strength-agent`) — unchanged from first draft

Lookup-first ordering, copy `total_for_quantity` verbatim ("never re-multiply"), prefer warning-free candidates, cite source + per-serving assumption in the reply, estimate only on miss and say so, save-to-pantry ask seeded with looked-up per-serving macros. Matching one-line nudge in the `log_nutrition` intent rules.

### Phase 1: Eval harness

#### Dataset — unchanged from first draft

`prog-strength-agent/evals/dataset/custom_meals.json`: ~75–100 cases across `chain`/`packaged`/`generic` with published ground truth, per-case `tolerance_pct`, auditable `source` URLs, and `verified_at` dates. Weighted toward quantity-scaling and modifier phrasing.

#### Runner — revised: real API, not a fake

`evals/run_eval.py` stands up the production stack hermetically:

1. **Real Go API**: `go build ./cmd/api` from a `--api-path` checkout, run with `DATABASE_URL=<tmpdir>/eval.db`, `JWT_SIGNING_KEY=<eval secret>`, `SERVER_ADDR=127.0.0.1:<port>`, and any provider keys forwarded from the environment. Verified boot surface: the API requires only the signing key; OAuth/Google config is optional and unmounted when absent.
2. **Real MCP server**: `uv run --project <--mcp-path>` with `PROG_STRENGTH_API_BASE_URL` pointed at the eval API.
3. **Real router + harnesses**: the same `ModelRouter`/`ModelHarness` classes production `/chat` uses.

Per (case, trial), the runner **mints a JWT** with `sub = eval-<case>-t<n>` (pyjwt, HS256 — the API's auth is stateless and trusts the subject claim, and nutrition rows have no FK to the users table, so no user bootstrapping is needed). The unique subject is the correlation id: after the trial drains the SSE stream, the runner reads that subject's rows directly from the temp SQLite database (`nutrition_log_entries WHERE user_id = ?`) and scores custom-meal entries against ground truth. An empty pantry falls out naturally from the empty database. Multiple custom entries in one trial are summed; zero entries is the `no_log` failure mode.

Because the real lookup endpoint (and its durable cache) sits inside the eval loop, the eval measures the true production path — including cache-hit behavior on later trials of the same case, which is representative, since real users repeat foods.

#### Scoring, comparison, sticky comment, history — unchanged from first draft

Median APE per macro across trials; case passes when median calorie APE ≤ tolerance and the agent logged in a majority of trials; composite = mean of per-category pass rates; verdict thresholds ±3 composite points pending observed variance; `<!-- macro-eval -->` sticky comment; `eval-baseline` artifact published on `main` pushes; `evals/history.jsonl` appended with `chore(eval): … [skip ci]` commits.

#### CI / PR integration — revised again: opt-in, smoke-first

One self-contained workflow in `prog-strength-agent` (`eval.yml`); the reusable workflow and the mcp/api callers were removed for cost and simplicity.

- **Triggers**: `pull_request` gated on the `eval:run` label, and `workflow_dispatch` with `full` / `trials` / `publish_baseline` inputs. No `push` triggers anywhere — merges never spend tokens.
- **Smoke subset**: ~24 dataset cases carry `"smoke": true` (8 per category, retaining quantity-scaling and modifier coverage). PR runs execute smoke × 1 trial at concurrency 2 (low-tier rate-limit friendly); the comparison restricts the published full baseline to the same case ids so verdicts stay apples-to-apples.
- **Baselines**: published only by an explicit dispatch from `main` with `publish_baseline` (full dataset, 3 trials recommended) — uploads the `eval-baseline` artifact and appends `evals/history.jsonl`. No baseline → the PR comment says so; nothing re-runs (the old inline-baseline fallback, which doubled cost, is gone).
- **Local-first loop**: `uv run python -m evals.run_eval --smoke --api-path … --mcp-path …` is the everyday iteration path; CI is for the shareable PR record.
- mcp/api PRs no longer trigger evals; when a tool- or API-side change needs an accuracy check, label the corresponding agent PR or run a dispatch. Per-ref cross-repo evals can return later if they earn their cost.

### Phase 3: Estimation tier routing — unchanged from first draft

One `ROUTER_SYSTEM_PROMPT` rule classifying external-meal logging as `(log_nutrition, complex)`; ships only if its own eval comment shows the win.

### Testing

- **`prog-strength-api`**: provider tests against `httptest` servers (OAuth token caching, description parsing, single-result quirk, branded serving scaling, dropped-row rules); service tests (merge order, fallback, plausibility); repository tests on a migrated temp SQLite (freshness TTL respected, stale-fallback, `last_used_at` bump, eviction sweep); handler tests (param validation, 503 shapes, envelope).
- **`prog-strength-mcp`**: forwarder tests pinning the exact query params sent and the 503 → structured-error adaptation (respx).
- **`prog-strength-agent`**: eval scorer/dataset/compare unit tests (no LLM); prompt and router-rule assertions.
- The eval itself is the integration test for the whole feature.

### Rollout

1. **api PR**: `internal/nutritionlookup` + migration + endpoint + config. Mergeable first; the endpoint is dormant until the MCP tool points at it.
2. **agent PR**: eval harness (real-API runner) + workflows + prompt + router rule. On merge, the first main run records baseline #1.
3. **mcp PR**: forwarder tool + its `eval.yml` caller. Its eval comment should show the chain-category jump once provider keys are configured (the tool is the last link).
4. **infra PR**: provider env on the api compose stack. Safe any time.

Secrets (as revised by the cost policy): no new required secrets — only the agent repo runs evals and already holds `ANTHROPIC_API_KEY`. FatSecret + USDA keys are optional, on the agent repo (eval lookups) and api repo (production deploy).

## Open Questions

1. **Live external APIs in CI, or recorded fixtures?** Live calls test the true path but add flake/quota/secret surface. The durable cache now does much of the damping (one provider pull per food per freshness window *per eval database* — though eval DBs are ephemeral, so CI still pulls each food once per run). Recommendation stands: live in v1; recorded fixtures only if flake shows up.
2. **FatSecret Basic-tier attribution.** Basic requires "Powered by fatsecret" attribution. Where does it live — agent reply text (noisy), web app footer on the nutrition page, or both? Needs a read of the actual license terms during implementation; if placement is unacceptable, USDA-only still covers `generic`/`packaged` and the eval quantifies what `chain` loses.
3. **Verdict noise thresholds.** ±3 composite points is a guess; recompute at ~2σ once `history.jsonl` has ~10 main-branch runs.
4. **Dataset staleness.** `verified_at` per case plus (proposed) a quarterly scheduled workflow flagging cases older than a year. Acceptable, or overkill for v1?
5. **Should `log_custom_meal` itself enforce the plausibility check?** Leaning warn-and-log in v1 — hard rejects on a heuristic risk blocking legitimate edge foods (fiber, sugar alcohols, alcohol at 7 cal/g).
6. **Cache freshness TTL value.** 7 days balances "fast repeat lookups" against "reformulations propagate within a week." Restaurant items change rarely; packaged labels change occasionally. If 7 days proves chatty against the FatSecret quota, 30 days is defensible — it's a one-constant change with the eval to confirm no accuracy cost.
