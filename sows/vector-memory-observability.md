---
status: draft
depends_on:
  - sows/agent-vector-memory.md
repos:
  - prog-strength-api
  - prog-strength-infra
  - prog-strength-docs
---

# Vector Memory Observability

**Status**: Draft · **Last updated**: 2026-06-21

## Introduction

Agent Vector Memory (`sows/agent-vector-memory.md`) added a background goroutine to `prog-strength-api` that, every five minutes, finds idle chat sessions, distills them into durable user facts via an LLM, embeds those facts, and writes them to a sqlite-vec table so the agent can retrieve them later and respond more personably. It is the kind of feature that **degrades silently**: if it stops running, or runs but stops producing memories, nothing breaks. No request 500s, no page goes blank. The agent just quietly gets less personal over time, and we'd have no way to know until we noticed responses feeling generic — weeks later, with no signal pointing at the cause.

Today that risk is unmonitored. The goroutine (`internal/vectormemory/job.go`, `service.go`) logs extensively via `slog`, but emits **zero Prometheus metrics**. There is nothing on any dashboard about it. Its failure modes are exactly the ones that hide in logs:

- The ticker goroutine dies (panic in a dependency, context cancelled, never restarted) → distillation silently stops; the only evidence is the *absence* of log lines, which nothing watches for.
- The Anthropic distill call fails (bad key, rate limit, model deprecation) → each affected session logs a WARN and is retried next tick, burning tokens on every retry, producing no memories.
- The OpenAI embed call fails → same: WARN, retry, no progress, silent token spend.
- The dedup query or a per-observation insert fails → WARN, session skipped or partially written; the memory set quietly goes incomplete.
- The model returns zero observations every time (prompt regression, model change) → the goroutine reports healthy success while writing nothing.

This SOW makes the goroutine observable. It instruments the pipeline with Prometheus metrics, builds a dedicated **`ps-vector-memory`** Grafana dashboard whose headline tile answers "is this running right now, and is it doing work?", and — since the agent has grown well past what its current flat dashboard explains — gives the existing **Agent Observability** dashboard the same sectioning, per-panel documentation, and alarm-threshold treatment the **API Health** dashboard (`monitoring/grafana/dashboards/api.json`) already has. The goal is that a single glance tells us the service is alive and earning its token spend, and that anything alarmable carries warning/critical lines ready to wire to notifications later.

This is observability only. It does not change distillation behavior, retrieval, models, or thresholds — it makes the existing behavior visible.

## Proposed Solution

Three parts, two code repos:

- **Part A — Instrument the goroutine** (`prog-strength-api`). A new `internal/vectormemory/metrics.go` defines Prometheus collectors following the established `internal/nutritionlookup/metrics.go` pattern (`prometheus.NewXVec` + `init()` `MustRegister`, `api_` domain prefix). They are populated at the existing decision points in `job.go` and `service.go`. Cost metrics additionally parse the `usage` blocks already returned (but currently ignored) by the Anthropic and OpenAI responses.
- **Part B — New `ps-vector-memory` dashboard** (`prog-strength-infra`). A standalone dashboard, structured like `api.json` (markdown header → `row` sections → per-panel `description` → dashed `vector(N)` warning/critical lines), dedicated to the distillation-and-write goroutine. Standalone rather than a section on the Agent dashboard because the goroutine physically runs in the `api` process and has enough surface (liveness, work, failures, latency, cost) to warrant its own board.
- **Part C — Section the Agent dashboard** (`prog-strength-infra`). Restructure the existing 16-panel flat `agent.json` into documented sections with a header and alarm thresholds, matching the API Health house style. No query changes to existing panels.

The principle throughout, borrowed from API Health: **for a single-user pre-launch app, the magnitude of a problem matters more than its ratio.** One distill error is worth a glance; a goroutine silent for twenty minutes is an outage. Thresholds favor raw, legible signals over percentages.

## Implementation Details

### Part A — Goroutine instrumentation (`prog-strength-api`)

A new `internal/vectormemory/metrics.go`. All names use the repo's `api_` domain prefix (matching `api_nutrition_lookup_*`, `api_timeline_*`). The collectors:

#### Liveness — the silent-failure detectors

