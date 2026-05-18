# Monitoring and Observability

**Status**: Draft · **Last updated**: 2026-05-18

## Introduction

**Prog Strength** is a serious side project. Two things make monitoring load-bearing for it: platform reliability — the API, MCP server, and agent need to stay up and fast for a credible product — and agent observability — the agent is the thing that makes **Prog Strength** distinctive, and there is no way to make it meaningfully better without measuring what it actually does in production.

The platform has neither today. There is no place to look at API latency or request rate; no place to see how the agent is performing across turns; no record of what tools the agent calls or how often the model router picks Sonnet over Haiku. Bugs are caught only when they break the UI hard enough to notice.

Past experience with CloudWatch and Datadog made custom metrics ruinously expensive, so a vendor stack is off the table. This work stands up a self-hosted Prometheus + Grafana setup on the same EC2 instance the rest of the backend runs on, plus a structured telemetry database the Go API owns. Once shipped, the answers to "is the API healthy?", "is the agent making good decisions?", and "where is it slow?" all live in one place — accessible over SSH tunnel, defined as code, and costs nothing beyond the host that's already running.

## Proposed Solution

Three layers of telemetry, each suited to the shape of its data:

1. **Prometheus + Grafana on the EC2 host** for time-series metrics: latency, request rate, error rate, resource usage. Both the Go API and the Python agent expose a `/metrics` endpoint; Prometheus scrapes both on a fixed interval. Disposable by design — if the host is reprovisioned and the Prometheus volume is lost, no application data goes with it.
2. **A new SQLite database, `telemetry.db`**, owned by the Go API alongside the existing `app.db`. Stores structured per-event records: one row per chat turn, one row per tool call, one row per message. The Go API is still the only service that opens a SQLite handle — the existing architectural rule holds — but now it manages two databases via separate connections.
3. **Internal-only HTTP endpoints** on the Go API for the agent to write telemetry. Caddy does not proxy any path under `/internal/`, so only Docker-network traffic between agent and api can reach these endpoints. Network isolation is the only auth layer; no shared secret needed at the single-host beta scale.

The FastAPI agent service writes telemetry in a fire-and-forget pattern: after every chat turn completes (success or error), it asynchronously POSTs the turn's metadata to the API. The user's chat flow never blocks on telemetry succeeding. If the API is down, the telemetry write fails silently and the user still gets their response.

Three Grafana dashboards, all defined as JSON in `prog-strength-infra/grafana/dashboards/` and provisioned by the Grafana container at boot:

- **Agent Observability** — turns/day, latency percentiles, token usage (input/output/cache), model-router distribution (Haiku vs Sonnet), tool-call breakdown, error rate.
- **API Health** — request rate by route, latency p50/p95/p99 by route, 4xx/5xx rate, top endpoints.
- **Host Health** — CPU, memory, disk, network from `node_exporter`; container restart counts.

Grafana is reachable only via SSH tunnel to the EC2 host. No public Grafana URL, no Caddy route, no external auth integration.

## Goals and Non-Goals

### Goals

- Stand up a Prometheus + Grafana + `node_exporter` stack as three new containers in the existing `docker-compose.yml` alongside the API, MCP server, and agent.
- Add `/metrics` exporters to the Go API and the FastAPI agent service.
- Create `telemetry.db` as a separate SQLite database, owned by the Go API, with its own migration runner.
- Add internal-only endpoints (`POST /internal/telemetry/turns`, `/tool-calls`, `/messages`) reachable only from inside the Docker network.
- Wire the FastAPI agent service to fire-and-forget telemetry writes after each turn, each tool call, and each persisted message.
- Add three Grafana dashboards defined as JSON in `prog-strength-infra`, provisioned by Grafana at boot.
- Implement a 90-day TTL job that NULLs out the `content` column on `agent_messages` while keeping metadata (token counts, timestamps, latency) indefinitely.

### Non-Goals

