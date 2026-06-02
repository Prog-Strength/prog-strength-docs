# Intent-Driven Context Enrichment

**Status**: Approved · **Last updated**: 2026-06-02

## Introduction

The **Prog Strength** agent today routes every chat turn through a one-call Haiku classifier that decides between a cheap simple-tier model (CRUD-shaped requests like "log my workout") and a more capable complex-tier model (planning, analysis). The decision is correct often enough — but it's the only thing the router does, and that turns out to be a ceiling on how good the agent feels for the most common interaction: logging meals.

The lived experience: "Log a protein shake for dinner" comes in and the agent dutifully starts asking — what's the brand, how many scoops, how many grams of protein, how many calories — as if the user hadn't already saved a "Whey Isolate" pantry item three months ago. The agent has the `list_pantry_items` tool available; it just doesn't reach for it on its own initiative. The user ends up spoon-feeding macros they already entered once, and a workflow that should be one sentence becomes an interrogation. At that point it's faster to fill out the form on `/nutrition` than to use the agent at all, which defeats the whole point.

The same shape recurs across the app's core write-paths. Logging a workout works, but the agent still calls `list_exercises` defensively even when the user said the exercise name plainly. Logging a bodyweight reading drops in as a number with no acknowledgement of trend. Asking the agent for progress analysis sometimes results in three tool calls before the model says anything substantive. In each case the agent is *capable* — the tools exist, the data is there — but the prompt-to-tool-call latency tax is paid every turn because the router has no notion of "this is a nutrition-logging conversation; bring the pantry along."

After this work ships, the router classifies each turn into both a tier AND an intent. Each known intent carries a small set of MCP prefetch calls and a behavioral rule block that the harness injects into the system prompt before the main model call. A `log_nutrition` turn arrives with the user's pantry items and recipes already in context plus rules like "assume one serving unless the user specifies otherwise"; the agent stops asking for things it should have looked up itself. Intent is also persisted on the chat session so multi-turn flows ("let me log dinner" → "what did you have?" → "a protein shake") keep the enrichment alive across follow-up turns.

## Proposed Solution

The existing single-output `ModelRouter` becomes a structured-output classifier emitting `{tier, intent}` in one Haiku call — same latency, same cost. The intent is one of a closed v1 set: `log_nutrition`, `log_workout`, `log_bodyweight`, `analyze_progress`, or `general` (fallback for everything else).

A new **`IntentRegistry`** module owns the per-intent behavior. For each known intent it declares:

1. A **prefetch** function that runs one or more MCP tool calls in parallel against the already-open MCP session.
2. A **rules** string — a behavioral addendum appended to the system prompt for that turn (e.g. "Assume one serving unless the user specifies otherwise.").
3. A **format** function that turns the prefetched data into a system-prompt context block ("USER'S CURRENT PANTRY: …").

`general` intent is a no-op (no prefetch, no rules) so the agent behaves exactly like today when the classifier can't identify a specific intent.

The `ModelHarness` is extended to accept the chosen intent. After `session.initialize()` and `list_tools()`, it calls `IntentRegistry.prefetch(intent, session)` to run the intent's tool calls concurrently via `asyncio.gather`. It then composes the per-turn system prompt as:

```
<today's date prefix>
<SYSTEM_PROMPT>
<intent rules block>     # omitted if intent == general
<intent data block>      # omitted if prefetch returned no data
```

…and runs the existing tool-use loop unchanged. The main model sees both the user's question AND the relevant slice of their data, and the rules nudge it toward fewer clarifying questions.

Multi-turn intent continuity rides on **`chat_sessions.last_intent`** — a new column on the API's existing `chat_sessions` table. The agent reads the prior intent via a new `GET /internal/chat-sessions/{id}/intent` endpoint at the start of each turn and passes it to the router as a hint ("Previous intent was `log_nutrition`. The user just said: '…'. Is this still nutrition logging, or did they switch?"). The classifier still runs every turn — the hint just biases it, so a user pivoting from logging to asking a planning question gets classified correctly. The new intent is written back to `chat_sessions` after the turn via the existing fire-and-forget telemetry pipeline.