| Metric | Type | Notes |
| --- | --- | --- |
| `api_vectormemory_last_sweep_timestamp_seconds` | Gauge | Unix seconds, stamped at the end of **every** sweep attempt (success or batch error). "The loop is executing." |
| `api_vectormemory_last_success_timestamp_seconds` | Gauge | Stamped only when a sweep completes without a batch-level (select) error. "The loop is executing *and* completing." The dashboard's headline tile derives from this. |
| `api_vectormemory_idle_sessions` | Gauge | Full count of idle, undistilled sessions at the start of each sweep — the true backlog, visible even when a single batch caps at 20. Set from a new lightweight `CountIdleUndistilled(cutoff)` repo method (one indexed `COUNT` per tick; negligible). |

#### Cadence and work — "is it doing anything?"

| Metric | Type | Labels | Notes |
| --- | --- | --- | --- |
| `api_vectormemory_sweeps_total` | Counter | `result="success\|partial\|error"` | One increment per tick. `error` = batch select failed (nothing processed); `partial` = sweep completed but ≥1 session hit a stage error; `success` = all selected sessions distilled (including the zero-selected case). |
| `api_vectormemory_sessions_selected_total` | Counter | — | Idle sessions pulled into a batch. |
| `api_vectormemory_sessions_distilled_total` | Counter | — | Sessions successfully processed and marked. |
| `api_vectormemory_observations_distilled_total` | Counter | — | Raw observations returned by the distiller. |
| `api_vectormemory_observations_inserted_total` | Counter | — | Observations actually written to `agent_memories` / `vec_agent_memories`. |
| `api_vectormemory_observations_deduped_total` | Counter | — | Observations skipped as near-duplicates (distance ≤ `dedup_threshold`). |

The gap between `observations_distilled_total` and `observations_inserted_total + observations_deduped_total` surfaces silent per-observation insert loss.

#### Per-stage failures — pinpoint the silent break

| Metric | Type | Labels | Notes |
| --- | --- | --- | --- |
| `api_vectormemory_stage_errors_total` | Counter | `stage="select\|load\|distill\|embed\|dedup\|insert\|mark"` | One series per failure point in the pipeline, mapped to the existing WARN/ERROR sites: `select` = `IdleUndistilled` (`job.go`), `load` = conversation load (`job.go`), `distill` = Anthropic call, `embed` = OpenAI call, `dedup` = nearest-neighbor query, `insert` = per-observation write, `mark` = `memory_distilled_at` stamp. |

#### Latency

| Metric | Type | Notes |
| --- | --- | --- |
| `api_vectormemory_sweep_duration_seconds` | Histogram | Wall time for a full tick. Buckets weighted toward seconds–minutes (e.g. `1, 2.5, 5, 10, 30, 60, 120`). |
| `api_vectormemory_distill_duration_seconds` | Histogram | Anthropic call latency. Buckets `0.25, 0.5, 1, 2, 4, 8, 16, 32`. |
| `api_vectormemory_embed_duration_seconds` | Histogram | OpenAI embed latency. Buckets `0.05, 0.1, 0.25, 0.5, 1, 2, 5`. |

#### Cost — token spend visibility

The distiller and embedder responses already carry `usage` blocks; the code currently discards them. We add minimal response-struct fields to parse them:

| Metric | Type | Labels | Notes |
| --- | --- | --- | --- |
| `api_vectormemory_distill_tokens_total` | Counter | `kind="input\|output"` | From the Anthropic response `usage.input_tokens` / `usage.output_tokens`. Requires adding a `Usage` field to the response struct in `anthropic.go` (the only API-client change in this SOW). |
| `api_vectormemory_embed_tokens_total` | Counter | — | From the OpenAI embeddings response `usage.total_tokens`. Adds a `Usage` field to the response struct in `openai.go`. |

This is the lever that makes "the goroutine is silently failing and retrying, burning tokens for nothing" visible as a cost line, which matters given the per-token budget.

#### Wiring

No control-flow changes — instrumentation is added at points the code already branches on:

- `job.go` `runDistill`/`distillOnce`: stamp `last_sweep_timestamp` each tick and `last_success_timestamp` on clean completion; set `idle_sessions` from the new count; time the sweep; increment `sweeps_total{result}` with the result classified from per-session outcomes; increment `sessions_selected_total` by batch size; `stage_errors_total{stage="select|load|mark"}` at those WARN/ERROR sites.
- `service.go` `DistillSession`: time and count the distill and embed calls; increment `observations_distilled/inserted/deduped_total`; `stage_errors_total{stage="distill|embed|dedup|insert"}`; on success bump `sessions_distilled_total`; record token counters.
- `anthropic.go` / `openai.go`: add `Usage` response fields; return counts to the service for the token counters.

Collectors register in `init()` and are scraped at the existing `/metrics` endpoint on the `api` job — no new exposition wiring.

