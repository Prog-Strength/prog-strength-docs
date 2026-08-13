---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Google Calendar Dashboard Tile

**Status**: Shipped · **Last updated**: 2026-08-12

## Introduction

Prog Strength already holds a Google Calendar grant for every user who has
connected one, and it has held it since
[`planned-workouts-google-calendar-sync`](planned-workouts-google-calendar-sync.md)
shipped. That grant has only ever been used in one direction. `internal/calendarsync`
writes planned workouts and logged activities *into* the user's `primary`
calendar; nothing has ever read one back.

The consequence is that the dashboard — the surface whose whole premise is
*"the state of your training, in one place"* — is blind to the single largest
source of constraint on a training day. The user knows they have a 7am upper
push planned, because we put it there. What the dashboard cannot tell them is
that they also have a 6:30am flight, a 12:30 lunch, and three hours of meetings
after it. Those facts decide whether the plan survives contact with the day,
and today they live in a different tab.

This SOW adds a read direction and spends it on one thing: a `calendar` mini-card
that prints today's events, tomorrow's, and a week-at-a-glance strip, behind the
paging idiom `weather` already established.

**The scope this feature does not need is the interesting part.** The granted
scope is `calendar.events`
(`internal/calendarsync/oauth.go`, `CalendarEventsScope`), which is read *and*
write on events — `events.list` is already permitted. The connection already
resolves to a calendar id, and it is `primary`
(`defaultCalendarID`). So every user who is connected today gets this tile the
moment it deploys: **no new OAuth scope, no re-consent prompt, no migration, no
new secret.** The entire cost is a read method on an HTTP client that already
talks to Google, a service that reuses a token resolver written for exactly this
kind of reuse, and a tile.

That constraint is also what fixes the product boundary. Reading `primary` needs
nothing new; reading *which calendars a user has* needs `calendar.readonly` or
`calendar.calendarlist.readonly`, which is a re-consent for every existing
connected user and a picker UI to spend it on. Multi-calendar is therefore a
deliberate non-goal — see *Non-Goals* and Open Question 1.

What changes for the user: a dashboard tile that answers *"what does today
actually look like, and does the plan fit in it?"* without leaving Prog Strength.

## Proposed Solution

A new self-fetching `calendar` tile in the mini-card grid, with three slides and
an expand panel.

1. **Today** — the day's events as `time · title` rows, Prog Strength-authored
   ones marked.
2. **Tomorrow** — identically rendered, so the user can plan the day ahead.
3. **Week** — seven columns Monday-first, each a busy bar and an event count,
   with a `Next: …` line beneath. Density, not text: a one-third-width cell
   cannot hold a week of event titles and should not pretend otherwise.

The ⤢ affordance opens a **week agenda panel** — every day, every event,
untruncated — which is what makes the density strip useful. The strip says
Wednesday is packed; the panel says with what. It fetches nothing; the tile
already holds the week.

Server-side, `GET /me/calendar/events` returns the window **grouped by local
date**, so the client never re-derives a day boundary. One request covers all
three slides and the panel.

Both catalogs gain the id. Following `sleep` and `resting_hr`, the tile ships
**tray-only** — not in the default layout.

## Goals and Non-Goals

### Goals

- A new `calendar` tile renders three slides and an expand panel,
  production-quality, conforming to [`design-system.md`](../design-system.md) v0.4.
- **No new OAuth scope and no re-consent.** Every currently-connected user gets
  the tile on deploy. A test pins that the read path requests
  `CalendarEventsScope` and nothing else.
- `GET /me/calendar/events` follows the house timezone contract
  (`internal/daterange`): a required IANA `timezone` plus `start_date`/`end_date`,
  converted server-side. The client never builds a UTC instant.
- **One request per tile mount** covers Today, Tomorrow, the week strip, and the
  panel.
- Prog Strength-authored events are marked by **stored `google_event_id`**, never
  by matching on event titles, and carry a deep link to their planned-workout page.
- The catalog gains `calendar` in **both** the Go source of truth
  (`internal/dashboard/tiles.go`) and its TypeScript mirror
  (`lib/dashboard-tiles.ts`), in the same position, so both contract tests pass.
