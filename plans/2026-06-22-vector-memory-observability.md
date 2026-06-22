# Vector Memory Observability Implementation Plan

> **For agentic workers:** Implement task-by-task. Each Go task runs the full
> gate (golangci-lint v2.12.2, `go vet ./...`, `go mod tidy` no-drift,
> `go test ./...`) before it is considered done. Steps use checkbox
> (`- [ ]`) syntax for tracking.

**Goal:** Make the `internal/vectormemory` background distillation goroutine
observable. It logs via `slog` but emits **zero Prometheus metrics**, so its
silent-failure modes (dead ticker, failing distill/embed call, empty output)
hide in logs. Add Prometheus instrumentation, a dedicated `ps-vector-memory`
Grafana dashboard, and bring the existing flat `agent.json` dashboard up to the
API Health house style.

**Non-goals:** No change to distillation behaviour, retrieval, models, or
thresholds. This is observability only. No Grafana alert rules — only the
threshold *lines* are drawn (alert wiring is a deliberate follow-up).

**Repos / branches:** All work lands on `feat/vector-memory-observability`.
- `prog-strength-api` — Part A (one PR).
- `prog-strength-infra` — Parts B + C (one PR).
- `prog-strength-docs` — this plan + the SOW status flip (one PR).

## Build-environment note (Go, read first)

The `sqlite-vec` cgo bindings `#include "sqlite3.h"` at compile time, which
mattn/go-sqlite3 does not expose. On this host there is no system
`libsqlite3-dev` and no sudo, so `sqlite3.h` was copied from the
mattn/go-sqlite3 module's `sqlite3-binding.h` into `/workspace/.sqlite-include`
and `go env -w CGO_CFLAGS="-I/workspace/.sqlite-include"` set. The symbols link
from mattn's static build in the same binary. **Do not commit this** — CI
installs `libsqlite3-dev` (see `.github/workflows/ci.yml`).

## Conventions

API: domain packages under `internal/`; Prometheus collectors follow the
`internal/nutritionlookup/metrics.go` pattern (`prometheus.NewXVec` +
`init()` `MustRegister`, `api_` domain prefix); comments explain *why*; tests
next to impl with `dbtest.New(t)` + `testutil.ToFloat64`/`CollectAndCompare`.
PR titles are lowercase conventional commits.