- **Eval-run storage.** Tracking how the agent does against a fixed test suite is a separate concern with its own schema, lifecycle, and trigger. Filed for a follow-up SOW.
- **Alerting.** No Alertmanager, no PagerDuty, no Slack notifications. Dashboards only — at single-user beta scale, push notifications about a side project would be more noise than signal.
- **Sampling.** Every chat turn is captured. At a single user generating maybe dozens of turns a day, full capture is trivially within SQLite write capacity.
- **MCP server `/metrics`.** Scoped out for MVP; the agent already times its own MCP tool calls, which covers the visible part of the agent → MCP path. The MCP server can grow its own exporter later.
- **Public Grafana.** SSH tunnel only. No Caddy reverse proxy, no Grafana SSO/OAuth, no basic auth front door.
- **Distributed tracing.** OpenTelemetry, Jaeger, Tempo all add complexity that a single-host beta does not need. Structured per-turn rows in `telemetry.db` give us the join key (`turn_id`) to reconstruct the order of events within a turn; that is enough until services span hosts.
- **Persisting `telemetry.db` to S3 via Litestream.** The whole point of separating it from `app.db` is that this data is replaceable. Litestream stays scoped to `app.db`.
- **Surfacing chat history to the user.** This SOW makes the agent's messages durable on the server for the first time, but it does not introduce a UI for the user to browse past chats. That is a product feature, not an observability one.

## Implementation Details

### Data Model

A new SQLite database file at `/data/telemetry.db`, opened by the Go API alongside `app.db`. Three tables, all managed by a dedicated migration runner identical in structure to the existing one but pointed at the new file.

#### `agent_turns`

One row per `/chat` request handled by the agent service.

| Column | Type | Description |
| --- | --- | --- |
| `id` | text | Primary key. |
| `user_id` | text | The authed user the turn belonged to. |
| `session_id` | text | Groups turns within a single conversation. Generated client-side when the chat is first opened and passed back with every `/chat` request. |
| `model` | text | The model that handled the main turn, e.g. `claude-sonnet-4-6`. |
| `routed_tier` | text | `simple` or `complex` — the model router's decision. |
| `router_model` | text | The model the router itself called for classification. |
| `router_latency_ms` | integer | How long the routing decision took. |
| `input_tokens` | integer | Anthropic-reported input tokens for the main turn. |
| `output_tokens` | integer | Anthropic-reported output tokens. |
| `cache_creation_tokens` | integer | Prompt cache write tokens. |
| `cache_read_tokens` | integer | Prompt cache hit tokens. |
| `total_latency_ms` | integer | Wall time from request received to stream completed. |
| `time_to_first_token_ms` | integer | Time to first SSE event from the model. |
| `completion_reason` | text | `end_turn`, `tool_use`, `max_tokens`, `error`. |
| `error` | text \| null | Error class or message when `completion_reason = 'error'`. |
| `started_at` | text (RFC3339) | When the agent service received the request. |
| `ended_at` | text (RFC3339) | When the response finished streaming. |
| `created_at` | text (RFC3339) | Row creation timestamp. |

Indexes:

- `(started_at DESC)` — "recent turns" queries.
- `(user_id, started_at DESC)` — per-user analyses (one user today, but the schema should not assume that).
- `(user_id, session_id, started_at)` — the dominant query for the future chat-history UI: "give me every turn in this conversation, oldest first."

`session_id` is the join key the future chat-history feature will use to render a list of past conversations and the messages inside each. Putting it on every turn now means that feature is a UI-only build — no schema migration needed when it ships.

#### `agent_tool_calls`

One row per MCP tool invocation made during a turn.

| Column | Type | Description |
| --- | --- | --- |
| `id` | text | Primary key. |
| `turn_id` | text | Foreign key to `agent_turns(id)`. Cascades on delete. |
| `tool_name` | text | E.g. `list_workouts`, `create_workout`. |
| `arguments_json` | text | The tool input as JSON. Kept for debugging; subject to the same 90-day TTL as message content. |
| `result_summary` | text \| null | First N characters of the tool's response, or null on error. |
| `latency_ms` | integer | Tool call duration. |
| `error` | text \| null | Error message if the tool call failed. |
| `started_at` | text (RFC3339) | |
| `ended_at` | text (RFC3339) | |
| `created_at` | text (RFC3339) | |

