---
type: sow
status: draft
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-mcp
  - prog-strength-docs
---

# WHOOP Sleep Ingestion & Sleep Tile

**Status**: Draft · **Last updated**: 2026-08-11

> Full-stack SOW. `prog-strength-api` does the scope, client, table, sync,
> webhook, endpoint, and dashboard-section work; `prog-strength-web` does the
> reconnect affordance and the tile; `prog-strength-mcp` adds one read tool;
> `prog-strength-docs` marks this shipped and records the scope-migration
> pattern. Visual scope is **`in-system`** — design-system v0.4 is decided and
> this SOW conforms to it, with one deliberate exception raised as Open
> Question 2 (sleep-stage color has no existing token set).

## Introduction

Prog Strength has no sleep data. Not "has it but doesn't render it" — the bytes
have never crossed the wire. The recovery integration
([`sows/whoop-integration.md`](./whoop-integration.md)) named sleep as an
explicit non-goal in v1, and every sleep-adjacent thing the app appears to know
is inferred from recovery: HRV, resting heart rate, and WHOOP's own recovery
score, which is itself partly computed from sleep upstream. The four
recovery-family tiles and the Recovery page are all reading that one 3-metric
row.

The `sleep_id` column on `user_whoop_recovery` is the most misleading artifact
of that gap. It is present — and even serialized out of `GET /whoop/recovery`
(`internal/whoopsync/read_handler.go:27`) — but it is **not a foreign key to
anything we hold**. It exists solely because WHOOP v2's `recovery.deleted`
webhook identifies the deleted record by its associated sleep UUID rather than
by a recovery id, so we need it to route the delete
(`internal/whoopsync/webhook.go:167`). Anyone reading the schema cold would
reasonably assume sleep data is one join away. It is not.

There are **two independent gates** on the WHOOP OAuth flow, and the project has
exactly one of them open:

1. **App configuration**, in the WHOOP developer dashboard — which scopes the
   app is *permitted* to request. `read:sleep`, `read:workout`, and
   `read:body_measurement` are all already enabled here.
2. **The authorization request**, in our code — which scopes the app *actually
   asks for* at consent. This is `ScopeString` at
   `internal/whoopsync/oauth.go:38`, and it reads
   `"read:recovery read:cycles read:profile offline"`.

Only the intersection is granted. The stored token therefore carries no
`read:sleep`, and `GET /v2/activity/sleep` would 403 today despite the dashboard
showing the scope as available. The fix on our side is one line — but that one
line has a migration consequence disproportionate to its size, covered in
"Scope Migration" below, because **OAuth scopes are fixed at consent and this
codebase has never added one to a live connection before.**

## Proposed Solution

Five pieces, in dependency order:

1. **Request `read:sleep`** and teach the connection model that a connection can
   be *connected but under-scoped*. This is the load-bearing piece: without it
   everything downstream 403s, and it is the only part that requires a human
   action (re-consent) to take effect.

2. **A sleep client method and a `user_whoop_sleep` table**, keyed by WHOOP's
   sleep UUID rather than by (user, date) — a deliberate departure from the
   steps-shaped recovery table, because naps mean a user can have several sleep
   records on one calendar day.

3. **An independent sleep sync path** that shares the token/refresh machinery
   with recovery but fails separately, so an under-scoped or degraded sleep
   fetch can never take recovery ingestion down with it.

4. **`sleep.updated` / `sleep.deleted` webhook handling** and a
   `GET /whoop/sleep` read endpoint on the house timezone+local-date convention.

5. **One dashboard tile** — a stage-composition bar plus a duration-vs-need
   headline — and one MCP tool.

## Goals and Non-Goals

### Goals

- A connected user's nightly sleep persists per sleep record: stage durations
  (REM, slow-wave, light, awake), time in bed, sleep/disturbance cycle counts,
  sleep need and its components, respiratory rate, and the performance /
  efficiency / consistency percentages.
