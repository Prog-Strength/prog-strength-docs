# WHOOP Sleep Ingestion & Sleep Tile — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ingest WHOOP sleep records into Prog Strength, surface them as one
dashboard tile and one MCP read tool, and teach the connection model that a
connection can be *connected but under-scoped*.

**Architecture:** `prog-strength-api` requests `read:sleep` at consent, adds a
`user_whoop_sleep` table keyed by WHOOP's sleep UUID, an independent sleep sync
path that fails separately from recovery, `sleep.updated`/`sleep.deleted` webhook
handling, `GET /whoop/sleep`, and a `sleep` dashboard section/tile id.
`prog-strength-web` renders the `missing_scopes` reconnect affordance and the
sleep tile. `prog-strength-mcp` adds `get_whoop_sleep`. `prog-strength-docs`
records the scope-migration pattern and flips the SOW to shipped.

**Tech Stack:** Go 1.25 (chi, SQLite, prometheus), Next.js 16 / React 19 /
TypeScript / Tailwind v4 / vitest, Python 3.12 (FastMCP, httpx, pytest).

**Source SOW:** [`sows/whoop-sleep-ingestion.md`](../sows/whoop-sleep-ingestion.md)

---

## Decisions taken on the SOW's Open Questions

- **OQ1 (request `read:workout` / `read:body_measurement` now?)** — **No.** The
  Implementation Details section specifies `ScopeString` verbatim and that is the
  normative spec; widening an OAuth grant to cover data nothing reads is an
  owner decision, not an implementer's. Implement the spec'd string; leave OQ1
  open and call it out in the API PR body.
- **OQ2 (sleep-stage color)** — **Take the recommendation:** a single-hue
  luminance ramp, deep → light → REM → awake, darkest to lightest. Implemented by
  **reusing the existing four-stop `--discipline-lift-1..4` ramp**, which
  design-system v0.4 already declares "the canonical encoding of graded
  intensity… reusable by any future per-intensity graphic". Zero new tokens, so
  the SOW's `in-system` scope holds.
- **OQ3 (verify field names against a live response)** — cannot be done from this
  environment. Structs follow the documented v2 `/v2/activity/sleep` score shape.
  The API PR body must state OQ3 is still open and that step 4 of the rollout is
  the verification.
- **OQ4 (section key)** — assume it holds; the key is `sleep`.

---

## File Structure

### `prog-strength-api`

| File | Responsibility |
| --- | --- |
| `internal/whoopsync/scopes.go` *(new)* | `RequiredScopes`, `MissingScopes` |
| `internal/whoopsync/scopes_test.go` *(new)* | scope-set unit tests |
| `internal/whoopsync/oauth.go` | `ScopeString` gains `read:sleep` |
| `internal/whoopsync/handler.go` | `missing_scopes` on `GET /me/whoop/connection`; mount `GET /whoop/sleep` |
| `internal/whoopadmin/handler.go` | `missing_scopes` on `connectionView` |
| `internal/whoopadmin/connections_gauge.go` | under-scoped-connections gauge |
| `internal/db/migrations/058_user_whoop_sleep.sql` *(new)* | the table |
| `internal/whoopsleep/` *(new pkg)* | `sleep.go`, `repository.go`, `sqlite_repository.go` (+ tests) |
| `internal/whoopsync/client.go` | `Sleep`/`SleepScore` structs, `Client.Sleeps` |
| `internal/whoopsync/service.go` | `syncSleepWindow`, failure isolation, `SyncResult` sleep counts |
| `internal/whoopsync/metrics.go` | `domain` label on `syncsTotal`; `sleepRowsTotal` |
| `internal/whoopsync/webhook.go` | `sleep.updated` / `sleep.deleted` |
| `internal/whoopsync/sleep_read_handler.go` *(new)* | `GET /whoop/sleep` |
| `internal/dashboard/tiles.go` | `TileSleep` |
| `internal/dashboard/dto.go` | `SleepSection`, `SleepNight` |
| `internal/dashboard/sleep.go` *(new)* | `buildSleep`, `pickNight` |
| `internal/server/server.go` | wire the sleep repo through handler/webhook/service/dashboard |

### `prog-strength-web`

| File | Responsibility |
| --- | --- |
| `lib/dashboard-tiles.ts` | `sleep` catalog entry |
| `lib/api.ts` | `missing_scopes` on `WhoopConnection`; `DashboardSleep*` raw types |
| `lib/dashboard.ts` | `SleepView` / `SleepNightView` + `adaptSleep` |
| `app/(app)/settings/_components/whoop-connection-row.tsx` | third state |
| `app/(app)/dashboard/_components/sleep/` *(new)* | `fixtures.ts`, `shared.ts`, `stage-bar.tsx`, `sleep-tile.tsx`, `reconnect-card.tsx` (+ tests) |
| `app/(app)/dashboard/_components/tile-renderer.tsx` | `case "sleep"` |

### `prog-strength-mcp`

| File | Responsibility |
| --- | --- |
| `src/prog_strength_mcp/api_client.py` | `get_whoop_sleep` |
| `src/prog_strength_mcp/whoop.py` | `get_whoop_sleep` tool |
| `tests/test_whoop_tools.py` | client + tool boundary tests |

### `prog-strength-docs`

| File | Responsibility |
| --- | --- |
| `notes/oauth-scope-migration.md` *(new)* | the reusable pattern |
| `sows/whoop-sleep-ingestion.md` | status flip |

---

## Task order

API 1 → 8 are sequential (each builds on the last). Web 1 → 4 depend only on the
API contract (already fixed by this plan), MCP 1 depends on nothing but the
endpoint shape. Docs 1 last.

---

## Task A1: Required scopes and `MissingScopes`

**Files:**
- Create: `internal/whoopsync/scopes.go`
- Create: `internal/whoopsync/scopes_test.go`
- Modify: `internal/whoopsync/oauth.go` (`ScopeString`, ~line 38)

- [ ] **Step 1: Write the failing test** — `internal/whoopsync/scopes_test.go`

Cover: exact match → nil; superset (extra granted scope) → nil; empty granted
string → all of `RequiredScopes`; granted scopes in a DIFFERENT order than
`RequiredScopes` → nil; a granted string missing only `read:sleep` → exactly
`["read:sleep"]`; result order follows `RequiredScopes` order, not the granted
string's. Use table-driven subtests and `slices.Equal`.

- [ ] **Step 2: Run it and watch it fail** — `go test ./internal/whoopsync/ -run Scopes -v`

- [ ] **Step 3: Implement** — `internal/whoopsync/scopes.go`

```go
package whoopsync

import "strings"

// RequiredScopes are the scopes a connection must carry for every ingestion
// path to function. A connection missing any of these is CONNECTED but
// DEGRADED: its tokens are valid and recovery still syncs, but the paths
// needing the absent scope are skipped rather than attempted.
//
// Deliberately NOT the same list as ScopeString: `offline` is a grant
// modifier rather than a read capability, and a connection that somehow lost
// it fails loudly at refresh rather than quietly under-scoped. Anything we
// request but do not yet consume must stay OUT of this list, or the
// under-scoped gauge lights up for a capability nothing reads.
var RequiredScopes = []string{"read:recovery", "read:cycles", "read:sleep", "read:profile"}

// MissingScopes returns the RequiredScopes absent from a connection's granted
// scope string, in RequiredScopes order. granted is WHOOP's echoed-back
// space-separated scope string as persisted on the connection row, so this is
// a pure function over data we already hold — no WHOOP call, no migration.
// The returned slice is nil (not empty) when nothing is missing, so callers
// can test it with len() or against nil interchangeably.
func MissingScopes(granted string) []string {
	have := make(map[string]bool, len(RequiredScopes))
	for _, s := range strings.Fields(granted) {
		have[s] = true
	}
	var missing []string
	for _, s := range RequiredScopes {
		if !have[s] {
			missing = append(missing, s)
		}
	}
	return missing
}
```

