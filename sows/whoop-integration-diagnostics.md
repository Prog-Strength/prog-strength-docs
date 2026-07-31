---
status: draft
repos:
  - prog-strength-api
  - prog-strength-tooling
  - prog-strength-infra
  - prog-strength-docs
---

# WHOOP Integration Diagnostics, Resync & Liveness Alerting

**Status**: Draft · **Last updated**: 2026-07-31

## Introduction

On 2026-07-31 a user reported that their WHOOP recovery scores had not updated in the
Prog Strength web client for several days. The `ps-whoop` Grafana dashboard showed no
problems — every failure panel read a healthy green `0`.

Both observations were correct. The integration had never worked.

WHOOP had been delivering webhooks the entire time, to a URL carrying a stray trailing
comma. Over 30 days, **97 of 97 deliveries hit `POST /webhooks/whoop,` and were answered
`404`**; the correctly-spelled path was hit exactly zero times by WHOOP. Every recovery
row in the database arrived via `Backfill`, which runs on OAuth connect — so data only
ever advanced on the three occasions the user manually reconnected:

```
2026-07-29 00:47  kind=backfill  recoveries_fetched=19  upserted=19
2026-07-28 04:50  kind=backfill  recoveries_fetched=18  upserted=18
2026-07-23 04:26  kind=backfill  recoveries_fetched=14  upserted=14
```

There is not one `kind=window` sync in the entire log history, and `kind=window` is the
only kind a webhook can produce.

The dashboard could not have shown this. Every `api_whoop_*` counter increments *inside*
the webhook handler; a request that 404s at the chi router never reaches instrumented
code. The panels were not stale or misconfigured — they were structurally incapable of
observing the failure, and `or vector(0)` rendered the resulting silence as a reassuring
zero. **Zero successes looked identical to zero problems.**

The immediate fix is a one-character edit in the WHOOP Developer Dashboard, and is owned
by the operator. This SOW addresses the durable gap: the integration had no way to answer
"is ingestion actually alive?", no way to recover missed days without a manual OAuth
reconnect, and no alert that would fire when the answer was no.

After this ships, a silent WHOOP failure becomes a Slack message within the hour, a single
CLI command explains which link in the chain broke, and a second command backfills the
days that were lost.

## Proposed Solution

Three coordinated parts, one per repo.

**`prog-strength-api` — an admin surface over connection state, plus the missing
instrumentation.** Two read endpoints expose the facts that live only in SQLite
(connection status, token expiry, newest stored recovery date) and one write endpoint
triggers an idempotent resync. A new `api_webhook_misroute_total` counter makes
delivery-to-a-path-we-do-not-serve observable for the first time, and an
`api_whoop_connections` gauge makes the alert conditional on there being anything to sync.

**`prog-strength-tooling` — `pst whoop doctor`.** A chain diagnostic that joins the new
admin endpoints with CloudWatch evidence and reports which link is broken. It is
fleet-first: bare invocation reports global ingestion health, `--user` drills into one
account. `pst whoop resync` is the remediation half.

**`prog-strength-infra` — two provisioned alert rules.** One fast and specific (misrouted
deliveries), one slow and cause-agnostic (dead ingestion). Plus a dashboard threshold so
absence of success renders red instead of green.

The organising principle throughout: **instrument and alert on the absence of success,
not only the presence of failure.** Every signal this integration had was a failure
counter, and failure counters cannot fire when the failure happens upstream of them.

## Goals and Non-Goals

### Goals

- Expose per-user WHOOP connection state (status, token expiry, scopes, newest recovery
  date) to an admin-gated API surface.
- Let an operator trigger an idempotent resync for a user over an explicit window,
  without an OAuth reconnect.
- Ship `pst whoop doctor`, which identifies which link in the ingestion chain is broken
  and prints the corrective action — including breakage that reaches no instrumented code.
- Ship `pst whoop resync` to recover days lost to an ingestion gap.
- Alert on misrouted webhook deliveries within the hour, naming the offending path.
- Alert on dead ingestion regardless of cause, including when the relevant metric series
  has never existed.
- Make zero-successes visually distinct from zero-problems on the `ps-whoop` dashboard.

### Non-Goals