Telemetry gains a new `intent` field on `TurnInstrumentation`, mirroring the shape of the existing `routed_tier` plumbing. Two new Prometheus series — `agent_intent_classifications_total{intent}` and `agent_intent_prefetch_duration_seconds{intent}` — feed two new panels on the agent Grafana dashboard (a time-series rate and a 24h share donut, mirroring the existing tier panels).

One small behavioral fix lands in the base `SYSTEM_PROMPT` independent of intent: **"When the user logs a food without specifying servings, assume one serving."** This is defense in depth — even on a `general` classification, the agent stops asking the single-serving question that drove the original complaint.

This is a contained refactor scoped to three repos: `prog-strength-api` (one column + one endpoint), `prog-strength-agent` (the bulk of the work — router output shape, IntentRegistry, harness wiring, telemetry), and `prog-strength-infra` (two Grafana panels). It ships as one SOW because splitting "add the field" from "use the field" leaves intermediate states that don't earn anyone anything.

## Goals and Non-Goals

### Goals

- Expand `ModelRouter` to emit both a tier (`simple` | `complex`) and an intent (`log_nutrition` | `log_workout` | `log_bodyweight` | `analyze_progress` | `general`) in a single structured-output Haiku call. No additional router latency.
- Introduce a static `IntentRegistry` module declaring, per intent: a prefetch function (parallel MCP tool calls), a rules string (system-prompt addendum), and a data formatter.
- Wire `ModelHarness` to run intent prefetch immediately after opening the MCP session and to compose the per-turn system prompt with the intent's rules and data blocks.
- Persist the most recent intent on each chat session via two new columns on the API's `chat_sessions` table (`last_intent`, `last_intent_at`). Pass it to the router as a hint on subsequent turns.
- Add `intent` to `TurnInstrumentation`, the agent → API telemetry payload, and the SQLite telemetry schema.
- Add `agent_intent_classifications_total{intent}` and `agent_intent_prefetch_duration_seconds{intent}` Prometheus series.
- Add two new panels to the agent Grafana dashboard: intent rate over time (stacked) and 24h intent share (donut).
- Update the base `SYSTEM_PROMPT` with the "assume one serving" convention so the fix lands even on `general` classifications.

### Non-Goals

- **Open / extensible intent set in v1.** The taxonomy is a hardcoded enum the classifier returns. Adding a new intent is intentionally a code change — registry entry, prompt update, tests — because each new intent's prefetch and rules deserve thought, not a config edit. Revisit if we accumulate more than ~10 intents.
- **Sub-intent routing or hierarchical intents.** `log_nutrition` covers logging any food event; there's no `log_breakfast` vs `log_dinner` split. The MCP `log_consumption` tool already takes a `meal` argument that the agent can derive from context.
- **Agent-side write paths for intent management.** The agent reads `chat_sessions.last_intent` but does not expose it to the user. There's no "switch this conversation to nutrition logging mode" affordance — intent is internal.
- **Speculative prefetch on the first turn of a session.** Until the classifier returns, we don't know the intent, so we don't prefetch. No "fan out and pre-warm every intent's prefetch concurrently with classification" — premature.
- **Token-budget caps on prefetched data in v1.** For the current single-user scale (dozens of pantry items, a handful of recipes), full prefetch is fine. We'll add caps when usage shows a real user with hundreds of items where the system prompt starts to bloat.
- **Cross-session intent learning.** Each session classifies independently. No "this user usually logs nutrition" personalization. The chat-session-level hint is enough; user-level priors are over-engineered for the problem at hand.
- **Replacing the tier classifier with intent-derived tiers.** Tier and intent stay orthogonal outputs of the same call. An `analyze_progress` intent doesn't always need Sonnet; a `log_nutrition` intent occasionally does (a user pasting a long restaurant menu wanting macro breakdowns). Letting the classifier decide both independently is the safer default.
- **Removing the existing single-serving prompt clutter elsewhere.** We add the convention to `SYSTEM_PROMPT`; we don't audit every prior nutrition-related rule to deduplicate. The classifier rule block is additive and short enough not to matter.
- **Voice-mode-specific intent behavior.** Voice mode (streaming TTS) and intent classification are independent. The router runs identically; the harness composes the same prompt; the voice streamer wraps the output. No special-casing.
- **Multi-worker deployment of the agent service.** The agent today is a single uvicorn worker per container, and that's sufficient for forecasted scale. Persisting intent on `chat_sessions` happens to make the agent multi-worker-safe as a free side effect, but we don't actively scale out as part of this SOW.