### Part B — `ps-vector-memory` dashboard (`prog-strength-infra`)

New `monitoring/grafana/dashboards/ps-vector-memory.json`, `uid: ps-vector-memory`, tag `prog-strength`, `refresh: 30s`, default range `now-6h`. Structure mirrors `api.json`: a transparent markdown header, `row`-delimited sections, a `description` (the ⓘ) on every panel explaining what it shows and why, and warning/critical lines rendered as constant `vector(N)` series styled with dashed lines (the established pattern). Queries scrape the `api` job. Threshold numbers below are **starting proposals**, trivially retunable.

**Header (text panel).** Explains the feature in two sentences, names the five-minute cadence, and states the rule: this service degrades silently, so the top-left **Time since last successful sweep** tile is the one to watch; everything else explains *why* it's behaving as it is.

**§ Liveness & cadence** — *is it running at all?*

- **Stat — Time since last successful sweep** *(headline)*. `time() - max(api_vectormemory_last_success_timestamp_seconds)`, unit seconds. Thresholds: green < 720s, **warning ≥ 720s** (more than two missed five-minute ticks), **critical ≥ 1800s** (30 min — the goroutine is effectively dead). This single tile is the answer to "is vector memory alive?"
- **Stat — Sweeps (24h)**. `sum(increase(api_vectormemory_sweeps_total[24h]))`. Expected ≈ 288 at a five-minute cadence; description states the expected value. Colored higher-is-better (red below ~240, green at full cadence).
- **Timeseries — Sweep outcomes**. `sum by (result) (increase(api_vectormemory_sweeps_total[15m]))` — sweeps per 15-min window by `result`, expected ≈ 3. Surfaces a rising `error`/`partial` share before the liveness tile would catch a full stop.
- **Stat — Idle backlog (now)**. `max(api_vectormemory_idle_sessions)`. Warning ≥ 50, critical ≥ 200 — the goroutine is running but falling behind.

**§ Work done** — *is it actually producing memories?*

- **Stat row — Sessions distilled / Observations inserted / Observations deduped (24h)**. `sum(increase(api_vectormemory_sessions_distilled_total[24h]))` and the two observation counters. These read zero when the loop runs but writes nothing — the "healthy success, empty output" failure.
- **Timeseries — Selected vs. distilled**. `sum(rate(api_vectormemory_sessions_selected_total[1h]))` vs. `..._sessions_distilled_total` — a persistent gap means sessions are selected but failing downstream.
- **Timeseries — Observations: distilled vs. inserted vs. deduped**. The three observation-counter rates together; distilled diverging above inserted+deduped is silent insert loss.

**§ Failures** — *where is it breaking?*