- **Scheduled server-side reconciliation.** A periodic sync would make webhook gaps
  self-healing and is the strongest long-term answer, but it is a product feature with
  scheduling and API-quota implications rather than diagnostic tooling. Rule A below makes
  a gap loud, which is the cheaper half of the same benefit. Filed as a follow-up.
- **Automating the WHOOP Developer Dashboard.** Webhook registration and app config stay
  manual and operator-owned. This work makes a misconfiguration *loud*, not impossible.
- **A general-purpose webhook-misroute framework.** The new counter recognises a small
  hard-coded registry of known provider fragments. Generalise when a second provider
  arrives, not before.
- **User-facing surfacing of a broken connection.** Prompting the user in Settings to
  reconnect is worthwhile and separate.
- **Changing the webhook-as-poke architecture.** The design is sound; it lacked
  observability, not a redesign.

## Implementation Details

### Data Model

**No schema changes.** No new tables, columns, or migrations. The diagnostic reads
existing rows through two new repository methods.

#### `whoopconn.Repository`

| Method | Signature | Notes |
| --- | --- | --- |
| `List` | `List(ctx) ([]Connection, error)` | All connections, any status, ordered by `updated_at DESC`. Returns metadata only — never token material, consistent with the existing `Get`. |

#### `whooprecovery.Repository`

| Method | Signature | Notes |
| --- | --- | --- |
| `Latest` | `Latest(ctx, userID string) (*Entry, error)` | Newest row by `date DESC LIMIT 1`. Returns `(nil, nil)` when the user has no rows — absence is an expected state for a freshly connected account, not an error. |
| `CountForUser` | `CountForUser(ctx, userID string) (int, error)` | Row count for the single-user detail view. |

Both interfaces have in-memory and SQLite implementations plus a shared conformance suite
(`repository_shared_test.go`); all three gain cases for the new methods.

`List` is deliberately unpaginated. The connection table holds one row per opted-in user
and is read only by an admin surface and a 5-minute exporter. Revisit above ~10,000
connections, at which point the gauge exporter should move to a `GROUP BY status`
aggregate query and the admin list should page.

### API Surface

New package `internal/whoopadmin`. It is kept out of `whoopsync` — that package's
`handler.go` is already ~14 KB across three distinct responsibilities (OAuth, connection
CRUD, recovery reads), and an admin surface with a different auth model does not belong in
it.

All three routes mount inside the **existing** `RequireAdmin` group in `server.go`
(currently wrapping `beta` and `vectormemory`), guarded by `whoopHandler != nil` so the
surface stays absent when the integration is disabled — the same pattern `vmHandler`
already uses. The handler does not import `auth`; the gate is applied by the enclosing
group, matching `beta.Handler`'s documented contract.

#### `GET /admin/whoop/connections`

Lists every connection. Response:

```json
{
  "message": "listed whoop connections",
  "data": {
    "connections": [
      {
        "user_id": "37cd1e90...",
        "whoop_user_id": 12345678,
        "status": "connected",
        "scopes": "read:recovery read:cycles offline",
        "token_expires_at": "2026-07-31T18:04:11Z",
        "token_expired": false,
        "connected_at": "2026-07-23T04:26:00Z",
        "updated_at": "2026-07-29T00:47:45Z",
        "latest_recovery_date": "2026-07-28",
        "recovery_row_count": 19
      }
    ]
  }
}
```

`token_expired` is computed server-side against the request clock so the client never has
to reimplement the comparison. `latest_recovery_date` is `null` when the user has no rows.

#### `GET /admin/whoop/connections/{userID}`

The same object for one user, `404` when no connection row exists.

#### `POST /admin/whoop/resync`

```json
{ "user_id": "37cd1e90...", "window_days": 30 }
```

`window_days` defaults to 30 and is clamped to `[1, 90]` — WHOOP's list endpoints are
paginated and an unbounded window is an easy way to burn API quota by typo. Returns the
sync outcome:

```json
{
  "message": "resynced whoop recovery",
  "data": { "upserted": 12, "skipped_unscored": 0, "skipped_no_cycle": 1, "skipped_bad_date": 0 }
}
```

Failure modes map to: `404` no connection, `409` connection not in `connected` status
(body names the status and that a reconnect is required), `502` WHOOP API failure, `500`
otherwise. `ErrReconnectNeeded` from the service maps to `409`, not `500` — it is a state
the operator must act on, not a transient fault to retry.