- The tile ships **tray-only**, pinned by a Go test mirroring
  `TestSummary_DefaultLayoutHasNoSleepTile`.
- Every degradation renders as a **calm muted line or a CTA**, never an error
  banner — the `weather` contract.
- A Google 401/403 flips the connection to revoked and surfaces the reconnect
  CTA. A 429/5xx does **not** touch the connection.
- The tile does not lie about a day it has clipped: overflow both directions is
  stated (`+2 earlier`, `+3 more`).
- A dashboard left open across local midnight refetches rather than showing
  yesterday.

### Non-Goals

- **No multi-calendar selection.** Deferred; it needs a new scope, a re-consent
  for every existing user, a picker, and persistence of the selection. See Open
  Question 1.
- **No write path change.** `Service`, `ActivityService`, the reconciler, and
  every existing sync test are untouched. This SOW adds a reader beside them.
- **No event creation or editing from the tile.** Read-only. The tile links out
  to a planned workout; it never PATCHes Google.
- **No Google push channels / `watch` subscriptions.** Polling on mount, behind
  a short TTL, is sufficient for a dashboard tile and avoids a webhook endpoint,
  a channel-renewal job, and the dead-ingestion class of bug documented in
  [`whoop-integration-diagnostics`](whoop-integration-diagnostics.md).
- **No `stale` serving.** See *Why this is not Weather* below.
- **No month view.** That is what `/calendar` is.
- **No `/calendar` page change.** The in-app month grid keeps showing Prog
  Strength's own data. Merging Google events onto it is a coherent follow-up and
  is not this.
- **No mobile change.** `prog-strength-mobile` carries no dashboard or tile
  catalog at all.
- **No default-layout change.** The default stays
  `[running, lifting, steps, nutrition, bodyweight, (recovery), streak]`.
- **No new env var or secret.** Knobs go in `config.toml`.

## Implementation Details

### Where the read path lives, and why it is not a new package

The reader goes in **`internal/calendarsync`**, not a new `internal/calendarevents`.

That package's unexported `connector` performs the five-step dance any Google
call needs — load the connection, reject a revoked one, fetch the encrypted
refresh token, decrypt it, mint an access token — plus the failure handling
around it that is easy to get wrong: *a refresh Google rejects must flip the
connection to revoked, or the UI never learns to prompt for re-consent.* Its own
doc comment records that it was extracted from `Service` precisely because a
second consumer appeared, and that duplicating it "would have been a correctness
risk, not just repetition."

A third consumer is the same argument. Putting the reader in another package
would mean exporting `connector`, `grant`, and the revoke-on-refresh-failure
behaviour across a package boundary — weakening the encapsulation that comment
defends, in exchange for a package name that reads slightly better. The name is
the lesser concern; `calendarsync` is where the Google grant lives.

| File | Change |
| --- | --- |
| `internal/calendarsync/client.go` | `ListEvents` added to the `CalendarClient` interface and its HTTP implementation. Reuses `calendarAPIBase`, the bearer header, and the existing 401/403 → `ErrTokenRejected` mapping. |
| `internal/calendarsync/events_service.go` | **New.** `EventsService`: resolve grant → list → filter → mark ours → group by local date. |
| `internal/calendarsync/events_cache.go` | **New.** The per-user TTL cache. |
| `internal/calendarsync/handler.go` | **Touched.** `MountAuthed` gains `r.Get("/me/calendar/events", h.events)`. |
| `internal/calendarsync/metrics.go` | **Touched.** Two new collectors. |

### The endpoint

```
GET /me/calendar/events?timezone=America/New_York&start_date=2026-08-10&end_date=2026-08-17
```

It sits beside `/me/calendar/connection`, which it depends on, and it takes the
house date contract verbatim — `daterange.LoadTimezone` then
`daterange.DayBoundsUTC` for each end of the window. **The client must never
compute `timeMin`/`timeMax`.** `daterange`'s package doc explains why at length,
and the DST case is not hypothetical here: a "week" is 167 or 169 hours twice a
year, and a tile that assumes 7×24 drops or duplicates a day's events for every
user not on UTC.

Response:

```json
{
  "status": "ok",
  "days": [
    {
      "date": "2026-08-12",
      "truncated": 0,
      "events": [
        {
          "id": "abc123",
          "title": "Upper Body Push",
          "start": "2026-08-12T11:00:00Z",
          "end": "2026-08-12T12:00:00Z",
          "all_day": false,
          "source": "prog_strength",
          "link": { "kind": "planned_workout", "id": "pw_123" }
        },
        {
          "id": "def456",
          "title": "Lunch w/ Sam",
          "start": "2026-08-12T16:30:00Z",
          "end": "2026-08-12T17:30:00Z",
          "all_day": false,
          "source": "google"
        }
      ]
    }
  ]
}
```

**`days` is dense, not sparse** — every date in the requested window appears,
with an empty `events` array for a free day. A client that has to distinguish
"no events" from "day missing from the payload" will eventually get it wrong,
and the week strip needs seven columns regardless.

`truncated` is how many events `max_events_per_day` cut from that day, and it is
almost always `0`. It exists so the cap is never silent: the week strip's count
prints `events.length + truncated`, so a conference day with 60 entries reports
sixty rather than the fifty that fit in the payload.

`status` is the closed degradation set, mirroring `weather`'s vocabulary:

| `status` | Meaning | Tile renders |
| --- | --- | --- |
| `ok` | Events read successfully (`days` may be all-empty) | The slides |
| `not_connected` | No connection row | `Connect Google Calendar →` |
| `reconnect_needed` | Connection revoked, or the refresh was rejected | `Reconnect Google Calendar →` |
| `disabled` | `[calendar_events].enabled = false` | `Calendar is off.` |
| `unavailable` | Google returned 429/5xx, or the call failed | `Calendar is unavailable.` |

`not_connected` is a **200 with a status**, not a 404. It is the ordinary state
of a user who never opted in, and the tile's job there is to invite, not to
handle an error. This mirrors `ErrNotConnected`'s existing treatment in the
write path, where it is explicitly "not an error condition at all."

### The Google call

```go
// events.list, with the four parameters that make the result usable.
q.Set("timeMin", start.Format(time.RFC3339))
q.Set("timeMax", end.Format(time.RFC3339))
q.Set("singleEvents", "true")  // expand recurring rules into real instances
q.Set("orderBy", "startTime")  // only legal WITH singleEvents=true
q.Set("showDeleted", "false")
q.Set("maxResults", …)
```

`singleEvents=true` is load-bearing and is the parameter most likely to be
dropped by someone simplifying the query. Without it Google returns the
recurring *rule* — a single event with an `RRULE` and the series' original start
date — and a user's daily standup either vanishes from the tile or appears on
the day the series began. `orderBy=startTime` is rejected by the API unless
`singleEvents` is set, so the two travel together.

Two filters run server-side, on the way out:

- **Declined events are dropped.** An event whose `attendees[]` contains the
  user with `responseStatus: "declined"` is a meeting they are not attending; it
  is noise on a tile this small.
- **Cancelled events** are excluded by `showDeleted=false`.

An event Google returns with no `summary` — a private event on a shared
calendar, or a busy block — renders as **`Busy`**, not an empty row. The user
cannot see the title in Google either; the tile should say so rather than draw a
row with nothing in it.

### Marking Prog Strength's own events

Prog Strength writes into `primary`, so its own events come back from
`events.list`. They are marked, not filtered: the planned workout is the single
most relevant item on a training day, and a tile that hides it stops being a
single place to see the day.

Marking is an **id-set lookup, never a title match**. Two tables already store
what Google gave us back:

| Source | Column | Link kind |
| --- | --- | --- |
| `planned_workouts` | `google_event_id` (migration 025) | `planned_workout` |
| `activity_calendar_sync` | `google_event_id` (migration 053) | `activity` |

One query per table over the window's user id, collected into a
`map[string]link`. Any returned event whose Google id is a key is `source:
"prog_strength"` and carries the link. Anything else is `source: "google"`.

Title matching is called out as forbidden because it is the tempting shortcut
and it is wrong in both directions: a user who renames our event in Google loses
its mark, and a user who names their own event "Upper Body Push" gets a deep
link to a planned workout that is not theirs.

**Accepted limitation.** The tile is a view of Google, and Google is downstream
of our own sync. A plan that has not synced yet — or any plan at all if
`[calendar_sync].enabled` is false — will not appear, because it is genuinely
not on the user's calendar. This is coherent, but it does couple the two
features in a way a user could notice. See Open Question 2.