## Implementation Details

### Data Model

One migration in `prog-strength-api/internal/db/migrations/010_chat_sessions_last_intent.sql`:

```sql
ALTER TABLE chat_sessions ADD COLUMN last_intent TEXT;
ALTER TABLE chat_sessions ADD COLUMN last_intent_at DATETIME;
```

Both nullable. A session created before this migration ships has `NULL` on both fields until its first post-migration turn classifies. No backfill required — `NULL` means "no hint, classify on conversation context alone," which is exactly the right behavior.

No index on `last_intent`. Reads are by `chat_sessions.id` (already the primary key); we don't query "give me all sessions where last_intent = log_nutrition" from the agent. SQL analysis ad-hoc queries against the column don't need an index at single-digit-thousands-of-sessions scale, and adding one we don't use is just churn.

The Grafana / Prometheus side gets two new series declared in `prog-strength-agent/src/prog_strength_agent/telemetry.py`:

```python
AGENT_INTENT_CLASSIFICATIONS_TOTAL = Counter(
    "agent_intent_classifications_total",
    "Intent classifications produced by the model router.",
    ["intent"],
)
AGENT_INTENT_PREFETCH_DURATION_SECONDS = Histogram(
    "agent_intent_prefetch_duration_seconds",
    "Time spent running an intent's prefetch tool calls (parallel asyncio.gather).",
    ["intent"],
    buckets=(0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.0, 5.0),
)
```

Cardinality: bounded by the closed intent enum (5 values today). Adding a new intent adds 1 series per metric. Safe.

The agent telemetry SQLite schema (already POSTed from the agent to the API) gains an `intent TEXT` column on the per-turn row, alongside the existing `routed_tier`. Migration on the API side bundled with the `chat_sessions` migration so the SOW ships as one DB change.

### Router

`ModelRouter.route` swaps from "answer with one word" to a structured-output tool call. The classifier system prompt and structure:

```python
ROUTER_TOOL = {
    "name": "classify_request",
    "description": "Classify the user's request by intent and required model tier.",
    "input_schema": {
        "type": "object",
        "properties": {
            "tier": {"type": "string", "enum": ["simple", "complex"]},
            "intent": {
                "type": "string",
                "enum": [
                    "log_nutrition",
                    "log_workout",
                    "log_bodyweight",
                    "analyze_progress",
                    "general",
                ],
            },
        },
        "required": ["tier", "intent"],
    },
}
```

The router prompt is rewritten to describe both axes and explicitly handles the multi-turn hint:

```
You are a routing classifier for the Prog Strength training assistant.
Given the user's most recent message (and the conversation context),
output BOTH:

- tier: "simple" (CRUD / lookup) or "complex" (multi-step reasoning).
- intent: one of log_nutrition, log_workout, log_bodyweight,
  analyze_progress, general.

Intents:
- log_nutrition — the user wants to record food they ate or drank.
- log_workout — the user is reporting a completed workout.
- log_bodyweight — the user is logging a bodyweight reading.
- analyze_progress — the user wants insight, trends, or planning advice.
- general — anything else (greetings, lookups, off-topic).

When uncertain on intent, pick "general". When uncertain on tier, pick
"simple". Both errs on the side of cheap and safe.

{optional} Previous intent for this session was {prior}. Use it as a
hint when the latest message is a short follow-up that only makes
sense in context — but switch when the user clearly pivots topic.
```