### Write Path

- **Resync requested** — `whoopadmin` calls a new `whoopsync.Service.SyncSince`. The
  service resolves a valid token (refreshing if needed, under the existing per-user
  mutex), fetches recoveries + cycles, and upserts. Idempotent: re-running overwrites the
  same `(user_id, date)` rows, latest wins.
- **Token rotated during resync** — unchanged existing behaviour; the rotated pair is
  persisted before use.

`SyncSince(ctx, userID string, window time.Duration, limit int) error` is a thin exported
wrapper over the existing unexported `syncWindow`. No new sync logic, no duplicated
date-derivation. It labels its metrics `kind="admin_resync"`.

**That label is load-bearing.** An operator-triggered resync must not increment
`syncs_total{kind="window"}`, or the liveness alert defined below — whose entire job is to
notice that organic webhook-driven syncs have stopped — would be silenced by the very act
of investigating the outage it reported.

### Instrumentation

#### `api_webhook_misroute_total{provider}`

A counter incremented from a chi `NotFound` handler when a 404 request path matches a
known webhook-provider fragment.

```go
// providerFragments maps a lowercase path fragment to the provider it
// identifies. Deliberately tiny and hard-coded: the counter's value is that
// it is silent except when a real provider is misdelivering, and matching
// anything broader (e.g. every /webhooks/ path) would readmit the scanner
// noise this metric exists to escape.
var providerFragments = map[string]string{"whoop": "whoop"}
```

Cardinality is bounded by the map's size. Paths that match nothing are not counted at
all — the counter is not a general 404 counter.

**Why a new metric rather than reusing `ps_http_requests_total`.** That counter *does*
already record these 404s, under `route="<unmatched>"` (`internal/server/metrics.go`
substitutes the literal when chi resolves no pattern). But the bucket is unusable for
alerting: over 30 days it collected roughly 1,100 hits, almost all internet background
noise — `.env` variants, `.git/config`, `_ignition/execute-solution`, `?s=captcha`,
`wp-config.php.bak`. Even restricted to `POST`, scanner traffic (~220 hits) outnumbered
the 97 genuine WHOOP deliveries. Any threshold low enough to catch a misrouted webhook
fires permanently on scanners. The measured noise floor is why this needs its own series.

#### `api_whoop_connections{status}`

A gauge, refreshed every 5 minutes by an exporter goroutine using the new
`whoopconn.List`. Values: `connected`, `revoked`, `error`.

It serves two purposes: it gates the liveness alert so an empty database cannot page, and
it gives `ps-whoop` a connection-health panel it currently lacks — today nothing on the
dashboard reveals that a user's connection has flipped to `error`.

The goroutine starts only when the WHOOP integration is enabled and stops on server
shutdown context cancellation.

### Diagnostic CLI (`prog-strength-tooling`)

```
pst whoop doctor [--user ID] [--since 7d] [--json]
pst whoop resync --user ID [--days 30]
```

Follows the repo's established layering — `commands/` (Typer) → domain module →
`render.py` — mirroring how `logs` is built.

| New module | Responsibility |
| --- | --- |
| `commands/whoop.py` | Typer sub-app, flag parsing, exit codes |
| `whoop.py` | Diagnosis engine: runs the checks, produces a `Diagnosis` |
| `whooplogs.py` | CloudWatch evidence: delivery scan + sync scan |

`whooplogs.py` uses `FilterLogEvents` with client-side aggregation rather than Logs
Insights, consistent with `cloudwatch.py`'s documented rationale (billed per call rather
than per GB scanned, synchronous, no start-query/poll/get-results loop). At the observed
volume — roughly a dozen deliveries a day — the aggregation is trivial client-side.

**Targeted refactor:** `cloudwatch.py`'s `_build_client` and `_describe_failure` become
shared (module-level, un-underscored) so `whooplogs.py` reuses the credential/permission/
region error guidance instead of duplicating it. This is the only change to existing code.

#### The checks

| # | Check | Source | Fires when |
| --- | --- | --- | --- |
| 1 | Deliveries arriving at all | logs | No `POST` whose URI contains `whoop` in the window |
| 2 | **Deliveries reaching the handler** | logs | Any delivery on a path other than `/webhooks/whoop`, or any non-2xx |
| 3 | Signatures accepted | logs | `401` responses present |
| 4 | Deliveries producing syncs | logs | Deliveries > 0 but no `kind=window` sync-complete lines |
| 5 | Syncs landing rows | logs | `upserted == 0` while `skipped_*` > 0 |
| 6 | Connection health | admin API | Status ≠ `connected`, or token expired |
| 7 | Data freshness | admin API | Newest recovery date older than 48h |