### Why this is not Weather

The tile borrows `weather`'s *shape* — self-fetching, no `/dashboard/summary`
section, calm degradations, paging. It deliberately does **not** borrow its cost
machinery, and the distinction should survive review.

`weather` carries a daily call budget, a hard ceiling, a paid-overage flag, a
`budget_exhausted` status, a durable cache, and `stale` serving. All of it exists
because OpenWeather **bills per call**, so a cache miss is money and a last-good
reading is worth serving.

Google Calendar's API is free, at a quota of one million queries per day. A
cache miss here is a free retry. Importing the budget machinery would be
cargo-culting a solution to a problem this integration does not have, and every
one of those knobs is a thing to configure, test, and misconfigure.

So:

- **Cache:** a per-user in-memory TTL, default 60s, and nothing else. Its only
  job is to keep dashboard remounts and slide changes off Google.
- **No durable cache, no budget ledger, no daily ceiling, no `budget_exhausted`.**
- **No `stale` serving.** When Google fails, the tile says `Calendar is
  unavailable.` and refetches on the next mount. Adding a status to the client's
  vocabulary to avoid a one-render blank, on data that is free to re-fetch, is
  not worth its weight.

Config, in a new `config.toml` block beside `[calendar_sync]` — public literals,
no env override, following that block's own precedent:

```toml
[calendar_events]
# Reading the user's Google Calendar for the dashboard tile. Public literals —
# not secrets, not env-overridden. The one real secret in the calendar feature
# is calendar_token_enc_key under [auth], and this reader shares it.
#
# enabled is the kill switch for the READ direction only. Turning it off makes
# every /me/calendar/events response status "disabled" and the tile say so
# plainly; it does not touch OAuth, planned-workout scheduling, or activity
# sync, which are governed by [calendar_sync].enabled.
enabled = true

# How long a user's fetched window is reused before Google is asked again.
# Small on purpose: this exists to absorb dashboard remounts and slide changes,
# not to be a cache in any meaningful sense. Google's quota is 1M/day and free,
# so there is nothing here to economise.
cache_ttl_seconds = 60

# Per-day cap on events returned to the client. A shared calendar can carry
# hundreds of entries on one day; the tile shows five and the panel shows a
# screenful, so an uncapped payload is bytes nobody reads. Overflow is REPORTED,
# never silently dropped — see each day's `truncated` count in the payload.
max_events_per_day = 50
```

### The catalog change

Both catalogs, same position, appended after `weather`:

| File | Change |
| --- | --- |
| `internal/dashboard/tiles.go` | New `TileCalendar TileID = "calendar"`; added to `Catalog` **after `TileWeather`**. |
| `internal/dashboard/tiles_test.go` | The `all` list and `TestCatalog_Order`'s `want` gain the id in the same position; `TestValidTileID` gains a `calendar` case. |
| `internal/dashboard/summary_layout_test.go` | New `TestSummary_DefaultLayoutHasNoCalendarTile`. |
| `lib/dashboard-tiles.ts` | `"calendar"` added to the `TileId` union and a `TILE_CATALOG` entry in the same position. |
| `lib/dashboard-tiles.test.ts` | Count `21 → 22`; the order list and the `ALL_TILE_IDS` exhaustiveness record gain the id. |
| `app/(app)/dashboard/_components/tile-renderer.tsx` | A pre-switch `if (id === "calendar") return <CalendarCard />;` — see below. |

The catalog entry has **no `href`**:

```ts
{
  id: "calendar",
  title: "Calendar",
  description: "Today and tomorrow from your Google Calendar, plus the week ahead.",
}
```

This is deliberate and it is the third such entry. `quote` has no page; `weather`
has no page; `calendar` has no page either — there is no single destination that
"the calendar tile" deep-links to, because its rows link to *different* places
(a planned workout) or to nowhere (a lunch). `TileCatalogEntry.href` is already
optional for exactly this, and `lib/dashboard-tiles.test.ts`'s href assertion
already tolerates it.