`ModelRouter.route` now accepts an optional `prior_intent: str | None` argument and returns a small dataclass:

```python
@dataclass(frozen=True)
class RouterDecision:
    tier: str
    intent: str
```

The fallback on any classifier exception or malformed output is `RouterDecision(tier="simple", intent="general")` — the same "default to cheap" semantics today's router uses, extended to intent.

The classifier sees the **last two user messages and the assistant turn between them** (when present) rather than only the latest user message. This is the bare minimum to make the "let me log dinner" → "a protein shake" case classify correctly when there's no prior_intent hint (e.g. brand-new sessions). For sessions with a prior_intent the hint usually carries the load; this fallback exists for cold starts.

### Intent Registry

A new module `prog_strength_agent/intents.py`:

```python
from dataclasses import dataclass
from typing import Callable, Awaitable, Any

PrefetchFn = Callable[[ClientSession], Awaitable[dict[str, Any]]]
FormatFn = Callable[[dict[str, Any]], str]


@dataclass(frozen=True)
class IntentSpec:
    intent: str
    rules: str           # appended to system prompt verbatim
    prefetch: PrefetchFn # async; runs MCP tool calls, returns dict
    format: FormatFn     # turns prefetch dict into a system-prompt block
```

Each known intent registers one `IntentSpec`. The registry exposes:

```python
def get(intent: str) -> IntentSpec | None: ...
async def run(intent: str, session: ClientSession) -> tuple[str, str]:
    """Returns (rules_block, data_block). Empty strings if intent
    is general / unknown or prefetch failed. Never raises."""
```

The per-intent prefetch functions use `asyncio.gather` to fan out tool calls so a multi-tool intent (e.g. `log_nutrition` calling `list_pantry_items` + `list_recipes`) pays max-of-N latency, not sum-of-N.

#### v1 intent specs

**`log_nutrition`**

- Prefetch: `list_pantry_items()` + `list_recipes()` concurrently.
- Rules: "The user is logging a meal or snack. Assume one serving unless they specify otherwise. The user's saved pantry items and recipes are listed below — search them by name first before asking the user for macros or creating a new pantry item. Only ask follow-up questions about details you genuinely cannot infer."
- Data block: `"USER'S CURRENT PANTRY (id · name · per-serving macros):\n…\nUSER'S CURRENT RECIPES (id · name · components):\n…"`.

**`log_workout`**

- Prefetch: `list_exercises()` (cached-friendly; ~50 items) + `list_workouts()` (returns ~50 by API default; the formatter slices to the most recent 5 for the prompt — for inferring plausible defaults when the user is terse).
- Rules: "The user is logging a completed workout. The exercise catalog is below — match the user's wording to an exercise slug without asking unless genuinely ambiguous. The user's last few workouts are also below; if they say 'same as last time' or use a shorthand, infer from those before asking."
- Data block: catalog + recent workouts as compact JSON-ish text.

**`log_bodyweight`**

- Prefetch: `list_bodyweight(since=<14 days ago>)`.
- Rules: "The user is logging a bodyweight reading. Default the unit to whatever they used most recently. If the new reading is meaningfully different from the recent trend, acknowledge it briefly; otherwise just confirm the log."
- Data block: last few bodyweight entries inline.

**`analyze_progress`**

- Prefetch: `list_workouts()` (returns ~50; formatter slices to the most recent 20) + `get_daily_macros(since=<30 days ago>, until=<today>)`.
- Rules: "The user wants analysis or planning advice. You already have a recent window of workouts and macros below. Favor citing specifics from this data over calling more tools; only fetch more if the user asks about a time range outside the included window."
- Data block: workouts + macros as compact text.