Infra: dashboards are provisioned JSON under
`monitoring/grafana/dashboards/`; mirror `api.json`'s house style (markdown
header text panel → `row` sections → per-panel `description` → dashed
`vector(N)` warning/critical series). Valid JSON, no trailing whitespace,
trailing newline (the repo's hygiene pre-commit hooks enforce these).

---

## Part A — Goroutine instrumentation (`prog-strength-api`)

### Task A1: token-usage parsing in the provider clients

**Files:** `internal/vectormemory/anthropic.go`, `openai.go`, `provider.go`,
`service.go` (+ test fakes).

- The `Distiller` and `Embedder` interfaces currently return only the result.
  Token metrics need the `usage` blocks the responses already carry. The
  `BatchDistiller`/`BatchEmbedder` use distinct method names (`DistillBatch`/
  `EmbedBatch`) so they are unaffected.
- Extend the interfaces to return a usage value alongside the result:
  - `Distill(ctx, conversation) ([]string, DistillUsage, error)` where
    `DistillUsage struct { InputTokens, OutputTokens int }`.
  - `Embed(ctx, inputs) ([][]float32, EmbedUsage, error)` where
    `EmbedUsage struct { TotalTokens int }`.
- Parse `usage.input_tokens`/`usage.output_tokens` from the Anthropic response
  and `usage.total_tokens` from the OpenAI response. A missing/zero usage block
  is not an error (returns a zero-value usage).
- Update `service.go` callers (`Retrieve` ignores usage; `DistillSession` keeps
  the distill + embed usage) and the test fakes (`fakeEmbedder`,
  `fakeDistiller`, `distillByConv`) to the new signatures.
- [ ] Provider clients parse and return usage; existing provider tests updated
  to assert the parsed token counts; gate green.

### Task A2: `CountIdleUndistilled` repo method

**Files:** `internal/chat/sqlite_repository.go` (+ test),
`internal/vectormemory/job.go` (`SessionSource` interface),
`internal/server/vectormemory.go` (adapter).

- Add `CountIdleUndistilled(ctx, cutoff) (int, error)` to the chat repo: one
  indexed `COUNT(*)` mirroring `IdleUndistilled`'s WHERE clause (no `LIMIT`) —
  the true backlog, not the batch-capped 20.
- Add it to the `vectormemory.SessionSource` interface and the `vmSessionSource`
  adapter, and to the `fakeSessionSource` test fake.
- [ ] Method added with a sqlite test asserting it counts all idle-undistilled
  rows ignoring the batch cap; gate green.

### Task A3: `metrics.go` collectors

**File:** new `internal/vectormemory/metrics.go`.

Define collectors per the SOW table, `api_vectormemory_` prefix, register in
`init()`:
- Gauges: `last_sweep_timestamp_seconds`, `last_success_timestamp_seconds`,
  `idle_sessions`.
- Counters: `sweeps_total{result}`, `sessions_selected_total`,
  `sessions_distilled_total`, `observations_distilled_total`,
  `observations_inserted_total`, `observations_deduped_total`,
  `stage_errors_total{stage}`, `distill_tokens_total{kind}`,
  `embed_tokens_total`.
- Histograms: `sweep_duration_seconds` (`1,2.5,5,10,30,60,120`),
  `distill_duration_seconds` (`0.25,0.5,1,2,4,8,16,32`),
  `embed_duration_seconds` (`0.05,0.1,0.25,0.5,1,2,5`).
- [ ] Collectors compile and register without a duplicate-registration panic.

### Task A4: wire collectors into `service.go`

**File:** `internal/vectormemory/service.go`.

In `DistillSession` (no control-flow changes): time + count the distill call
(`distill_duration`, `distill_tokens{kind}`, `observations_distilled` by len);
on distiller error `stage_errors{stage="distill"}`. Time + count the embed call
(`embed_duration`, `embed_tokens`); on embed error `stage_errors{stage="embed"}`.
On dedup-probe error `stage_errors{stage="dedup"}`; on a skipped near-duplicate
`observations_deduped`. On a per-observation insert error
`stage_errors{stage="insert"}`; on a successful insert `observations_inserted`.

- [ ] Service-level metrics recorded at the existing branch points; gate green.

### Task A5: wire collectors into `job.go`

**File:** `internal/vectormemory/job.go`.

`distillOnce`: stamp `last_sweep_timestamp` at the end of **every** attempt;
set `idle_sessions` from `CountIdleUndistilled` at the start; time the sweep
(`sweep_duration`); `sessions_selected` += batch size. On select error
`stage_errors{stage="select"}`, `sweeps_total{result="error"}`, return (no
`last_success` stamp). Track a per-sweep `stageErr` flag set on load error
(`stage_errors{stage="load"}`), `DistillSession` error, or mark error
(`stage_errors{stage="mark"}`). On clean completion stamp `last_success` and
classify `sweeps_total{result}`: `partial` if `stageErr`, else `success`
(including the zero-selected and zero-observation cases). `sessions_distilled`
is incremented on each marked session.

- [ ] Job-level metrics + result classification wired; gate green.

### Task A6: metrics tests

**File:** new `internal/vectormemory/metrics_test.go` (before/after counter
deltas, `testutil.ToFloat64`, reusing the job/service fakes).

Assert: a clean sweep advances `last_success_timestamp` and increments
`sweeps_total{success}` + `sessions_distilled` + observation counters; a
distill/embed/insert failure increments `stage_errors{stage}` and classifies
the sweep `partial`; a select failure classifies `error` and leaves
`last_success` unchanged; `idle_sessions` reflects the count method; token
counters parse the usage blocks.

- [ ] Tests green; full gate green.

---

## Part B — `ps-vector-memory` dashboard (`prog-strength-infra`)

**File:** new `monitoring/grafana/dashboards/ps-vector-memory.json`,
`uid: ps-vector-memory`, tag `prog-strength`, `refresh: 30s`, range `now-6h`.
Mirror `api.json`: transparent markdown header, `row` sections, a `description`
on every non-row panel, dashed `vector(N)` warning/critical series via byName
overrides. Queries scrape `job="api"` implicitly (the metrics are on the api
job's `/metrics`).

Sections + panels exactly per SOW Part B: **Liveness & cadence** (headline Stat
*Time since last successful sweep* `time() - max(...last_success...)` warn ≥720s
crit ≥1800s; Stat *Sweeps (24h)*; Timeseries *Sweep outcomes* by result; Stat
*Idle backlog (now)* warn ≥50 crit ≥200), **Work done** (Stats sessions
distilled / observations inserted / deduped 24h; Timeseries selected vs
distilled; Timeseries observations distilled/inserted/deduped), **Failures**
(Timeseries stage errors by stage warn 1 crit 3; Stats distill errors 24h /
embed errors 24h warn ≥1 crit ≥5), **Latency & cost** (Timeseries distill
latency p50/p95 warn 10s crit 20s; embed latency warn 2s crit 5s; sweep
duration warn 60s crit 120s; token spend/sec, informational).

- [ ] JSON parses; `uid` unique across the dashboards dir; every non-row panel
  has a non-empty `description`; threshold `vector(N)` series present on the
  alarmable panels.

## Part C — Section the Agent dashboard (`prog-strength-infra`)

**File:** `monitoring/grafana/dashboards/agent.json`. **Existing panel queries
unchanged.** Add a markdown header text panel, `row` separators, a
`description` on every panel that lacks one, and warning/critical `vector(N)`
lines on the alarmable panels. Re-lay `gridPos` so rows and panels stack
cleanly.

Sections per SOW Part C: **Overview** (4 stats), **Routing & intent** (routing
over time, tier split, intent over time, intent share), **Conversation** (chat
turn rate; chat latency p50/p95 warn 3s crit 8s; errors per tier warn 1 crit 3),
**Tools** (tool calls/sec; top tools; tool call latency p95 warn 2s crit 5s;
tool errors/sec warn just above 0), **Voice** (time to first audio p50/p95 warn
1.5s crit 3s — keep/extend the existing 2s SLO note).

- [ ] JSON parses; `uid` still `ps-agent`; every non-row panel has a
  `description`; all 15 original queries preserved verbatim.

---

## Rollout

Deploy API first (metrics exist), then infra (dashboards get data). Decoupled —
either PR reverts independently. Verify `/metrics` exposes `api_vectormemory_*`,
the headline tile reads a small steady value within ~10 min, and exercising a
failure lights the matching `stage_errors{stage}`.