The renderer follows `weather`'s placement, **before** the `href` resolution and
outside the `switch`, for the same stated reason — a tile with no `href` cannot
go through a path that requires one, and the switch stays a pure switch:

```tsx
// Calendar self-fetches, like weather: third-party data with its own auth and
// failure modes has no business inside the single /dashboard/summary
// round-trip, where a slow Google would delay every other tile on the grid.
// Like quote and weather it has no page behind it, so it resolves before href.
if (id === "calendar") {
  return <CalendarCard />;
}
```

**Default layout: tray-only.** `defaultLayout` in `layout_resolve.go` is
unchanged. The tile is useless — a permanent connect CTA — for any user who has
not connected a calendar, and unlike `recovery` (which is gated on
`hasConnectedWhoop`) gating it would mean a second connection probe on every
dashboard load to save a tray click. `sleep` and `resting_hr` both set the
opt-in precedent. The Go test pinning this is part of the work, so the choice is
deliberate rather than incidental.

### File layout (web)

| File | Responsibility |
| --- | --- |
| `app/(app)/dashboard/_components/calendar/calendar-tile.tsx` | **New.** `CalendarCard`: the fetch, the three slides, paging, the states. |
| `app/(app)/dashboard/_components/calendar/day-slide.tsx` | **New.** One day's rows, including the windowing. |
| `app/(app)/dashboard/_components/calendar/week-strip.tsx` | **New.** The seven-column density strip. |
| `app/(app)/dashboard/_components/calendar/calendar-week-modal.tsx` | **New.** The expand panel. |
| `app/(app)/dashboard/_components/calendar/shared.ts` | **New.** Pure logic — window math, the visible-row selection, strip normalisation, time formatting. No React. |
| `app/(app)/dashboard/_components/calendar/fixtures.ts` | **New.** The states below. |
| `lib/api.ts` | **Extended.** `getCalendarEvents`, beside the existing `getCalendarConnection`. |

The pure module is separated for the reason `hrv-chart.ts` and `resting-rank.ts`
were: the row-windowing rule below is where the tests earn their keep, and it is
where a well-meaning simplification would do the most damage.

### The panel, and why the tile is not a `MiniCard`

`CalendarCard` composes `MINI_CARD_PANEL` + `MINI_CARD_HOVER` by hand rather
than wrapping `MiniCard`. `mini-card.tsx` exports those two constants for
precisely this case and says so: a tile with its own buttons inside cannot be
wrapped in a single `<a>`, because a button inside an anchor is invalid markup
that swallows its own clicks. This tile has pager buttons, an expand button, and
per-row links.

**There is no gear icon.** `weather`'s gear manages saved locations; `primary`-only
means this tile has nothing to manage. The header carries the ⤢ affordance and
nothing else. (An early mockup showed a gear; it is dropped.)

### Paging

Lifted from `weather-tile.tsx`, idiom for idiom: ‹ › buttons, dot indicators,
`ArrowLeft`/`ArrowRight` on the focused `role="group"`, touch swipe past a 40px
threshold, and page state that is **ephemeral by design** — the tile always
opens on Today. A user who left it on the week strip yesterday wants today's
events today.

Three slides, fixed: `Today`, `Tomorrow`, `Week`.

### The request window

One request feeds everything:

```ts
// The union of what the three slides need. The max() matters on Sunday, when
// "tomorrow" is next Monday and falls OUTSIDE the current week — a window of
// just the week would render an empty Tomorrow slide one day in seven.
const start = min(weekStart /* Monday */, today);
const end   = max(weekEnd /* Sunday */, tomorrow);
```

`weekStart` is **Monday**, matching `/calendar`'s existing `WEEKDAYS` constant
and its `mondayOffset` grid math. A tile whose week starts on Sunday sitting one
click away from a month grid that starts on Monday is the kind of drift that is
invisible in review and jarring in use.

### Day slides — the windowing rule

Header (`Today · Wed Aug 12`), then up to **five** rows of `time · title`.
All-day events pin to the top with no time column.

The rule that needs stating: **the visible window anchors on the first upcoming
event**, taking it plus the four after it, backfilled with earlier events if the
day is nearly over.