**`general`**

- No prefetch, no rules. The harness composes the system prompt without any addendum — behavior identical to today's `/chat` for unrecognized intents.

#### Failure semantics inside prefetch

`IntentRegistry.run` wraps the prefetch call in a broad `except` that:

1. Logs the failure with intent + tool name.
2. Returns the intent's rules block unchanged, but with an empty data block.
3. Records `intent_prefetch_failed=True` on `TurnInstrumentation` for observability.

Crucially, the rules string is written so the model handles either case ("listed below" reads naturally as "nothing listed" when the block is empty), and the model still has access to every MCP tool — it can call `list_pantry_items` itself if it needs to. Failure of enrichment is a graceful degradation to today's agent behavior, not a turn-killing error.

### ModelHarness changes

`ModelHarness.stream_chat` accepts a new `intent: str` parameter (passed by the server). Inside the existing flow, after `await session.initialize()` and `await session.list_tools()`, before the first model call:

```python
rules_block, data_block = await intent_registry.run(intent, session)
composed_prompt = _compose_system_prompt(
    base=system_prompt or SYSTEM_PROMPT,
    rules=rules_block,
    data=data_block,
)
```

`_compose_system_prompt` concatenates with double-newline separators and skips empty sections. The composed prompt is passed in place of `system_prompt` to the existing `self.client.messages.stream(...)` call. No other change to the tool-use loop.

The harness records on `TurnInstrumentation`:

- `intent` (string) — mirrors `routed_tier`.
- `intent_prefetch_duration_ms` (int) — measured around the `intent_registry.run` call.
- `intent_prefetch_failed` (bool) — true if any prefetch tool call failed.

The Grafana panels read these directly via the existing telemetry → SQLite → ad-hoc query path, and the Prometheus counters fire from `record_prometheus_metrics`.

### Server changes

`server.py`'s `_route_and_stream` gains one new step before the router call: fetch the prior intent for the session.

```python
prior_intent = await api_client.get_session_intent(req.session_id, auth.token)
decision = await router_obj.route(messages, telemetry, prior_intent=prior_intent)
harness = HARNESSES.get(decision.tier, HARNESSES["simple"])
async for chunk in harness.stream_chat(
    messages, auth.token, telemetry, system_prompt=system_prompt,
    intent=decision.intent,
):
    yield chunk
```

`api_client.get_session_intent` is a thin wrapper over `httpx.AsyncClient` against the new `GET /internal/chat-sessions/{id}/intent` endpoint. It's best-effort: on any failure (404, network error, 5xx) it returns `None` and the router classifies without a hint. Timeout: 200ms — fast enough that a single sluggish call doesn't add user-visible latency on top of the ~500ms classifier.

The write side rides on the existing telemetry POST in `TelemetryClient.record_turn`. The payload already includes per-turn fields; we add `intent` to the payload and the API's telemetry receiver UPSERTs `chat_sessions.last_intent` + `last_intent_at = now` whenever a non-general intent is recorded for a known session_id.

Fire-and-forget remains intentional: a missed write means the next turn's hint is stale, which falls back to conversation-context classification — exactly the graceful-degradation property we already rely on for telemetry today.

### API changes

Two new endpoints in `prog-strength-api/internal/chat/`:

- `GET /internal/chat-sessions/{id}/intent` (Bearer-auth via the JWT signing key; same trust boundary as the existing internal telemetry endpoints). Returns `{"intent": "log_nutrition", "intent_at": "2026-06-01T17:23:00Z"}` or `{"intent": null, "intent_at": null}` for a session with no prior intent (or no session — same shape on 404 to keep the client trivial).
- Existing internal telemetry POST handler extended: if the inbound payload includes `intent` and `session_id`, UPSERT `chat_sessions.last_intent` + `last_intent_at` in the same transaction as the per-turn telemetry insert. Only updates when intent ≠ `general` — sticking with the last *meaningful* classification is more useful as a hint than overwriting with `general` on a brief greeting mid-conversation.