- **Timeseries — Stage errors**. `sum by (stage) (increase(api_vectormemory_stage_errors_total[1h]))` per hour by stage. Warning line at 1, critical at 3 (mirroring API Health's raw-count posture). The stage label points straight at the failing dependency.
- **Stat — Distill errors (24h)** and **Stat — Embed errors (24h)**. `..._stage_errors_total{stage="distill"}` / `{stage="embed"}` over 24h. Warning ≥ 1, critical ≥ 5 — these retry on a budget, so any sustained count means wasted spend and an unhealthy upstream (bad key, rate limit, deprecated model).

**§ Latency & cost**

- **Timeseries — Distill latency (p50/p95)**. `histogram_quantile(…, sum by (le) (rate(api_vectormemory_distill_duration_seconds_bucket[10m])))`. Warning 10s, critical 20s.
- **Timeseries — Embed latency (p50/p95)**. Same shape over `..._embed_duration_seconds_bucket`. Warning 2s, critical 5s.
- **Timeseries — Sweep duration (p50/p95)**. Over `..._sweep_duration_seconds_bucket`. Warning 60s, critical 120s (a sweep approaching the five-minute tick is a problem).
- **Timeseries — Token spend / sec**. `sum(rate(api_vectormemory_distill_tokens_total[1h]))` by `kind` plus `..._embed_tokens_total`. Informational (no hard threshold) — the cost line that makes silent retry-storms legible. Description notes that a spend climb without a matching `observations_inserted` climb is the signature of a failing-and-retrying loop.

### Part C — Section the Agent dashboard (`prog-strength-infra`)

Restructure the existing `monitoring/grafana/dashboards/agent.json` (16 flat panels, no header, no rows, no descriptions) into the API Health house style. **Existing panel queries are unchanged**; this adds a header, `row` separators, a `description` on every panel, and warning/critical lines on the alarmable ones.

- **Header (text panel)** — what the dashboard covers (chat turns, routing, intent, tools, voice) and how the sections are organized, with the "hover ⓘ for threshold meanings" note.
- **§ Overview** — the four existing stats (Total turns, Tokens in, Tokens out, Cache reads, 24h).
- **§ Routing & intent** — Routing decisions over time, Tier split, Intent classifications over time, Intent share.
- **§ Conversation** — Chat turn rate; **Chat latency (p50/p95)** with warning 3s / critical 8s lines; **Errors per tier** with warning 1 / critical 3 (raw count, per window).
- **§ Tools** — Tool calls/sec; Top tools; **Tool call latency p95** with warning 2s / critical 5s; **Tool errors/sec** with a warning line just above zero.
- **§ Voice** — **Time to first audio (p50/p95)** with warning 1.5s / critical 3s.

Each panel gains a one-to-three-sentence `description` explaining what it shows and why it's included — the same standard `api.json` panels meet. Alarm thresholds are starting proposals, retunable.

## Tests

| Repo | Coverage |
| --- | --- |
| `prog-strength-api` | Unit tests in `internal/vectormemory` asserting the new collectors move on the right events, using the existing job/service test harness (`job_test.go` already exercises the failure scenarios): a clean sweep advances `last_success_timestamp` and increments `sweeps_total{result="success"}`, `sessions_distilled_total`, and the observation counters; a distill/embed/insert failure increments `stage_errors_total{stage=…}` and classifies the sweep `partial`; a select failure classifies `error` and leaves `last_success_timestamp` unchanged; `idle_sessions` reflects the count method; token counters parse the `usage` blocks. Use `testutil.ToFloat64` / `CollectAndCompare` from `client_golang`. Regression: distillation behavior (what gets written, retry policy) is unchanged. |
| `prog-strength-infra` | Both dashboard JSON files parse and pass the repo's dashboard lint/validation; `uid`s are unique; every non-row panel has a non-empty `description`; threshold series resolve. Manual check against a live Grafana that panels render and the headline tile reads sensibly. |
| `prog-strength-docs` | This SOW; status transitions. |

## Rollout

1. **API** (`prog-strength-api`): add `internal/vectormemory/metrics.go`, the `CountIdleUndistilled` repo method, the `Usage` response fields in `anthropic.go`/`openai.go`, and wire the collectors. Ships behind no flag — metrics are inert until scraped and harmless if the feature is disabled (`vectormemory.enabled = false` simply means the counters stay flat, which the dashboard renders as a clearly-dead service, correctly).
2. **Infra** (`prog-strength-infra`): add `ps-vector-memory.json` and restructure `agent.json`. Dashboards are provisioned from the repo, so they appear on the next provisioning cycle / deploy.
3. Deploy API first so the metrics exist, then infra so the dashboard has data. If infra lands first, the new dashboard shows "No data" until the API deploy — cosmetic, not breaking.
4. **Verify after deploy**: `/metrics` on the api job exposes the `api_vectormemory_*` series; within ~10 minutes (two ticks) the headline tile shows a small, steady "time since last sweep" and the work counters climb; deliberately exercising a failure (e.g. an invalid distill model in a throwaway test) lights the matching `stage_errors_total` stage. Roll back by reverting either PR independently (metrics and dashboards are decoupled).

Once verified, the warning/critical lines are ready to be wired to Grafana alert rules / notifications in a follow-up — out of scope here, but the thresholds are placed with that next step in mind.

## Open Questions

1. **Headline thresholds.** Warning at 720s (2 missed ticks) / critical at 1800s assume the five-minute `session_idle_minutes`/tick cadence stays put. If the tick interval changes, these should track it. Reasonable as proposed?
2. **Backlog alarm levels.** Idle-backlog warning 50 / critical 200 are guesses for a single-user app; real numbers will become obvious after a week of data. Fine to ship as placeholders and retune.
3. **Cost metrics — keep token parsing?** Including `distill_tokens_total` / `embed_tokens_total` is the one change touching the API-client structs. Worth it for spend visibility (and the retry-storm signature), but cuttable for a leaner first pass if you'd rather not touch `anthropic.go`/`openai.go` yet.
4. **Standalone vs. section.** This builds `ps-vector-memory` as its own dashboard. If you'd prefer it as a section appended to the Agent dashboard instead, the panel definitions are identical — only the host file changes.
5. **Alert wiring.** This SOW only draws the threshold lines. Turning them into actual Grafana notifications (and choosing a channel) is a deliberate follow-up.