```ts
/**
 * Which five of a day's events to show.
 *
 * A naive slice(0, 5) is chronological and wrong after lunch: at 6pm it fills
 * the tile with things the user has already done, and buries the one event
 * they can still act on. Anchoring on the first UPCOMING event keeps the tile
 * about the rest of the day, and backfilling means a 9pm view is not blank.
 *
 * Everything clipped is REPORTED — see earlierCount/laterCount. A tile that
 * silently drops half a day reads as an empty afternoon.
 */
export function visibleEvents(events: Event[], now: Date, limit = 5): {
  visible: Event[];
  earlierCount: number;
  laterCount: number;
}
```

Past events that do render are `--muted` rather than hidden, so the day still
reads as a whole. `+2 earlier` and `+3 more` are both muted lines that open the
panel on that day.

**Tomorrow is not "now"-relative.** It has no past, so it renders the first five
chronologically. `visibleEvents` handles this without a special case — with `now`
before every event, the anchor is index 0 — but it gets a test, because it is
the kind of thing a refactor breaks silently.

An empty day renders `Nothing scheduled.` in `--muted`. **No CTA, no
suggestion.** The tile does not nag a user for having a free afternoon, and it
must not become a surface that sells planning a workout.

### Week slide

Seven columns, Monday-first. Each: the weekday initial, a bar whose height is
that day's event count against the week's busiest day, and the count. Today's
column is marked. A day carrying a Prog Strength event takes the accent tint on
its bar.

```ts
/**
 * Bar heights, normalised against the week's busiest day rather than a fixed
 * ceiling. The strip answers "which days are heavy RELATIVE to my week" — an
 * absolute scale makes an ordinary week look empty and a conference week look
 * identical to a normal one.
 *
 * An all-zero week yields all-zero heights, not NaN. This is the guard the
 * division needs and it is a real state, not a defensive flourish: it is
 * exactly what a new user's first Sunday looks like.
 */
export function stripHeights(counts: number[]): number[]
```

Beneath the strip, one line: `Next: 12:30p Lunch w/ Sam` — the next upcoming
event anywhere in the remaining week, or `Nothing left this week.`

**The strip is decoration; the meaning is the caption and the counts.** Following
`morning-ledger`'s `railLabel` precedent, the strip container takes `role="img"`
with a composed label ("Monday 1 event, Tuesday 3 events, …") and the bars are
`aria-hidden`.

### The states

Each renders correctly at a constant card height and gets a test.

- **default** — a mixed weekday: a marked planned workout, two meetings, one
  all-day event. Reads calm.
- **empty-day** — today is free. `Nothing scheduled.`, no CTA. The week strip
  still renders from the other days.
- **busy-day** — eleven events, six of them past. The window anchors correctly,
  `+6 earlier` and `+0 more` render honestly, and the card does not grow.
- **late-evening** — `now` is after every event today. The window backfills to
  the last five, all muted, `+3 earlier`, no `+N more`.
- **sunday** — the union window's edge case. Tomorrow is next Monday, outside
  the week strip, and the Tomorrow slide is populated.
- **all-day-only** — a holiday. One pinned row, no time column, and the week
  strip counts it as one event.
- **empty-week** — every day zero. `stripHeights` returns zeros, not `NaN`, the
  strip renders as a flat rail, and the caption reads `Nothing left this week.`
- **not-connected** — the CTA, wired to
  `/auth/google/calendar/connect?return_to=<dashboard>`, the same call the
  Settings page already makes.
- **reconnect-needed** — identical layout, `Reconnect Google Calendar →`.
- **unavailable / disabled** — quiet muted lines.
- **loading** — `MiniCardSkeleton`, the house shimmer.
- **both breakpoints** — full-width single column on mobile, one-third on
  desktop. Five rows plus a header plus the pager must fit the one-third cell
  without the grid row growing.

### Midnight rollover

A dashboard left open past local midnight would otherwise show yesterday as
"Today" indefinitely. On `visibilitychange` (and on window focus) the tile
compares the local date against the one it rendered and refetches if they
differ. No interval timer — a tile that polls a clock every minute to catch one
transition a day is a background cost for a foreground problem.

### Failure handling