- [ ] **Step 4: Widen `ScopeString`** in `internal/whoopsync/oauth.go`

```go
	ScopeString = "read:recovery read:cycles read:sleep read:profile offline"
```

Extend the existing doc comment above it to say that `read:sleep` was added
after launch, that a refresh cannot widen a grant so connected users must
re-consent, and to point at `RequiredScopes`/`MissingScopes` as the machinery
that makes that state visible instead of silent.

- [ ] **Step 5: Run tests** — `go test ./internal/whoopsync/`. Fix any existing
  test that asserted the old scope string.

- [ ] **Step 6: Commit** — `feat(whoop): request read:sleep and model required scopes`

---

## Task A2: Surface `missing_scopes` on both read surfaces

**Files:**
- Modify: `internal/whoopsync/handler.go` (`connectionResponse`, `getConnection`, `callback`)
- Modify: `internal/whoopsync/handler_test.go`
- Modify: `internal/whoopadmin/handler.go` (`connectionView`, `buildView`)
- Modify: `internal/whoopadmin/handler_test.go`
- Modify: `internal/whoopadmin/connections_gauge.go`
- Modify: `internal/whoopadmin/connections_gauge_test.go`

- [ ] **Step 1: Write the failing tests**

  - `handler_test.go`: `GET /me/whoop/connection` for a connection whose
    `Scopes` omits `read:sleep` returns `missing_scopes: ["read:sleep"]`; a
    fully-scoped connection returns `missing_scopes: []` (**not** absent — the
    client branches on length, and an omitted key would read as "fine" on an
    older payload); the `absent` case still returns just `{"status":"absent"}`.
  - `whoopadmin/handler_test.go`: `connectionView` carries `missing_scopes`,
    non-nil so the JSON is `[]` rather than `null`.
  - `connections_gauge_test.go`: with two connected connections, one
    under-scoped, and one revoked-and-under-scoped, the new gauge reads **1** —
    revoked rows do not count, matching the existing liveness-gauge reasoning.

- [ ] **Step 2: Run them and watch them fail.**

- [ ] **Step 3: Implement.**

`internal/whoopsync/handler.go` — add to `connectionResponse`:

```go
	// MissingScopes are the RequiredScopes this connection did not consent to.
	// Non-omitempty and always non-nil for a real connection so the client can
	// branch on length: an omitted key is indistinguishable from "nothing
	// missing" and would silently hide the reconnect affordance.
	MissingScopes []string `json:"missing_scopes,omitempty"`
```

Keep `omitempty` **only** because the `absent` branch must stay
`{"status":"absent"}`; in `getConnection`, build it as:

```go
	missing := MissingScopes(conn.Scopes)
	if missing == nil {
		missing = []string{}
	}
```

so a connected-and-fully-scoped connection serializes `"missing_scopes":[]`.
(An empty non-nil slice is not omitted by `encoding/json`'s `omitempty` for
slices — verify this in the test rather than assuming.) If `omitempty` does
drop it, remove `omitempty` and give the `absent` branch its own response type
or an explicit nil.

Do the same in the `callback` success payload.

`internal/whoopadmin/handler.go` — add to `connectionView`:

```go
	MissingScopes []string `json:"missing_scopes"` // never null; [] when fully scoped
```

populated in `buildView` from `whoopsync.MissingScopes(conn.Scopes)`, normalized
to a non-nil empty slice.

`internal/whoopadmin/connections_gauge.go` — add:

```go
// api_whoop_connections_missing_scope counts CONNECTED connections that did
// not consent to every scope ingestion needs. This is what turns "users are
// silently not getting sleep" from an invisible state into a dashboard number,
// and it generalizes to every future scope addition: widen RequiredScopes and
// this gauge starts reporting the backlog without further work.
//
// Only CONNECTED rows count — a revoked connection is not ingesting anything,
// so counting it would report a reconnect backlog that does not exist. Same
// reasoning as lastWindowSyncGauge above.
var missingScopeGauge = prometheus.NewGauge(...)
```

Register it in the existing `init()` and set it in `refresh()` from
`len(whoopsync.MissingScopes(c.Scopes)) > 0`. Note the import direction:
`whoopadmin` already imports `whoopsync`, so this adds no cycle.

- [ ] **Step 4: Run tests** — `go test ./internal/whoopsync/ ./internal/whoopadmin/`
- [ ] **Step 5: Commit** — `feat(whoop): expose missing_scopes on the connection read surfaces`

---

## Task A3: Migration 058 and the `whoopsleep` package

**Files:**
- Create: `internal/db/migrations/058_user_whoop_sleep.sql`
- Create: `internal/whoopsleep/sleep.go`
- Create: `internal/whoopsleep/repository.go`
- Create: `internal/whoopsleep/sqlite_repository.go`
- Create: `internal/whoopsleep/repository_test.go`

Mirror `internal/whooprecovery/` file-for-file. Read those four files first —
this task is deliberately shaped like them.

- [ ] **Step 1: Write the migration** — `internal/db/migrations/058_user_whoop_sleep.sql`

Use the SOW's SQL verbatim (SOW → "Data Model"). Prefix it with a comment block
explaining the two departures from `040_user_whoop_recovery.sql`:

1. **One row per WHOOP sleep record, not per (user, date).** WHOOP emits a
   separate record with `nap: true` for daytime sleep, so `(user_id, date)` is
   genuinely not unique. Uniqueness is `(user_id, whoop_sleep_id)`.
2. **Durations stored in milliseconds exactly as WHOOP sends them.** Storing
   the wire value keeps ingest dumb and reversible; presentation rounding is the
   tile's job.

Also note that every score column is nullable because a `PENDING` record has a
start, an end, and nothing else — and it is still stored, so the row exists when
the score arrives.

Match the house column types: `created_at`/`updated_at` are `DATETIME NOT NULL`
(as in 040), not `TEXT`, so `database/sql` scans them into `time.Time`.

- [ ] **Step 2: Write the failing repository test** — `internal/whoopsleep/repository_test.go`

Follow `internal/whooprecovery/repository_test.go`'s harness (it uses
`internal/db/dbtest` — read it and reuse the same helper). Cover:

- `Upsert` inserts; a second `Upsert` with the same `(user_id, whoop_sleep_id)`
  updates in place — same row id, `created_at` preserved, `updated_at` advanced,
  metrics replaced.
- Two records for the same user on the same `date` (one `is_nap` true) **both**
  persist — the nap case the whole keying decision exists for.
- `ListRange` honours inclusive `since`/`until`, either bound `""` for
  unbounded, ordered `date DESC, ended_at DESC`.
- Nullable score fields round-trip as `nil` when absent.
- `DeleteByWhoopSleepID` removes exactly the matching row, is a no-op for an
  unknown UUID, and never touches another user's row with the same UUID.
