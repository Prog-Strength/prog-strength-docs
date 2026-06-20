---
status: draft
depends_on:
  - sows/centralized-api-config.md
repos:
  - prog-strength-api
  - prog-strength-agent
  - prog-strength-docs
---

# Agent Vector Memory

**Status**: Draft · **Last updated**: 2026-06-20

## Introduction

The **Prog Strength** coach has no memory of the user beyond what's in front of it. Every `/chat` turn ships the current conversation's message array and nothing else; the agent answers, the turn is persisted to `chat_messages`, and that history is never consulted again unless the user reopens that exact thread. The agent already recalls **structured** facts well — macro goals, step goals, logged workouts, bodyweight — because those live in typed application tables that MCP tools read on demand. What it can't recall is the **unstructured** signal that never becomes a row in a domain table: "I travel for work most weeks and can only train in hotel gyms," "my left shoulder flares up on overhead pressing," "I'm cutting for a wedding in August," "I hate running but I'll do the bike." Today that context is stated once, scrolls out of the conversation, and is gone.

This work gives the agent **semantic memory** over the user's past interactions. A cheap background pass distills durable observations from completed conversations, embeds them, and stores them as vectors. At request time, the incoming prompt is embedded and the most similar stored memories — and only those above a relevance threshold — are injected into the agent's context as background. The coach starts to feel like it *knows* the user across conversations, without the user having to repeat themselves.

This is deliberately **not** a second structured-recall system. Structured facts and preferences are already handled by existing MCP tools over the application DB, and that boundary stays. Vector memory targets the fuzzy contextual signal those tools can't capture. It is also deliberately **boring**: no new datastore, no new container, no new service. Memories live in the existing `app.db` SQLite file via the `sqlite-vec` extension, the Go API remains the sole reader and writer of vectors, and the whole feature is backed up by the existing Litestream→S3 replication with zero new infrastructure.

## Proposed Solution

A new `vectormemory` domain in `prog-strength-api` owns the entire vector lifecycle — distillation, embedding, storage, and retrieval — because the API is the single writer of `app.db` and that invariant must hold. The MCP server is not involved; it never touches vectors. The agent gains exactly one new dependency: a best-effort call to an internal retrieval endpoint that runs in parallel with the existing router.

**Storage.** Two tables in `app.db`: a plain `agent_memories` table (the distilled text plus provenance and model metadata, readable with ordinary SQL) and a `vec_agent_memories` virtual table provided by the `sqlite-vec` extension (the float vectors, queried by approximate nearest-neighbour). The extension is loaded once via a connection hook in `db.Open`, so it is available process-wide on every SQLite connection with no per-query wiring. Because everything is in `app.db`, the source link to the originating conversation is a real foreign key, not a soft reference.

**Source of truth.** Distillation reads from `chat_messages` — the durable, indefinitely-retained system-of-record for conversation content that already backs the chat history UI. It does **not** read from `telemetry.db`'s `agent_messages`, which is an observability copy with a 90-day content TTL. This keeps the whole feature inside one database and means a memory always traces back to a real, non-expiring message.

**Write path (distillation).** A scheduled background job in the API — modelled on the existing telemetry content-TTL sweep — finds chat sessions that have gone idle (no new message for a configurable window) and have not yet been distilled. For each, it makes a single cheap Haiku call that reads the conversation and returns zero or more **atomic durable observations**. Each observation is embedded with OpenAI `text-embedding-3-small` and stored as its own memory row, tagged with the embedding model name and dimension. A write-time dedup check skips observations that are near-identical to something the user already has, so the index doesn't accumulate restatements of the same fact.

**Read path (retrieval).** The agent's existing `_route_and_stream` already runs a Haiku router to classify intent. Retrieval is added as a sibling task that runs *alongside* it under `asyncio.gather`: the agent posts the latest user message to `POST /internal/memory/retrieve`, the API embeds it, runs a per-user nearest-neighbour search, drops everything below the relevance threshold, caps the rest at top-k, and returns the matching distilled texts. Those are injected into the system prompt through the same `compose_system_prompt(base, rules, data)` seam the intent layer already uses, as a clearly-labelled **background** block — context for the coach, not instructions. The call is best-effort with a tight timeout: on error, timeout, or no matches above threshold, nothing is injected and behaviour is unchanged. Retrieval's latency hides underneath the router call, so the feature adds no perceptible turn latency.