| Google says | Service does | Tile shows |
| --- | --- | --- |
| 401 / 403 | `ErrTokenRejected` → connection flipped to `revoked` | `Reconnect Google Calendar →` |
| 404 / 410 on the calendar | `unavailable` | `Calendar is unavailable.` |
| 429, 5xx, timeout, transport error | `unavailable`, **connection untouched** | `Calendar is unavailable.` |

The 429 row is the one that matters. Google rate-limits; a rate limit is not a
revoked grant, and flipping the connection over one would present a re-consent
prompt to a user whose authorisation is perfectly good — and, because the write
path shares that connection row, would **also stop their planned-workout sync**.
A read-side blip must never cost the user their grant. This gets a test.

### Observability

Two collectors in `metrics.go`, following the existing closed-label convention:

```go
// api_calendar_event_reads_total counts every read of a user's calendar for
// the dashboard tile, by outcome and by whether the cache answered it.
//
// cache is split out because it is the only thing that makes the Google-call
// rate interpretable: a healthy tile serves most reads from the 60s TTL, and
// a sudden collapse in the cache-hit ratio means remounts, not usage.
var eventReadsTotal = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "api_calendar_event_reads_total",
        Help: "Google Calendar reads for the dashboard tile by result and cache outcome.",
    },
    []string{"result", "cache"}, // result: ok|not_connected|reconnect_needed|disabled|unavailable
)

var eventReadDuration = prometheus.NewHistogram(…) // api_calendar_event_read_seconds
```

**Deliberately no last-success stamp and no absence-of-success alert**, in
contrast to `calendarconn.Connection.LastSuccessfulSyncAt` and its long doc
comment. That stamp exists because the write path is a *background* process
whose silence is a symptom — the Whoop webhook lesson. This read path is
*user-triggered*: a day with no reads means nobody opened a dashboard, which is
a fact about traffic, not a fault. Alerting on its absence would reproduce
exactly the false-positive pattern that stamp was introduced to fix, in the one
place where it does not apply.

### Testing

**Go — `events_service_test.go`**, against an `httptest` server via the existing
`calendarAPIBase` package var. No Google in tests.

- The window is built from `daterange`, and a **DST-transition week** yields the
  correct `timeMin`/`timeMax` — the test that fails if anyone reintroduces
  `24 * 7 * time.Hour`.
- `singleEvents=true` and `orderBy=startTime` are both present on the outgoing
  query.
- A declined attendee's event is filtered; an accepted and a needs-action one
  are not.
- An event with no `summary` renders as `Busy`.
- A multi-day all-day event appears on **each** day it covers; a timed event
  appears only on its start day.
- An event whose id is in `planned_workouts.google_event_id` is marked
  `prog_strength` with a `planned_workout` link; one in
  `activity_calendar_sync.google_event_id` is marked with an `activity` link; an
  unknown id is `google`.
- **An event titled exactly like a planned workout but with an unknown id is
  `google`** — the pin against title matching.
- `days` is dense: a window with no events at all still returns one entry per
  date.
- `max_events_per_day` truncates and reports the overflow rather than dropping
  silently.
- 401 → connection status becomes `revoked`, response is `reconnect_needed`.
- **429 → connection status is UNCHANGED**, response is `unavailable`.
- No connection row → `200` with `not_connected`, and **no Google call is made**.
- `enabled = false` → `disabled`, and no Google call is made.
- Cache: two reads inside the TTL make one Google call; a read after it makes
  two; two different users never share a cache entry.
- The OAuth config used by the reader requests `CalendarEventsScope` and nothing
  else — the pin that this feature never silently grows a scope.

**Go — catalog.** `tiles_test.go` for the new constant and order;
`summary_layout_test.go` for the tray-only pin; `layout_handler_test.go` gains a
case proving `calendar` validates on write.

**Web — `calendar/shared.test.ts`** (the pure module):

- `visibleEvents` anchors on the first upcoming event and reports both counts.
- `visibleEvents` backfills when the day is over, and reports `earlierCount`
  with `laterCount` zero.
- `visibleEvents` with `now` before everything returns the first five — the
  Tomorrow case.
- `visibleEvents` on a day with ≤5 events returns them all and both counts zero.
- The request window is a Monday–Sunday union, and **on a Sunday it extends to
  the following Monday**.
- `stripHeights` normalises against the busiest day and returns zeros — not
  `NaN` — for an all-zero week.