- `DeleteForUser` removes all of one user's rows and none of another's.
- `Latest` returns the newest row, `(nil, nil)` when the user has none.
- `CountForUser` counts.

- [ ] **Step 3: Run and watch it fail.**

- [ ] **Step 4: Implement `sleep.go`**

```go
// Package whoopsleep stores WHOOP sleep records, one row per WHOOP sleep
// record rather than one per (user, day).
//
// This is the deliberate departure from internal/whooprecovery (and the
// steps-shaped table pattern generally): WHOOP emits a separate record with
// nap=true for daytime sleep, so (user_id, date) is genuinely not unique.
// Keying by WHOOP's own UUID also makes a sleep.deleted webhook a direct
// delete rather than a date-derivation round trip.
//
// Durations are milliseconds exactly as WHOOP sent them — the ingest layer
// does no unit conversion, so a wire value is always recoverable and
// presentation rounding stays the tile's job.
package whoopsleep

import "time"

// Entry is one WHOOP sleep record. Date is the YYYY-MM-DD local WAKE date
// (derived from `end` localized by TimezoneOffset — see the SOW's "Dating a
// Night"). Every score field is a pointer because a PENDING or UNSCORABLE
// record carries a start, an end, and nothing else, and is still stored so the
// row exists when the score arrives.
type Entry struct {
	ID             string
	UserID         string
	WhoopSleepID   string
	Date           string // YYYY-MM-DD local wake date
	IsNap          bool
	StartedAt      string // RFC3339, as WHOOP sent it
	EndedAt        string // RFC3339, as WHOOP sent it
	TimezoneOffset string // e.g. "-06:00", as WHOOP sent it
	ScoreState     string // SCORED | PENDING | UNSCORABLE

	InBedMilli         *int64
	AwakeMilli         *int64
	NoDataMilli        *int64
	LightSleepMilli    *int64
	SlowWaveSleepMilli *int64
	REMSleepMilli      *int64
	SleepCycleCount    *int64
	DisturbanceCount   *int64

	NeedBaselineMilli       *int64
	NeedFromSleepDebtMilli  *int64
	NeedFromStrainMilli     *int64
	NeedFromNapMilli        *int64

	RespiratoryRate *float64
	PerformancePct  *float64
	ConsistencyPct  *float64
	EfficiencyPct   *float64

	CreatedAt time.Time
	UpdatedAt time.Time
}
```

`StartedAt`/`EndedAt` stay strings — they are stored as WHOOP's RFC3339 text and
the only consumer that needs ordering compares them lexicographically, which
RFC3339 in a fixed zone supports. Do not parse-and-reformat at the storage
boundary; that would silently normalize away WHOOP's own representation.

- [ ] **Step 5: Implement `repository.go`** — the interface, doc-commented per
  method in `whooprecovery/repository.go`'s style:

```go
type Repository interface {
	Upsert(ctx context.Context, e Entry, now time.Time) error
	ListRange(ctx context.Context, userID, since, until string) ([]Entry, error)
	Latest(ctx context.Context, userID string) (*Entry, error)
	CountForUser(ctx context.Context, userID string) (int, error)
	DeleteByWhoopSleepID(ctx context.Context, userID, whoopSleepID string) error
	DeleteForUser(ctx context.Context, userID string) error
}
```

- [ ] **Step 6: Implement `sqlite_repository.go`**

`Upsert` uses `ON CONFLICT(user_id, whoop_sleep_id) DO UPDATE SET …` touching
every mutable column plus `updated_at`, preserving `id` and `created_at` —
exactly the shape of `whooprecovery`'s. Use `internal/id`.New() for the insert
path. Add `nullInt` alongside a copied `nullFloat` helper. `ListRange` orders
`date DESC, ended_at DESC`.

- [ ] **Step 7: Run tests** — `go test ./internal/whoopsleep/ ./internal/db/`
- [ ] **Step 8: Commit** — `feat(whoop): add the user_whoop_sleep table and repository`

---

## Task A4: WHOOP sleep client method

**Files:**
- Modify: `internal/whoopsync/client.go`
- Modify: `internal/whoopsync/client_test.go`