Check 2 is the reason this command exists. It groups every `POST` whose URI contains
`whoop` by `(uri, status)` and flags anything that is not `/webhooks/whoop` → 2xx. This is
the segment no `api_whoop_*` metric can observe, because those counters live downstream of
the router.

Each finding prints the symptom, the evidence, and the fix:

```
✗ Delivery path    97 deliveries → /webhooks/whoop, (404)
                   WHOOP is posting to a path chi does not serve.
                   Fix: WHOOP Developer Dashboard → webhook URL.
                   Expected: https://api.progstrength.fitness/webhooks/whoop
```

Exit codes follow `commands/logs.py`: `0` healthy, `1` findings present, `2`
configuration/AWS/API error. The distinction matters for scripting — "broken tool" must
be distinguishable from "working tool, unhealthy integration".

`--json` emits the full `Diagnosis` for piping.

Auth: the admin endpoints need `PST_TOKEN` (same admin JWT as `pst memory`); the
CloudWatch half uses the operator's AWS credentials. `doctor` degrades gracefully — if
`PST_TOKEN` is absent it runs the five log-derived checks, prints the two API-derived ones
as skipped, and says why. A missing token must not block the diagnosis that would have
caught this outage.

### Alerting (`prog-strength-infra`)

New `monitoring/grafana/provisioning/alerting/rules-whoop.yml`, structured like
`rules-vector-memory.yml`, with `__dashboardUid__: ps-whoop` annotations.

#### Rule A — dead ingestion (cause-agnostic backstop)

```promql
(
  sum(increase(api_whoop_syncs_total{kind="window",result="ok"}[36h])) or vector(0)
)
and on()
(sum(api_whoop_connections{status="connected"}) > 0)
```

Threshold `< 0.5`, `for: 0s`, severity `critical`, `noDataState: Alerting`.

**`or vector(0)` is the mechanism that makes this rule work at all.** The obvious form of
this alert — the one the original whoop-integration SOW proposed —

```promql
increase(api_whoop_syncs_total{kind="window",result="ok"}[36h]) == 0
```

would have silently never fired against this outage. `{kind="window"}` has never existed
as a series, so the expression returns *no data* rather than zero, and this repo's
standing convention (`noDataState: OK`, documented in the alerting README because absence
of an error series must not page) would have converted that into silence. The naive alert
would have reproduced the dashboard's blind spot exactly. `or vector(0)` materialises a
zero when the series is absent, turning "never happened" into a firing condition;
`noDataState: Alerting` is a deliberate, commented departure from the file-level
convention, justified because this is a **liveness** monitor, not an error counter.

The `and on()` clause gates on there being at least one connected account, so a fresh
environment or a fully disconnected user base cannot page.

This rule is cause-agnostic by design: it fires whether the webhook URL is wrong, the
subscription is disabled, a refresh token died, WHOOP is down, or a regression ships. That
property is the point — it is the rule that catches the failure nobody predicted.

**Known false positive:** a user who stops wearing the strap for more than 36 hours
generates no recoveries and therefore no webhooks. At current scale (one connected
account, pre-launch) this is an acceptable trade for cause-agnostic coverage. Mitigation
if it nags: widen to 48h. Once there are many users, change the shape from "no syncs at
all" to "fraction of connected users with no sync in 36h > 0.5", which is robust to any
individual's habits.

#### Rule B — misrouted deliveries (fast and specific)

```promql
sum by (provider) (increase(api_webhook_misroute_total[1h]))
```

Threshold `> 0.5`, `for: 0s`, severity `warning`, `noDataState: OK` (standard for a
counter that should normally not exist).

Fires within roughly 15 minutes of a misconfiguration, and the firing series names the
provider. The `0.5` threshold rather than `0` follows the alerting README's guidance that
`increase()` extrapolates and can report ~0.97 for a single event.

**A and B are complementary, not alternatives.** B catches this specific failure class
almost immediately and points straight at the cause; A catches every other cause of dead
ingestion, slowly but unconditionally.