- A user whose connection predates the scope change sees an explicit, actionable
  **Reconnect to enable sleep** state in Settings → Integrations — not silence,
  and not a broken tile.
- Sleep sync degrades independently of recovery sync: a 403, a rate-limit, or a
  WHOOP outage on the sleep endpoint leaves recovery ingestion untouched, and
  vice versa.
- New sleep records land within minutes of WHOOP scoring them, via webhook, with
  no polling and no new infrastructure — the same poke-not-payload pattern
  recovery already uses.
- Reconnecting backfills the trailing ~30 nights so the tile is non-empty
  immediately.
- One new dashboard tile (`sleep`) shows last night's stage composition and
  duration against WHOOP's computed need; it is addable/removable through the
  existing customizable-grid machinery and absent for users with no connection.
- `GET /whoop/sleep` follows the house date-window convention — the client
  sends timezone + local dates, the API does every UTC conversion.
- The agent can read sleep through a new MCP tool using the standard
  bearer-forwarding pattern.
- Ingestion liveness for sleep is observable **by absence of success**, not only
  by error rate.

### Non-Goals

- **A Sleep page.** The Recovery page precedent
  ([`sows/recovery-page.md`](./recovery-page.md)) says a dedicated page earns its
  place once there is history worth charting. Ship the tile, accumulate a month,
  then decide. The tile's `href` deep-links to `/recovery` in the interim.
- **A sleep-family tile spread.** [`sows/recovery-tile-family.md`](./recovery-tile-family.md)
  produced five tiles from a Design Exploration with a binding
  no-two-tiles-hero-the-same-figure constraint. Sleep deserves the same
  treatment eventually; it does not deserve it before any sleep data exists to
  design against. **One** tile here.
- **Strain, workouts, and body measurements.** `read:workout` and
  `read:body_measurement` are enabled upstream and would need no WHOOP dashboard
  work — but each is its own product surface. See Open Question 1, which is
  specifically about whether to *request* them now while a reconnect is already
  being forced, without building anything on them yet.
- **Nap surfacing.** Naps are **ingested** (they are sleep records and dropping
  them would corrupt the sleep-need math) but the v1 tile shows the night's
  main sleep only.
- **Sleep-aware coaching or agent proactivity.** The agent gets a read tool.
  Prompting it to volunteer sleep-aware advice is later tuning.
- **Recomputing or second-guessing WHOOP's scores.** Sleep need, performance,
  and efficiency are stored as WHOOP computes them. Prog Strength does not
  derive its own sleep model.
- **Mobile.** Picked up in a later parity phase alongside the recovery tiles.
- **Backfilling beyond WHOOP's retention or beyond 30 days at connect.** An
  operator-triggered wider resync exists (`whoopadmin`); an automatic deep
  backfill does not.

## Implementation Details

### Scope Migration (`prog-strength-api`, `prog-strength-web`)

This is the part with no code-only solution, so it is specified first.

`ScopeString` becomes:

```go
ScopeString = "read:recovery read:cycles read:sleep read:profile offline"
```

Three consequences, none of which are optional:

**Existing tokens never gain the scope.** `RefreshToken` re-sends the scope
string on every refresh (`oauth.go:193`), which keeps `offline` alive across
WHOOP's single-use refresh rotation — but a refresh cannot *widen* a grant. The
authorization-code flow must be run again. For a pre-launch app with one
connected user this is a single manual reconnect; the machinery below exists so
it does not become a silent failure for beta users later.

**The connection model learns about required scopes.** Add to `whoopsync`:

```go
// RequiredScopes are the scopes a connection must carry for every ingestion
// path to function. A connection missing any of these is CONNECTED but
// DEGRADED: its tokens are valid and recovery still syncs, but the paths
// needing the absent scope are skipped rather than attempted.
var RequiredScopes = []string{"read:recovery", "read:cycles", "read:sleep", "read:profile"}

// MissingScopes returns the RequiredScopes absent from a connection's granted
// scope string, in RequiredScopes order.
func MissingScopes(granted string) []string
```