Both endpoints are auth-gated under the same internal/JWT middleware the existing telemetry endpoints use. No public surface.

### Grafana dashboard

Two new panels in `prog-strength-infra/monitoring/grafana/dashboards/agent.json`, mirroring the existing tier panels (the structure to copy lives at lines ~214 and ~255 of that file):

1. **"Intent classifications over time"** — time-series, stacked.
   - PromQL: `sum by (intent) (rate(agent_intent_classifications_total[5m]))`
   - Legend: `{{intent}}`
   - Placed adjacent to the existing "Routing decisions over time (by tier)" panel for visual alignment.

2. **"Intent share (24h)"** — donut.
   - PromQL: `sum by (intent) (increase(agent_intent_classifications_total[24h]))`
   - Legend: `{{intent}}`
   - Placed adjacent to the existing "Routing share (24h)" donut.

A third panel for prefetch latency is optional v1, deferrable:

3. **"Intent prefetch latency p95"** — time-series.
   - PromQL: `histogram_quantile(0.95, sum by (intent, le) (rate(agent_intent_prefetch_duration_seconds_bucket[5m])))`
   - Useful for catching a slow prefetch tool, but not load-bearing for the launch.

### Base system prompt change

A single line added to `prog_strength_agent/prompt.SYSTEM_PROMPT` under the existing "Conventions you must follow" section:

> **Default to one serving for nutrition logs.** When the user logs a food without specifying servings ("log a protein shake," "had two eggs for breakfast" — wait, that one specifies), assume `quantity=1`. Only ask for servings if the user's wording is genuinely ambiguous. The serving-size unit on the pantry item itself tells you what "one serving" means.

Lands in the base prompt rather than `log_nutrition`'s rule block specifically because: (a) it's a useful default even when the classifier misses, and (b) the rule is short and uncontroversial — keeping it always-on costs nothing.

### Telemetry instrumentation

`TurnInstrumentation` in `prog_strength_agent/telemetry.py` gains:

```python
intent: str = ""
intent_prefetch_duration_ms: int = 0
intent_prefetch_failed: bool = False
```

The telemetry POST payload constructed in `TelemetryClient._payload` adds the three fields. The API's telemetry receiver schema gains three matching columns on its per-turn table — a small ALTER bundled into the same migration as `chat_sessions.last_intent`.

`record_prometheus_metrics` is extended to:

```python
if t.intent:
    AGENT_INTENT_CLASSIFICATIONS_TOTAL.labels(intent=t.intent).inc()
    if t.intent_prefetch_duration_ms > 0:
        AGENT_INTENT_PREFETCH_DURATION_SECONDS.labels(intent=t.intent).observe(
            t.intent_prefetch_duration_ms / 1000.0,
        )
```

Empty `intent` (router failed completely) is excluded so we don't pollute the series with empty-label samples.

### Testing

Three layers:

- **Unit tests in `prog-strength-agent/tests/test_intents.py`** — one test per intent spec. Mocks an `MCPSession` whose `call_tool` returns fixture data; asserts the format function produces the expected system-prompt block; asserts that a failing tool call returns an empty data block without raising.
- **`tests/test_model_router.py`** — tests for structured-output parsing, the `RouterDecision` fallback on a malformed response, prior_intent hint plumbing, and the multi-turn message-history slicing.
- **Integration test against a stub MCP server** (mirrors the pattern in the existing harness tests) — verifies that for a `log_nutrition` classified message, the harness's first model call sees a system prompt containing both the rules block and the pantry/recipes data block.

API-side: standard handler tests for `GET /internal/chat-sessions/{id}/intent` (auth, 200, 404), plus a test that the internal telemetry handler correctly UPSERTs `chat_sessions.last_intent` only for non-`general` intents.

