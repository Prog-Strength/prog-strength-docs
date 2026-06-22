---
status: draft
repos:
  - prog-strength-api
  - prog-strength-mcp
  - prog-strength-agent
  - prog-strength-docs
---

# Holistic Training Analysis: The `analyze_training` Intent and the Training Snapshot

**Status**: Draft · **Last updated**: 2026-06-21

## Introduction

When a user asks the **Prog Strength** agent "How did I do with training this
week?", they expect their AI coach to look at *everything* — lifts, runs, steps,
bodyweight, and nutrition — and give back a holistic read on the week. Today it
does not. The router classifies the message as `analyze_progress`, which does
**not** force the capable model and whose prefetch pulls only `list_workouts`
plus `get_daily_macros`. So a real session looked like this:

- The turn ran on **Haiku 4.5** (the cheap tier), not Sonnet.
- The only tool called was **`list_workouts`** — strength sessions only.
- Running activities, daily steps, and bodyweight were **never fetched** (and
  there is no MCP tool that lists running activities at all, so the agent
  *cannot* see runs as a list even if it tried).
- The response confidently narrated a "great week" from a fraction of the data,
  with no grounding in cardio volume, step consistency, bodyweight trend, or how
  the week's nutrition actually tracked against goals.

Analyzing training over a period — most often a week at a glance — is a **core
use case** of Prog Strength, the moment the product is supposed to feel like a
coach rather than a logger. It deserves the best reasoning the system can offer,
fed the full picture of what the user actually did.

This work does three things:

1. Adds a purpose-built **`GET /training-snapshot`** endpoint to the Go API that
   composes a single, compact, pre-aggregated view of a date window across
   strength, running, steps, bodyweight, and nutrition. This is the API-side
   abstraction of "a snapshot of the user's training" — the one place that
   assembles the holistic picture.
2. Exposes it to the agent through a new MCP forwarder tool,
   **`get_training_snapshot`**.
3. Renames the `analyze_progress` intent to **`analyze_training`**, routes it to
   the **complex** model always (as `plan_workout` already is), and rewires its
   prefetch to pull the snapshot for the current week — so "How did I do this
   week?" lands on Sonnet with the whole dataset in hand, and grounds its read in
   the durable memories the agent already retrieves each turn.

## Goals and Non-Goals

### Goals