`whoopconn.Connection.Scopes` already persists what WHOOP echoed back at consent
(written at `handler.go:214`), so this is a pure function over data we hold — no
migration, no extra WHOOP call.

**Both read surfaces expose it.** `GET /me/whoop/connection` gains
`missing_scopes: []string` alongside the existing `scopes`. `whoopadmin`'s
`connectionView` gains the same, so an operator can see at a glance which
connections are under-scoped without decrypting anything.

> **Deliberately not a new `Status`.** It is tempting to add
> `StatusScopesStale` next to `connected` / `revoked` / `error`. Resist it: the
> connection *is* connected, its tokens *are* valid, and recovery ingestion *is*
> working. Folding a partial-capability signal into the lifecycle enum would
> make every existing `status == connected` check subtly wrong, including the
> webhook's not-connected gate (`webhook.go:145`) — which would start dropping
> valid recovery events. Capability is a separate axis from lifecycle. Keep it
> that way.

**Frontend.** The WHOOP card on Settings → Integrations renders a third state
between connected and error: connected, but with a one-line explanation and a
**Reconnect** action when `missing_scopes` is non-empty. The copy names what is
missing in product terms ("Reconnect to enable sleep tracking"), not in scope
terms — `read:sleep` is not a user-facing noun. The existing Reconnect path is
reused verbatim; this is a new *reason* to show it, not a new flow.

### Data Model (`prog-strength-api`)

One migration, `058_user_whoop_sleep.sql`.

`user_whoop_sleep` — **one row per WHOOP sleep record**, not one row per user
per day. This is the significant departure from `user_whoop_recovery` (040) and
from the steps-shaped table pattern generally, and the reason is naps: WHOOP
emits a separate sleep record with `nap: true` for daytime sleep, so
(`user_id`, `date`) is genuinely not unique. Keying by WHOOP's own UUID also
makes `sleep.deleted` a direct delete rather than a date-derivation round trip.

```sql
CREATE TABLE IF NOT EXISTS user_whoop_sleep (
    id                  TEXT PRIMARY KEY,
    user_id             TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    whoop_sleep_id      TEXT NOT NULL,      -- v2 UUID; the webhook delete key
    date                TEXT NOT NULL,      -- YYYY-MM-DD local WAKE date (see below)
    is_nap              INTEGER NOT NULL,   -- 0/1
    started_at          TEXT NOT NULL,      -- RFC3339
    ended_at            TEXT NOT NULL,      -- RFC3339
    timezone_offset     TEXT NOT NULL,      -- e.g. "-06:00", as WHOOP sent it
    score_state         TEXT NOT NULL,      -- SCORED | PENDING | UNSCORABLE

    -- stage summary (nullable: absent unless SCORED)
    in_bed_milli            INTEGER,
    awake_milli             INTEGER,
    no_data_milli           INTEGER,
    light_sleep_milli       INTEGER,
    slow_wave_sleep_milli   INTEGER,
    rem_sleep_milli         INTEGER,
    sleep_cycle_count       INTEGER,
    disturbance_count       INTEGER,

    -- sleep need (nullable)
    need_baseline_milli         INTEGER,
    need_from_sleep_debt_milli  INTEGER,
    need_from_strain_milli      INTEGER,
    need_from_nap_milli         INTEGER,

    -- scored percentages + respiratory rate (nullable REALs)
    respiratory_rate            REAL,
    performance_pct             REAL,
    consistency_pct             REAL,
    efficiency_pct              REAL,

    created_at          TEXT NOT NULL,
    updated_at          TEXT NOT NULL
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_user_whoop_sleep_record
    ON user_whoop_sleep(user_id, whoop_sleep_id);
CREATE INDEX IF NOT EXISTS idx_user_whoop_sleep_date
    ON user_whoop_sleep(user_id, date);
```