No new e2e tests; the existing chat e2e suite catches regressions in the overall flow.

### Rollout

The change is single-deploy, single-direction. The migration is additive (two nullable columns), the new API endpoint is auth-gated and only the agent calls it, and the agent's classifier output shape is internal. The first deploy ships:

1. API migration + new internal endpoint.
2. Agent with the new router, IntentRegistry, harness wiring.
3. Infra PR with the two new Grafana panels (and Prometheus picks up the new series automatically — no scrape-config change).

If anything goes sideways post-deploy, the agent's router has the existing "default to simple" failure mode extended to "default to (simple, general)," which collapses the new code path to today's behavior. No feature flag needed; the failure mode IS the off switch.

### Scale considerations

The forward question: what changes if the agent scales beyond one uvicorn worker per container, or if the API moves off single-instance SQLite?

- **Multi-worker agent**: works without modification because `chat_sessions.last_intent` is the source of truth and the agent reads/writes it via the API. No sticky load balancing needed; any worker can serve any turn.
- **Postgres / managed DB on the API**: the migration is one-line ALTER TABLE on the new DB; nothing in the agent changes.
- **Cross-VPC API call latency**: the GET-intent round-trip is currently <1ms (Docker network); at scale on a real LB it's ~5–10ms. Still dwarfed by the ~500ms classifier and the multi-second main model call. A future optimization (not in this SOW) is to bundle the intent read into the auth/session bootstrap the agent already does, removing the extra round-trip entirely.

None of these are work items here; they're noted so a future scaling SOW can reference the assumptions.

## Open Questions

1. **Should `analyze_progress` intent force the complex tier regardless of the classifier's tier output?** Today the classifier is allowed to pick `(analyze_progress, simple)` for short questions; this lets Haiku answer "did I bench last week?" while a real progression question gets Sonnet. The risk is the classifier under-routing a nuanced analysis question to Haiku and producing a worse answer. Option (a): leave tier and intent fully orthogonal. Option (b): hard-code `analyze_progress → complex`. Recommendation: (a) — let the classifier do its job; revisit if telemetry shows under-routing on analysis.

2. **Token budget for `log_nutrition` prefetch.** Current single-user has ~30 pantry items and ~10 recipes; full prefetch is ~2–4 KB of text, trivial. A power user with 200 pantry items could push the block to ~20 KB — still well under the model's context limit but a noticeable fraction of the prompt cache. Option (a): cap pantry items in the block at top-50 by recent log frequency (which we already track via `nutrition_log_entries`). Option (b): ship uncapped and revisit if anyone reports slow turns. Recommendation: (b) — ship the simple thing; the cap is a one-day follow-up if real usage shows pressure.

3. **Whether `last_intent` should be reset on session deletion.** `chat_sessions` already cascades-deletes its `chat_messages`; the columns go with the row, so semantically there's nothing to do. Worth confirming during implementation that the cascade is intact and adding a test if it isn't.

4. **Should the intent classifier prompt include the list of intents' definitions explicitly, or rely on enum-named semantics?** Today's tier classifier prompt has examples; the intent classifier prompt above has short one-liners. Option (a): keep it terse — the names + one-liners are self-explanatory at Haiku scale. Option (b): expand each intent with 2–3 example user messages, ~50% larger prompt, possibly better classification on edge cases. Recommendation: ship (a) and watch the Grafana intent-share panel; if any one intent collapses to 0% in production we'll know the classifier doesn't understand it and (b) becomes the next iteration.

5. **Whether to include the prefetch failure as a visible signal to the user.** Today MCP tool failures in the main loop surface as `tool_result.is_error` to the client. Prefetch failures are silent. A muted "context partially unavailable" indicator could be added but probably noise; the model still has every tool. Recommendation: silent for v1; revisit if users report confusion.