Index: `(turn_id)` for joining tool calls to their turn.

#### `agent_messages`

One row per user or assistant message in the conversation. This is the only table that stores the actual conversation content.

| Column | Type | Description |
| --- | --- | --- |
| `id` | text | Primary key. |
| `turn_id` | text | Foreign key to `agent_turns(id)`. Cascades on delete. |
| `role` | text | `user` or `assistant`. |
| `content` | text \| null | The message text. NULL'd out by the TTL job after 90 days. |
| `token_count` | integer \| null | Per-message token count when available. |
| `created_at` | text (RFC3339) | |

Index: `(turn_id, created_at)` for ordered retrieval of a turn's messages.

### Write Path

Every write to `telemetry.db` originates from the FastAPI agent service via the API's `/internal/telemetry/*` endpoints. The Go API is the sole writer; the agent does not open a SQLite handle.

`session_id` enters the system at the frontend: when the chat page first mounts, it generates a UUID and stores it in component state. Every `/chat` request includes that ID in its body. The agent passes it through to the telemetry write unchanged. A new session begins when the user opens a fresh chat — for the MVP UI that means a page refresh; a "new conversation" button is the natural product follow-up. Requests without a `session_id` (older clients, scripted calls) get one generated server-side so no turn lands in the database unowned.

- **Turn started** — the agent generates a turn ID, holds it locally alongside the inbound `session_id`, and processes the chat request. No row written yet.
- **Turn completed (success or error)** — the agent fires three async POSTs in parallel, none of which block returning the SSE stream to the user:
  - `POST /internal/telemetry/turns` with the turn row (session_id, model, tokens, latency, completion reason, etc.).
  - `POST /internal/telemetry/tool-calls` with the array of tool calls made during the turn (zero or more).
  - `POST /internal/telemetry/messages` with the user message and the assistant message.
- **API receives the writes** — each handler validates the payload and inserts into `telemetry.db` inside a transaction. The endpoints are not chained — each one is independently retryable, idempotent on `id`.

Failure semantics: if any telemetry write fails (API down, schema mismatch, network blip), the agent logs locally and moves on. Telemetry loss is preferable to user-visible chat failure. No retries on the agent side.

### Algorithms

**90-day TTL on message content.** A scheduled job inside the Go API runs once a day and NULLs out aged content:

```sql
UPDATE agent_messages
SET content = NULL
WHERE content IS NOT NULL
  AND created_at < datetime('now', '-90 days');

UPDATE agent_tool_calls
SET arguments_json = NULL, result_summary = NULL
WHERE arguments_json IS NOT NULL
  AND created_at < datetime('now', '-90 days');
```

NULL'ing the content rather than deleting the row preserves the metadata signal (token counts, timestamps, role, turn linkage) forever, while dropping the actual text after the retention window. Same approach for `agent_tool_calls.arguments_json` and `result_summary` — the tool name, latency, and error fields stay; the inputs and outputs age out.

The job runs as a `time.Ticker` inside the API process. No external scheduler. At single-user scale the rows-to-update count per day is tiny; the update completes in milliseconds and locks nothing user-facing.

### API Surface

Three internal-only endpoints on the Go API, all `POST`, all JSON, all reachable only from inside the Docker network because Caddy strips `/internal/*` paths from public routing.

- **`POST /internal/telemetry/turns`** — accepts one turn record. Returns 201 on success, 400 on validation error.
- **`POST /internal/telemetry/tool-calls`** — accepts an array of tool calls for a single `turn_id`. Returns 201 on success.
- **`POST /internal/telemetry/messages`** — accepts an array of messages for a single `turn_id`. Returns 201 on success.