Durations are stored in **milliseconds as WHOOP sends them**, not converted to
minutes at ingest. Storing the wire value keeps the ingest layer dumb and
reversible; presentation rounding is the tile's job. Every score field is
nullable for the same reason recovery's are — a `PENDING` record has a start,
an end, and nothing else, and it should still be stored so the row exists when
the score arrives.

New package `internal/whoopsleep`, mirroring `internal/whooprecovery`'s shape:
`Entry`, `Repository` (interface), `sqlite_repository.go`, with
`Upsert`, `ListRange`, `DeleteByWhoopSleepID`, and `DeleteForUser`.

### Dating a Night (`prog-strength-api`)

**A sleep record is dated by its `end` (wake time), localized by the record's
own `timezone_offset`.**

This matters more than it looks, and it is worth stating why rather than leaving
it to be rediscovered. Recovery went through exactly this bug: migration 041
exists (`041_wipe_whoop_recovery_redate.sql`) because recoveries were originally
dated by their cycle's `start`, and WHOOP cycles run sleep-onset → sleep-onset,
so every recovery landed on the day *before* the user woke up with it and
"today" never had data.

Sleep has the same trap in a more obvious form: a night starting 22:40 Monday
and ending 06:15 Tuesday is, to any user, **Tuesday's sleep** — it is the night
they are recovering from when they look at the dashboard on Tuesday. Dating by
`start` would put it on Monday and, worse, would put it on a *different day than
the recovery computed from it*, making the dashboard internally inconsistent
between two tiles reading the same night.

Two simplifications fall out, both worth having:

- **No cycle join.** Recovery's `syncWindow` fetches `/v2/cycle` purely to
  obtain `timezone_offset` (`service.go:203`). A sleep record carries its own
  `timezone_offset`, so the sleep path needs one endpoint, not two — fewer
  calls, and no `skipped_no_cycle` failure mode.
- **DST is handled by construction.** The offset is the one in force at that
  record's end, as WHOOP recorded it.

`deriveDate(instantRFC3339, tzOffset string)` already exists in `service.go:371`
and does exactly this. Reuse it; do not write a second one.

### Sync Path (`prog-strength-api`)

Add `Client.Sleeps(ctx, accessToken, start, end, limit)` following the existing
paginated-envelope pattern (`recoveryEnvelope` / `cycleEnvelope` →
`sleepEnvelope`), against `GET /v2/activity/sleep` with `start`/`end`/`limit`
and `next_token` paging under the same `maxPages` cap.

Add `Service.syncSleepWindow(ctx, kind, userID, start, end, limit)` as a
**sibling** of `syncWindow`, not a branch inside it. Both are called from
`Backfill`, `SyncWindow`, and `SyncSince`; both go through the same
`validToken` (so the per-user keyed mutex still serializes the single-use
refresh-token rotation correctly across both).

The scope gate runs first:

```go
// Skip the sleep path entirely for a connection that never consented to
// read:sleep. Calling anyway would 403 on every sync for every under-scoped
// user, burning rate limit and filling the error metric with a condition that
// is not an error — it is a user who has not reconnected yet.
if slices.Contains(missing, "read:sleep") { /* record skip, return */ }
```

**Failure isolation is a requirement, not an optimization.** The caller runs
recovery and sleep in sequence and joins their outcomes such that:

- a sleep failure is logged, counted, and returned in the `SyncResult`, but does
  **not** fail the overall sync if recovery succeeded;
- the webhook consequently does **not** return 500 for a sleep-only failure,
  because a 500 makes WHOOP redeliver the recovery data we already stored
  successfully — the exact reasoning already recorded for `MarkWindowSync` at
  `service.go:253`;
- a *recovery* failure retains its current behavior unchanged.