- Add `GET /training-snapshot` to `prog-strength-api`: one endpoint that
  composes a **date-windowed, LLM-shaped, pre-aggregated** view across strength,
  running, steps, bodyweight, and nutrition, with **defensive degradation** (one
  domain's failure nulls *its* section, never the whole response) modeled on the
  existing `dashboard.summary` handler.
- Add a `get_training_snapshot` MCP tool that is a **pure forwarder** to the new
  endpoint, following the established domain-module pattern.
- Rename the `analyze_progress` intent to **`analyze_training`**, **force the
  complex tier** for it in the router, and rewire its prefetch to call
  `get_training_snapshot` for the user's current local week.
- Give the agent a list-level view of **running activities for a period**, which
  it has no tool for today — folded into the snapshot rather than added as a
  standalone tool.
- Keep the snapshot **compact by default** (per-session headlines and per-day
  totals, not raw sets or trackpoints) and preserve a **drill-down escape
  hatch**: existing detail endpoints/tools stay available for the model to call
  only when it needs more.
- Ground the analysis in **retrieved vector memories**, which the agent already
  injects per turn (the `agent-vector-memory` feature, PR #18), by directing the
  new intent's prompt to use that remembered context.

### Non-Goals

- **Do not touch `GET /dashboard/summary`.** It serves the web dashboard with a
  fixed rolling window and UI-shaped fields (sparklines, headline 1RM). The
  snapshot is a separate, agent-facing surface; the two intentionally do not
  share a response shape.
- **No planned-vs-actual adherence in v1.** Comparing the week's logged sessions
  against `planned_workouts` is a natural follow-up but is out of scope here; the
  snapshot answers "what happened," not "what was scheduled."
- **No new vector-memory infrastructure.** Retrieval and injection already exist
  (PR #18). This SOW only consumes that seam; it does not change how memories are
  stored, embedded, or retrieved (beyond an optional per-intent top-k tuning,
  noted as an open question).
- **No raw detail in the snapshot payload.** Individual sets and per-second
  trackpoints stay behind the existing detail endpoints; the snapshot never
  streams them to the model.
- **No change to the single-item read tools** (`list_workouts`, `get_steps`,
  `list_bodyweight`, `get_daily_macros`, `get_running_best_efforts`). They remain
  for their existing intents and as drill-down options.

## Proposed Solution

A single new API endpoint owns the aggregation. Everything upstream is thin: the
MCP tool forwards, and the agent intent prefetches the tool and routes to the
capable model. The compact-by-default payload plus opt-in drill-down keeps the
per-analysis token cost bounded and predictable even for a heavy training week,
while the complex tier fires only for this intent (and the existing
plan-workout, external-meal, and vision cases).

The request flow after this ships:

```
"How did I do with training this week?"
  → router classifies intent=analyze_training, tier=complex (always)
  → prefetch: get_training_snapshot(timezone, current local week)
       → GET /training-snapshot → snapshot.Service fans out to
         workout / activity / steps / bodyweight / nutrition repos
  → memories retrieved concurrently (existing #18 seam), injected as Background
  → Sonnet reasons over the full snapshot + remembered goals/constraints
  → holistic, grounded read across all modalities
```

## Implementation Details

### `prog-strength-api`: `GET /training-snapshot`

A new `internal/snapshot` package, sibling to `internal/dashboard`, holding a
`Handler` and a `Service`. It introduces **no new tables, migrations, or
repositories** — it composes the repos that already exist.

**Route.** Register inside the JWT-gated group in `internal/server/server.go`
(the same group `dashboard` mounts under):

```go
r.Route("/training-snapshot", func(r chi.Router) {
    r.Get("/", h.snapshot)
})
```

**Query contract.** `timezone` (required, IANA) plus either `date` or
`start_date`+`end_date` (YYYY-MM-DD), resolved through the existing
`internal/daterange.ParseQuery`, which returns the UTC half-open window plus the
`*time.Location` and is already DST-correct. **Default window:** when neither
`date` nor a range is supplied, the handler defaults to the **trailing 7 local
days** ending today, so the agent can request "this week" without computing
bounds itself.

**Service composition.** `snapshot.Service.Build(ctx, userID, start, end, loc)`
fans out to the existing repository methods, each wrapped defensively (a failure
logs and nulls that section, mirroring `dashboard`'s `defer1` pattern):

- Strength — `workout.Repository.ListByUser(ctx, userID, ListOptions{Since,
  Until})`, plus `ListPersonalRecordEventsByWorkouts` for PR headlines.
- Running — `activity.Repository.ListInRange(ctx, userID, &since, &until)` and
  the running best-effort reads. This is the data the agent has no list tool for
  today.
- Steps — `steps.Repository.List(ctx, userID, since, until, 0, nil)` (range
  mode, `limit == 0`).
- Bodyweight — `bodyweight.Repository.List(ctx, userID, since, until)`.
- Nutrition — `nutrition.Repository.DailyMacros(ctx, userID, start, end, loc)`
  plus `GetMacroGoals`.

**Response shape** (`httpresp.OK(w, "training snapshot", snap)`), compact and
pre-aggregated — per-session headlines and per-day totals, never raw sets or
trackpoints:

```jsonc
{
  "period": { "start_date": "2026-06-15", "end_date": "2026-06-21",
              "timezone": "America/Denver", "days": 7 },
  "strength": {                       // null if the workout repo errors
    "session_count": 3,
    "total_volume": 48250,
    "unit": "lb",
    "by_muscle_group": [ { "muscle_group": "chest", "sets": 12, "volume": 9800 } ],
    "sessions": [
      { "date": "2026-06-16", "name": "Chest & Back",
        "top_sets": [ { "exercise": "barbell-bench-press", "weight": 265,
                        "reps": 8, "est_1rm": 331 } ],
        "prs": [ { "exercise": "barbell-bench-press", "kind": "est_1rm" } ] }
    ],
    "headline_prs": [ "335 lb high-bar back squat (est. 1RM PR)" ]
  },
  "running": {                        // null if the activity repo errors
    "run_count": 2, "total_distance_m": 14200, "total_duration_s": 4380,
    "avg_pace_sec_per_km": 308,
    "runs": [ { "date": "2026-06-17", "name": "Easy 5",
                "distance_m": 8000, "duration_s": 2520,
                "avg_pace_sec_per_km": 315 } ],
    "new_best_efforts": [ { "distance_key": "5k", "time_seconds": 1320 } ]
  },
  "steps": {                          // null if the steps repo errors
    "days_logged": 6, "avg": 9120, "total": 54720, "goal": 10000,
    "by_day": [ { "date": "2026-06-15", "steps": 8800 } ]
  },
  "bodyweight": {                     // null if the bodyweight repo errors
    "unit": "lb", "start": 184.2, "end": 183.4, "delta": -0.8,
    "readings": [ { "date": "2026-06-15", "weight": 184.2 } ]
  },
  "nutrition": {                      // null if the nutrition repo errors
    "days_logged": 5,
    "avg": { "calories": 2680, "protein_g": 198, "fat_g": 82, "carbs_g": 290 },
    "goals": { "calories": 2700, "protein_g": 200, "fat_g": 80, "carbs_g": 295 },
    "by_day": [ { "date": "2026-06-15", "calories": 2710, "protein_g": 205,
                  "fat_g": 79, "carbs_g": 288 } ]
  },
  "consistency": { "active_days": 5, "window_days": 7 }
}
```

A section is `null` only when its underlying repo errors; an empty-but-healthy
domain (e.g. no runs logged) renders zeros/empty arrays so the model can state
"no runs this week" rather than guess. Aggregation (volume sums, per-muscle-group
rollups, est-1RM, pace, averages, deltas, active-day count) lives in the
`snapshot` service, not in handlers or the model.

### `prog-strength-mcp`: `get_training_snapshot`

A new domain module `src/prog_strength_mcp/training_snapshot.py` exposing one
**pure-forwarder** tool, following the `steps.py` / `nutrition.py` pattern
exactly:

```python
@mcp.tool
async def get_training_snapshot(
    timezone: str,
    date: str | None = None,
    start_date: str | None = None,
    end_date: str | None = None,
) -> dict[str, Any]:
    """Holistic training snapshot for a date window — strength, running,
    steps, bodyweight, and nutrition, pre-aggregated for analysis. Supply an
    IANA `timezone` plus either a single `date` or a `start_date`/`end_date`
    range (YYYY-MM-DD); omit dates for the trailing 7 days. Use this for any
    "how did my training go" question over a period before reaching for the
    per-domain list tools."""
```

It pulls the inbound `Authorization` header via the shared
`_auth_header_or_raise()` guard, calls a new `APIClient.get_training_snapshot`
in `api_client.py` (GET `/training-snapshot` with the query params), unwraps the
`{service, message, data}` envelope, and converts `APIError` to `RuntimeError`.
Registered in `server.py` alongside the other domains (`training_snapshot.register(mcp, api)`).

### `prog-strength-agent`: rename to `analyze_training`, force complex, snapshot prefetch

This builds on the current `origin/main`, which already retrieves vector
memories per turn and injects them through `compose_system_prompt`
(`memory.format_memory_block`, PR #18). The new intent inherits that seam
unchanged.

- **`intents.py`** — rename `analyze_progress` to `analyze_training` in
  `KNOWN_INTENTS`. Replace the old prefetch (which called `list_workouts` +
  `get_daily_macros`) with a single `get_training_snapshot` call for the user's
  **current local week**, computed from the injected local date + timezone (the
  same timezone plumbing the old prefetch already used to call
  `get_daily_macros`). A new `_analyze_training_format` renders the snapshot into
  a compact data block (per-section summaries + headlines). The prefetch stays
  fail-soft: on error the agent falls back to its base behavior with the tool
  still available.
- **`_ANALYZE_TRAINING_RULES`** — direct the model to: give a holistic read
  across **all** modalities present in the snapshot (not just lifting); cite
  specifics from the snapshot rather than generalize; **use remembered goals,
  constraints, and injuries** from the Background memory block when relevant;
  call `get_training_snapshot` again with explicit `start_date`/`end_date` if the
  user asked about a period other than the prefetched week ("last month"); reach
  for the per-domain detail tools only when it needs more than the snapshot
  carries; and never fabricate data for a domain the snapshot reports as empty.
- **`model_router.py`** — rename the intent in `KNOWN_INTENTS` / the router
  prompt's intent list, and add `analyze_training` to the **always-complex** set
  next to `plan_workout`, with a one-line rule: any request to assess, review, or
  reflect on training over a period routes to `analyze_training` + complex.
- The harness, server, and config need **no changes** — intent discovery,
  prompt composition, and memory injection are already generic over the intent
  name.

### `prog-strength-docs`: this SOW

This file; status transitions draft → ready_for_implementation → shipped as the
work lands.

### Tests

| Repo | Coverage |
| --- | --- |
| `prog-strength-api` | **Service:** multi-repo fan-out aggregation (volume/macro/pace rollups, active-day count, bodyweight delta); **defensive degradation** (one repo erroring nulls only its section); **empty-but-healthy** domains render zeros/empty arrays, not null. **Handler:** query parsing (`date` vs range vs default trailing-7), DST window correctness via `daterange`, auth required. |
| `prog-strength-mcp` | Forwarder maps params to the GET query and returns unwrapped `data`; missing `Authorization` raises the guard error; `APIError` surfaces as `RuntimeError`. respx-mocked, following `test_steps_tools.py`. |
| `prog-strength-agent` | Router classifies period-analysis phrasings → `analyze_training` + `complex`; intent prefetch calls `get_training_snapshot` for the current week and the format renders compactly; prefetch failure falls back soft; the Background memory block still composes alongside the new data block. Update existing `analyze_progress` tests to the renamed intent. |
| `prog-strength-docs` | This SOW; status transitions. |

### Rollout

Ship in dependency order, one PR per repo:

1. `prog-strength-api` — `GET /training-snapshot`. Independently deployable; no
   consumer yet.
2. `prog-strength-mcp` — `get_training_snapshot` forwarder, once the endpoint is
   live.
3. `prog-strength-agent` — rename + reroute + snapshot prefetch, once the tool is
   available in the MCP session.

Each step is backward-compatible: until the agent PR merges, the old
`analyze_progress` behavior is untouched. There is no data migration and no
infra change, so rollback is reverting the relevant PR.

## Open Questions

1. **Retrieval depth for analysis turns.** Memory retrieval currently returns a
   default top-k tuned for logging/CRUD turns. A holistic review may benefit from
   a larger top-k (more remembered goals/constraints in context). Tune per-intent
   now, or leave the global default and revisit after seeing real analysis turns?
   Leaning toward leaving the default for v1.
2. **Default window — trailing 7 days vs. current calendar week.** The prefetch
   defaults to trailing 7 days. "This week" sometimes means Monday-to-today.
   Trailing-7 is simpler and the model can always re-call with explicit dates; is
   that the right default, or should the snapshot support a `week` convenience
   mode?
3. **Best-effort PR detection in the snapshot.** Strength PR headlines reuse the
   existing per-workout PR events. Running "new best efforts" needs the snapshot
   to flag efforts set within the window — confirm the activity repo exposes
   enough to determine "new this window" without a heavier query.