These endpoints do not validate that the calling identity is "the agent"; they trust the network layer. Anyone with shell access to the EC2 host inside the Docker network can hit them, but that's the same threat model as direct SQLite access.

Additionally, two `/metrics` endpoints, scraped by Prometheus on a 15-second interval:

- **Go API**: `/metrics` on the API container's listening port, served by `prometheus/client_golang`. Standard HTTP middleware records request count, latency histogram, and error count by route and status code. Process metrics (goroutines, GC pauses, heap size) come for free.
- **FastAPI agent**: `/metrics` on the agent container, served by `prometheus-fastapi-instrumentator`. Same shape: request count, latency, error rate by route.

Both `/metrics` endpoints are also `/internal/`-stripped at the Caddy layer so the public can't scrape them.

### Container Layout

Three new services in `docker-compose.yml`:

- **`prometheus`** — official `prom/prometheus` image. Config mounted from `prog-strength-infra/prometheus/prometheus.yml`. Default 15-day TSDB retention. Volume mounted at `/prometheus` for the time-series store.
- **`node_exporter`** — official `prom/node-exporter` image. Mounts `/proc`, `/sys`, and `/` read-only for host metrics. Scraped by Prometheus alongside the application services.
- **`grafana`** — official `grafana/grafana` image. Dashboards and datasources provisioned via files mounted from `prog-strength-infra/grafana/`. Volume mounted at `/var/lib/grafana` for the SQLite-backed Grafana config store. Bound to `127.0.0.1:3000` only on the host, so the public internet cannot reach it; SSH tunnel forwards a local port to access the UI.

No service depends on Prometheus or Grafana for startup. If the metrics stack is down, the API and agent keep serving traffic — `/metrics` endpoints just don't get scraped.

### Backfill or Migration

- **Mechanism**: a separate Go migration runner pointed at `/data/telemetry.db`. Same embedded-SQL pattern as the existing migrations, but mounted under a different `//go:embed` directive (e.g. `internal/db/telemetry_migrations/`). Initial migration creates the three tables and their indexes.
- **Recoverability**: telemetry data is replaceable by design. If `telemetry.db` is corrupted or lost, truncate it and let it repopulate from new traffic. No backfill from `app.db` is possible — pre-feature chat history was never persisted.
- **Scale boundary**: at single-user scale the database stays small for a long time. If it later grows past a few GB or if SQLite locking ever shows up on the latency dashboards, the natural escalation is a dedicated write process (e.g. a small Litestream-style writer-process for telemetry) or moving telemetry to a columnar store (DuckDB). Both are far away; current decision is the simplest one that works.

## Open Questions

1. **What goes in `agent_messages` exactly?** The user's prompt and the assistant's response are obvious. Does the system prompt count? Does each tool result message Anthropic injects into the next turn count? **Lean**: log only the human-facing pair — user message and final assistant response — to keep the table understandable and the content small. System prompt is captured separately (by hash + version) as a turn-level field if it becomes useful.

2. **Tool arguments storage**. The `arguments_json` column carries the full tool input. For `create_workout` that includes the entire structured workout. At single-user scale this is fine; at higher volume the column could dominate database size before the TTL kicks in. **Lean**: store full JSON, rely on the 90-day TTL, revisit if writes become hot.

3. **Grafana SQLite datasource for telemetry.db**. Grafana can read SQLite directly via a community datasource plugin. The alternative is for the API to expose `/internal/telemetry/query` endpoints that the Grafana panels hit. **Lean**: SQLite datasource plugin — simpler, no API surface to maintain — but worth confirming the plugin is healthy enough for production use before committing.

4. **Sensitive content in chats**. Lifters could share private health information in the chat. The 90-day TTL contains the long-tail exposure, but anyone with EC2 SSH access can read recent content. **Lean**: out of scope for this SOW; raise as a separate threat-modeling exercise once the platform actually has more than one user.