Metrics: extend `syncsTotal` with a `domain` label (`recovery` | `sleep`) rather
than adding a parallel counter, and add `sleep` variants of the row counters
(`upserted`, `skipped_unscored`, `skipped_bad_date`, `skipped_no_scope`). Note
there is deliberately no `skipped_no_cycle` for sleep — it cannot occur.

`MarkWindowSync` stays a single per-connection liveness stamp advanced only by
`kind == "window"`, unchanged. Sleep liveness is not tracked separately: the
same webhook path carries both, so a dead path is dead for both, and a second
stamp would only create a second thing to forget to alert on.

### Webhooks (`prog-strength-api`)

Two cases join the switch at `webhook.go:151`:

- **`sleep.updated`** — poke, not payload. Re-sync the recent window for that
  user, same as `recovery.updated`. Idempotent upserts make redelivery,
  duplicates, and out-of-order arrival safe.
- **`sleep.deleted`** — the event `id` is the sleep UUID; call
  `DeleteByWhoopSleepID(ctx, conn.UserID, event.ID)`. This is the first webhook
  where that id maps to a record we actually own.

The existing HMAC validation, unknown-user routing, and not-connected gate all
apply unchanged. Unknown event types keep falling through to the ignore branch.

> **Registration is the thing most likely to silently break this.** Per
> [`sows/whoop-integration-diagnostics.md`](./whoop-integration-diagnostics.md),
> webhook ingestion was dead from ship until
> 2026-07-31 because of a trailing comma in the registered URL — a 404 that
> nobody noticed because the alerting watched for errors rather than for absence
> of success. If sleep events require any change to the WHOOP-side webhook
> registration, verify by **observing a real `sleep.updated` arrive**, not by
> reading the config back.

### API Surface (`prog-strength-api`)

`GET /whoop/sleep` (authed), mirroring `GET /whoop/recovery`
(`read_handler.go`) exactly: required `timezone` (IANA), optional `since` /
`until` as inclusive local `YYYY-MM-DD`, converted via `internal/daterange`.
Per the house date-window convention (`internal/daterange`) the client never
constructs UTC bounds. Returns `{ "sleep": [ ... ] }`, most recent first, one object per sleep
record with `is_nap` present so a client can filter.

Routes registered alongside the existing WHOOP routes in `server.go`; the
route-presence contract test (`server_test.go:78-82`) gains the entry.

### Dashboard Section (`prog-strength-api`)

New `TileSleep TileID = "sleep"`, appended to `Catalog` after the recovery
family and mirrored in `lib/dashboard-tiles.ts` — the Go/TS catalog contract
test asserts identical id set **and order**, so both move together in one PR.

`SleepSection` on the summary DTO, `omitempty`, gated on a connected WHOOP
connection exactly as the recovery family is (`handler.go:278`). Unlike the
recovery family, sleep is a single tile reading a single section, so it needs no
shared-section fan-out.

```go
type SleepSection struct {
    LastNight *SleepNight  `json:"last_night"`  // nil when no non-nap record for today
    Nights    []SleepNight `json:"nights"`      // date-aligned trailing window, oldest→newest
}
```

`Nights` follows the honest date-aligned convention the recovery payload
established ([`sows/dashboard-recovery-metrics-payload.md`](./dashboard-recovery-metrics-payload.md)):
every date in the window present, missing nights carrying null metrics — never
omitted, never zero-filled. A night with no data and a night of zero sleep are
different facts.

**Which record is "the night".** For a given local date, the night is the
non-nap record; if more than one non-nap record shares a date (rare — a
fragmented night WHOOP split, or travel across a date line), take the one with
the **longest in-bed duration**, tie-broken by latest `ended_at`. The rule lives
in one function with its own test; it is not open-coded at call sites.

### Frontend (`prog-strength-web`)

One tile, `app/(app)/dashboard/_components/sleep/`, following the recovery-tile
component conventions (co-located `fixtures.ts`, `shared.ts`, per-component
tests).

