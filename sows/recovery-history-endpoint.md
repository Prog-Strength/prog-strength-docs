---
status: draft
repos:
  - prog-strength-api
  - prog-strength-docs
---

# Recovery History Endpoint

**Status**: Draft · **Last updated**: 2026-08-12

## Introduction

[`dx/recovery-page-refresh`](../dx/recovery-page-refresh.md) explored four
compositions of `/recovery` and **`aligned-deck` was selected** (DX comparison PR
Prog-Strength/prog-strength-web#167, closed un-merged). Every one of the four
variants was blocked on the same thing, which the DX records as **prerequisite
P1**: *the page cannot draw the HRV chart at all.* This SOW is that
prerequisite, and it ships before the page SOW
([`sows/recovery-page-refresh.md`](./recovery-page-refresh.md)) rather than
inside it.

The gap is a routing accident, not a missing computation. `internal/recoverytrend`
already derives everything the page needs — per-day trailing baselines, per-day
balanced bounds, per-day z-scores and statuses, the HRV spread, the 7-day mean,
and the baseline's drift against its own past — and
[`sows/recovery-baseline-drift-payload.md`](./recovery-baseline-drift-payload.md)
put all of it on the wire. But it reaches exactly one consumer, through exactly
one door: `GET /dashboard/summary`'s `recovery` section, built for a tile and
windowed like one. The window is hard-coded to `2 × baseline_window_days + 1`
local dates **ending today** — 61 fetched, 31 charted — because that is what a
3-column card draws.

The page's own endpoint, `GET /whoop/recovery`, returns bare storage rows:

```ts
type WhoopRecoveryDay = {
  date: string;                    // local YYYY-MM-DD
  recovery_score: number | null;   // 0–100
  resting_heart_rate: number | null;
  hrv_rmssd_milli: number | null;
};
```

No baselines, no per-day bands, no drift, no z-scores. So the shipped HRV chart —
the best thing in the recovery surface, and the one element the DX held constant
across all four variants precisely because it is decided and liked — is
unreachable from the page that most wants it. `prepareHrvChart` guards on
`days`, `baseline`, `hrv` and `baselineTrend` and returns `null` when any is
absent; fed `/whoop/recovery`'s rows it would return `null` forever. `hrv-chart.ts`'s
own header anticipated this endpoint: its machinery is *"the natural import for
the `/recovery` page when that chart follows."* The chart is ready. The read is
not.

What changes for the user when this alone ships: **nothing visible.** This is
scaffolding, and saying so plainly is the point — it exists so the page SOW is a
pure front-end job against a shape the web adapter already speaks, rather than an
API change and a redesign entangled in one review.

## Proposed Solution

One new authed endpoint, `GET /recovery/history`, returning **the exact
`RecoverySection` shape** that `GET /dashboard/summary` already serves under
`sections.recovery` — same keys, same nullability, same semantics — over an
arbitrary local-date window instead of the tile's fixed 31 days.

Two moves get it there, and neither adds a line of derivation logic:

1. **The pure builder learns an explicit window.** `buildWhoop` currently
   materializes its charted dates implicitly: "the last `win + 1` local dates
   ending today, behind `win` dates of lead-in." That rule becomes a parameter —
   the caller states the first and last charted date, the builder prepends the
   lead-in itself, and the dashboard passes the window it always used. The
   dashboard's output is unchanged **byte for byte**, and its existing tests are
   the proof.

2. **A second read path calls the same builder over a wider window.** Same
   connection gate, same single indexed `ListRange`, same engine, same DTO.

`GET /whoop/recovery` is **not touched**. That is a deliberate choice with a
named reason: it is an MCP tool surface (`get_whoop_recovery` in
`prog-strength-mcp` returns its JSON verbatim into the model's context), and
per-day bands, z-scores and statuses are exactly the kind of payload the agent
neither reads nor needs but would pay for on every call. A page read and an agent
read want different amounts of the same data; giving them different doors is
cheaper than giving one door a mode switch.

## Goals and Non-Goals

### Goals

- A new authed `GET /recovery/history?timezone=&since=&until=` returns
  `{"recovery": <RecoverySection>}` — the **identical shape** to
  `/dashboard/summary`'s `sections.recovery`, so `prog-strength-web`'s existing
  `adaptRecovery` maps it with no new adapter.
- The response carries **every derived block** the page needs: `days[]` with
  per-day `baseline_avg` / `balanced_low` / `balanced_high` / `z_score` /
  `status`, plus `baseline`, `hrv`, and `baseline_trend`.
- The charted window is the caller's `[since, until]`, defaulting to **all stored
  history** when `since` is omitted, capped by a config knob.
- Every charted day still carries a **full `baseline_window_days` of lead-in**
  behind it, so the oldest charted morning is measured against real history and
  not against a truncated sample. A caller asking for March gets March's bands,
  computed from February.
- `timezone` is **required** and is the only thing that decides which local dates
  the window covers — the house convention from
  [`internal/daterange`](https://github.com/Prog-Strength/prog-strength-api/tree/main/internal/daterange):
  the client sends a timezone and local dates, never a UTC instant.
- The endpoint honours the same **connection gate** as the dashboard section: an
  absent, revoked, or errored Whoop connection yields an empty-but-well-formed
  payload the page's existing render gates already handle, never a 500.
- `GET /dashboard/summary`'s recovery section is **provably unchanged** — same
  window, same values, same key order — pinned by the existing
  `internal/dashboard` tests passing unmodified.
- `GET /whoop/recovery` and every MCP tool over it are **byte-identical** to
  today.

### Non-Goals

- **No new derivation.** `internal/recoverytrend` gains nothing. Every figure
  this endpoint serves is one the engine already computes for the dashboard;
  this SOW moves reach, not maths.
- **No resting-HR band, spread, or per-day classification.** That is the DX's
  **prerequisite P2**, and `aligned-deck` explicitly does not carry it — the
  winning variant answers resting HR with the trailing average that already
  exists. `resting-hr-tile`'s Open Question 3 stays open, deliberately. See Open
  Questions.
- **No change to `GET /whoop/recovery`.** No new query param, no enriched mode,
  no shape that varies by argument. The reasoning is above.
- **No MCP change.** `prog-strength-mcp` is not in `repos:` and gains no tool.
  An agent that wants recovery context still calls `get_whoop_recovery` and still
  gets four fields per day.
- **No web change.** The client function, the page, and the fetch wiring belong
  to [`sows/recovery-page-refresh.md`](./recovery-page-refresh.md). This SOW
  ships an endpoint with no consumer, on purpose.
- **No pagination or cursor.** One row per day means a decade of Whoop history is
  ~3,650 rows. See *Scale boundary*.
- **No caching layer.** One indexed range read per page load, exactly as the
  dashboard does it.
- **No sleep equivalent.** `GET /whoop/sleep` has the same shape of gap and is
  not in scope.

## Implementation Details

### Why a new endpoint rather than an enriched `/whoop/recovery`

Three alternatives were live; the trade is worth recording because the cheapest
one is wrong for a reason that is invisible from inside the API.

| Option | Rejected because |
| --- | --- |
| Enrich `/whoop/recovery` unconditionally | It is an MCP tool surface. `get_whoop_recovery` returns the API's JSON straight into the model's context, so five extra fields per day × 90 days is pure token cost on every agent call that touches recovery, for figures no coaching answer reads. |
| `/whoop/recovery?include=derived` | A response shape that changes with a query param is a contract that has to be described twice, tested twice, and mirrored twice in the web types. It also leaves the MCP tool one typo away from paying the cost anyway. |
| **New `GET /recovery/history`** | **Chosen.** One shape per door. The page's door serves the page's shape; the agent's door is untouched; neither can regress the other. |

The precedent is already in the codebase: `/activities` serves raw rows while
`/activities/running-metrics` and `/activities/running/best-efforts` serve the
derived, page-scale reads beside it.

### The windowed builder — `internal/dashboard/whoop.go`

`buildWhoop` is already pure (no clock, no DB) and already separates a charted
window from its lead-in. The change is to make both explicit:

```go
// buildRecoveryView assembles a RecoverySection over an EXPLICIT charted window
// of local dates [from, to], inclusive, oldest→newest. The builder prepends
// eng.BaselineWindowDays() dates of lead-in itself, so every charted day —
// including the first — is measured against a full trailing sample.
func buildRecoveryView(
    entries []whooprecovery.Entry,
    eng *recoverytrend.Engine,
    from, to string,          // local YYYY-MM-DD, inclusive
    now time.Time,            // for Today; unchanged semantics
    loc *time.Location,
) *RecoverySection
```

`buildWhoop` stays as a thin wrapper computing its own `from`/`to` from `now`
and `loc` — the last `win + 1` local dates ending today — and delegating. That
keeps the dashboard's call site and its tests untouched and makes "the dashboard
window is one particular window" a fact stated in one place.

Everything below the window computation is unchanged and must not be re-derived:

- The date materialization loop still emits **every** date in
  `[from − win, to]`, with absent days carried as `recoverytrend.Day{Date: ds}`
  and null metrics. A missing morning is a present slot with nothing in it —
  never omitted, never zero-filled.
- `eng.ComputeSeries(engineDays)` still returns exactly one `DayResult` per
  charted day, index-aligned by construction.
- `eng.Compute(charted)` still derives `Baseline` and `HRV` from the charted
  window's **final** day, which is why the window must end on the day the caller
  means by "today". See the invariant below.

### The `Today`-relative blocks, and the invariant they impose

`Baseline`, `HRV`, and `BaselineTrend` answer *"how does the most recent morning
stand?"*. `Compute` takes `days[len(days)-1]` as today and measures it against
the window behind it. That is correct for the dashboard, and it stays correct for
the page **only while `until` is today**.

So the endpoint states the rule rather than hiding it:

- `until` defaults to today in the caller's timezone, and when it is today the
  three derived blocks mean exactly what they mean on the dashboard.
- When a caller passes an **earlier** `until`, the blocks describe *that* day
  against *its* history — which is a coherent answer, not a wrong one, but it is
  a different question. It is documented on the DTO and in the handler comment,
  and the page never asks it (`aligned-deck`'s windows all end on the last stored
  morning).

This is not a hypothetical nicety: `prepareHrvChart` overwrites the final point
of its rolling curve with the server's `hrv.short_avg` so the curve's endpoint
and the printed 7-day figure are one number rather than two that agree by
coincidence. A window whose last day is not the day `short_avg` describes would
put a stale figure on the end of a curve. The page SOW carries the matching
client-side invariant.

### Read path — `internal/dashboard/handler.go`

A second handler beside `summary`, registered on the authed router:

```go
r.Get("/recovery/history", h.recoveryHistory)
```

It reuses `buildRecoverySection`'s gate and read verbatim, differing only in the
window it asks for:

1. **Auth** — `auth.UserIDFrom`; missing user → `401`.
2. **Timezone** — `timezone` is **required**; missing or unloadable → `400
   invalid timezone`. (This differs from `GET /whoop/recovery`, which accepts and
   ignores it because it does no date arithmetic. This endpoint materializes
   dates, so it cannot.)
3. **Window** — `since`/`until` are optional `YYYY-MM-DD`; either malformed →
   `400`. `until` defaults to today in `loc`. `since` defaults to the oldest
   stored row for the user — i.e. **all history** — and is clamped forward when
   the requested span exceeds `page_window_max_days`. `since > until` → `400`.
4. **Connection gate** — `whoopConns.Get`; not connected → the empty payload
   below, `200`. A *read error* on the connection is logged and treated as not
   connected, matching `buildRecoverySection`.
5. **Fetch** — one `ListRange(ctx, userID, sinceStr − win, untilStr)`, covered by
   `idx_user_whoop_recovery_user_date`. A failed read degrades to an empty window
   (never a 500), matching every other section read.
6. **Build** — `buildRecoveryView(entries, h.recovery, from, to, now, loc)`.
7. **Respond** — `httpresp.OK(w, "read recovery history", recoveryHistoryDTO{...})`.

The `since`-defaults-to-oldest-row step needs a bound the repository does not
have today. Prefer the cheap version: skip the extra query and default `since` to
`until − page_window_max_days`, letting the range read return whatever exists.
That is one query, not two, and a user with three months of history simply gets
three months of rows inside a wider requested window — the builder already
materializes absent dates honestly, but a window of null lead-in mornings before
the athlete owned a Whoop is noise on the wire. If the emptiness turns out to
matter visually, trim leading all-null days in the builder rather than adding a
`MinDate` query. See Open Questions.

### Disconnected and empty states

A connected user with no rows already yields a well-formed section (today `nil`,
empty spark, a full null-metric window, unknown baseline/HRV). The page's render
gates read the **connection** (`GET /me/whoop/connection`) for connect/reconnect,
so this endpoint does not need to restate connection status in its payload.

For a **not connected** user, `/dashboard/summary` omits the section entirely.
An endpoint whose entire subject is that section cannot omit its own body, so it
returns the empty-but-valid shape: `today: null`, `resting_hr_spark: []`,
`days: []`, and the three derived blocks in their unknown/zero state. `days: []`
rather than a window of nulls, because a user with no connection has no history
to be aligned to — and the page never gets there anyway, its connect gate fires
first.

### API Surface

**`GET /recovery/history`** (authed)

| Param | Required | Format | Default |
| --- | --- | --- | --- |
| `timezone` | **yes** | IANA name (`America/Denver`) | — |
| `since` | no | `YYYY-MM-DD` local, inclusive | `until − page_window_max_days` |
| `until` | no | `YYYY-MM-DD` local, inclusive | today in `timezone` |

Response `200`:

```json
{
  "message": "read recovery history",
  "data": {
    "recovery": {
      "today": { "date": "2026-08-12", "resting_heart_rate": 69, "recovery_score": 13, "hrv_rmssd_milli": 45 },
      "resting_hr_spark": [51, 47, 50, 69],
      "days": [
        {
          "date": "2026-07-15",
          "resting_heart_rate": 52, "recovery_score": 61, "hrv_rmssd_milli": 88,
          "baseline_avg": 91.2, "balanced_low": 78.6, "balanced_high": 103.8,
          "z_score": -0.25, "status": "balanced"
        }
      ],
      "baseline": { "window_days": 30, "resting_hr_avg": 54.1, "resting_hr_days": 29, "hrv_avg": 89.4, "hrv_std_dev": 12.6, "hrv_days": 28, "recovery_score_avg": 61.3, "recovery_score_days": 28 },
      "hrv": { "status": "suppressed", "balanced_low": 76.8, "balanced_high": 102.0, "z_score": -3.5, "trend": "falling", "short_avg": 71.2 },
      "baseline_trend": { "direction": "rising", "delta_ms": 6.1, "from_avg": 83.3, "over_days": 28 }
    }
  }
}
```

Errors: `400` on a missing/invalid `timezone`, an unparseable `since`/`until`, or
`since > until`; `401` unauthenticated. No `404` — a user without a Whoop
connection gets `200` and the empty shape.

### Config

One knob in the existing `[recovery]` block of `config.toml`, and it is a
non-secret tuning value, so it belongs in version control rather than in an env
var (house convention):

```toml
# page_window_max_days: the widest charted window GET /recovery/history will
#   serve, and the default span when `since` is omitted. Sizes the response, not
#   the truth: a longer history is still stored, it just takes a second request.
#   1825 is five years — comfortably past any current user's Whoop history and
#   still ~450 KB of enriched JSON at worst.
page_window_max_days = 1825
```

Mirrored in `internal/config`'s `RecoveryConfig` and its raw TOML struct, with
validation matching the block's neighbours: must be > 0, and > `baseline_window_days`
(a max window narrower than the lead-in is incoherent). A request exceeding it is
**clamped, not rejected** — the page asks for "everything" and should get as much
as the server is willing to serve, not a `400`.

### Scale boundary

Whoop writes one row per day. Enriched, a day serializes to roughly 190 bytes:

| History | Rows | Approx. response |
| --- | --- | --- |
| 3 months | 92 | ~18 KB |
| 14 months (the DX fixture) | 425 | ~80 KB |
| 5 years (the cap) | 1,825 | ~350 KB |

The DB read is one indexed range scan and `ComputeSeries` is O(days ×
baseline_window_days) — ~55k float operations at the cap, microseconds. Gzip over
the wire takes the numbers down further. The strategy needs revisiting only if
recovery ever becomes intra-day, at which point the row count stops being one per
morning and a cursor becomes the right shape.

### Testing

Go, table-driven, in the packages that own the behaviour:

**`internal/dashboard/whoop_test.go`** — the builder:

- The existing `buildWhoop` tests pass **unmodified**. This is the regression
  contract: the refactor is provably shape-preserving.
- An explicit window narrower than the dashboard's produces exactly its own
  charted dates, with per-day bands present on the first charted day — the
  lead-in proof.
- A window whose lead-in reaches before the oldest stored row still emits every
  charted date, with `status: "unknown"` and null band fields where the sample is
  short.
- A window spanning a month boundary and a DST transition materializes the right
  local dates.
- Missing mornings inside a wide window are present with null metrics, in order.

**`internal/dashboard/recovery_history_handler_test.go`** — the read path:

- `401` without a user.
- `400` for missing `timezone`, malformed `since`, malformed `until`,
  `since > until`.
- Not connected → `200` with the empty shape (`days: []`), not a 500 and not an
  omitted key.
- Connected with rows → `days` covers the requested window inclusive, ordered
  oldest→newest, with the derived blocks populated.
- A request wider than `page_window_max_days` is **clamped** and returns `200`.
- A recovery repository error degrades to the empty window at `200`.
- The response's recovery object is **key-for-key identical** to the one
  `/dashboard/summary` emits for the same user and the same window — asserted by
  marshalling both and comparing, so the two doors cannot drift.

**`internal/server/server_test.go`** — `GET /recovery/history` is in the mounted
route list.

**`internal/config/config_test.go`** — the new knob loads, defaults, and fails
validation at `0` and below `baseline_window_days`.

### Documentation

- `config.toml`'s `[recovery]` comment block gains the `page_window_max_days`
  paragraph, in the same voice as its neighbours.
- The handler carries the *why a separate endpoint* reasoning as a doc comment,
  so the next reader does not helpfully "simplify" it into `/whoop/recovery`.
- `RecoverySection`'s doc comment notes that it is now served by two routes and
  that `Baseline` / `HRV` / `BaselineTrend` describe the window's **final** day.
- This SOW's `status` moves to `shipped` when the PR merges; the page SOW links
  to it as its prerequisite.

## Open Questions

1. **Should `since` default to the user's oldest stored row rather than to
   `until − page_window_max_days`?** Options: a second cheap `MIN(date)` query
   per request; a `MinDate` method on the repository; or the leading-null trim in
   the builder. **Lean: ship the flat default and trim leading all-null days in
   the builder** — it costs one query fewer and produces the same visible result,
   and the ledger's "N mornings stored" count is computed from real rows either
   way.
2. **Should the endpoint live under `/whoop/`?** `GET /whoop/recovery/history`
   would group it with its sibling; `/recovery/history` reads as a product
   surface rather than a vendor one. **Lean: `/recovery/history`** — the page is
   named Recovery, not Whoop, and if a second provider ever lands the vendor
   prefix becomes a lie. Cheap to change before it has a consumer; not after.
3. **Resting-HR spread and per-day banding (the DX's P2).** `aligned-deck` does
   not need it — it draws resting HR as bars diverging from `resting_hr_avg`,
   which exists — and picking that variant was, per the DX, a deliberate answer
   to `resting-hr-tile`'s Open Question 3. **Lean: leave it unbuilt.** If the
   shipped page makes the case that resting HR wants a band after all, it is an
   additive `recoverytrend` change against a payload that already has room for
   it.
4. **Should `resting_hr_spark` and `today` be dropped from this endpoint's
   payload?** Both are tile-shaped: the spark is a 7-day gap-omitting array the
   page will never draw, and `today` duplicates the last element of `days`.
   **Lean: keep both.** Shape identity with `/dashboard/summary` is what lets the
   web reuse `adaptRecovery` unchanged, and ~100 bytes is a cheap price for one
   adapter instead of two.