**Threshold is the product.** Top-k is never injected unconditionally — a distance cutoff decides what counts as a real match versus noise, and that cutoff is the primary thing to tune. To tune it empirically the API ships first-class inspection and probing tools: a flat admin dump of all memories, and an admin search endpoint that runs the **exact same retrieval code path the agent uses** and returns ranked results *with their distance scores and the active threshold*. A thin CLI wraps both.

**Configuration.** The only secrets are the two API keys (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`), which stay in the environment. Every tuning knob — the threshold, top-k, the idle window, model names, the dedup cutoff, the feature on/off switch — is registered as a `[vectormemory]` section in the centralized, version-controlled `config.toml` introduced by `sows/centralized-api-config.md` (a hard dependency of this work), so changes are auditable and reviewable, not in an ever-growing list of environment variables.

## Goals and Non-Goals

### Goals

- Add the `sqlite-vec` extension to `prog-strength-api`, loaded once via a connection hook in `db.Open`, available on every `app.db` connection. The Docker build links the extension cleanly.
- Persist distilled memories in `app.db`: an `agent_memories` table (distilled text, `user_id`, source FKs, embedding model + dimension, timestamps, dedup/supersede metadata) and a `vec_agent_memories` `sqlite-vec` virtual table holding the float vectors, scoped for per-user search.
- A scheduled distillation job that finds idle, not-yet-distilled chat sessions; makes one Haiku call per session to extract atomic durable observations from `chat_messages`; embeds each with `text-embedding-3-small`; and stores it with a write-time near-duplicate check.
- An internal retrieval endpoint `POST /internal/memory/retrieve` that embeds a query, runs a threshold-gated, top-k-capped per-user nearest-neighbour search, and returns the matching distilled texts.
- Agent integration: retrieval runs in parallel with the existing router via `asyncio.gather`, is best-effort (failure/timeout/empty ⇒ inject nothing), and the results are composed into the system prompt as a labelled background block.
- First-class inspection + probing tooling: `GET /admin/memories` (flat paginated dump) and `POST /admin/memories/search` (the production retrieval path, returning distances + the active threshold), both admin-gated. A `cmd/memctl` CLI wraps them (curl + jq to start).
- A one-time backfill command that distills and embeds historical conversations, using the Anthropic Message Batches API and the OpenAI Batch embeddings API (half price, async) for the one-shot.
- Store embedding model name + dimension on every vector, and have retrieval only compare vectors produced by the currently-active model, so a future model swap can never silently compare incomparable vectors.
- Keep all non-secret configuration (threshold, top-k, idle window, dedup cutoff, model names, enable flag) in a checked-in config file; only `OPENAI_API_KEY` and `ANTHROPIC_API_KEY` are environment secrets.
- MCP stays entirely uninvolved: it never reads or writes vectors.

### Non-Goals

- **Structured fact / preference recall.** Macro goals, step goals, workouts, bodyweight, etc. are already served by existing MCP tools over the application DB. Vector memory does not replace, duplicate, or wrap them.
- **Exposing the probe as an MCP tool** (e.g. `debug_search_memory`). A nice future convenience for the agent, but not foundational. Probing is an admin/operator tool in v1.
- **Migrating off `sqlite-vec` / standing up Qdrant.** `sqlite-vec` in `app.db` is the chosen store. Qdrant is the noted clean upgrade path *if* scale or metadata-filtering needs ever demand it — explicitly not now.
- **Moving `agent_messages` out of `telemetry.db`.** The canonical transcript is already `chat_messages` in `app.db`; the telemetry copy is observability data with its own retention policy and stays put.
- **Real-time / per-turn distillation.** Distillation is batched per idle session, not run on every turn. (See Open Questions for the idle-window value.)
- **User-visible memory management.** No UI for users to view, edit, or delete their own memories in v1. Memories are an internal context-enrichment mechanism, inspected by the operator, not a user-facing surface.
- **Cross-user or shared memory.** Memories are strictly per `user_id` and never visible across users.
- **Retrieval inside the active session.** The current conversation's recent turns are already in the agent's context; memory targets cross-conversation recall. (Flagged in Open Questions.)

## Implementation Details

The sections below follow lifecycle order: data model → extension loading → config → write path → read path → agent integration → admin tooling → backfill → model migration → tests → rollout.

### Data Model

One new migration, `internal/db/migrations/034_agent_memory.sql`, adds two tables to `app.db`. (Numbered after `033_workout_tcx_association.sql`, the latest on `main`; rebase before implementing in case the ledger has advanced further.)

`agent_memories` — one row per distilled observation. Plain table, readable with ordinary SQL (this is what the inspection dump reads, and what catches bad distillation or a mixed-model index):

| Column | Type | Description |
| --- | --- | --- |
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | Stable row id; doubles as the `rowid` joined to the vector table. |
| `user_id` | TEXT NOT NULL | Owner. Soft reference, consistent with the rest of the schema. Indexed. |
| `distilled_text` | TEXT NOT NULL | The atomic durable observation (the thing that gets embedded). |
| `source_session_id` | TEXT NOT NULL REFERENCES `chat_sessions(id)` ON DELETE CASCADE | Originating conversation. Real FK — same DB. Memory dies with its session. |
| `source_message_id` | INTEGER REFERENCES `chat_messages(id)` ON DELETE SET NULL | Best-effort pointer to the turn that triggered the observation, where attributable. |
| `embedding_model` | TEXT NOT NULL | e.g. `text-embedding-3-small`. Stored per row so a model swap is detectable. |
| `embedding_dim` | INTEGER NOT NULL | e.g. `1536`. Guards against comparing incomparable vectors. |
| `superseded_at` | DATETIME | Set when a re-embed or dedup-merge retires this row; non-null rows are excluded from retrieval. |
| `created_at` | DATETIME NOT NULL | Distillation time. |

Indexes: `(user_id)` for per-user scans in the dump and dedup; `(source_session_id)` for cascade and provenance lookups.

`vec_agent_memories` — the `sqlite-vec` virtual table holding the float vectors:

```sql
CREATE VIRTUAL TABLE vec_agent_memories USING vec0(
    memory_id INTEGER PRIMARY KEY,   -- == agent_memories.id
    user_id   TEXT,                  -- partition/filter column for per-user KNN
    embedding float[1536]
);
```

The `user_id` filter column keeps nearest-neighbour search scoped to one user. The exact mechanism for efficient per-user filtered KNN in `sqlite-vec` (partition key vs. metadata column vs. over-fetch-and-join) is an Open Question for the implementer to benchmark; the schema above expresses intent. `embedding_dim` in `agent_memories` must match the `float[N]` declared here; if the active embedding model changes dimension, that's a new vector table (see Model-Version Migration).

`chat_sessions` gains one nullable column (small alter in the same migration) to track distillation state:

| Column | Type | Description |
| --- | --- | --- |
| `memory_distilled_at` | DATETIME | NULL = not yet distilled. Set when the distillation job has processed this session. The job's work-list is "idle and `memory_distilled_at IS NULL`." |

### Loading the `sqlite-vec` Extension

The API uses `mattn/go-sqlite3` (CGO), which supports SQLite extensions. Use the official `github.com/asg017/sqlite-vec-go-bindings/cgo` package, which registers the extension with the driver. Loading happens once, via a `sql.Register`ed driver with a `ConnectHook` (or the binding's auto-registration) in `internal/db/db.go`, *before* `db.Open` returns — so every pooled connection has `vec0` available. Litestream is unaffected: it replicates the SQLite file at the WAL level and is oblivious to which virtual tables exist.

**This is the single biggest technical risk and must be de-risked first.** The Docker image must compile/link the extension against the right libc (the build base is musl/glibc-sensitive). The first unit of work is a throwaway spike: build the image, open a DB, `CREATE VIRTUAL TABLE … USING vec0`, insert and KNN-query a vector, in CI. Nothing else proceeds until that passes.

### Configuration

The feature reads two env secrets and a set of non-secret knobs registered as a `[vectormemory]` section in the centralized `config.toml` (`sows/centralized-api-config.md`). It does **not** introduce its own config file. **Secrets (GitHub secrets, `${VAR}` labels in the central file):**

- `OPENAI_API_KEY` — embeddings. New to the API.
- `ANTHROPIC_API_KEY` — Haiku distillation. New to the API (already anticipated for other API-side LLM work).

**Non-secret tuning knobs — the `[vectormemory]` section of the central `config.toml`**, public literals, version-controlled and annotated:

| Key | Default (starting point) | Meaning |
| --- | --- | --- |
| `enabled` | `true` | Master kill-switch. `false` ⇒ no distill, no retrieve, no behaviour change. Version-controlling its state is intentional — git shows when it flipped. |
| `distance_threshold` | TBD (tune) | Cosine distance cutoff. Matches at or above this distance are treated as noise and excluded. **The primary tuning knob.** |
| `top_k` | `5` | Max memories injected per turn (return all above threshold, capped here). |
| `dedup_threshold` | TBD (tune) | Write-time cutoff: a new observation nearer than this to an existing one for the same user is skipped/merged. |
| `session_idle_minutes` | `30` | A session with no new message for this long is eligible for distillation. |
| `distill_model` | `claude-haiku-4-5-20251001` | Model for the distillation pass. |
| `embed_model` | `text-embedding-3-small` | Embedding model. Persisted on every vector. |
| `embed_dim` | `1536` | Vector dimension. Must match the `vec0` table declaration. |

These knobs live alongside the rest of the API's configuration in the central `config.toml`, not in a feature-specific file — the centralization SOW is what makes that the single home for non-secret settings.

### Go API: `internal/vectormemory` package

Follows the established domain shape (`models.go`, `repository.go` + `sqlite_repository.go`, `service.go`, `handler.go`):

- **`provider.go`** — two thin HTTP clients behind interfaces, matching the existing `nutritionlookup` provider pattern (single injected `*http.Client` with a bounded timeout, context-aware, no retry loops):
  - `Embedder` — `Embed(ctx, []string) ([][]float32, error)` against OpenAI embeddings. One implementation for live calls; the Batch variant is used only by the backfill command.
  - `Distiller` — `Distill(ctx, conversation) ([]string, error)` against Anthropic Haiku, forced to return a JSON array of atomic observations (possibly empty) via a tool-call schema, mirroring how the agent's router forces structured output.
- **`repository.go` / `sqlite_repository.go`** — insert a memory (text row + vector row in one transaction), nearest-neighbour search scoped by `user_id` with a distance cap and limit, the flat dump query, the dedup probe, and the supersede/re-embed writes.
- **`service.go`** — the orchestration that both the internal retrieval endpoint and the admin search endpoint call. **This is the single retrieval code path** the probe tool must reuse; it must not be reimplemented.
- **`handler.go`** — mounts `POST /internal/memory/retrieve` (internal, Docker-network, ungated like the telemetry endpoints) and the two `/admin/memories*` routes (admin-gated).

### Write Path: Distillation Job

A background goroutine started at server boot, modelled on `telemetry.StartContentTTL` (ticker loop, context-cancelled on shutdown, errors logged not fatal). Each tick:

1. **Find work.** Select `chat_sessions` where `last_message_at < now − session_idle_minutes` AND `memory_distilled_at IS NULL`, owned by any user. Bounded batch per tick.
2. **Read the conversation.** Load that session's `chat_messages` (durable content, no TTL) in order.
3. **Distill.** One Haiku call → 0..N atomic observations. The prompt instructs the model to extract only *durable, stable* facts/preferences/context worth remembering long-term (training constraints, injuries, equipment access, goals, dietary patterns, stated dislikes), and to return an empty array when a conversation holds nothing durable (most "log my workout" turns will). Prompt design and the definition of "durable" will be iterated against real data via the probe tool (Open Question).
4. **Dedup + embed + store.** For each observation: embed it; nearest-neighbour probe against that user's existing memories; if the nearest is within `dedup_threshold`, skip (or bump/merge — see Open Questions) rather than insert; otherwise insert the text row and vector row in one transaction.
5. **Mark done.** Set `memory_distilled_at = now` on the session regardless of how many observations it yielded (including zero), so it isn't re-examined every tick.

Cost is one Haiku call per conversation (not per turn) plus negligible embedding cost — the cheapest useful posture.

### Read Path: Retrieval

`POST /internal/memory/retrieve`

Request: `{ "user_id": "...", "query": "<latest user message text>", "k": 5, "threshold": 0.0 }` (`k` and `threshold` optional; default to config; a per-request override exists for the probe).

Behaviour: embed `query` with `embed_model`; run per-user KNN over `vec_agent_memories` restricted to rows whose `embedding_model` matches the active model and `superseded_at IS NULL`; drop matches beyond `distance_threshold`; cap at `k`; return the surviving distilled texts (with distances — the agent ignores them, the probe surfaces them).

Response: `{ "memories": [ { "text": "...", "distance": 0.18, "source_session_id": "...", "created_at": "..." } ] }`. Empty array when nothing clears the threshold.

### Agent Integration (`prog-strength-agent`)

The agent change is small and contained in `_route_and_stream`. Today it `await`s the router, then streams the chosen harness. Change:

```python
# run retrieval alongside intent classification, not instead of it
decision_task = router_obj.route(messages, telemetry=telemetry, prior_intent=prior_intent)
memory_task   = retrieve_memories(api_client, user_id, _last_user_text(messages))
decision, memories = await asyncio.gather(
    decision_task, memory_task, return_exceptions=True,
)
if isinstance(memories, Exception) or not memories:
    memories = []   # best-effort: never block or fail the turn on memory
```

`retrieve_memories` is a new `APIClient` method calling `POST /internal/memory/retrieve` with a bounded timeout (a little larger than the existing 0.2s intent-hint call, to cover the embedding round-trip; still well under the router's ~500ms so it adds no perceptible latency). The returned texts are formatted into a labelled background block and threaded through to the harness, where they join the system prompt via the existing `compose_system_prompt(base=..., rules=..., data=...)` composition — concretely, the memory block is appended to the `data` channel (or added as a third, explicitly-labelled section) so the model reads it as background context about the user, not as instructions. When `memories` is empty, nothing is added and the prompt is byte-for-byte what it is today.

The agent never embeds, never queries `sqlite-vec`, and gains no new persistence — the single-writer invariant holds. Config: the agent already holds the API base URL; no new secrets.

### Admin Tooling: Inspection + Probing

Both routes are admin-gated using the existing `auth.RequireAdmin(userRepo, cfg.AdminEmails)` pattern (the same gate as `/admin/beta-emails`).

- **`GET /admin/memories`** — inspection. A flat, paginated dump straight from `agent_memories`: `distilled_text`, `user_id`, `source_session_id`, `embedding_model`, `embedding_dim`, `created_at`, `superseded_at`. Plain SQL read, no embedding. Supports `?user_id=` and `?limit/offset`. This is what catches bad distillation output and a mixed-model index at a glance.
- **`POST /admin/memories/search`** — probing. Body `{ "query": "...", "user_id": "...", "k": 10, "threshold": <override> }`. It calls the **same `service` retrieval method the agent's endpoint calls** — no hand-rolled query logic — and returns ranked results *with distance scores and the active threshold echoed back*, so the cutoff can be tuned empirically against real queries. The `threshold` override lets the operator sweep cutoffs without redeploying.

- **`cmd/memctl`** — a thin CLI wrapper over the two admin routes (curl + jq is acceptable to start; promote to `cobra` only if it grows). Subcommands `list` and `search`. It exists so tuning the threshold and eyeballing the index is a one-liner, not a manual HTTP dance.

### Backfill (one-time)

A `cmd/`-level command (run once, manually) that seeds the index from history:

1. Enumerate historical `chat_sessions` not yet distilled.
2. Distill each via the Anthropic **Message Batches API** (half price, async is fine for a one-shot) → observations.
3. Embed all observations via the OpenAI **Batch embeddings API** (half price, async).
4. Insert text + vector rows with the same dedup check as the live path; set `memory_distilled_at`.

Only sessions whose `chat_messages.content` is present are eligible (it is, for `chat_messages` — no TTL there; this is a non-issue but stated for clarity). Recoverable and idempotent: re-running skips already-distilled sessions via `memory_distilled_at`.

### Model-Version Migration

Every vector carries `embedding_model` + `embedding_dim`, and retrieval filters to the active model — so vectors from two different models are never compared, even transiently. Swapping the embedding model is a re-embed-in-place backfill: read each live memory's `distilled_text`, re-embed with the new model, write the new vector (new `vec0` table if the dimension changed), flip the active model in config, and mark the old rows `superseded_at`. No dual-index/transition window is needed at this scale — the filter makes a mixed index correct (if slower) during the migration, and the supersede sweep cleans up after. Distilled text is retained precisely so re-embedding never requires re-running Haiku.

### Tests

| Repo | Coverage |
| --- | --- |
| `prog-strength-api` | **Build/extension spike** in CI: image builds, `vec0` table creates, vector insert + KNN round-trips. Repository: insert text+vector in one txn; per-user KNN respects `user_id` scoping, threshold cutoff, and `k` cap; dedup probe skips near-duplicates; supersede excludes rows from retrieval. Service: retrieval returns ordered-by-distance, threshold-gated results; the admin search path and the internal retrieve path return identical rankings for the same input (proves shared code path). Distillation job: idle+undistilled selection logic; marks `memory_distilled_at` even on zero observations; cross-checks it reads `chat_messages` not telemetry. Handlers: `/internal/memory/retrieve` shape; `/admin/memories*` admin-gating (403 without admin); per-request `threshold`/`k` overrides. Providers tested against fakes (no live OpenAI/Anthropic in unit tests). |
| `prog-strength-agent` | `_route_and_stream` runs retrieval and routing concurrently; a retrieval exception/timeout/empty result injects nothing and never blocks the turn; non-empty memories are composed into the system prompt as a background block; prompt is unchanged when memory is empty or `enabled=false`. `retrieve_memories` client: timeout + error handling returns `[]`. |
| `prog-strength-docs` | This SOW; status transitions. |

Follow the existing conventions: stdlib `testing`, `httptest`, per-test temp `app.db` via `dbtest.NewTestDB` (extended to load the `vec0` extension), fakes for external providers. No live API calls in the test suite.

### Rollout

1. **Extension spike** lands first (CI proves `vec0` builds + queries in the Docker image). Gate everything else on it.
2. **API schema + package + config file**, behind `enabled=false`. Ship dark — no distillation, no retrieval, zero behaviour change.
3. **Distillation job + backfill**, still `enabled=false` for retrieval but distillation populating the index. Use the inspection dump to eyeball distillation quality; iterate the distill prompt.
4. **Tune the threshold** with `POST /admin/memories/search` / `memctl search` against the now-populated index before any injection happens.
5. **Agent retrieval** lands; flip `enabled=true`. All users, always-on, with the kill-switch one config commit away if anything looks off.

Because the index can be populated and the threshold tuned while retrieval is still dark, the user-facing behaviour change (step 5) ships only after the cutoff is calibrated against real data.

## Open Questions

1. **Efficient per-user filtered KNN in `sqlite-vec`.** Partition key vs. metadata column vs. over-fetch-then-join-and-filter. Benchmark on a realistic index size; pick the one that stays correct and fast. The schema in Data Model expresses intent, not the final mechanism.
2. **Distillation prompt + the definition of "durable."** What counts as worth remembering long-term, and how aggressively to extract. Must be iterated against real conversations via the inspection dump; the starting prompt is a first draft, not the spec.
3. **`distance_threshold` and `dedup_threshold` defaults.** Both marked TBD above; set empirically via the probe before go-live. The whole rollout is sequenced so this happens before injection.
4. **`session_idle_minutes`.** 30 minutes is a starting point. Too short and an active conversation gets distilled mid-stream and re-touched; too long and memory is stale. Validate against real session cadence.
5. **Dedup semantics: skip vs. merge vs. bump.** When a near-duplicate observation arrives, do we drop it, update the existing row's timestamp (signal of reinforcement), or merge text? Start with the simplest (skip) and revisit if the index shows drift.
6. **Retrieval within the active session.** Currently out of scope (recent turns are already in context). Reconsider if memories from earlier in a very long single session prove useful.
7. **`k` fixed vs. threshold-only.** Shipping as "return all above threshold, capped at `k=5`." If real usage shows the threshold alone is sufficient and the cap never binds (or binds badly), revisit.
8. **Model-swap mechanics at scale.** Re-embed-in-place is specified for current scale. If the index grows large enough that a re-embed sweep is heavy, reconsider a staged dual-index — explicitly deferred until the numbers demand it.