- [ ] **Step 1: Write the failing test.** Mirror the existing `Recoveries` /
  `Cycles` tests exactly (they repoint `whoopAPIBase` at an `httptest.Server`).
  Cover: the request path is `/v2/activity/sleep`; `start`/`end` are RFC3339 UTC
  and `limit` is set; `nextToken` paging accumulates across two pages and stops
  when `next_token` is empty; the `maxPages` cap holds against a server that
  always returns a token; a full documented record decodes into every struct
  field (assert `SlowWaveSleepMilli`, `NeedFromStrainMilli`,
  `SleepPerformancePercentage` specifically — a mis-transcribed JSON tag is the
  failure mode Open Question 3 names); a `PENDING` record decodes with
  `Score == nil`; 429 → `ErrRateLimited`; 401 → `ErrTokenRejected`; **403 →
  a generic error carrying the status** (there is no dedicated sentinel — see
  Task A5's scope gate for why we never expect to see one).

- [ ] **Step 2: Run and watch it fail.**

- [ ] **Step 3: Implement** in `internal/whoopsync/client.go`, next to `Cycle`:

```go
// SleepStageSummary is WHOOP's per-stage duration breakdown for a scored
// sleep. Every field is milliseconds as WHOOP sends them; nothing is converted
// at this boundary.
type SleepStageSummary struct {
	TotalInBedTimeMilli        *int64 `json:"total_in_bed_time_milli"`
	TotalAwakeTimeMilli        *int64 `json:"total_awake_time_milli"`
	TotalNoDataTimeMilli       *int64 `json:"total_no_data_time_milli"`
	TotalLightSleepTimeMilli   *int64 `json:"total_light_sleep_time_milli"`
	TotalSlowWaveSleepTimeMilli *int64 `json:"total_slow_wave_sleep_time_milli"`
	TotalRemSleepTimeMilli     *int64 `json:"total_rem_sleep_time_milli"`
	SleepCycleCount            *int64 `json:"sleep_cycle_count"`
	DisturbanceCount           *int64 `json:"disturbance_count"`
}

// SleepNeeded is WHOOP's computed sleep need and its components. Prog Strength
// stores these as WHOOP computes them and derives no sleep model of its own.
type SleepNeeded struct {
	BaselineMilli           *int64 `json:"baseline_milli"`
	NeedFromSleepDebtMilli  *int64 `json:"need_from_sleep_debt_milli"`
	NeedFromRecentStrainMilli *int64 `json:"need_from_recent_strain_milli"`
	NeedFromRecentNapMilli  *int64 `json:"need_from_recent_nap_milli"`
}

// SleepScore is the scored portion of a sleep record, present only when
// ScoreState is "SCORED".
type SleepScore struct {
	StageSummary               *SleepStageSummary `json:"stage_summary"`
	SleepNeeded                *SleepNeeded       `json:"sleep_needed"`
	RespiratoryRate            *float64           `json:"respiratory_rate"`
	SleepPerformancePercentage *float64           `json:"sleep_performance_percentage"`
	SleepConsistencyPercentage *float64           `json:"sleep_consistency_percentage"`
	SleepEfficiencyPercentage  *float64           `json:"sleep_efficiency_percentage"`
}

// Sleep is a WHOOP v2 sleep record. Unlike Recovery it carries its own
// timezone_offset, which is why the sleep sync path needs one endpoint where
// recovery needs two (see the SOW's "Dating a Night").
type Sleep struct {
	ID             string      `json:"id"`              // v2 UUID; the webhook delete key
	Start          string      `json:"start"`           // RFC3339
	End            string      `json:"end"`             // RFC3339
	TimezoneOffset string      `json:"timezone_offset"` // e.g. "-06:00"
	Nap            bool        `json:"nap"`
	ScoreState     string      `json:"score_state"` // SCORED | PENDING | UNSCORABLE
	Score          *SleepScore `json:"score"`       // absent when not SCORED
}

// sleepEnvelope is the v2 paginated list wrapper for sleeps.
type sleepEnvelope struct {
	Records   []Sleep `json:"records"`
	NextToken string  `json:"next_token"`
}
```

and `Sleeps` as a copy of `Cycles` against `/v2/activity/sleep`.

> **Note for the implementer:** these JSON tags are transcribed from WHOOP's
> published v2 schema, not from a captured live response — SOW Open Question 3.
> Do not "tidy" them. A single wrong tag silently produces a column of nulls
> that looks like "WHOOP didn't score it".

- [ ] **Step 4: Run tests, gofmt.** `gofmt` will realign the struct tags — let it.
- [ ] **Step 5: Commit** — `feat(whoop): add the v2 sleep list client method`

---

## Task A5: The independent sleep sync path

**Files:**
- Modify: `internal/whoopsync/service.go`
- Modify: `internal/whoopsync/metrics.go`
- Modify: `internal/whoopsync/service_test.go`
- Modify: `internal/whoopsync/observability_test.go`
- Modify: `internal/whoopadmin/handler.go` (`resyncOutcome` gains the sleep counts)
- Modify: `internal/server/server.go` (pass the sleep repo to `NewService`)

This is the load-bearing task. Read `service.go` end to end before starting.

**Shape:**

- `SyncResult` gains a nested sleep block so an operator resync reports both
  domains without the caller guessing which counter is which:

```go
type SyncResult struct {
	Upserted        int
	SkippedUnscored int
	SkippedNoCycle  int
	SkippedBadDate  int

	// Sleep is the sleep domain's outcome for the same window. Separate rather
	// than summed: the two domains fail independently, so a single set of
	// counters could not express "recovery landed, sleep 403'd".
	Sleep SleepSyncResult
}

// SleepSyncResult reports the sleep domain's outcome. There is deliberately no
// SkippedNoCycle: a sleep record carries its own timezone_offset, so the sleep
// path never joins to a cycle and that failure mode cannot occur. Err carries
// the sleep-side failure when the sleep path failed but the overall sync did
// not — the isolation the SOW requires.
type SleepSyncResult struct {
	Upserted        int
	SkippedUnscored int
	SkippedBadDate  int
	SkippedNoScope  bool
	Err             error
}
```

- `Service` gains a `sleep whoopsleep.Repository` field and `NewService` a
  parameter for it (positioned after `rec`). Update the one production call site
  in `server.go` and every test constructor.
- `whoopAPI` gains `Sleeps(ctx, accessToken string, start, end time.Time, limit int) ([]Sleep, error)`.
- `syncWindow` is **renamed nothing** and left structurally unchanged; extract
  the token acquisition so both paths share it, or simply let `syncSleepWindow`
  call `validToken` itself — the per-user keyed mutex makes that safe and it is
  the simpler change. Prefer the simpler change.

- [ ] **Step 1: Write the failing tests** in `service_test.go` (extend the
  existing fake client with a `Sleeps` func and a **call log**):

  1. **Happy path:** a SCORED sleep record upserts one `whoopsleep.Entry` with
     the date derived from `End` + `TimezoneOffset` (not `Start`), `IsNap` false,
     and every millisecond field carried through unconverted.
  2. **A night crossing midnight lands on the WAKE date.** `start`
     `2026-03-02T22:40:00-06:00`, `end` `2026-03-03T06:15:00-06:00`, offset
     `-06:00` → `2026-03-03`. This is the regression migration 041 exists to
     commemorate; give it its own named test.
  3. **A night crossing a DST boundary** uses the offset WHOOP sent (assert two
     records whose offsets differ by an hour land on the dates their own offsets
     imply).
  4. **A nap at 14:00 lands on its own day** and is stored with `IsNap` true.
  5. **PENDING / UNSCORABLE records are still upserted** with nil score fields
     and counted under `SkippedUnscored`… — **decide and pin the semantics
     here:** the SOW says a PENDING record "should still be stored so the row
     exists when the score arrives", so it is stored AND counted as
     `skipped_unscored` (the counter means "no score to store", not "no row
     written"). Say so in a comment; the two readings are otherwise a coin flip
     for the next reader.
  6. **A record with an unparseable `timezone_offset` or `end`** is skipped with
     `skipped_bad_date`, warned, and does not fail the sync.
  7. **Isolation, 403:** the fake `Sleeps` returns a generic 403 error. Assert
     recovery rows ARE upserted, `syncWindow`'s error is nil, `SyncResult.Sleep.Err`
     is non-nil, and `syncsTotal{domain="sleep",result="error"}` incremented
     while `{domain="recovery",result="ok"}` did too.
  8. **Isolation, 429:** same, with `ErrRateLimited`.
  9. **A recovery failure retains current behavior** — returns the error, and
     the sleep path is not even attempted (assert the call log has no `Sleeps`).
  10. **Under-scoped connection:** the connection's `Scopes` omits `read:sleep`;
      assert the fake client's call log contains **no** `Sleeps` call at all,
      `SyncResult.Sleep.SkippedNoScope` is true, and
      `sleepRowsTotal{disposition="skipped_no_scope"}` incremented.
  11. **`MarkWindowSync` is still advanced only by `kind == "window"`**, and is
      still advanced when the sleep path failed but recovery succeeded.

- [ ] **Step 2: Run and watch them fail.**

- [ ] **Step 3: Implement `syncSleepWindow`** — a sibling of `syncWindow`, not a
  branch inside it:

```go
// syncSleepWindow fetches sleep records for [start, end] and upserts one row
// per WHOOP sleep record, dated by its END (wake time) localized by the
// record's own timezone_offset.
//
// It is a SIBLING of syncWindow rather than a branch inside it because the two
// domains must fail independently: an under-scoped or degraded sleep fetch can
// never take recovery ingestion down with it. Both go through the same
// validToken, so the per-user keyed mutex still serializes WHOOP's single-use
// refresh-token rotation correctly across both.
func (s *Service) syncSleepWindow(ctx context.Context, kind, userID string, start, end time.Time, limit int) SleepSyncResult
```

The scope gate runs FIRST, before `validToken`, reading the connection the
caller already loaded — or re-reading it; a second `conns.Get` is cheap and
keeps the signature narrow:

```go
	// Skip the sleep path entirely for a connection that never consented to
	// read:sleep. Calling anyway would 403 on every sync for every under-scoped
	// user, burning rate limit and filling the error metric with a condition
	// that is not an error — it is a user who has not reconnected yet.
	if slices.Contains(MissingScopes(conn.Scopes), "read:sleep") {
		sleepRowsTotal.WithLabelValues("skipped_no_scope").Inc()
		return SleepSyncResult{SkippedNoScope: true}
	}
```

Then fetch, derive each date with the EXISTING `deriveDate(r.End, r.TimezoneOffset)`
(SOW: "Reuse it; do not write a second one"), map to a `whoopsleep.Entry`, and
upsert.

- [ ] **Step 4: Join the outcomes in the three callers.** `SyncWindow`,
  `Backfill`, and `SyncSince` each call `syncWindow` then `syncSleepWindow`, in
  that order, and join such that:

```go
	// A sleep failure is logged, counted, and returned in the SyncResult, but
	// does NOT fail the overall sync when recovery succeeded. The webhook
	// consequently does not return 500 for a sleep-only failure: a 500 makes
	// WHOOP redeliver recovery data we already stored successfully — the same
	// reasoning already recorded for MarkWindowSync above.
```

A **recovery** failure short-circuits before the sleep path and returns exactly
as it does today.

- [ ] **Step 5: Metrics** — in `metrics.go`:
  - `syncsTotal` gains a `domain` label (`recovery` | `sleep`) as its FIRST
    label, ahead of `kind`. Update every `WithLabelValues` call site and the
    existing observability test. Extend the doc comment to explain that a
    parallel counter was rejected: one series with a domain label keeps every
    existing dashboard query working with a label selector rather than a rename.
  - Add `sleepRowsTotal` (`api_whoop_sleep_rows_total`, label `disposition`)
    with dispositions `upserted`, `skipped_unscored`, `skipped_bad_date`,
    `skipped_no_scope`. Doc-comment that there is deliberately **no**
    `skipped_no_cycle` for sleep because it cannot occur.

- [ ] **Step 6: The summary log line.** Add the sleep counts to the existing
  `whoopsync: sync complete` line (`sleeps_fetched`, `sleep_upserted`,
  `sleep_skipped_unscored`, `sleep_skipped_bad_date`, `sleep_skipped_no_scope`,
  and `sleep_error` when non-nil) so the one-line "did this user's data land"
  answer covers both domains.

- [ ] **Step 7: `whoopadmin`'s `resyncOutcome`** gains a nested `sleep` object
  with the same four counts plus `skipped_no_scope`, and its handler test is
  extended. `SyncSince`'s signature is unchanged.

- [ ] **Step 8: Run** `go test ./internal/whoopsync/ ./internal/whoopadmin/ ./internal/server/`
- [ ] **Step 9: Commit** — `feat(whoop): sync sleep on a path that fails independently of recovery`

---

## Task A6: `sleep.updated` / `sleep.deleted` webhooks

**Files:**
- Modify: `internal/whoopsync/webhook.go`
- Modify: `internal/whoopsync/webhook_test.go`
- Modify: `internal/server/server.go` (pass the sleep repo to `NewWebhookHandler`)

- [ ] **Step 1: Write the failing tests** in `webhook_test.go`, alongside the
  existing `recovery.updated` / `recovery.deleted` cases:
  - `sleep.updated` triggers a window sync (the fake syncer records the call) and
    returns 204.
  - `sleep.updated` whose sync fails returns 500 (so WHOOP retries) and counts
    `sync_error`.
  - `sleep.deleted` calls `DeleteByWhoopSleepID(userID, event.ID)` and returns 204.
  - `sleep.deleted` for an unknown UUID is a no-op 204 (the repository's
    idempotent delete), and a redelivery of the same event is also 204.
  - Both are still gated by the existing HMAC check, the unknown-user route, and
    the not-connected gate — assert one of each for a sleep event so the gates
    are pinned as applying to the new types too.
  - An unhandled type (e.g. `workout.updated`) still falls through to `ignored`.

- [ ] **Step 2: Run and watch them fail.**

- [ ] **Step 3: Implement.** `WebhookHandler` gains a `sleep whoopsleep.Repository`
  field and constructor parameter. Two cases join the switch:

```go
	case "sleep.updated":
		// Poke, not payload — re-sync the recent window, same as
		// recovery.updated. Idempotent upserts make redelivery, duplicates, and
		// out-of-order arrival safe. Note this is the SAME SyncWindow call the
		// recovery case makes: one sync covers both domains, which is why sleep
		// gets no liveness stamp of its own.
	case "sleep.deleted":
		// The event id IS the sleep UUID — the first webhook where that id maps
		// to a record we actually own. A WHOOP-initiated delete is a data
		// correction, so the row goes; a user disconnect is an account action
		// and leaves ingested rows alone.
```

- [ ] **Step 4: Run tests.**
- [ ] **Step 5: Commit** — `feat(whoop): handle sleep.updated and sleep.deleted webhooks`

---

## Task A7: `GET /whoop/sleep`

**Files:**
- Create: `internal/whoopsync/sleep_read_handler.go`
- Create: `internal/whoopsync/sleep_read_handler_test.go`
- Modify: `internal/whoopsync/handler.go` (`Handler` gains `sleep`; `MountAuthed`)
- Modify: `internal/server/server.go`
- Modify: `internal/server/server_test.go` (route-presence contract, ~line 78)

- [ ] **Step 1: Write the failing tests.** Mirror `read_handler_test.go`:
  - missing `timezone` → 400 with the `daterange` message ("timezone is required")
  - unknown IANA name → 400
  - malformed `since` / `until` → 400 "invalid since (expected YYYY-MM-DD)"
  - a valid call returns `{"sleep":[…]}` most-recent-first with `is_nap` present
    on every object and every duration in **milliseconds**
  - a user with no rows returns `{"sleep":[]}` (an empty array, never null)
  - nullable score fields serialize as JSON `null`, not `0`
  - `server_test.go` gains `"GET /whoop/sleep"` to the `want` list.

- [ ] **Step 2: Run and watch them fail.**

- [ ] **Step 3: Implement `sleep_read_handler.go`.** Define `sleepDTO` explicitly
  (do NOT serialize `whoopsleep.Entry`) so the snake_case JSON is the API's
  contract independent of the repo struct:

```go
type sleepDTO struct {
	WhoopSleepID   string `json:"whoop_sleep_id"`
	Date           string `json:"date"`
	IsNap          bool   `json:"is_nap"`
	StartedAt      string `json:"started_at"`
	EndedAt        string `json:"ended_at"`
	TimezoneOffset string `json:"timezone_offset"`
	ScoreState     string `json:"score_state"`

	InBedMilli         *int64 `json:"in_bed_milli"`
	AwakeMilli         *int64 `json:"awake_milli"`
	NoDataMilli        *int64 `json:"no_data_milli"`
	LightSleepMilli    *int64 `json:"light_sleep_milli"`
	SlowWaveSleepMilli *int64 `json:"slow_wave_sleep_milli"`
	RemSleepMilli      *int64 `json:"rem_sleep_milli"`
	SleepCycleCount    *int64 `json:"sleep_cycle_count"`
	DisturbanceCount   *int64 `json:"disturbance_count"`

	NeedBaselineMilli      *int64 `json:"need_baseline_milli"`
	NeedFromSleepDebtMilli *int64 `json:"need_from_sleep_debt_milli"`
	NeedFromStrainMilli    *int64 `json:"need_from_strain_milli"`
	NeedFromNapMilli       *int64 `json:"need_from_nap_milli"`

	RespiratoryRate *float64 `json:"respiratory_rate"`
	PerformancePct  *float64 `json:"performance_pct"`
	ConsistencyPct  *float64 `json:"consistency_pct"`
	EfficiencyPct   *float64 `json:"efficiency_pct"`

	CreatedAt time.Time `json:"created_at"`
	UpdatedAt time.Time `json:"updated_at"`
}

type sleepListDTO struct {
	Sleep []sleepDTO `json:"sleep"`
}
```

**On `timezone`:** the SOW asks for a required IANA `timezone` per the house
date-window convention. Sleep rows are already keyed by local wake date, so
`since`/`until` compare lexicographically and no UTC bounds are constructed —
exactly as `GET /whoop/recovery` does. Validate the name with
`daterange.LoadTimezone` and **require** it (unlike recovery, which accepts and
ignores it), and say in the doc comment WHY validating-but-not-converting is the
right shape here: the client must not learn that this endpoint would work
without a timezone, because the day-boundary contract is the API's to own and a
future re-dating would need it.

- [ ] **Step 4: Mount** `r.Get("/whoop/sleep", h.getSleep)` in `MountAuthed`,
  wire the repo through `NewHandler` in `server.go`, and update the log line at
  `server.go:591` to mention sleep reads.
- [ ] **Step 5: Run** `go test ./internal/whoopsync/ ./internal/server/`
- [ ] **Step 6: Commit** — `feat(whoop): add the GET /whoop/sleep read endpoint`

---

## Task A8: The `sleep` dashboard section and tile id

**Files:**
- Modify: `internal/dashboard/tiles.go`, `tiles_test.go`
- Modify: `internal/dashboard/dto.go`
- Create: `internal/dashboard/sleep.go`, `internal/dashboard/sleep_test.go`
- Modify: `internal/dashboard/handler.go`
- Modify: `internal/dashboard/summary_layout_test.go`
- Modify: `internal/server/server.go`

- [ ] **Step 1: Write the failing tests.**

  - `tiles_test.go`: add `TileSleep` to BOTH the `all` list and the `Order`
    want-list, **after** the recovery family (i.e. after `TileRecoveryLog`,
    before `TileStreak`).
  - `sleep_test.go` — **night selection** (`pickNight`), its own function with
    its own test per the SOW:
    - a single non-nap record for a date is the night
    - nap records are never selected
    - two non-nap records on one date → the one with the **longest in-bed
      duration** wins
    - equal in-bed durations → the one with the **latest `ended_at`** wins
    - a date with only naps, or no records, → no night
    - a record with a nil `InBedMilli` loses to one that has it, and two nil
      records fall through to the `ended_at` tie-break (state and pin this — an
      unscored record has no in-bed duration at all)
  - `sleep_test.go` — **payload shape**: `Nights` is date-aligned over the
    trailing window, oldest→newest, EVERY date present, missing nights carrying
    null metrics — never omitted, never zero-filled. Assert an interior gap
    explicitly. `LastNight` is nil when today has no non-nap record.
  - `summary_layout_test.go`: a layout containing `sleep` yields a `sleep` key
    for a connected user and NO `sleep` key for a user with no connection.

- [ ] **Step 2: Run and watch them fail.**

- [ ] **Step 3: Implement.**

`tiles.go` — add after `TileRecoveryLog`:

```go
	// TileSleep is the sleep tile. It is its own section rather than a member
	// of the recovery family's one-section-many-tiles arrangement because it is
	// a single tile reading a single section. The key is `sleep` (not
	// `sleep_tile`) so a later sleep tile SPREAD can expand into this section
	// with no retirement mapping.
	TileSleep TileID = "sleep"
```

and to `Catalog`, in the same position.

`dto.go`:

```go
// SleepSection is the sleep tile. nil at the Summary level unless a connected
// Whoop connection exists — the same gate the recovery family uses.
type SleepSection struct {
	// LastNight is today's main sleep: nil when there is no non-nap record for
	// today. Deliberately not "the most recent night whenever it was" —
	// promoting a two-day-old night into today's slot is the bug the recovery
	// tile's no-reading branch exists to avoid.
	LastNight *SleepNight `json:"last_night"`
	// Nights is the date-aligned trailing window, oldest→newest: every date in
	// the window present, missing nights carrying null metrics. A night with no
	// data and a night of zero sleep are different facts.
	Nights []SleepNight `json:"nights"`
}

// SleepNight is one local date's main sleep. Date is always populated; every
// other field is nullable, because a date in the window with no record — or a
// record WHOOP has not scored — is represented, not omitted.
type SleepNight struct {
	Date string `json:"date"`
	// … the millisecond stage fields, need fields, respiratory rate, and the
	// three percentages, all *int64 / *float64 with snake_case tags matching
	// the sleepDTO names in Task A7.
	InBedMilli *int64 `json:"in_bed_milli"`
	// …
}
```

Add `Sleep *SleepSection \`json:"sleep,omitempty"\`` to `Summary`, documented
like `Recovery`.

`sleep.go` — `buildSleep(entries []whoopsleep.Entry, now time.Time, loc *time.Location) *SleepSection`,
pure (no `time.Now`, no DB), plus:

```go
// pickNight returns THE night for a local date: the non-nap record, or — when
// more than one non-nap record shares a date (a fragmented night WHOOP split,
// or travel across a date line) — the one with the longest in-bed duration,
// tie-broken by latest ended_at.
//
// It lives in one function with its own test rather than open-coded at call
// sites precisely because the tie-break is the part a second implementation
// would get subtly different.
func pickNight(records []whoopsleep.Entry) *whoopsleep.Entry
```

Window width: `sleepWindowDays = 30` (a named constant with a comment tying it
to the SOW's "reconnecting backfills the trailing ~30 nights so the tile is
non-empty immediately").

`handler.go` — a `buildSleepSection` mirroring `buildRecoverySection`: gate on a
CONNECTED connection (nil section otherwise), one `defer1`-wrapped `ListRange`
over the trailing window, then `buildSleep`. Emit under `out[string(TileSleep)]`
when `enabled[TileSleep]`. Add the `whoopSleep whoopsleep.Repository` field and
constructor parameter; update `server.go` and every dashboard test constructor.

Sleep is **not** added to `defaultLayout` — the SOW's rollout adds the tile by
hand at step 5.

- [ ] **Step 4: Run** `go test ./...`
- [ ] **Step 5: Commit** — `feat(dashboard): add the sleep tile section`

---

## Task A9: API gate

- [ ] `gofmt -l .` clean, `go vet ./...`, `go build ./...`
- [ ] `golangci-lint run --timeout=5m` (v2.12.2, the CI-pinned version) clean —
  **no `//nolint`, no rule disables.** `gosec` and `shadow` are on.
- [ ] `go mod tidy` produces no `go.mod` / `go.sum` diff
- [ ] `go test ./...` green
- [ ] `go test -race ./internal/whoopsync/` green (the keyed mutex is on this path)

---

## Task W1: Catalog entry + raw API types (`prog-strength-web`)

**Files:**
- Modify: `lib/dashboard-tiles.ts`, `lib/dashboard-tiles.test.ts`
- Modify: `lib/api.ts`

- [ ] **Step 1: Write the failing test.** In `lib/dashboard-tiles.test.ts`:
  bump `has exactly 19 tiles` → 20, and add `"sleep"` to the order array
  **after `"recovery_log"`, before `"streak"`** — the Go catalog contract test
  asserts identical id set AND order, so the two move together.
- [ ] **Step 2: Run `npm run test -- dashboard-tiles` and watch it fail.**
- [ ] **Step 3: Implement.** Add `"sleep"` to the `TileId` union in catalog
  position, and the catalog entry:

```ts
  {
    id: "sleep",
    title: "Sleep",
    href: "/recovery",
    description: "Last night's stages and how it measured against your need.",
  },
```

`href` is `/recovery` deliberately: the SOW's non-goals rule out a Sleep page
until there is history worth charting, and the tile deep-links to the recovery
page in the interim. Leave a comment saying so, otherwise it reads as a typo.

- [ ] **Step 4: Add the raw payload types** to `lib/api.ts`, next to
  `DashboardRecovery`, using the exact snake_case keys from Task A8's
  `SleepNight`, plus:

```ts
export type DashboardSleep = {
  last_night: DashboardSleepNight | null;
  nights: DashboardSleepNight[];
};
```

and `sleep?: DashboardSleep | null;` on `DashboardSummary`.

- [ ] **Step 5: Extend `WhoopConnection`:**

```ts
export type WhoopConnection = {
  status: "connected" | "revoked" | "error" | "absent";
  connected_at?: string;
  /**
   * Scopes the connection did not consent to. Non-empty means CONNECTED BUT
   * UNDER-SCOPED: the tokens are valid and recovery still syncs, but the paths
   * needing the absent scope are skipped. Optional because an older API build
   * omits the key entirely; absent reads as "nothing missing".
   */
  missing_scopes?: string[];
};
```

- [ ] **Step 6: Run** `npm run test`, `npm run typecheck`
- [ ] **Step 7: Commit** — `feat(dashboard): add the sleep tile to the catalog`

---

## Task W2: The under-scoped state on Settings → Integrations

**Files:**
- Modify: `app/(app)/settings/_components/whoop-connection-row.tsx`
- Modify: `app/(app)/settings/_components/whoop-connection-row.test.tsx`

- [ ] **Step 1: Write the failing tests.**
  - `status: "connected"` with `missing_scopes: ["read:sleep"]` renders the
    under-scoped copy AND a **Reconnect** button (not Disconnect).
  - `status: "connected"` with `missing_scopes: []` (and with the key absent)
    renders today's connected copy and the Disconnect button — unchanged.
  - `status: "error"` still renders its own attention copy and Reconnect, and is
    NOT overridden by the new branch.
  - Clicking Reconnect in the under-scoped state navigates to the same
    `/auth/whoop/connect?return_to=…` URL the existing Reconnect uses — this is
    a new *reason* to show the existing flow, not a new flow.

- [ ] **Step 2: Run and watch them fail.**

- [ ] **Step 3: Implement.** A third state between connected and error:

```tsx
  const underScoped = connected && (conn?.missing_scopes?.length ?? 0) > 0;
```

Copy names what is missing in **product terms**, never in scope terms —
`read:sleep` is not a user-facing noun:

> "Reconnect to enable sleep tracking — your Whoop connection predates it."

Render the Reconnect button (the same `connect()` navigation) instead of
Disconnect while under-scoped, so the affordance the user needs is the one they
see. Order the branches `underScoped → connected → errored → default` and leave
a comment saying the under-scoped test must come first, because it is a REFINEMENT
of `connected` rather than a sibling status — folding it into the status enum is
what the SOW explicitly forbids.

- [ ] **Step 4: Run tests.**
- [ ] **Step 5: Commit** — `feat(settings): surface the under-scoped whoop connection state`

---

## Task W3: Sleep view model + adapter

**Files:**
- Modify: `lib/dashboard.ts`
- Modify: `lib/dashboard.test.ts` (or the existing adapter test file — find it first)

- [ ] **Step 1: Write the failing test.** `adaptDashboard` maps a `sleep`
  payload to `{ present: true, … }` and a missing/null one to
  `{ present: false }`; nulls are preserved as null (never coerced to 0); the
  `nights` array keeps its gaps as null-metric entries and its oldest→newest
  order.
- [ ] **Step 2: Run and watch it fail.**
- [ ] **Step 3: Implement.**

```ts
/** One night in the sleep history. Every metric is nullable: a date in the
 *  window with no record, and a record WHOOP has not scored, are both
 *  represented rather than omitted. Durations are MILLISECONDS as stored —
 *  the tile formats, the adapter does not convert. */
export type SleepNightView = {
  date: string;
  inBedMilli: number | null;
  awakeMilli: number | null;
  lightSleepMilli: number | null;
  slowWaveSleepMilli: number | null;
  remSleepMilli: number | null;
  noDataMilli: number | null;
  sleepCycleCount: number | null;
  disturbanceCount: number | null;
  needBaselineMilli: number | null;
  needFromSleepDebtMilli: number | null;
  needFromStrainMilli: number | null;
  needFromNapMilli: number | null;
  respiratoryRate: number | null;
  performancePct: number | null;
  consistencyPct: number | null;
  efficiencyPct: number | null;
};

export type SleepView = {
  lastNight: SleepNightView | null;
  nights: SleepNightView[];
};
```

plus `sleep: Section<SleepView>` on `DashboardData`, `sleep: { present: false }`
in the null-summary branch, and `adaptSleep`. Keep the adapter a pure rename —
"the server already did the aggregation" is this layer's whole contract.

- [ ] **Step 4: Run** `npm run test`, `npm run typecheck`
- [ ] **Step 5: Commit** — `feat(dashboard): adapt the sleep summary section`

---

## Task W4: The sleep tile

**Files:**
- Create: `app/(app)/dashboard/_components/sleep/shared.ts` (+ test)
- Create: `app/(app)/dashboard/_components/sleep/fixtures.ts`
- Create: `app/(app)/dashboard/_components/sleep/stage-bar.tsx` (+ test)
- Create: `app/(app)/dashboard/_components/sleep/sleep-tile.tsx` (+ test)
- Create: `app/(app)/dashboard/_components/sleep/reconnect-card.tsx`
- Modify: `app/(app)/dashboard/_components/tile-renderer.tsx` (+ test)

Follow the recovery-tile component conventions exactly: co-located `fixtures.ts`,
`shared.ts`, per-component tests, `MiniCard` furniture, CSS vars by name and
never a raw hex.

- [ ] **Step 1: `shared.ts` + its test (pure functions, no React).**

```ts
/**
 * Sleep-stage color and duration formatting, single-sourced for the same
 * reason recovery/shared.ts exists: two hand-rolled copies of a stage palette
 * is exactly how REM ends up a different blue on the bar than in the legend.
 *
 * THE STAGE RAMP. Design-system v0.4 has no four-way categorical set that
 * would read correctly here — the one accent is app chrome and never a data
 * hue, the per-discipline hues would say "sleep is a discipline", and the
 * status colors would say "light sleep is a warning". SOW Open Question 2's
 * recommendation is an ordinal single-hue luminance ramp, which is also the
 * more honest encoding: sleep stages have a natural ordering by depth.
 *
 * So the bar reuses the EXISTING four-stop `--discipline-lift-1..4` ramp,
 * which the design system already names "the canonical encoding of graded
 * intensity … reusable by any future per-intensity graphic". Zero new tokens,
 * so the SOW's `in-system` scope holds. Deep is darkest, awake lightest.
 */
export const STAGE_ORDER = ["slowWave", "light", "rem", "awake"] as const;
export type SleepStage = (typeof STAGE_ORDER)[number];

export function stageColor(stage: SleepStage): string; // → var(--discipline-lift-N)
export function stageLabel(stage: SleepStage): string; // "Deep" | "Light" | "REM" | "Awake"

/** "7h 12m" from milliseconds; "—" for null. Rounds to the nearest minute —
 *  the ms are stored so nothing is lost, and a tile that prints seconds of
 *  sleep is claiming a precision the user does not have. */
export function formatSleepDuration(ms: number | null): string;

/** Asleep time = in-bed minus awake minus no-data, or null when the pieces
 *  aren't all present. NOT a re-derivation of a server figure: WHOOP sends no
 *  "total asleep" field, so this is the one arithmetic the tile must do. */
export function asleepMilli(night: SleepNightView): number | null;

/** Total sleep need = baseline + debt + strain + nap components, or null.
 *  The nap component is legitimately NEGATIVE (a nap discharges need), so
 *  sum signed — clamping it to zero would overstate the need. */
export function sleepNeedMilli(night: SleepNightView): number | null;
```

Test each: the ramp maps in order; formatting rounds and handles null, sub-hour,
and >24h; `asleepMilli` returns null when any piece is missing; `sleepNeedMilli`
sums a negative nap component correctly.

- [ ] **Step 2: `stage-bar.tsx` + test.** A single horizontal **stacked** bar,
  four segments proportional to the night, each segment `title`/`aria-label`
  carrying its stage name and duration for hover and focus. A stacked bar is the
  right mark because the question is *composition of a whole*.
  Tests: segment widths are proportional and sum to 100%; a zero-duration stage
  renders no segment (not a zero-width sliver with a tooltip); a night with no
  stage data renders the bar's own empty treatment rather than four NaN widths;
  each segment carries its ramp token, not a hex.

- [ ] **Step 3: `sleep-tile.tsx` + test.** Two questions, no number another tile
  already heroes:
  1. **Did you get enough?** `<BigNum>` duration (e.g. `7h 12m`) with WHOOP's
     computed need as the qualifier and `performancePct` beside it.
  2. **What kind of sleep was it?** The `StageBar`, plus a compact legend row.
  States: no data yet → the tile's own empty state matching `connect-card.tsx`
  precedent (`MiniCardEmpty`, muted CTA, never an error).
  Tests: renders last night; renders the no-data empty state; prints `—` rather
  than `NaN` for a scored-but-partial night; the durations shown are the ones
  `formatSleepDuration` produces from the fixture's raw ms.

- [ ] **Step 4: `reconnect-card.tsx`.** The under-scoped affordance, because
  the dashboard is where the user will notice the absence:

```tsx
export function SleepReconnectCard({ href }: { href: string })
```

Same **Reconnect to enable sleep** language as the Settings card. It links to
`/settings?tab=integrations` (the place the action lives) rather than to the API
connect endpoint directly — the tile is a signpost, the Settings card is the
control, and duplicating the OAuth navigation in two components is how they
drift.

- [ ] **Step 5: `fixtures.ts`.** Hand-authored, test-only, never imported by
  production code. Provide: a full scored night; a night with an interior gap in
  the `nights` window; a no-data view; a partially-scored night. Header comment
  in the style of `recovery/fixtures.ts` explaining what each exists to exercise.

- [ ] **Step 6: `tile-renderer.tsx`.** Add `case "sleep":`. The tile has three
  branches, and the order matters:

```tsx
    case "sleep":
      // Three states, and the order is the point: an under-scoped connection is
      // CONNECTED, so `present` is true and the section exists — it is simply
      // empty forever until the user reconnects. Checking `present` first would
      // render the ordinary empty state and the user would wait for data that
      // is never coming.
```

The renderer needs to know the connection is under-scoped. `DashboardData` has
no connection state, so the tile reads it the same way the Settings row does —
via `getWhoopConnection`, in a small client component that renders the reconnect
card or the tile. Keep the fetch inside `sleep-tile.tsx`'s wrapper so
`tile-renderer.tsx` stays a pure switch, and give it the same
treat-a-failure-as-"not under-scoped" default the Settings row uses: a failed
read must never hide real data behind a reconnect prompt.

- [ ] **Step 7: Run** `npm run lint`, `npm run format:check`, `npm run typecheck`,
  `npm run test`, `npm run build`
- [ ] **Step 8: Commit** — `feat(dashboard): add the sleep tile`

---

## Task M1: `get_whoop_sleep` (`prog-strength-mcp`)

**Files:**
- Modify: `src/prog_strength_mcp/api_client.py`
- Modify: `src/prog_strength_mcp/whoop.py`
- Modify: `tests/test_whoop_tools.py`

- [ ] **Step 1: Write the failing tests**, mirroring the three existing
  `get_whoop_recovery` client tests plus the two tool-boundary tests:
  path `/whoop/sleep`, `timezone` forwarded and required, `since`/`until`
  forwarded only when set, `data` unwrapped verbatim, non-dict `data` → `{}`,
  `APIError` surfaced with its status, and the auth guard firing before any HTTP
  (extend `_ExplodingAPI` / `_FailingAPI` with a `get_whoop_sleep`).
- [ ] **Step 2: Run `pytest` and watch them fail.**
- [ ] **Step 3: Implement** `APIClient.get_whoop_sleep` as a copy of
  `get_whoop_recovery` against `/whoop/sleep`, and the tool in `whoop.py` next
  to `get_whoop_recovery` with the same required-`timezone` + optional
  `since`/`until` signature and the same `_auth_header_or_raise()` forwarding.

  The docstring must carry three things:
  - the same "empty result probably means not connected → point at
    **Settings → Integrations**" note;
  - **the new failure mode**: an empty result for a user whose *recovery* does
    return data most likely means they have not reconnected since sleep was
    added — the same Settings → Integrations destination, a different reason;
  - **durations are in MILLISECONDS**, stated explicitly, so the agent does not
    infer minutes and be wrong by 60×.

  Also update the module docstring, which currently says the module is about
  recovery only.
- [ ] **Step 4: Run** `pytest` and `ruff check .` / `ruff format --check .`
- [ ] **Step 5: Commit** — `feat(mcp): add the get_whoop_sleep tool`

---

## Task D1: Docs (`prog-strength-docs`)

**Files:**
- Create: `notes/oauth-scope-migration.md`
- Modify: `sows/whoop-sleep-ingestion.md`

- [ ] **Step 1: Write `notes/oauth-scope-migration.md`** — the reusable pattern,
  in the register of `notes/hrzones-engine.md`. It records: the two independent
  gates (dashboard config vs. the authorization request); that a refresh
  re-sends the scope string but cannot WIDEN a grant, so the authorization-code
  flow must run again; that capability is a separate axis from lifecycle and
  must NOT become a new `Status` (folding it in makes every existing
  `status == connected` check subtly wrong, including the webhook's
  not-connected gate); the three moving parts (`RequiredScopes`,
  `MissingScopes`, the under-scoped gauge); and the skip-don't-call rule for the
  path whose scope is absent. Point at this SOW as the provenance.
- [ ] **Step 2: Flip the SOW.** In `sows/whoop-sleep-ingestion.md`:
  frontmatter `status: shipped`; body header `**Status**: Shipped` and
  `**Last updated**: 2026-08-11`.
- [ ] **Step 3: Commit** — `docs: mark whoop-sleep-ingestion as shipped`