The tile answers two questions and prints no number that another tile already
heroes:

1. **Did you get enough?** Headline duration (`7h 12m`) against WHOOP's computed
   need, with performance percentage as the qualifier. This is the figure the
   user asked for and no existing tile shows.
2. **What kind of sleep was it?** A single horizontal **stacked stage bar** —
   slow-wave / REM / light / awake — proportional to the night, with durations
   on hover/focus. This is the "time in different sleep zones" ask, and a
   stacked bar is the right form because the question is *composition of a
   whole*, which is exactly what a part-to-whole mark is for. A four-series
   line chart would answer a question nobody asked.

Empty and degraded states are first-class: no connection → no tile (grid
handles it); connected but under-scoped → the tile shows the same
**Reconnect to enable sleep** affordance as the Settings card, because the
dashboard is where the user will notice the absence; connected and scoped but no
data yet → the tile's own empty state, matching `connect-card.tsx` precedent.

Design-system conformance is `in-system` (v0.4): no new tokens, no accent
reuse for data, existing card furniture. The one genuine gap is stage color —
see Open Question 2.

### MCP (`prog-strength-mcp`)

One tool, `get_whoop_sleep`, in `src/prog_strength_mcp/whoop.py` next to
`get_whoop_recovery` — same required-`timezone` + optional `since`/`until`
signature, same `_auth_header_or_raise()` bearer forwarding, same
"empty result probably means not connected → point at Settings → Integrations"
note in the docstring. Its docstring adds the equivalent for the new failure
mode: an empty result for a user whose recovery *does* return data most likely
means they have not reconnected since sleep was added.

Durations are returned in milliseconds as stored; the docstring states the unit
explicitly so the agent does not have to infer it and get it wrong by 60×.

### Observability (`prog-strength-api`)

- Sleep rows in the sync summary log line (`service.go:235`) alongside the
  recovery counts, so the one-line "did this user's data land" answer covers
  both domains.
- `domain`-labelled sync counters as above.
- A gauge for **connections missing a required scope**, exported by the
  `whoopadmin` exporter next to the existing liveness gauge. This is what turns
  "users are silently not getting sleep" from an invisible state into a
  dashboard number — and it generalizes to every future scope addition.
- Alerting follows the absence-of-success direction already established: alert
  when no successful window sync has been recorded within the expected interval,
  **not** on error rate. That is the lesson from
  [`sows/whoop-integration-diagnostics.md`](./whoop-integration-diagnostics.md)
  and it applies unchanged here.

### Backfill or Migration

No data migration — the table is new and starts empty.

The rollout sequence is load-bearing and cannot be reordered:

1. Merge and deploy the API with the new scope, table, sync, webhook, and
   endpoint. **Nothing changes for the existing connection**: `read:sleep` is
   missing, the sleep path skips cleanly, recovery keeps working, and the new
   gauge reads 1 under-scoped connection.
2. Deploy the web reconnect affordance. The Settings card now explains the
   state.
3. **Reconnect the account.** Disconnect and re-run consent. The new grant
   includes `read:sleep`; `Backfill` runs at connect and pulls ~30 nights.
4. Verify a real `sleep.updated` webhook arrives and upserts — observed, not
   assumed (see the webhook note above).
5. Add the tile to the dashboard layout and confirm it renders against real
   data.

Disconnect semantics match recovery's: ingested sleep rows are the user's data
and survive disconnect; only the connection and tokens are destroyed. A
WHOOP-initiated `sleep.deleted` **does** delete the row — that is a data
correction, not an account action.

### Testing

- `deriveDate` applied to sleep: a night crossing midnight lands on the **wake**
  date; a night crossing a DST boundary uses the offset WHOOP sent; a nap at
  14:00 lands on its own day. This is the regression that migration 041 exists
  to commemorate — it gets explicit tests, not incidental coverage.