- Time formatting respects the browser locale and does not print `12:00p` for
  noon.

**Web — `calendar/calendar-tile.test.tsx`**: one test per state above, plus
paging by button, keyboard, and swipe; the pager not rendering below two slides
is not applicable (there are always three); the not-connected CTA's `href`; the
midnight-rollover refetch on `visibilitychange`; and the panel's open, close,
and focus-return-to-⤢ paths.

**Web — `lib/dashboard-tiles.test.ts`** and `tile-renderer.test.tsx` updated for
the new id. `dashboard/page.test.tsx` must pass **unmodified**.

### Design system

`scope: in-system`. Every colour is a v0.4 token — `--foreground`, `--muted`,
`--faint`, `--border`, `--border-strong`, `--surface`, `--surface-2`, and
`--accent` for the Prog Strength mark. No raw hex in any component. Manrope,
`tabular-nums` on every time and count, `--radius-card` via `MINI_CARD_PANEL`.

Two conformance notes:

- **`--accent` (periwinkle) is the mark for "this one is ours"**, and it is the
  only colour on the card. It is a *provenance* signal, not a status: a planned
  workout is not painted warm for being soon or green for being done.
- **No `--warning` and no `--danger` anywhere.** A busy day is not an alarm. The
  tile reports a calendar; it does not have an opinion about it.

No new tokens. No `design-system.md` change.

### Documentation

- `calendar-tile.tsx`'s file header explains the three slides, why the tile
  self-fetches rather than joining `/dashboard/summary`, why one request covers
  everything, and why there is no gear.
- `shared.ts`'s header explains the windowing rule with the "chronological is
  wrong after lunch" argument, since that is the piece most likely to be
  simplified back into a `slice(0, 5)`.
- `events_service.go`'s header explains why the reader lives in `calendarsync`,
  that the existing scope already permits reads, and why a 429 must not revoke.
- `client.go`'s `ListEvents` documents `singleEvents=true` and what breaks
  without it.
- Amend [`planned-workouts-google-calendar-sync`](planned-workouts-google-calendar-sync.md)
  with a line noting the grant now has a reader, and pointing here.

## Open Questions

1. **Should the tile eventually read more than `primary`?** As specified it
   reads `primary` only, which is what the existing grant permits and is why
   this ships without re-consent. The gap is real: a user whose training lives
   on a separate "Training" calendar, or whose partner's shared calendar carries
   half their commitments, sees an incomplete day. Options: leave it and see
   whether anyone notices; add `calendar.readonly` with a picker
   (`weather-locations-popover`'s shape applies almost directly) and re-consent
   every connected user; read all calendars with no picker. **Tentative lean:
   leave it, and let real use decide.** The re-consent is a genuine cost — it
   interrupts users who already said yes, and a grant that lapses mid-prompt
   takes planned-workout sync down with it — and it should be spent on a
   demonstrated need rather than an anticipated one. The picker is cheap to add
   later; the scope prompt is not cheap to spend twice.

2. **Should the tile fall back to unsynced planned workouts?** The tile shows
   Google's events, so a plan that has not reached Google yet is missing from a
   card the user reasonably reads as "my day". We hold that plan in our own
   database. Options: leave it, on the grounds that the tile is honestly a view
   of the calendar; merge unsynced plans in client-side from the existing
   `listPlannedWorkouts` call; make the endpoint itself merge them server-side.
   **Tentative lean: leave it and watch, but instrument it.** The merge is not
   hard, but it makes the tile a second, subtly different renderer of planned
   workouts — with its own dedupe rule against the synced copy — and that is a
   real complexity for a window that is usually seconds wide. If
   `api_calendar_syncs_total{result="failed"}` shows the window is routinely
   long, the answer changes.

3. **Is a week strip the right third slide, or should it be a three-day
   agenda?** The strip trades text for coverage: it answers "which day is
   heavy" but never "heavy with what" without opening the panel. A rolling
   three-day agenda would keep real event titles on every slide at the cost of
   the week overview. **Tentative lean: ship the strip.** The panel is the
   escape hatch and it is in scope, so the strip is never a dead end — and
   "which day this week can I move the long run to" is a question nothing else
   in the product answers. Worth revisiting if the panel goes unopened.
