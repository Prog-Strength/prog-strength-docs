# Adding an OAuth scope to a live integration

**Status**: Living note · **Last updated**: 2026-08-11 · **Provenance**: [`sows/whoop-sleep-ingestion.md`](../sows/whoop-sleep-ingestion.md)

Adding a scope to a provider integration looks like a one-line change and is
not. `read:sleep` was added to WHOOP in the sleep-ingestion SOW; this note
records the shape of that migration so the next scope addition — WHOOP's
`read:workout`, or a scope on any other provider — does not have to rediscover
it. Nothing here is WHOOP-specific except the names.

## Two gates, and only the intersection is granted

There are two independent gates on any OAuth scope, and they are configured in
different places by different people.

1. **The provider's app configuration** — which scopes the app is *permitted*
   to request. For WHOOP this is the developer dashboard, and `read:sleep`,
   `read:workout`, and `read:body_measurement` were all enabled there long
   before any of them was used.
2. **The authorization request** — which scopes the app *actually asks for* at
   consent. In our code this is `ScopeString` in
   `internal/whoopsync/oauth.go`, today
   `"read:recovery read:cycles read:sleep read:profile offline"`.

Only the intersection is granted, and only what was granted lands in the stored
token. A dashboard showing a scope as "enabled" therefore says nothing about
what any given connection carries — before this work `read:sleep` was enabled
upstream and `GET /v2/activity/sleep` would still have 403'd on every
connection we held. When a provider call fails on permissions, check the
persisted grant, not the provider's console.

## A refresh cannot widen a grant

`RefreshToken` re-sends the scope string on every refresh, which is what keeps
`offline` alive across WHOOP's single-use refresh-token rotation. It does not
widen the grant. Scopes are fixed at consent, so the only way an existing
connection gains one is by running the authorization-code flow again.

That makes adding a scope **a migration with a human step**, not a code change.
The deploy is inert for everyone already connected: they stay on the old grant
until they personally re-consent, and no operator action, backfill, or token
surgery can change that. Ship the API so the new path skips cleanly, ship the
affordance that explains the state, then reconnect — and assume some users sit
in the intermediate state indefinitely, because some will.

## Capability is a separate axis from lifecycle

This is the load-bearing lesson. The tempting move is a new `Status` —
`scopes_stale` next to `connected` /
`revoked` / `error`. Resist it. An under-scoped connection *is* connected: its
tokens are valid, they refresh, and every ingestion path whose scope it does
hold keeps working. Folding a partial-capability signal into the lifecycle enum
makes every existing `status == connected` check subtly wrong — including the
webhook's not-connected gate (`webhook.go`), which would start dropping valid
recovery events for a user whose only problem is that they have not reconnected
yet. A user-visible regression in a working feature, caused by adding a
different feature.

Capability is its own axis. `status` answers "does this connection work at
all"; `missing_scopes` answers "what can it do". Both read surfaces carry both:
`GET /me/whoop/connection` and `whoopadmin`'s connection view each expose
`missing_scopes` alongside `scopes`. It is deliberately not `omitempty` — an
omitted key is indistinguishable from "nothing missing", which would silently
hide the reconnect affordance, so a fully-scoped connection serializes `[]`.

## Three moving parts make the state visible instead of silent

The mechanism is small and worth copying wholesale.

**`RequiredScopes`** (`internal/whoopsync/scopes.go`) is the list of scopes
ingestion actually needs — deliberately *not* the same list as `ScopeString`.
`offline` is a grant modifier rather than a read capability, and a connection
that lost it fails loudly at refresh instead of quietly under-scoped. Anything
we request but do not yet consume must stay out, or the gauge below lights up
for a capability nothing reads.

**`MissingScopes(granted string)`** is a pure function over the scope string
WHOOP echoed back at consent, which `whoopconn.Connection.Scopes` already
persists. No migration, no extra provider call, no state to keep in sync — the
answer is derived from data we hold, every time it is asked for.

**The under-scoped gauge** (`api_whoop_connections_missing_scope`, in
`internal/whoopadmin/connections_gauge.go`) counts connected connections missing
at least one required scope. This is the piece that pays for itself: it turns
"some users are silently not getting data" from an invisible state into a
dashboard number, and it generalizes for free — widen `RequiredScopes` and the
gauge starts reporting the reconnect backlog with no further work. It counts
only `connected` rows, because a revoked connection is not ingesting anything
and counting it would report a backlog that does not exist.

## Skip, don't call

The ingestion path whose scope is absent must be **skipped, not attempted**.
`syncSleepWindow` checks `MissingScopes` before it even fetches a token and
returns early. Calling anyway would 403 on every sync for every under-scoped
user, burning rate limit and filling the error metric with a condition that is
not an error — it is a user who has not reconnected yet.

The skip is counted as its own row disposition (`skipped_no_scope`) and
deliberately *not* as an `ok` in the sync counter. An "ok" there would make the
path read as alive while nothing is being fetched, which is the same
absence-of-success blindness that hid the dead webhook registration for four
months (see [`sows/whoop-integration-diagnostics.md`](../sows/whoop-integration-diagnostics.md)).

## The user-facing half

The affordance names what is missing in **product** terms and never in scope
terms: "Reconnect to enable sleep tracking", because `read:sleep` is not a
user-facing noun. It reuses the existing reconnect path verbatim — the same
OAuth navigation as Connect — so this is a new *reason* to show a control, not
a new flow. On Settings the Reconnect button appears *alongside* Disconnect,
not instead of it: the connection still works, so revoking it has to stay
possible.

Two rules in `lib/whoop.ts` carry forward. A surface asks about the scope it
needs **by name** (`missingSleepScope`), never about the array's length — a
connection missing only `read:workout` ingests sleep perfectly, and a length
check would hide that user's real data behind a false prompt. And the sentence
lives in one place, shared by the Settings row and the dashboard tile, rather
than two literals that drift. Where a surface genuinely speaks for every
capability, as the Settings row does, it falls back to a truthful generic line
("Reconnect to enable the rest of your Whoop sync") rather than naming the
wrong one: a vague true sentence beats a specific wrong one.

## Where this stands today

Open Question 1 of the SOW proposed also requesting `read:workout` and
`read:body_measurement` while a reconnect was already being forced. The shipped
`ScopeString` does **not** include them, so that consent is still owed and
adding either is another run of this same migration. When they arrive they
belong in `ScopeString` immediately and in `RequiredScopes` only once something
reads them.