#### Dashboard change

`monitoring/grafana/dashboards/whoop.json`:

- Add red-below-1 thresholds to the "Successful syncs (24h)" and "Recovery rows upserted
  (24h)" stat panels. They currently render `or vector(0)` results as a healthy green
  zero, which is precisely the misreading that let this outage persist.
- Add a connection-health panel driven by the new `api_whoop_connections{status}` gauge.

### Testing

**`prog-strength-api`**

- Conformance cases for `whoopconn.List` and `whooprecovery.Latest`/`CountForUser` in the
  shared suite, exercised against both in-memory and SQLite implementations. `Latest`
  covers the no-rows case returning `(nil, nil)`.
- `whoopadmin` handler tests: happy path for all three routes; `404` unknown user; `409`
  when the connection is not `connected`; `window_days` clamping at both bounds; and a
  `403` case confirming the routes are unreachable without the admin gate.
- `SyncSince` test asserting it records `kind="admin_resync"` and **not** `kind="window"` —
  this is the regression guard for the metric-pollution hazard described above.
- Misroute counter test: a 404 on `/webhooks/whoop,` increments
  `{provider="whoop"}`; a 404 on `/.env` and on `/webhooks/incoming/stripe.json`
  increment nothing.
- Gauge exporter test with a fake clock and repository.

**`prog-strength-tooling`**

- `respx` for the admin HTTP calls, including the token-absent degraded path.
- Stubbed boto3 paginator pages for the log scan.
- **A regression fixture replaying this outage**: 97 `POST /webhooks/whoop,` 404 lines and
  zero `kind=window` syncs must produce a check-2 finding naming the offending path, and
  exit `1`.
- A healthy fixture must exit `0` and report no findings.

**`prog-strength-infra`**

- Rules are provisioned YAML with no test harness in-repo; validation is a Grafana restart
  plus confirming both rules appear and evaluate. Rule A can be verified end-to-end by
  querying the expression in Grafana's Explore against live data — with ingestion
  currently dead it should evaluate to `0` and satisfy the firing condition today.

## Open Questions

1. **Should `pst whoop doctor` verify the registered URL against WHOOP's API rather than
   inferring it from logs?** WHOOP exposes no endpoint to read back a webhook
   registration, so the log-derived inference is the only available evidence. If that
   changes, an explicit check would be strictly better. Tentative lean: keep the
   inference, revisit if the API gains the capability.

2. **Should Rule A's window be 36h or 48h?** 36h catches a real outage a day sooner; 48h
   tolerates a strap-free weekend. Tentative lean: ship 36h and widen if it produces a
   false positive in the first month — a nagging alert that gets tuned is recoverable, an
   alert too slow to matter is not.

3. **Should the resync endpoint be reachable from the web client's admin surface rather
   than only the CLI?** A button is friendlier than a terminal, but the audience is one
   operator who already lives in `pst`. Tentative lean: CLI only; revisit if a support
   workflow emerges that involves someone who does not have AWS credentials.

4. **Should `api_webhook_misroute_total` also fire for *unsigned* deliveries to the
   correct path?** A rotated client secret would produce `401`s that Rule B would miss and
   Rule A would catch only after 36h. Check 3 in the doctor covers it interactively.
   Tentative lean: leave it to Rule A for now; add a `bad_signature` rule if the secret is
   ever rotated in anger.

## Appendix: Immediate Operator Action

Independent of this SOW, and required before any of it can help:

**Edit the webhook URL in the WHOOP Developer Dashboard to remove the trailing comma.**
The registered value ends `/webhooks/whoop,`; it must be
`https://api.progstrength.fitness/webhooks/whoop`.

Verification, in order:

```bash
# 1. The route is healthy and the signature gate is live (401, not 404):
curl -s -o /dev/null -w '%{http_code}\n' -X POST \
  https://api.progstrength.fitness/webhooks/whoop

# 2. After the dashboard edit, deliveries should answer 204 rather than 404,
#    and kind=window syncs should appear within a few hours.
```

WHOOP will not replay the 97 dropped deliveries. The days lost between 2026-07-29 and the
fix must be recovered — by `pst whoop resync --user <id> --days 30` once this SOW ships,
or by disconnecting and reconnecting the account in Settings → Integrations before then.