- `MissingScopes`: exact match, superset, empty string, and scopes in a
  different order than `RequiredScopes`.
- Sync isolation: a sleep fetch returning 403 leaves recovery rows upserted and
  the overall sync non-failing; a sleep fetch returning 429 does the same; a
  recovery failure retains current behavior.
- Under-scoped connection: sleep path is skipped without an HTTP call at all
  (assert against the fake client's call log, not just the outcome).
- Webhook: `sleep.updated` triggers a window sync; `sleep.deleted` removes the
  row by UUID and is a no-op for an unknown UUID; both are safe on redelivery.
- Night-selection: multiple non-nap records on one date resolve deterministically
  by the stated rule.
- Payload: `Nights` is date-aligned with nulls for missing nights, never
  zero-filled.
- Catalog contract: Go `Catalog` and `TILE_CATALOG` stay identical in id set and
  order (the existing test covers this once `sleep` is added to both).
- Tile: renders last night, renders the under-scoped reconnect state, renders
  the no-data empty state.

## Open Questions

1. **Should `read:workout` and `read:body_measurement` be requested in the same
   consent, before anything consumes them?** A reconnect is a manual,
   user-visible action, and this SOW forces one. Adding those two scopes now
   means the eventual strain/workout SOW and any body-composition work need
   **zero** further reconnects; deferring means a second forced reconnect later,
   and a third after that. Against it: requesting scopes nothing reads violates
   least privilege, and the consent screen lists them, so the user is agreeing to
   share data the app has no use for yet.

   *Recommendation: request them.* For a pre-launch app with one connected user
   the least-privilege cost is close to zero, and the deferred cost — forcing
   beta users through repeat reconnects post-launch, each one a chance to churn
   or silently end up under-scoped — is the larger risk. If the answer is yes,
   `RequiredScopes` should still list **only** what ingestion actually needs, so
   the under-scoped gauge does not light up for scopes nothing consumes.

2. **Sleep-stage color has no home in design-system v0.4.** The stage bar needs
   four visually distinct segments. The decided palette offers one accent
   (app-chrome only, never a data hue), per-discipline activity hues (which
   would misread — sleep is not a discipline), and desaturated status colors
   (which would misread harder — light sleep is not a warning). Introducing a
   new four-way categorical set is a design-system *change*, which this SOW's
   `in-system` scope explicitly excludes.

   *Recommendation: a single-hue luminance ramp* — deep → light → REM → awake as
   four steps of one hue, darkest to lightest. Sleep stages have a natural
   ordering by depth, so an ordinal ramp is arguably the more honest encoding
   than four categorical hues anyway, and a ramp derived from an existing hue
   stays in-system. If the owner would rather have four true categorical hues,
   that needs a Design Exploration first and this SOW should ship the tile
   behind the ramp in the interim rather than block on it.

3. **Exact WHOOP v2 sleep field names must be verified against the live API
   before the client struct is written.** The column set above reflects the
   documented v2 `/v2/activity/sleep` score shape (`stage_summary`,
   `sleep_needed`, `sleep_performance_percentage`, and siblings), but this SOW
   was written from the existing integration code plus prior knowledge of the
   API, **not** from a live response captured during drafting. A single
   mis-transcribed JSON tag silently produces a column of nulls that looks like
   "WHOOP didn't score it." Capture one real response first and diff it against
   the struct; treat any discrepancy as a correction to this document.

4. **Does the sleep tile belong in the recovery family's shared-section
   pattern?** Today it is its own section. If a sleep tile *spread* later
   arrives (non-goal here), the recovery family's one-section-many-tiles
   arrangement (`handler.go:278`) is the precedent to follow, and the section
   key should be `sleep` from the start so that expansion needs no retirement
   mapping. This SOW assumes that and names the key `sleep` rather than
   `sleep_tile` for exactly that reason — worth confirming the assumption holds.
