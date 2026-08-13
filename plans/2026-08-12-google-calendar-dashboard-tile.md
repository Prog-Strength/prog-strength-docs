# Google Calendar Dashboard Tile — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a read direction to the existing Google Calendar grant and spend it on one thing — a tray-only `calendar` dashboard mini-card that prints today's events, tomorrow's, and a week-at-a-glance strip, with an expand panel showing the full week agenda.

**Architecture:** Two slices. (1) **API** — the reader lives in `internal/calendarsync` (beside the existing write path, so it reuses the unexported `connector` and its revoke-on-refresh-failure behaviour): `ListEvents` on the existing HTTP client, a new `EventsService` that resolves the grant → lists → filters → marks Prog Strength's own events by stored id → groups by local date, a per-user TTL cache, and `GET /me/calendar/events` on the existing authed mount. (2) **Web** — a self-fetching `CalendarCard` in the mini-card grid, composed from `MINI_CARD_PANEL` by hand (it has buttons inside), with a pure `shared.ts` carrying the windowing rule. Both tile catalogs gain the id.

**No new OAuth scope, no re-consent, no migration, no new secret.** The granted `calendar.events` scope already permits `events.list`, and the connection already resolves to `primary`.

**Tech Stack:** Go 1.25 (chi, SQLite, stdlib `testing`, `httptest`, prometheus); Next.js 16 / React 19 / TypeScript / Tailwind v4; Vitest + Testing Library.

**Design system:** `scope: in-system` against design-system **v0.4**. Conform only — no new tokens, no `design-system.md` change. Use `--foreground`, `--muted`, `--faint`, `--border`, `--border-strong`, `--surface`, `--surface-2`, and `--accent` for the Prog Strength provenance mark. **No `--warning`, no `--danger` anywhere.** No raw hex. `tabular-nums` on every time and count.

---

## Decisions made while planning (record these in the PR bodies)

1. **"Upcoming" means *has not ended*, not *has not started*.** `visibleEvents` anchors on the first event whose `end` is after `now`. A meeting you are currently in is the most relevant row on the tile; anchoring on `start` would push it into the `+N earlier` bucket at exactly the moment it matters most. The SOW's stated intent ("keeps the tile about the rest of the day") is served by this reading, and every state in the SOW's list still behaves as specified.
2. **All-day events are pinned outside the window math.** `visibleEvents` partitions the day into all-day and timed, pins the all-day rows at the top, and runs the anchor/backfill rule over the timed remainder with the leftover row budget. Otherwise a holiday marker would anchor the window at index 0 and fill a 6pm tile with the morning.
3. **The events service takes a local `EventsConfig`, not `config.CalendarEventsConfig`.** `internal/calendarsync` imports no `internal/config` today (unlike `internal/weather`); `ActivityServiceDeps` is the package's established shape for injected collaborators. `internal/server` maps the typed config across. Keeps the package's tests free of the config package.
4. **The request window is capped at 31 days** (`400` beyond it). The tile asks for 8 days; an unbounded window is a Google call and a response size a client should not be able to choose.
5. **Go comments use US spelling.** `golangci-lint` runs `misspell` with `locale: US`, so `behaviour`, `economise`, `normalise` and friends are lint errors *in Go files* (not in TOML, and not in the web repo). Some prose in this plan is quoted from the SOW, which uses British spelling — Americanize it when it lands in a `.go` file. This already bit once.
6. **Time formatting uses `toLocaleTimeString`.** The SOW's mock writes `12:30p`; a hand-rolled `h % 12` formatter is exactly what prints `0:00p` for noon, which the SOW's test forbids. Locale formatting is the correct answer to "respects the browser locale".

---

## File Structure

**`prog-strength-api`**

- Modify: `config.toml` — new `[calendar_events]` block beside `[calendar_sync]`.
- Modify: `internal/config/config.go` — `CalendarEventsConfig` type, `fileConfig` block, mapping.
- Modify: `internal/config/config_test.go` — the literals assertion gains the block.
- Modify: `internal/calendarsync/client.go` — `ListEvents` on the `CalendarClient` interface + HTTP impl, `ListedEvent` wire type.
- Modify: `internal/calendarsync/client_test.go` — query params, all-day vs timed parsing, declined detection, status mapping.
- Create: `internal/calendarsync/events_links.go` — `EventLink`, `EventLinkRepository`.
- Create: `internal/calendarsync/events_links_sqlite.go` — the two-table SQLite implementation.
- Create: `internal/calendarsync/events_links_sqlite_test.go`.
- Create: `internal/calendarsync/events_cache.go` — per-user TTL cache.
- Create: `internal/calendarsync/events_cache_test.go`.
- Create: `internal/calendarsync/events_service.go` — `EventsService`, `EventsConfig`, the status set, day grouping.
- Create: `internal/calendarsync/events_service_test.go`.
- Modify: `internal/calendarsync/metrics.go` — `eventReadsTotal`, `eventReadDuration`, `ObserveEventRead`.
- Modify: `internal/calendarsync/handler.go` — `AttachEvents`, `MountAuthed` gains `GET /me/calendar/events`, the response DTOs.
- Modify: `internal/calendarsync/handler_test.go` — endpoint tests.
- Modify: `internal/server/server.go` — construct the link repo, the cache, the events service; attach to the handler.
- Modify: `internal/dashboard/tiles.go` — `TileCalendar`, appended to `Catalog` after `TileWeather`.
- Modify: `internal/dashboard/tiles_test.go` — `all`, `TestCatalog_Order` `want`, `TestValidTileID`.
- Modify: `internal/dashboard/summary_layout_test.go` — `TestSummary_DefaultLayoutHasNoCalendarTile`.
- Modify: `internal/dashboard/layout_handler_test.go` — `TestPutLayout_CalendarAccepted`.

**`prog-strength-web`**

- Modify: `lib/api.ts` — `CalendarEventsResponse` / `CalendarDay` / `CalendarEvent` types + `getCalendarEvents`.
- Create: `app/(app)/dashboard/_components/calendar/shared.ts` — the pure module. No React.
- Create: `app/(app)/dashboard/_components/calendar/shared.test.ts`.
- Create: `app/(app)/dashboard/_components/calendar/fixtures.ts` — the states.
- Create: `app/(app)/dashboard/_components/calendar/day-slide.tsx`.
- Create: `app/(app)/dashboard/_components/calendar/week-strip.tsx`.
- Create: `app/(app)/dashboard/_components/calendar/calendar-week-modal.tsx`.
- Create: `app/(app)/dashboard/_components/calendar/calendar-tile.tsx`.
- Create: `app/(app)/dashboard/_components/calendar/calendar-tile.test.tsx`.
- Modify: `lib/dashboard-tiles.ts` — `TileId` union + `TILE_CATALOG` entry after `weather`.
- Modify: `lib/dashboard-tiles.test.ts` — count 21 → 22, order list, `PAGELESS_TILE_IDS`, `ALL_TILE_IDS`.
- Modify: `app/(app)/dashboard/_components/tile-renderer.tsx` — pre-switch `calendar` branch.
- Modify: `app/(app)/dashboard/_components/tile-renderer.test.tsx` — the new id.

**`prog-strength-docs`**

- Modify: `sows/planned-workouts-google-calendar-sync.md` — a line noting the grant now has a reader.
- Modify: `sows/google-calendar-dashboard-tile.md` — status flip (done by the operator workflow, not a task here).

---

## Shared contract (every task depends on this — read it first)

### Wire shape of `GET /me/calendar/events`

```
GET /me/calendar/events?timezone=America/New_York&start_date=2026-08-10&end_date=2026-08-17
```

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
        }
      ]
    }
  ]
}
```

- `days` is **dense**: one entry per date in the requested window, `events: []` for a free day.
- `truncated` is how many events `max_events_per_day` cut from that day.
- `status` ∈ `ok | not_connected | reconnect_needed | disabled | unavailable`. Everything but `status` is omitted when there is nothing to say; `days` is always present and non-null on `ok`.
- `link.kind` ∈ `planned_workout | activity`. Absent when `source` is `google`.
- All-day events carry `all_day: true` and a `start`/`end` at the local day's bounds in UTC.

### Go types (defined in Task 4, referenced by Tasks 2, 3, 5, 6)

```go
// EventsStatus is the closed degradation set, mirroring weather's vocabulary.
type EventsStatus string

const (
	EventsStatusOK              EventsStatus = "ok"
	EventsStatusNotConnected    EventsStatus = "not_connected"
	EventsStatusReconnectNeeded EventsStatus = "reconnect_needed"
	EventsStatusDisabled        EventsStatus = "disabled"
	EventsStatusUnavailable     EventsStatus = "unavailable"
)
```

---

## Task 1: Config — the `[calendar_events]` block

**Files:**
- Modify: `config.toml` (insert immediately after the `[calendar_sync]` block, before `[cors]`)
- Modify: `internal/config/config.go`
- Test: `internal/config/config_test.go`

- [ ] **Step 1: Add the TOML block**

In `config.toml`, immediately after `max_attempts = 5` (the last line of `[calendar_sync]`) and before `[cors]`, insert a blank line then:

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

- [ ] **Step 2: Add the typed config**

In `internal/config/config.go`, add the field to `Config` immediately after the `CalendarSync CalendarSyncConfig` field:

```go
	// CalendarEvents configures the READ direction of the calendar
	// integration — the dashboard tile's /me/calendar/events endpoint.
	// Separate from CalendarSync because the two kill switches are
	// genuinely independent. See CalendarEventsConfig.
	CalendarEvents CalendarEventsConfig
```

Add the type immediately after `CalendarSyncConfig`:

```go
// CalendarEventsConfig groups the calendar READ knobs (the dashboard tile).
// All are non-secret public literals, mirroring CalendarSyncConfig — the read
// path shares the write path's one real secret, CALENDAR_TOKEN_ENC_KEY.
//
// Enabled is a kill switch for reads only: false makes every
// /me/calendar/events response status "disabled" without touching OAuth or
// the write path. CacheTTLSeconds bounds how long one user's fetched window
// is reused; it exists to absorb dashboard remounts, not to economise on a
// free quota. MaxEventsPerDay caps the per-day payload; overflow is reported
// as each day's `truncated` count rather than silently dropped.
type CalendarEventsConfig struct {
	Enabled         bool
	CacheTTLSeconds int
	MaxEventsPerDay int
}
```

Add the `fileConfig` block immediately after the `CalendarSync struct { … } \`toml:"calendar_sync"\`` block:

```go
	CalendarEvents struct {
		Enabled         bool `toml:"enabled"`
		CacheTTLSeconds int  `toml:"cache_ttl_seconds"`
		MaxEventsPerDay int  `toml:"max_events_per_day"`
	} `toml:"calendar_events"`
```

Add the mapping immediately after the `CalendarSync: CalendarSyncConfig{…},` literal:

```go
		CalendarEvents: CalendarEventsConfig{
			Enabled:         fc.CalendarEvents.Enabled,
			CacheTTLSeconds: fc.CalendarEvents.CacheTTLSeconds,
			MaxEventsPerDay: fc.CalendarEvents.MaxEventsPerDay,
		},
```

- [ ] **Step 3: Extend the literals test**

`internal/config/config_test.go` has `TestLiteralsDecodeToTypedFields`, which asserts the shipped `config.toml` decodes into the expected struct values (search for `CalendarSync: CalendarSyncConfig{` around line 626 for the shape it uses). Add the matching expectation for `CalendarEvents`:

```go
		CalendarEvents: CalendarEventsConfig{
			Enabled:         true,
			CacheTTLSeconds: 60,
			MaxEventsPerDay: 50,
		},
```

Follow whatever comparison idiom that test already uses — if it compares field-by-field rather than whole structs, add three field assertions instead.

- [ ] **Step 4: Run the tests**

```
go test ./internal/config/...
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add config.toml internal/config/config.go internal/config/config_test.go
git commit -m "feat: add [calendar_events] config block"
```

---

## Task 2: `ListEvents` on the Google Calendar client

**Files:**
- Modify: `internal/calendarsync/client.go`
- Test: `internal/calendarsync/client_test.go`

**Context:** `client.go` already carries `calendarAPIBase` (a package var so tests can repoint it at an `httptest.Server`), the bearer-header `do` helper, and `classifyStatus` (2xx → nil, 404/410 → `ErrEventGone`, 401/403 → `ErrTokenRejected`, else a generic error). Reuse all four. Read the existing `InsertEvent` for the exact idiom before writing this.

- [ ] **Step 1: Write the failing tests**

Add to `internal/calendarsync/client_test.go`. Read the top of that file first — it already has a helper pattern for standing up an `httptest.Server` and repointing `calendarAPIBase`; reuse it rather than writing a second one.

```go
func TestListEvents_QueryParams(t *testing.T) {
	var got url.Values
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		got = r.URL.Query()
		if auth := r.Header.Get("Authorization"); auth != "Bearer tok" {
			t.Errorf("Authorization = %q, want %q", auth, "Bearer tok")
		}
		w.Write([]byte(`{"items":[]}`))
	}))
	defer srv.Close()
	restore := calendarAPIBase
	calendarAPIBase = srv.URL
	defer func() { calendarAPIBase = restore }()

	c := NewGoogleCalendarClient(srv.Client())
	start := time.Date(2026, 8, 10, 4, 0, 0, 0, time.UTC)
	end := time.Date(2026, 8, 18, 4, 0, 0, 0, time.UTC)
	if _, err := c.ListEvents(context.Background(), "tok", "primary", start, end, 400); err != nil {
		t.Fatalf("ListEvents: %v", err)
	}

	// singleEvents=true is load-bearing: without it Google returns the
	// recurring RULE, not its instances, and a daily standup either vanishes
	// or lands on the day the series began. orderBy=startTime is rejected by
	// the API unless singleEvents is set, so the two travel together.
	for k, want := range map[string]string{
		"timeMin":      start.Format(time.RFC3339),
		"timeMax":      end.Format(time.RFC3339),
		"singleEvents": "true",
		"orderBy":      "startTime",
		"showDeleted":  "false",
		"maxResults":   "400",
	} {
		if got.Get(k) != want {
			t.Errorf("query %s = %q, want %q", k, got.Get(k), want)
		}
	}
}

func TestListEvents_ParsesTimedAndAllDay(t *testing.T) {
	body := `{"items":[
		{"id":"t1","summary":"Standup","start":{"dateTime":"2026-08-12T09:00:00-04:00"},"end":{"dateTime":"2026-08-12T09:15:00-04:00"}},
		{"id":"a1","summary":"Holiday","start":{"date":"2026-08-14"},"end":{"date":"2026-08-16"}},
		{"id":"n1","start":{"dateTime":"2026-08-12T13:00:00-04:00"},"end":{"dateTime":"2026-08-12T14:00:00-04:00"}}
	]}`
	events := listEventsWithBody(t, body)

	if len(events) != 3 {
		t.Fatalf("got %d events, want 3", len(events))
	}
	if events[0].AllDay || events[0].Summary != "Standup" {
		t.Errorf("timed event parsed wrong: %+v", events[0])
	}
	if !events[0].Start.Equal(time.Date(2026, 8, 12, 13, 0, 0, 0, time.UTC)) {
		t.Errorf("start = %v, want 13:00Z", events[0].Start)
	}
	if !events[1].AllDay || events[1].StartDate != "2026-08-14" || events[1].EndDate != "2026-08-16" {
		t.Errorf("all-day event parsed wrong: %+v", events[1])
	}
	// A missing summary is preserved as empty — rendering "Busy" is the
	// service's decision, not the transport's.
	if events[2].Summary != "" {
		t.Errorf("summary = %q, want empty", events[2].Summary)
	}
}

func TestListEvents_DeclinedByTheUser(t *testing.T) {
	body := `{"items":[
		{"id":"d1","summary":"Declined","start":{"dateTime":"2026-08-12T09:00:00Z"},"end":{"dateTime":"2026-08-12T10:00:00Z"},
		 "attendees":[{"self":true,"responseStatus":"declined"},{"responseStatus":"accepted"}]},
		{"id":"a2","summary":"Accepted","start":{"dateTime":"2026-08-12T11:00:00Z"},"end":{"dateTime":"2026-08-12T12:00:00Z"},
		 "attendees":[{"self":true,"responseStatus":"accepted"}]},
		{"id":"o1","summary":"Someone else declined","start":{"dateTime":"2026-08-12T13:00:00Z"},"end":{"dateTime":"2026-08-12T14:00:00Z"},
		 "attendees":[{"responseStatus":"declined"}]}
	]}`
	events := listEventsWithBody(t, body)

	if !events[0].Declined {
		t.Error("the user's own declined attendance must set Declined")
	}
	if events[1].Declined {
		t.Error("an accepted event must not be Declined")
	}
	if events[2].Declined {
		t.Error("ANOTHER attendee declining is not the user declining")
	}
}

func TestListEvents_StatusMapping(t *testing.T) {
	for _, tc := range []struct {
		status int
		want   error
	}{
		{http.StatusUnauthorized, ErrTokenRejected},
		{http.StatusForbidden, ErrTokenRejected},
		{http.StatusNotFound, ErrEventGone},
	} {
		srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			w.WriteHeader(tc.status)
		}))
		restore := calendarAPIBase
		calendarAPIBase = srv.URL
		_, err := NewGoogleCalendarClient(srv.Client()).
			ListEvents(context.Background(), "tok", "primary", time.Now(), time.Now(), 10)
		calendarAPIBase = restore
		srv.Close()
		if !errors.Is(err, tc.want) {
			t.Errorf("status %d: err = %v, want %v", tc.status, err, tc.want)
		}
	}
}

// listEventsWithBody stands up a server returning body and returns the parsed
// events.
func listEventsWithBody(t *testing.T, body string) []ListedEvent {
	t.Helper()
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(body))
	}))
	t.Cleanup(srv.Close)
	restore := calendarAPIBase
	calendarAPIBase = srv.URL
	t.Cleanup(func() { calendarAPIBase = restore })

	events, err := NewGoogleCalendarClient(srv.Client()).
		ListEvents(context.Background(), "tok", "primary", time.Now(), time.Now(), 10)
	if err != nil {
		t.Fatalf("ListEvents: %v", err)
	}
	return events
}
```

- [ ] **Step 2: Run to verify it fails**

```
go test ./internal/calendarsync/ -run TestListEvents
```
Expected: FAIL — `c.ListEvents undefined`, `undefined: ListedEvent`.

- [ ] **Step 3: Implement**

In `internal/calendarsync/client.go`, add `ListEvents` to the `CalendarClient` interface:

```go
// CalendarClient reads and writes events on a user's Google calendar.
// Fakeable for tests.
type CalendarClient interface {
	InsertEvent(ctx context.Context, accessToken, calendarID string, ev GoogleEvent) (eventID string, err error)
	PatchEvent(ctx context.Context, accessToken, calendarID, eventID string, ev GoogleEvent) error
	DeleteEvent(ctx context.Context, accessToken, calendarID, eventID string) error
	// ListEvents reads the events between timeMin and timeMax. See the
	// implementation for why singleEvents=true is not optional.
	ListEvents(ctx context.Context, accessToken, calendarID string, timeMin, timeMax time.Time, maxResults int) ([]ListedEvent, error)
}
```

Add the types and the method:

```go
// ListedEvent is one event as events.list returned it, normalized just enough
// that the service never touches Google's wire shape.
//
// Google models all-day events as CALENDAR DAYS (`date`, with an exclusive
// end) and timed events as instants (`dateTime`). The two are kept apart here
// rather than flattened into a single instant pair: an all-day event has no
// time zone of its own, so converting it to an instant means inventing one,
// and a multi-day all-day event has to be expanded across the days it covers
// by date arithmetic, not by clock arithmetic.
type ListedEvent struct {
	ID      string
	Summary string
	AllDay  bool
	// Timed events only, in UTC.
	Start time.Time
	End   time.Time
	// All-day events only, YYYY-MM-DD. EndDate is EXCLUSIVE, as Google sends it.
	StartDate string
	EndDate   string
	// Declined reports that THIS user's own attendee entry says "declined".
	// Another attendee declining is not this user declining.
	Declined bool
}

// listedEventTime is Google's {dateTime | date} union on an event's start/end.
type listedEventTime struct {
	DateTime string `json:"dateTime"`
	Date     string `json:"date"`
}

type listedAttendee struct {
	Self           bool   `json:"self"`
	ResponseStatus string `json:"responseStatus"`
}

type listedEventBody struct {
	ID        string           `json:"id"`
	Summary   string           `json:"summary"`
	Start     listedEventTime  `json:"start"`
	End       listedEventTime  `json:"end"`
	Attendees []listedAttendee `json:"attendees"`
}

type listResponse struct {
	Items []listedEventBody `json:"items"`
}

// ListEvents reads the user's events in [timeMin, timeMax).
//
// singleEvents=true is LOAD-BEARING and is the parameter most likely to be
// dropped by someone simplifying this query. Without it Google returns the
// recurring RULE — one event carrying an RRULE and the series' original start
// date — so a daily standup either vanishes from the tile or appears on the
// day the series began, once. orderBy=startTime is rejected by the API unless
// singleEvents is set, so the two travel together.
//
// A 401/403 maps to ErrTokenRejected exactly as the write path's calls do; the
// caller decides whether that should revoke the connection.
func (c *googleCalendarClient) ListEvents(ctx context.Context, accessToken, calendarID string, timeMin, timeMax time.Time, maxResults int) ([]ListedEvent, error) {
	q := url.Values{}
	q.Set("timeMin", timeMin.Format(time.RFC3339))
	q.Set("timeMax", timeMax.Format(time.RFC3339))
	q.Set("singleEvents", "true")
	q.Set("orderBy", "startTime")
	q.Set("showDeleted", "false")
	q.Set("maxResults", strconv.Itoa(maxResults))

	u := fmt.Sprintf("%s/calendars/%s/events?%s", calendarAPIBase, url.PathEscape(calendarID), q.Encode())
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, u, nil)
	if err != nil {
		return nil, fmt.Errorf("calendarsync: build list request: %w", err)
	}
	resp, err := c.do(req, accessToken)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	if err := classifyStatus(resp.StatusCode); err != nil {
		return nil, err
	}
	var out listResponse
	if err := json.NewDecoder(resp.Body).Decode(&out); err != nil {
		return nil, fmt.Errorf("calendarsync: decode list response: %w", err)
	}

	events := make([]ListedEvent, 0, len(out.Items))
	for _, item := range out.Items {
		ev, ok := toListedEvent(item)
		if !ok {
			continue
		}
		events = append(events, ev)
	}
	return events, nil
}

// toListedEvent normalizes one wire item. It reports false for an event with
// neither a dateTime nor a date on its start — a shape Google should never
// send, and one with nowhere to be rendered.
func toListedEvent(item listedEventBody) (ListedEvent, bool) {
	ev := ListedEvent{ID: item.ID, Summary: item.Summary}
	for _, a := range item.Attendees {
		if a.Self && a.ResponseStatus == "declined" {
			ev.Declined = true
			break
		}
	}
	switch {
	case item.Start.Date != "":
		ev.AllDay = true
		ev.StartDate = item.Start.Date
		ev.EndDate = item.End.Date
	case item.Start.DateTime != "":
		start, err := time.Parse(time.RFC3339, item.Start.DateTime)
		if err != nil {
			return ListedEvent{}, false
		}
		ev.Start = start.UTC()
		// A missing end is not fatal: treat a zero-length event as ending
		// when it starts rather than dropping it off the tile.
		ev.End = ev.Start
		if item.End.DateTime != "" {
			if end, endErr := time.Parse(time.RFC3339, item.End.DateTime); endErr == nil {
				ev.End = end.UTC()
			}
		}
	default:
		return ListedEvent{}, false
	}
	return ev, true
}
```

Add `strconv` to the import block.

- [ ] **Step 4: Run the tests**

```
go test ./internal/calendarsync/ -run TestListEvents -v
```
Expected: PASS. Then `go test ./internal/calendarsync/...` — the whole package, since any test fake implementing `CalendarClient` now needs the new method. Add `ListEvents` to every existing fake in the package's tests (search for `InsertEvent(ctx context.Context` in `*_test.go`); a fake that does not exercise reads returns `(nil, nil)`.

- [ ] **Step 5: Commit**

```bash
git add internal/calendarsync/client.go internal/calendarsync/client_test.go
git commit -m "feat: add ListEvents to the Google Calendar client"
```

---

## Task 3: The link lookup and the TTL cache

**Files:**
- Create: `internal/calendarsync/events_links.go`
- Create: `internal/calendarsync/events_links_sqlite.go`
- Create: `internal/calendarsync/events_cache.go`
- Test: `internal/calendarsync/events_links_sqlite_test.go`
- Test: `internal/calendarsync/events_cache_test.go`

**Context:** Two tables already store the Google ids Prog Strength wrote: `planned_workouts.google_event_id` (migration 025, with `scheduled_start_utc` and a `deleted_at` soft-delete column, indexed on `(user_id, scheduled_start_utc)`) and `activity_calendar_sync.google_event_id` (migration 053, indexed on `(user_id, sync_status)`, no time column). **No migration is needed.** Read `internal/calendarsync/activity_sync_sqlite.go` for the repository idiom this package uses; read `internal/db/dbtest` for how package tests get a migrated SQLite handle.

- [ ] **Step 1: Write `events_links.go`**

```go
package calendarsync

import "context"

// Link kinds. These are the wire values of a marked event's `link.kind`.
const (
	LinkKindPlannedWorkout = "planned_workout"
	LinkKindActivity       = "activity"
)

// EventLink points a Google event back at the Prog Strength row that wrote it.
type EventLink struct {
	Kind string
	ID   string
}

// EventLinkRepository answers "which of these Google event ids are ours, and
// what do they point at".
//
// Marking is an ID-SET LOOKUP and never a title match. Title matching is the
// tempting shortcut and it is wrong in both directions: a user who renames our
// event in Google loses its mark, and a user who names their own event "Upper
// Body Push" gets a deep link to a planned workout that is not theirs.
type EventLinkRepository interface {
	// LinksForUser returns the user's Google event ids mapped to what they
	// point at. from/to bound the planned-workout side by scheduled start;
	// the activity side has no time column and is bounded by user alone.
	// Extra entries are harmless — the map is only ever probed by id.
	LinksForUser(ctx context.Context, userID string, from, to time.Time) (map[string]EventLink, error)
}
```

(Add `"time"` to that import block.)

- [ ] **Step 2: Write the failing repository test**

`internal/calendarsync/events_links_sqlite_test.go`:

```go
func TestLinksForUser(t *testing.T) {
	db := dbtest.NewDB(t) // use whatever helper the package's other sqlite tests use
	ctx := context.Background()
	now := time.Date(2026, 8, 12, 12, 0, 0, 0, time.UTC)

	mustExec(t, db, `INSERT INTO planned_workouts
		(id, user_id, name, activity_kind, scheduled_start_utc, scheduled_end_utc, timezone, status, google_event_id, created_at, updated_at)
		VALUES ('pw_1','u1','Upper Push','lift',?,?,'UTC','planned','g_pw1',?,?)`,
		now, now.Add(time.Hour), now, now)
	// Soft-deleted plans must not mark anything.
	mustExec(t, db, `INSERT INTO planned_workouts
		(id, user_id, name, activity_kind, scheduled_start_utc, scheduled_end_utc, timezone, status, google_event_id, created_at, updated_at, deleted_at)
		VALUES ('pw_2','u1','Deleted','lift',?,?,'UTC','planned','g_pw2',?,?,?)`,
		now, now.Add(time.Hour), now, now, now)
	// Another user's plan must not leak.
	mustExec(t, db, `INSERT INTO planned_workouts
		(id, user_id, name, activity_kind, scheduled_start_utc, scheduled_end_utc, timezone, status, google_event_id, created_at, updated_at)
		VALUES ('pw_3','u2','Theirs','lift',?,?,'UTC','planned','g_pw3',?,?)`,
		now, now.Add(time.Hour), now, now)
	mustExec(t, db, `INSERT INTO activity_calendar_sync
		(activity_id, user_id, google_event_id, sync_status, attempts, updated_at)
		VALUES ('act_1','u1','g_act1','synced',0,?)`, now)
	// A row that never got an event id is not a link.
	mustExec(t, db, `INSERT INTO activity_calendar_sync
		(activity_id, user_id, google_event_id, sync_status, attempts, updated_at)
		VALUES ('act_2','u1',NULL,'pending',0,?)`, now)

	repo := NewSQLiteEventLinkRepository(db)
	links, err := repo.LinksForUser(ctx, "u1", now.Add(-24*time.Hour), now.Add(24*time.Hour))
	if err != nil {
		t.Fatalf("LinksForUser: %v", err)
	}

	want := map[string]EventLink{
		"g_pw1":  {Kind: LinkKindPlannedWorkout, ID: "pw_1"},
		"g_act1": {Kind: LinkKindActivity, ID: "act_1"},
	}
	if !reflect.DeepEqual(links, want) {
		t.Errorf("links = %#v, want %#v", links, want)
	}
}

func TestLinksForUser_WindowExcludesFarPlans(t *testing.T) {
	db := dbtest.NewDB(t)
	now := time.Date(2026, 8, 12, 12, 0, 0, 0, time.UTC)
	mustExec(t, db, `INSERT INTO planned_workouts
		(id, user_id, name, activity_kind, scheduled_start_utc, scheduled_end_utc, timezone, status, google_event_id, created_at, updated_at)
		VALUES ('pw_far','u1','Next month','lift',?,?,'UTC','planned','g_far',?,?)`,
		now.AddDate(0, 1, 0), now.AddDate(0, 1, 0).Add(time.Hour), now, now)

	links, err := NewSQLiteEventLinkRepository(db).
		LinksForUser(context.Background(), "u1", now, now.Add(48*time.Hour))
	if err != nil {
		t.Fatalf("LinksForUser: %v", err)
	}
	if _, ok := links["g_far"]; ok {
		t.Error("a plan outside the window must not be in the link map")
	}
}
```

Write `mustExec` as a small `t.Helper()` wrapper over `db.Exec` that fails the test on error, unless the package's test files already have one — check first.

- [ ] **Step 3: Implement `events_links_sqlite.go`**

```go
package calendarsync

import (
	"context"
	"database/sql"
	"time"
)

var _ EventLinkRepository = (*SQLiteEventLinkRepository)(nil)

// SQLiteEventLinkRepository reads the two tables that already record what
// Prog Strength wrote to Google: planned_workouts.google_event_id (migration
// 025) and activity_calendar_sync.google_event_id (migration 053). It adds no
// schema of its own.
type SQLiteEventLinkRepository struct {
	db *sql.DB
}

func NewSQLiteEventLinkRepository(db *sql.DB) *SQLiteEventLinkRepository {
	return &SQLiteEventLinkRepository{db: db}
}

// LinksForUser runs one query per table.
//
// The plan side is bounded by scheduled start, which rides the existing
// (user_id, scheduled_start_utc) index. The activity side is bounded by user
// alone: activity_calendar_sync carries no timestamp of the session it
// describes, and joining `activities` to get one would buy a narrower map
// that is probed only by id anyway. Its (user_id, sync_status) index makes
// the user-scoped read cheap, and a user has one row per synced activity.
func (r *SQLiteEventLinkRepository) LinksForUser(ctx context.Context, userID string, from, to time.Time) (map[string]EventLink, error) {
	links := make(map[string]EventLink)

	planRows, err := r.db.QueryContext(ctx, `
		SELECT id, google_event_id
		FROM planned_workouts
		WHERE user_id = ?
		  AND google_event_id IS NOT NULL AND google_event_id != ''
		  AND deleted_at IS NULL
		  AND scheduled_start_utc >= ? AND scheduled_start_utc < ?
	`, userID, from.UTC(), to.UTC())
	if err != nil {
		return nil, err
	}
	defer planRows.Close()
	for planRows.Next() {
		var id, eventID string
		if err := planRows.Scan(&id, &eventID); err != nil {
			return nil, err
		}
		links[eventID] = EventLink{Kind: LinkKindPlannedWorkout, ID: id}
	}
	if err := planRows.Err(); err != nil {
		return nil, err
	}

	actRows, err := r.db.QueryContext(ctx, `
		SELECT activity_id, google_event_id
		FROM activity_calendar_sync
		WHERE user_id = ?
		  AND google_event_id IS NOT NULL AND google_event_id != ''
	`, userID)
	if err != nil {
		return nil, err
	}
	defer actRows.Close()
	for actRows.Next() {
		var id, eventID string
		if err := actRows.Scan(&id, &eventID); err != nil {
			return nil, err
		}
		// A plan that handed its event over to the activity that completed it
		// leaves BOTH rows pointing at the same id. The activity is what
		// actually happened, so it wins the link.
		links[eventID] = EventLink{Kind: LinkKindActivity, ID: id}
	}
	return links, actRows.Err()
}
```

- [ ] **Step 4: Write the failing cache test**

`internal/calendarsync/events_cache_test.go`:

```go
func TestEventsCache_HitInsideTTL(t *testing.T) {
	now := time.Date(2026, 8, 12, 9, 0, 0, 0, time.UTC)
	c := newEventsCache(60*time.Second, func() time.Time { return now })
	key := eventsCacheKey{UserID: "u1", Start: "2026-08-10", End: "2026-08-17", Timezone: "UTC"}
	days := []Day{{Date: "2026-08-10"}}

	c.put(key, days)
	if _, ok := c.get(key); !ok {
		t.Fatal("a read inside the TTL must hit")
	}

	now = now.Add(61 * time.Second)
	if _, ok := c.get(key); ok {
		t.Error("a read past the TTL must miss")
	}
}

func TestEventsCache_IsolatesUsersAndWindows(t *testing.T) {
	now := time.Date(2026, 8, 12, 9, 0, 0, 0, time.UTC)
	c := newEventsCache(60*time.Second, func() time.Time { return now })
	base := eventsCacheKey{UserID: "u1", Start: "2026-08-10", End: "2026-08-17", Timezone: "UTC"}
	c.put(base, []Day{{Date: "2026-08-10"}})

	other := base
	other.UserID = "u2"
	if _, ok := c.get(other); ok {
		t.Error("two users must never share a cache entry")
	}
	window := base
	window.End = "2026-08-18"
	if _, ok := c.get(window); ok {
		t.Error("a different window must not read another window's entry")
	}
	zone := base
	zone.Timezone = "America/New_York"
	if _, ok := c.get(zone); ok {
		t.Error("a different timezone groups days differently and must not share")
	}
}

func TestEventsCache_ReturnsACopy(t *testing.T) {
	now := time.Now()
	c := newEventsCache(time.Minute, func() time.Time { return now })
	key := eventsCacheKey{UserID: "u1", Start: "a", End: "b", Timezone: "UTC"}
	c.put(key, []Day{{Date: "2026-08-10", Events: []Event{{ID: "x"}}}})

	got, _ := c.get(key)
	got[0].Events[0].ID = "mutated"

	again, _ := c.get(key)
	if again[0].Events[0].ID != "x" {
		t.Error("a caller mutating its result must not corrupt the cache")
	}
}
```

- [ ] **Step 5: Implement `events_cache.go`**

```go
package calendarsync

import (
	"sync"
	"time"
)

// eventsCacheKey identifies one user's fetched window. Every field is part of
// the identity: the timezone decides which local day an instant lands on, so
// two zones over the same dates are genuinely different answers.
type eventsCacheKey struct {
	UserID   string
	Start    string
	End      string
	Timezone string
}

type eventsCacheEntry struct {
	days     []Day
	storedAt time.Time
}

// eventsCache is a per-user in-memory TTL cache and nothing else.
//
// Its only job is to keep dashboard remounts and slide changes off Google.
// There is deliberately no durable cache, no budget ledger, no daily ceiling,
// and no stale serving — all of which the weather integration carries because
// OpenWeather BILLS PER CALL. Google Calendar's API is free at a quota of one
// million queries a day, so a cache miss here is a free retry and importing
// that machinery would be cargo-culting a solution to a problem this
// integration does not have.
type eventsCache struct {
	ttl time.Duration
	now func() time.Time

	mu      sync.Mutex
	entries map[eventsCacheKey]eventsCacheEntry
}

func newEventsCache(ttl time.Duration, now func() time.Time) *eventsCache {
	if now == nil {
		now = time.Now
	}
	return &eventsCache{ttl: ttl, now: now, entries: make(map[eventsCacheKey]eventsCacheEntry)}
}

// get returns a deep copy of the cached days, or false when absent or expired.
// The copy is the point: the caller renders and may sort or truncate what it
// gets back, and a shared slice would let one request corrupt the next.
func (c *eventsCache) get(key eventsCacheKey) ([]Day, bool) {
	if c == nil || c.ttl <= 0 {
		return nil, false
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	entry, ok := c.entries[key]
	if !ok {
		return nil, false
	}
	if c.now().Sub(entry.storedAt) >= c.ttl {
		delete(c.entries, key)
		return nil, false
	}
	return cloneDays(entry.days), true
}

func (c *eventsCache) put(key eventsCacheKey, days []Day) {
	if c == nil || c.ttl <= 0 {
		return
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	c.entries[key] = eventsCacheEntry{days: cloneDays(days), storedAt: c.now()}
}

func cloneDays(days []Day) []Day {
	out := make([]Day, len(days))
	for i, d := range days {
		out[i] = d
		out[i].Events = append([]Event(nil), d.Events...)
	}
	return out
}
```

Note this references `Day` and `Event`, which Task 4 defines. Implement Task 4's type block first if the compiler complains — or, if working strictly in order, add the `Day`/`Event`/`EventsStatus` type declarations from Task 4's Step 3 now and leave the service for Task 4.

- [ ] **Step 6: Run the tests**

```
go test ./internal/calendarsync/ -run 'TestLinksForUser|TestEventsCache' -v
```
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add internal/calendarsync/events_links.go internal/calendarsync/events_links_sqlite.go \
        internal/calendarsync/events_links_sqlite_test.go \
        internal/calendarsync/events_cache.go internal/calendarsync/events_cache_test.go
git commit -m "feat: add calendar event link lookup and read cache"
```

---

## Task 4: `EventsService` and its metrics

**Files:**
- Create: `internal/calendarsync/events_service.go`
- Modify: `internal/calendarsync/metrics.go`
- Test: `internal/calendarsync/events_service_test.go`

**Context:** This is the core of the API side. It resolves the grant through the existing unexported `connector` (which already flips a connection to revoked when a REFRESH is rejected), calls `ListEvents`, filters, marks, and groups by local date. Read `internal/calendarsync/connector.go` and `internal/calendarsync/service.go` before starting — `Service.recordFailure` shows the existing `ErrTokenRejected` → `conns.SetStatus(revoked)` handling this must mirror for the LIST call.

- [ ] **Step 1: Write the file header and types**

`internal/calendarsync/events_service.go` starts with:

```go
// EventsService reads a user's Google Calendar for the dashboard tile.
//
// WHY THIS LIVES IN calendarsync AND NOT A NEW PACKAGE. The unexported
// `connector` performs the five-step dance any Google call needs — load the
// connection, reject a revoked one, fetch the encrypted refresh token, decrypt
// it, mint an access token — plus the failure handling around it that is easy
// to get wrong: a refresh Google rejects must flip the connection to revoked,
// or the UI never learns to prompt for re-consent. Its own doc comment records
// that it was extracted from Service precisely because a second consumer
// appeared. A third consumer is the same argument. Putting the reader
// elsewhere would mean exporting `connector`, `grant`, and the
// revoke-on-refresh-failure behavior across a package boundary, in exchange
// for a package name that reads slightly better.
//
// NO NEW SCOPE. The granted scope is CalendarEventsScope
// (calendar.events), which is read AND write on events — events.list is
// already permitted — and the connection already resolves to `primary`. Every
// user connected today gets this the moment it deploys: no re-consent, no
// migration, no new secret.
//
// WHY A 429 MUST NOT REVOKE. Google rate-limits. A rate limit is not a revoked
// grant, and flipping the connection over one would present a re-consent
// prompt to a user whose authorisation is perfectly good — and, because the
// write path shares that connection row, would ALSO stop their planned-workout
// sync. Only ErrTokenRejected (401/403) touches the connection; everything
// else degrades to "unavailable" and leaves it alone.
package calendarsync
```

(The `package` clause goes after the comment block only if the comment is intended as a package doc — it is not; put the comment immediately above `type EventsService struct` instead, and let the file open with a plain `package calendarsync`.)

Types:

```go
// EventsStatus is the closed degradation set, mirroring weather's vocabulary.
// Every value renders as a calm muted line or a CTA on the tile — never an
// error banner.
type EventsStatus string

const (
	EventsStatusOK              EventsStatus = "ok"
	EventsStatusNotConnected    EventsStatus = "not_connected"
	EventsStatusReconnectNeeded EventsStatus = "reconnect_needed"
	EventsStatusDisabled        EventsStatus = "disabled"
	EventsStatusUnavailable     EventsStatus = "unavailable"
)

// Event sources. prog_strength marks an event Prog Strength itself wrote.
const (
	EventSourceProgStrength = "prog_strength"
	EventSourceGoogle       = "google"
)

// busyTitle is what an event Google returned with no summary renders as — a
// private event on a shared calendar, or a busy block. The user cannot see the
// title in Google either, so the tile should say so rather than draw an empty
// row.
const busyTitle = "Busy"

// Event is one calendar entry on the wire.
type Event struct {
	ID     string
	Title  string
	Start  time.Time
	End    time.Time
	AllDay bool
	Source string
	Link   *EventLink
}

// Day is one local calendar date. days is DENSE — every date in the requested
// window appears, with an empty Events slice for a free day — because a client
// that has to distinguish "no events" from "day missing from the payload" will
// eventually get it wrong, and the week strip needs seven columns regardless.
type Day struct {
	Date string
	// Truncated is how many events MaxEventsPerDay cut from this day. It
	// exists so the cap is never silent: the week strip prints
	// len(events)+truncated, so a conference day with 60 entries reports sixty
	// rather than the fifty that fit.
	Truncated int
	Events    []Event
}

// EventsResult is what the handler renders.
type EventsResult struct {
	Status EventsStatus
	Days   []Day
}

// EventsConfig is the [calendar_events] block, injected rather than imported
// so this package keeps its zero dependency on internal/config.
type EventsConfig struct {
	Enabled         bool
	CacheTTL        time.Duration
	MaxEventsPerDay int
}

// EventsWindow is one request's resolved date contract: the UTC half-open
// interval bracketing the user's local days, the zone those days are in, and
// the YYYY-MM-DD bounds themselves.
//
// The client NEVER computes timeMin/timeMax. internal/daterange's package doc
// explains why at length, and the DST case is not hypothetical here: a "week"
// is 167 or 169 hours twice a year, and a reader that assumes 7*24 drops or
// duplicates a day's events for every user not on UTC.
type EventsWindow struct {
	StartUTC  time.Time
	EndUTC    time.Time
	Loc       *time.Location
	StartDate string
	EndDate   string
}

// EventsServiceDeps carries the service's collaborators, mirroring
// ActivityServiceDeps.
type EventsServiceDeps struct {
	Conns  calendarconn.Repository
	Cipher *tokencrypt.Cipher
	Tokens *TokenSource
	Client CalendarClient
	Links  EventLinkRepository
	Config EventsConfig
	Now    func() time.Time
}

type EventsService struct {
	conns  calendarconn.Repository
	conn   *connector
	client CalendarClient
	links  EventLinkRepository
	cache  *eventsCache
	cfg    EventsConfig
	now    func() time.Time
}

func NewEventsService(d EventsServiceDeps) *EventsService {
	now := d.Now
	if now == nil {
		now = time.Now
	}
	return &EventsService{
		conns:  d.Conns,
		conn:   &connector{conns: d.Conns, cipher: d.Cipher, tokens: d.Tokens, now: now},
		client: d.Client,
		links:  d.Links,
		cache:  newEventsCache(d.Config.CacheTTL, now),
		cfg:    d.Config,
		now:    now,
	}
}
```

- [ ] **Step 2: Write the failing service tests**

`internal/calendarsync/events_service_test.go`. Build a harness that stands up an `httptest.Server` for `calendarAPIBase` plus a fake token endpoint, an in-memory `calendarconn.Repository` (the package's existing tests already have one — reuse it, do not write a second), and a stub `EventLinkRepository`. Every test below is required by the SOW.

```go
func TestEvents_WindowFromDaterange_DSTWeek(t *testing.T)
// America/New_York, start_date 2026-11-01 (the US DST fall-back), end_date
// 2026-11-08. Assert the outgoing timeMin/timeMax are exactly the UTC instants
// daterange.DayBoundsUTC yields for those dates in that zone — i.e. the window
// spans 169 hours, not 168. THIS IS THE TEST THAT FAILS if anyone
// reintroduces `24 * 7 * time.Hour`.

func TestEvents_QueryCarriesSingleEventsAndOrderBy(t *testing.T)
// singleEvents=true and orderBy=startTime are both on the outgoing query.

func TestEvents_DropsDeclinedKeepsAcceptedAndNeedsAction(t *testing.T)
// A declined attendee's event is filtered; an accepted one and a
// needs-action one are not.

func TestEvents_NoSummaryRendersBusy(t *testing.T)
// An event with no summary comes back titled "Busy".

func TestEvents_MultiDayAllDaySpansEveryDay(t *testing.T)
// An all-day event with start.date 2026-08-14 and end.date 2026-08-16 appears
// on 08-14 and 08-15 and NOT on 08-16 (Google's end is exclusive). A timed
// event appears only on its start day.

func TestEvents_MarksOurOwnByID(t *testing.T)
// An id in the planned-workout link map is source prog_strength with a
// planned_workout link; one in the activity map carries an activity link; an
// unknown id is google with no link.

func TestEvents_TitleMatchDoesNotMark(t *testing.T)
// An event titled EXACTLY like a planned workout but with an unknown id is
// source "google" with no link. This is the pin against title matching.

func TestEvents_DaysAreDense(t *testing.T)
// A window with no events at all returns one entry per date, each with an
// empty (non-nil) Events slice.

func TestEvents_TruncatesAndReports(t *testing.T)
// With MaxEventsPerDay = 2 and a day carrying 5 events, that day returns 2
// events and Truncated == 3.

func TestEvents_401RevokesConnectionAndReportsReconnect(t *testing.T)
// Google 401 → the stored connection status becomes revoked, and the result
// status is reconnect_needed.

func TestEvents_429LeavesConnectionUntouched(t *testing.T)
// Google 429 → the stored connection status is UNCHANGED (still connected)
// and the result status is unavailable. Assert the status explicitly rather
// than asserting SetStatus was not called, so the test survives a refactor of
// how the flip happens.

func TestEvents_5xxIsUnavailableAndLeavesConnection(t *testing.T)
// Same as 429 for a 500 and for a transport error.

func TestEvents_NoConnectionRowMakesNoGoogleCall(t *testing.T)
// status not_connected, and the httptest server recorded ZERO requests.

func TestEvents_DisabledMakesNoGoogleCall(t *testing.T)
// EventsConfig.Enabled = false → status disabled, zero requests.

func TestEvents_CacheServesInsideTTLAndRefetchesAfter(t *testing.T)
// Two reads inside the TTL make one Google call; a read after it makes two.

func TestEvents_CacheIsPerUser(t *testing.T)
// Two different users never share a cache entry (two calls, both populated).

func TestEvents_ReaderRequestsOnlyTheEventsScope(t *testing.T) {
	cfg := NewCalendarConfig("id", "secret", "https://example.test/cb")
	if len(cfg.Scopes) != 1 || cfg.Scopes[0] != CalendarEventsScope {
		t.Fatalf("scopes = %v, want exactly [%s]", cfg.Scopes, CalendarEventsScope)
	}
}
// The pin that this feature never silently grows a scope. calendar.events is
// read AND write on events, which is why the tile ships without re-consent;
// reading WHICH calendars a user has would need calendar.readonly and a
// re-consent for every existing connected user.
```

- [ ] **Step 3: Run to verify they fail**

```
go test ./internal/calendarsync/ -run TestEvents
```
Expected: FAIL — `undefined: NewEventsService`.

- [ ] **Step 4: Implement the service**

```go
// Events reads the user's calendar for the given window.
//
// It never returns an error: every failure is a status the tile can render
// calmly. not_connected in particular is the ORDINARY state of a user who
// never opted in — the tile's job there is to invite, not to handle an error —
// which mirrors ErrNotConnected's existing treatment in the write path, where
// it is explicitly not an error condition at all.
func (s *EventsService) Events(ctx context.Context, userID string, w EventsWindow) EventsResult {
	if !s.cfg.Enabled {
		observeEventRead(string(EventsStatusDisabled), cacheMiss, 0)
		return EventsResult{Status: EventsStatusDisabled}
	}

	key := eventsCacheKey{UserID: userID, Start: w.StartDate, End: w.EndDate, Timezone: w.Loc.String()}
	if days, ok := s.cache.get(key); ok {
		observeEventRead(string(EventsStatusOK), cacheHit, 0)
		return EventsResult{Status: EventsStatusOK, Days: days}
	}

	started := s.now()
	days, status := s.fetch(ctx, userID, w)
	observeEventRead(string(status), cacheMiss, s.now().Sub(started).Seconds())
	if status != EventsStatusOK {
		return EventsResult{Status: status}
	}
	s.cache.put(key, days)
	return EventsResult{Status: EventsStatusOK, Days: days}
}

// fetch does the uncached work: resolve the grant, list, filter, mark, group.
func (s *EventsService) fetch(ctx context.Context, userID string, w EventsWindow) ([]Day, EventsStatus) {
	g, err := s.conn.resolve(ctx, userID)
	if err != nil {
		switch {
		case errors.Is(err, ErrNotConnected):
			return nil, EventsStatusNotConnected
		case errors.Is(err, ErrReconnectNeeded):
			// connector.resolve has already flipped the connection when the
			// refresh itself was rejected.
			return nil, EventsStatusReconnectNeeded
		default:
			return nil, EventsStatusUnavailable
		}
	}

	maxResults := s.maxResults(w)
	raw, err := s.client.ListEvents(ctx, g.AccessToken, g.CalendarID, w.StartUTC, w.EndUTC, maxResults)
	if err != nil {
		if errors.Is(err, ErrTokenRejected) {
			// 401/403 only. This is the one failure that means the grant is
			// gone; see the type's doc comment for why a 429 must not land here.
			_ = s.conns.SetStatus(ctx, userID, calendarconn.StatusRevoked, s.now())
			return nil, EventsStatusReconnectNeeded
		}
		return nil, EventsStatusUnavailable
	}

	// A link lookup failure degrades to an UNMARKED tile rather than no tile:
	// the events are the payload, the provenance mark is a garnish.
	links, err := s.links.LinksForUser(ctx, userID, w.StartUTC, w.EndUTC)
	if err != nil {
		links = nil
	}

	return s.group(raw, links, w), EventsStatusOK
}

// maxResults bounds the Google page. Asking for the per-day cap across every
// day in the window means the per-day cap is what actually truncates, so the
// `truncated` counts stay meaningful; the ceiling is Google's own maximum.
func (s *EventsService) maxResults(w EventsWindow) int {
	days := len(datesInWindow(w))
	n := s.cfg.MaxEventsPerDay * days
	if n < 1 {
		n = 1
	}
	if n > googleMaxResults {
		n = googleMaxResults
	}
	return n
}

// googleMaxResults is the largest page events.list will return.
const googleMaxResults = 2500
```

Grouping:

```go
// group buckets the listed events into dense local days.
func (s *EventsService) group(raw []ListedEvent, links map[string]EventLink, w EventsWindow) []Day {
	dates := datesInWindow(w)
	byDate := make(map[string][]Event, len(dates))

	for _, ev := range raw {
		// A meeting the user declined is one they are not attending; it is
		// noise on a tile this small. (Cancelled events never arrive —
		// showDeleted=false.)
		if ev.Declined {
			continue
		}
		out := Event{
			ID:     ev.ID,
			Title:  ev.Summary,
			AllDay: ev.AllDay,
			Source: EventSourceGoogle,
		}
		if out.Title == "" {
			out.Title = busyTitle
		}
		if link, ok := links[ev.ID]; ok {
			out.Source = EventSourceProgStrength
			l := link
			out.Link = &l
		}

		if ev.AllDay {
			// Google's end.date is EXCLUSIVE, so a 14th→16th all-day event
			// covers the 14th and the 15th.
			for _, date := range allDayDates(ev, w.Loc) {
				day := out
				start, end, err := daterange.DayBoundsUTC(date, w.Loc)
				if err != nil {
					continue
				}
				day.Start, day.End = start, end
				byDate[date] = append(byDate[date], day)
			}
			continue
		}
		out.Start, out.End = ev.Start, ev.End
		byDate[out.Start.In(w.Loc).Format(dateLayout)] = append(byDate[out.Start.In(w.Loc).Format(dateLayout)], out)
	}

	days := make([]Day, 0, len(dates))
	for _, date := range dates {
		events := byDate[date]
		sortDayEvents(events)
		truncated := 0
		if s.cfg.MaxEventsPerDay > 0 && len(events) > s.cfg.MaxEventsPerDay {
			truncated = len(events) - s.cfg.MaxEventsPerDay
			events = events[:s.cfg.MaxEventsPerDay]
		}
		if events == nil {
			// Dense, and non-null on the wire.
			events = []Event{}
		}
		days = append(days, Day{Date: date, Truncated: truncated, Events: events})
	}
	return days
}

const dateLayout = "2006-01-02"

// datesInWindow lists every local calendar date the window covers, inclusive
// of both ends. Iteration is by AddDate in the user's zone, never by adding
// 24h — a DST day is 23 or 25 hours long.
func datesInWindow(w EventsWindow) []string {
	start, err := time.ParseInLocation(dateLayout, w.StartDate, w.Loc)
	if err != nil {
		return nil
	}
	end, err := time.ParseInLocation(dateLayout, w.EndDate, w.Loc)
	if err != nil {
		return nil
	}
	var dates []string
	for d := start; !d.After(end); d = d.AddDate(0, 0, 1) {
		dates = append(dates, d.Format(dateLayout))
	}
	return dates
}

// allDayDates expands an all-day event across the dates it covers.
func allDayDates(ev ListedEvent, loc *time.Location) []string {
	start, err := time.ParseInLocation(dateLayout, ev.StartDate, loc)
	if err != nil {
		return nil
	}
	end, err := time.ParseInLocation(dateLayout, ev.EndDate, loc)
	if err != nil || !end.After(start) {
		// A malformed or single-day span still shows on its start date.
		return []string{ev.StartDate}
	}
	var dates []string
	for d := start; d.Before(end); d = d.AddDate(0, 0, 1) {
		dates = append(dates, d.Format(dateLayout))
	}
	return dates
}

// sortDayEvents pins all-day events to the top, then orders the rest by start.
// The tile renders in this order and the truncation cap slices off the end, so
// the sort decides what a capped day keeps.
func sortDayEvents(events []Event) {
	sort.SliceStable(events, func(i, j int) bool {
		if events[i].AllDay != events[j].AllDay {
			return events[i].AllDay
		}
		return events[i].Start.Before(events[j].Start)
	})
}
```

Imports: `context`, `errors`, `sort`, `time`, `calendarconn`, `daterange`, `tokencrypt`.

- [ ] **Step 5: Add the metrics**

Append to `internal/calendarsync/metrics.go`:

```go
// api_calendar_event_reads_total counts every read of a user's calendar for
// the dashboard tile, by outcome and by whether the cache answered it.
//
// cache is split out because it is the only thing that makes the Google-call
// rate interpretable: a healthy tile serves most reads from the 60s TTL, and a
// sudden collapse in the cache-hit ratio means remounts, not usage.
var eventReadsTotal = prometheus.NewCounterVec(
	prometheus.CounterOpts{
		Name: "api_calendar_event_reads_total",
		Help: "Google Calendar reads for the dashboard tile by result and cache outcome.",
	},
	[]string{"result", "cache"},
)

// api_calendar_event_read_seconds times the uncached read path.
//
// DELIBERATELY NO last-success stamp and no absence-of-success alert, in
// contrast to calendarconn.Connection.LastSuccessfulSyncAt. That stamp exists
// because the WRITE path is a background process whose silence is a symptom.
// This read path is USER-TRIGGERED: a day with no reads means nobody opened a
// dashboard, which is a fact about traffic, not a fault. Alerting on its
// absence would reproduce exactly the false-positive pattern that stamp was
// introduced to fix.
var eventReadDuration = prometheus.NewHistogram(
	prometheus.HistogramOpts{
		Name:    "api_calendar_event_read_seconds",
		Help:    "Duration of an uncached Google Calendar read for the dashboard tile.",
		Buckets: prometheus.DefBuckets,
	},
)

// Cache label values, closed like every other label set here.
const (
	cacheHit  = "hit"
	cacheMiss = "miss"
)

// observeEventRead records one read. seconds is 0 for a read the cache or a
// short-circuit answered, and those are not timed — the histogram measures the
// Google round-trip, and folding in cache hits would flatter it into
// uselessness.
func observeEventRead(result, cache string, seconds float64) {
	eventReadsTotal.WithLabelValues(result, cache).Inc()
	if cache == cacheMiss && seconds > 0 {
		eventReadDuration.Observe(seconds)
	}
}
```

Register both in the existing `init()`'s `prometheus.MustRegister(...)` call.

- [ ] **Step 6: Run the tests**

```
go test ./internal/calendarsync/... -v
```
Expected: PASS, including every pre-existing test in the package.

- [ ] **Step 7: Commit**

```bash
git add internal/calendarsync/events_service.go internal/calendarsync/events_service_test.go internal/calendarsync/metrics.go
git commit -m "feat: add calendar events read service"
```

---

## Task 5: `GET /me/calendar/events`

**Files:**
- Modify: `internal/calendarsync/handler.go`
- Test: `internal/calendarsync/handler_test.go`

**Context:** The route sits beside `/me/calendar/connection`, which it depends on, inside the existing `MountAuthed`. It takes the house date contract verbatim: `daterange.ParseQuery` on the raw query values. Read `internal/calendarsync/handler.go`'s `getConnection` for the auth + `httpresp` idiom.

- [ ] **Step 1: Write the failing handler tests**

Add to `internal/calendarsync/handler_test.go`, reusing the file's existing router/auth harness:

```go
func TestGetEvents_RequiresTimezone(t *testing.T)
// No ?timezone → 400, body error "timezone is required" (daterange's verbatim
// message — handlers forward err.Error(), so the string is the contract).

func TestGetEvents_RejectsUnknownTimezone(t *testing.T)
// ?timezone=Mars/Olympus → 400.

func TestGetEvents_RequiresBothDates(t *testing.T)
// start_date without end_date → 400 "end_date is required when start_date is
// supplied".

func TestGetEvents_RejectsAnOversizeWindow(t *testing.T)
// A 90-day window → 400 mentioning the cap.

func TestGetEvents_NotConnectedIs200(t *testing.T)
// A user with no connection row gets HTTP 200 and {"status":"not_connected"},
// NOT a 404. It is the ordinary state of a user who never opted in.

func TestGetEvents_OKShapeIsDenseAndTyped(t *testing.T)
// A connected user over a 3-day window with one timed event on day 2 gets
// 200, status "ok", exactly 3 day entries in date order, events non-null on
// every day, and the populated event carrying id/title/start/end/all_day/
// source. RFC3339 UTC timestamps.

func TestGetEvents_NoAuthIs401(t *testing.T)
```

- [ ] **Step 2: Run to verify they fail**

```
go test ./internal/calendarsync/ -run TestGetEvents
```
Expected: FAIL — the route 404s.

- [ ] **Step 3: Implement**

In `internal/calendarsync/handler.go`, add the field to `Handler`:

```go
	// eventsSvc serves GET /me/calendar/events. It is attached separately
	// (see AttachEvents) rather than passed to NewHandler because it depends
	// on collaborators the OAuth handler has no other use for, and because a
	// deploy without it must still mount the connection routes.
	eventsSvc *EventsService
```

Add:

```go
// AttachEvents wires the read-side events service into the handler.
func (h *Handler) AttachEvents(svc *EventsService) { h.eventsSvc = svc }
```

Add the route to `MountAuthed`, after the connection routes:

```go
	r.Get("/me/calendar/events", h.getEvents)
```

Add the DTOs and the handler:

```go
// maxEventWindowDays caps the requested window. The tile asks for eight days;
// an unbounded window is a Google call and a response size the client should
// not get to choose.
const maxEventWindowDays = 31

// eventLinkPayload deep-links a marked event back into the app.
type eventLinkPayload struct {
	Kind string `json:"kind"`
	ID   string `json:"id"`
}

type eventPayload struct {
	ID     string            `json:"id"`
	Title  string            `json:"title"`
	Start  time.Time         `json:"start"`
	End    time.Time         `json:"end"`
	AllDay bool              `json:"all_day"`
	Source string            `json:"source"`
	Link   *eventLinkPayload `json:"link,omitempty"`
}

// dayPayload is one local date. Events is never null — see EventsService.Day
// for why the array is dense.
type dayPayload struct {
	Date      string         `json:"date"`
	Truncated int            `json:"truncated"`
	Events    []eventPayload `json:"events"`
}

// eventsResponse mirrors weather's shape: a status that is always present, and
// a payload that collapses to just the status on every degradation.
type eventsResponse struct {
	Status EventsStatus `json:"status"`
	Days   []dayPayload `json:"days,omitempty"`
}

// getEvents (authed) returns the user's calendar for a window, grouped by
// LOCAL DATE so the client never re-derives a day boundary.
//
// The date contract is daterange's, verbatim: a required IANA timezone plus
// start_date/end_date, converted server-side. The client must never build a
// UTC instant — a "week" is 167 or 169 hours twice a year, and a tile that
// assumes 7x24 drops or duplicates a day's events for every user not on UTC.
func (h *Handler) getEvents(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	userID, ok := auth.UserIDFrom(ctx)
	if !ok || userID == "" {
		httpresp.Error(w, http.StatusUnauthorized, "missing authenticated user")
		return
	}

	query := r.URL.Query()
	start, end, loc, err := daterange.ParseQuery(query)
	if err != nil {
		httpresp.Error(w, http.StatusBadRequest, err.Error())
		return
	}
	startDate, endDate := query.Get("start_date"), query.Get("end_date")
	if startDate == "" {
		// The single-`date` form is legal to daterange but meaningless here:
		// the tile always asks for a range.
		httpresp.Error(w, http.StatusBadRequest, "start_date and end_date are required")
		return
	}
	if end.Sub(start) > maxEventWindowDays*24*time.Hour {
		httpresp.Error(w, http.StatusBadRequest,
			fmt.Sprintf("window is too long (max %d days)", maxEventWindowDays))
		return
	}

	if h.eventsSvc == nil {
		// Not wired — a misconfiguration, not the kill switch. The tile says
		// "unavailable" and refetches on the next mount.
		httpresp.OK(w, "calendar events", eventsResponse{Status: EventsStatusUnavailable})
		return
	}

	res := h.eventsSvc.Events(ctx, userID, EventsWindow{
		StartUTC:  start,
		EndUTC:    end,
		Loc:       loc,
		StartDate: startDate,
		EndDate:   endDate,
	})
	httpresp.OK(w, "calendar events", eventsResponse{Status: res.Status, Days: dayPayloads(res.Days)})
}

func dayPayloads(days []Day) []dayPayload {
	if len(days) == 0 {
		return nil
	}
	out := make([]dayPayload, 0, len(days))
	for _, d := range days {
		events := make([]eventPayload, 0, len(d.Events))
		for _, e := range d.Events {
			p := eventPayload{
				ID:     e.ID,
				Title:  e.Title,
				Start:  e.Start.UTC(),
				End:    e.End.UTC(),
				AllDay: e.AllDay,
				Source: e.Source,
			}
			if e.Link != nil {
				p.Link = &eventLinkPayload{Kind: e.Link.Kind, ID: e.Link.ID}
			}
			events = append(events, p)
		}
		out = append(out, dayPayload{Date: d.Date, Truncated: d.Truncated, Events: events})
	}
	return out
}
```

Add `fmt` and the `daterange` import to `handler.go`.

- [ ] **Step 4: Run the tests**

```
go test ./internal/calendarsync/... -v
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/calendarsync/handler.go internal/calendarsync/handler_test.go
git commit -m "feat: add GET /me/calendar/events"
```

---

## Task 6: Wire the reader in `internal/server`

**Files:**
- Modify: `internal/server/server.go`

**Context:** The calendar OAuth block sits around line 489–545 and is gated on `cfg.CalendarTokenEncKey != "" && cfg.GoogleCalendarRedirectURL != "" && cfg.GoogleClientID != "" && cfg.GoogleClientSecret != ""`. `tokenSource` and `calendarEventClient` are already constructed inside it. The events service needs a `*sql.DB` for the link repository — check what the surrounding code calls the handle (`database`, per `calendarsync.NewSQLiteActivitySyncRepository(database)` further down) and whether it is in scope at this point; if it is not, construct the link repo where `calendarSyncState` is built and attach the events service there instead.

- [ ] **Step 1: Construct and attach**

Inside the calendar block, immediately after `calendarScheduler = calendarsync.NewService(...)`, add:

```go
			// The READ direction: GET /me/calendar/events for the dashboard
			// tile. It shares the grant, the token source, and the HTTP client
			// with the write path above and adds no scope of its own — the
			// granted calendar.events scope already permits events.list, so
			// every connected user gets the tile on deploy with no re-consent.
			// In-memory mode (no SQLite handle) has no rows to mark events
			// against, so the reader is left unattached and the endpoint
			// reports "unavailable".
			if database != nil {
				calendarSyncHandler.AttachEvents(calendarsync.NewEventsService(calendarsync.EventsServiceDeps{
					Conns:  calendarConnRepo,
					Cipher: cipher,
					Tokens: tokenSource,
					Client: calendarEventClient,
					Links:  calendarsync.NewSQLiteEventLinkRepository(database),
					Config: calendarsync.EventsConfig{
						Enabled:         cfg.CalendarEvents.Enabled,
						CacheTTL:        time.Duration(cfg.CalendarEvents.CacheTTLSeconds) * time.Second,
						MaxEventsPerDay: cfg.CalendarEvents.MaxEventsPerDay,
					},
				}))
			}
```

If `database` is not in scope there, hoist the `AttachEvents` call down to the block where `calendarSyncState = calendarsync.NewSQLiteActivitySyncRepository(database)` is built (guarding on `calendarSyncHandler != nil`), and say so in a comment.

Update the block's closing log line to mention the reader:

```go
			log.Println("calendar-sync: enabled (google calendar oauth + connection + event writing + dashboard reads)")
```

- [ ] **Step 2: Build and run the server tests**

```
go build ./... && go test ./internal/server/...
```
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add internal/server/server.go
git commit -m "feat: wire the calendar events reader into the server"
```

---

## Task 7: The Go tile catalog

**Files:**
- Modify: `internal/dashboard/tiles.go`
- Test: `internal/dashboard/tiles_test.go`
- Test: `internal/dashboard/summary_layout_test.go`
- Test: `internal/dashboard/layout_handler_test.go`

- [ ] **Step 1: Write the failing tests**

In `internal/dashboard/tiles_test.go`, append `TileCalendar` to the `all` slice in `TestCatalog_EveryConstantAppearsExactlyOnce` and to the `want` slice in `TestCatalog_Order` — in both cases **after `TileWeather`**. Add to `TestValidTileID`:

```go
	if !ValidTileID("calendar") {
		t.Error("calendar should be valid")
	}
```

In `internal/dashboard/summary_layout_test.go`, add beside `TestSummary_DefaultLayoutHasNoRestingHRTile`:

```go
// TestSummary_DefaultLayoutHasNoCalendarTile pins the rollout, mirroring sleep
// and resting_hr: calendar is a catalog tile but NOT part of the default
// layout. The tile is useless — a permanent connect CTA — for any user who has
// not connected a calendar, and unlike recovery (gated on hasConnectedWhoop)
// gating it would mean a second connection probe on every dashboard load to
// save a tray click.
func TestSummary_DefaultLayoutHasNoCalendarTile(t *testing.T) {
	r, rp, userID := newTestEnv(t)
	seedWhoopConnected(t, rp, userID)

	layout, _ := dataEnvelope(t, r, userID, "?timezone=UTC")

	if indexOf(layout, string(TileCalendar)) >= 0 {
		t.Errorf("default layout must not contain %q; got %v", TileCalendar, layout)
	}
}
```

In `internal/dashboard/layout_handler_test.go`, add a case mirroring `TestPutLayout_RestingHRAccepted` (read that test and copy its shape exactly):

```go
// TestPutLayout_CalendarAccepted proves a client may place the calendar tile.
// It is tray-only, so this is the only way it reaches a layout.
func TestPutLayout_CalendarAccepted(t *testing.T) { … }
```

- [ ] **Step 2: Run to verify they fail**

```
go test ./internal/dashboard/ -run 'TestCatalog|TestValidTileID|TestSummary_DefaultLayoutHasNoCalendarTile|TestPutLayout_CalendarAccepted'
```
Expected: FAIL — `undefined: TileCalendar`.

- [ ] **Step 3: Implement**

In `internal/dashboard/tiles.go`, add after `TileWeather`:

```go
	// TileCalendar has NO summary section, for the same reason TileWeather
	// does not: the tile self-fetches from GET /me/calendar/events, so a slow
	// Google can never delay the single /dashboard/summary round-trip that
	// every other tile shares. The catalog entry exists only so layouts can
	// place and validate the tile. It is tray-only — see
	// TestSummary_DefaultLayoutHasNoCalendarTile.
	TileCalendar TileID = "calendar"
```

Append it to `Catalog`, after `TileWeather`:

```go
	TileSleep, TileStreak, TileQuote, TileWeather, TileCalendar,
```

- [ ] **Step 4: Run the tests**

```
go test ./internal/dashboard/...
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/dashboard/
git commit -m "feat: add the calendar tile to the dashboard catalog"
```

---

# Web (`prog-strength-web`)

Everything below is in `prog-strength-web`, on its own `feat/google-calendar-dashboard-tile` branch. Run `npm run typecheck && npm run lint && npm run test` before every commit.

---

## Task 8: `lib/api.ts` types and the pure `shared.ts`

**Files:**
- Modify: `lib/api.ts`
- Create: `app/(app)/dashboard/_components/calendar/shared.ts`
- Test: `app/(app)/dashboard/_components/calendar/shared.test.ts`

**Context:** `lib/api.ts` already has `getCalendarConnection` (~line 4093) and its `CalendarConnection` type; put the new types and fetcher directly beneath them. `unwrap<T>(resp, empty)` (~line 5225) is the house envelope helper — every fetcher uses it. `getWeather` (~line 5129) shows the `URLSearchParams` + `Authorization` idiom.

The pure module is separated for the reason `recovery/hrv-chart.ts` and `recovery/resting-rank.ts` were: the row-windowing rule is where the tests earn their keep, and where a well-meaning simplification would do the most damage.

- [ ] **Step 1: Add the API types and fetcher**

In `lib/api.ts`, immediately after `getCalendarConnection`:

```ts
/** One event on the calendar tile. Timestamps are RFC3339 UTC. */
export type CalendarEvent = {
  id: string;
  title: string;
  start: string;
  end: string;
  all_day: boolean;
  /** "prog_strength" marks an event Prog Strength itself wrote. */
  source: "prog_strength" | "google";
  link?: { kind: "planned_workout" | "activity"; id: string };
};

/**
 * One local calendar date. `days` from the API is DENSE — every date in the
 * requested window appears, with an empty `events` array for a free day — so
 * a client never has to distinguish "no events" from "day missing".
 */
export type CalendarDay = {
  date: string;
  /** How many events the server's per-day cap cut. Usually 0, never silent. */
  truncated: number;
  events: CalendarEvent[];
};

/** The closed degradation set, mirroring the weather tile's vocabulary. */
export type CalendarEventsStatus =
  | "ok"
  | "not_connected"
  | "reconnect_needed"
  | "disabled"
  | "unavailable";

export type CalendarEventsResponse = {
  status: CalendarEventsStatus;
  days?: CalendarDay[];
};

/**
 * GET /me/calendar/events. One request covers Today, Tomorrow, the week strip,
 * and the expand panel. The window is sent as a required IANA timezone plus
 * start_date/end_date and converted server-side — the client never builds a
 * UTC instant, because a week is 167 or 169 hours twice a year.
 */
export async function getCalendarEvents(
  token: string,
  timezone: string,
  startDate: string,
  endDate: string,
): Promise<CalendarEventsResponse | null> {
  const params = new URLSearchParams();
  params.set("timezone", timezone);
  params.set("start_date", startDate);
  params.set("end_date", endDate);
  const resp = await fetch(`${config.apiUrl}/me/calendar/events?${params.toString()}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  return unwrap<CalendarEventsResponse | null>(resp, null);
}
```

- [ ] **Step 2: Write the failing `shared.test.ts`**

`app/(app)/dashboard/_components/calendar/shared.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import type { CalendarEvent } from "@/lib/api";
import {
  formatEventTime,
  localDateKey,
  nextUpcoming,
  requestWindow,
  stripHeights,
  visibleEvents,
  weekColumns,
} from "./shared";

/** A timed event at the given local hour on 2026-08-12. */
function ev(id: string, hour: number, minutes = 0): CalendarEvent {
  const start = new Date(2026, 7, 12, hour, minutes);
  const end = new Date(start.getTime() + 30 * 60_000);
  return {
    id,
    title: `Event ${id}`,
    start: start.toISOString(),
    end: end.toISOString(),
    all_day: false,
    source: "google",
  };
}

function allDay(id: string): CalendarEvent {
  return {
    id,
    title: `All day ${id}`,
    start: new Date(2026, 7, 12, 0, 0).toISOString(),
    end: new Date(2026, 7, 13, 0, 0).toISOString(),
    all_day: true,
    source: "google",
  };
}

describe("visibleEvents", () => {
  it("anchors on the first upcoming event and reports both counts", () => {
    const events = [6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16].map((h) => ev(`e${h}`, h));
    const now = new Date(2026, 7, 12, 11, 30); // after the 11:00, before the 12:00
    const { visible, earlierCount, laterCount } = visibleEvents(events, now);

    expect(visible.map((e) => e.id)).toEqual(["e12", "e13", "e14", "e15", "e16"]);
    expect(earlierCount).toBe(6);
    expect(laterCount).toBe(0);
  });

  it("backfills when the day is over, reporting earlier with later zero", () => {
    const events = [6, 7, 8, 9, 10, 11, 12, 13].map((h) => ev(`e${h}`, h));
    const now = new Date(2026, 7, 12, 21, 0);
    const { visible, earlierCount, laterCount } = visibleEvents(events, now);

    expect(visible.map((e) => e.id)).toEqual(["e9", "e10", "e11", "e12", "e13"]);
    expect(earlierCount).toBe(3);
    expect(laterCount).toBe(0);
  });

  it("returns the first five when now is before everything — the Tomorrow case", () => {
    const events = [6, 7, 8, 9, 10, 11, 12].map((h) => ev(`e${h}`, h));
    const now = new Date(2026, 7, 11, 5, 0);
    const { visible, earlierCount, laterCount } = visibleEvents(events, now);

    expect(visible.map((e) => e.id)).toEqual(["e6", "e7", "e8", "e9", "e10"]);
    expect(earlierCount).toBe(0);
    expect(laterCount).toBe(2);
  });

  it("returns everything with both counts zero for a day of five or fewer", () => {
    const events = [8, 9, 12].map((h) => ev(`e${h}`, h));
    const { visible, earlierCount, laterCount } = visibleEvents(
      events,
      new Date(2026, 7, 12, 20, 0),
    );
    expect(visible).toHaveLength(3);
    expect(earlierCount).toBe(0);
    expect(laterCount).toBe(0);
  });

  it("counts an event in progress as upcoming", () => {
    const events = [ev("a", 9), ev("b", 11), ev("c", 13)];
    // 11:15 — "b" started at 11:00 and ends at 11:30.
    const { visible } = visibleEvents(events, new Date(2026, 7, 12, 11, 15), 2);
    expect(visible.map((e) => e.id)).toEqual(["b", "c"]);
  });

  it("pins all-day events above the window and spends their row budget", () => {
    const events = [allDay("h"), ...[6, 7, 8, 9, 10, 11].map((h) => ev(`e${h}`, h))];
    const { visible, earlierCount } = visibleEvents(events, new Date(2026, 7, 12, 9, 30));
    expect(visible[0].id).toBe("h");
    expect(visible).toHaveLength(5);
    expect(visible.slice(1).map((e) => e.id)).toEqual(["e10", "e11"].concat([]).length
      ? ["e8", "e9", "e10", "e11"]
      : []);
    expect(earlierCount).toBe(2);
  });
});

describe("stripHeights", () => {
  it("normalises against the busiest day", () => {
    expect(stripHeights([0, 1, 4, 2, 0, 0, 0])).toEqual([0, 0.25, 1, 0.5, 0, 0, 0]);
  });

  it("returns zeros, not NaN, for an all-zero week", () => {
    const heights = stripHeights([0, 0, 0, 0, 0, 0, 0]);
    expect(heights).toEqual([0, 0, 0, 0, 0, 0, 0]);
    expect(heights.every((h) => Number.isFinite(h))).toBe(true);
  });
});

describe("requestWindow", () => {
  it("is the Monday-to-Sunday union with today and tomorrow", () => {
    // Wednesday 2026-08-12.
    expect(requestWindow(new Date(2026, 7, 12, 9, 0))).toEqual({
      startDate: "2026-08-10",
      endDate: "2026-08-16",
    });
  });

  it("extends to the FOLLOWING Monday on a Sunday", () => {
    // Sunday 2026-08-16 — "tomorrow" is next Monday, outside this week. A
    // window of just the week would render an empty Tomorrow slide one day in
    // seven.
    expect(requestWindow(new Date(2026, 7, 16, 9, 0))).toEqual({
      startDate: "2026-08-10",
      endDate: "2026-08-17",
    });
  });
});

describe("formatEventTime", () => {
  it("does not print 12:00p for noon", () => {
    const noon = new Date(2026, 7, 12, 12, 0).toISOString();
    const formatted = formatEventTime(noon);
    expect(formatted).not.toMatch(/^0[:.]/);
    expect(formatted).toMatch(/12/);
  });

  it("formats an afternoon time in the browser locale", () => {
    expect(formatEventTime(new Date(2026, 7, 12, 14, 30).toISOString())).toMatch(/2[:.]30/);
  });
});

describe("weekColumns", () => {
  it("returns seven Monday-first columns with counts including truncation", () => {
    const days = [
      { date: "2026-08-10", truncated: 0, events: [] },
      { date: "2026-08-11", truncated: 3, events: [ev("a", 9)] },
      { date: "2026-08-12", truncated: 0, events: [ev("b", 9), ev("c", 10)] },
      { date: "2026-08-13", truncated: 0, events: [] },
      { date: "2026-08-14", truncated: 0, events: [] },
      { date: "2026-08-15", truncated: 0, events: [] },
      { date: "2026-08-16", truncated: 0, events: [] },
      { date: "2026-08-17", truncated: 0, events: [ev("d", 9)] },
    ];
    const cols = weekColumns(days, new Date(2026, 7, 16, 9, 0));
    expect(cols).toHaveLength(7);
    expect(cols[0].date).toBe("2026-08-10");
    expect(cols[6].date).toBe("2026-08-16");
    // The cap is never silent: the count is events.length + truncated.
    expect(cols[1].count).toBe(4);
  });
});

describe("nextUpcoming", () => {
  it("finds the next event anywhere in the remaining week", () => {
    const days = [
      { date: "2026-08-12", truncated: 0, events: [ev("past", 8)] },
      { date: "2026-08-13", truncated: 0, events: [ev("next", 9)] },
    ];
    // ev() builds everything on 2026-08-12, so build the day-2 event by hand.
    const later = { ...ev("next", 9), start: new Date(2026, 7, 13, 9, 0).toISOString() };
    days[1].events = [later];
    expect(nextUpcoming(days, new Date(2026, 7, 12, 10, 0))?.id).toBe("next");
  });

  it("returns null when nothing is left", () => {
    const days = [{ date: "2026-08-12", truncated: 0, events: [ev("past", 8)] }];
    expect(nextUpcoming(days, new Date(2026, 7, 12, 23, 0))).toBeNull();
  });
});
```

> The implementer should tidy the deliberately awkward `visible.slice(1)` expectation in the all-day test into a plain `expect(visible.slice(1).map((e) => e.id)).toEqual(["e8", "e9", "e10", "e11"])` — it is written above only to make the expected values explicit. Verify the expectation against the rule before "fixing" it: with one all-day pin the timed budget is 4, `now` is 09:30 so the anchor is `e10` (09:00's event ends at 09:30, so it is past), the tail is `e10, e11`, and backfill pulls `e8, e9` in front.

- [ ] **Step 3: Run to verify they fail**

```
npx vitest run app/\(app\)/dashboard/_components/calendar/shared.test.ts
```
Expected: FAIL — module not found.

- [ ] **Step 4: Implement `shared.ts`**

```ts
/**
 * Pure logic for the calendar tile: the request window, the row-windowing
 * rule, week-strip normalisation, and time formatting. No React, so every rule
 * below is directly testable — this module is separated for the reason
 * `recovery/hrv-chart.ts` was.
 *
 * THE WINDOWING RULE IS THE POINT OF THIS FILE. A naive slice(0, 5) is
 * chronological and wrong after lunch: at 6pm it fills the tile with things
 * the user has already done and buries the one event they can still act on.
 * Anchoring on the first UPCOMING event keeps the tile about the rest of the
 * day, and backfilling means a 9pm view is not blank. Everything clipped is
 * REPORTED — a tile that silently drops half a day reads as an empty
 * afternoon.
 */
import type { CalendarDay, CalendarEvent } from "@/lib/api";

/** How many rows a day slide shows. */
export const DAY_ROW_LIMIT = 5;

/** Monday-first weekday initials, matching /calendar's WEEKDAYS constant. */
export const WEEKDAY_INITIALS = ["M", "T", "W", "T", "F", "S", "S"] as const;

/** Full weekday names, for the strip's composed accessible label. */
export const WEEKDAY_NAMES = [
  "Monday",
  "Tuesday",
  "Wednesday",
  "Thursday",
  "Friday",
  "Saturday",
  "Sunday",
] as const;

/**
 * YYYY-MM-DD for a Date in the BROWSER's local zone. Not toISOString().slice —
 * that is UTC, and would name the wrong day for most of the world for part of
 * every day.
 */
export function localDateKey(d: Date): string {
  const month = `${d.getMonth() + 1}`.padStart(2, "0");
  const day = `${d.getDate()}`.padStart(2, "0");
  return `${d.getFullYear()}-${month}-${day}`;
}

/** Midnight local on the given date. */
export function startOfLocalDay(d: Date): Date {
  return new Date(d.getFullYear(), d.getMonth(), d.getDate());
}

export function addDays(d: Date, n: number): Date {
  return new Date(d.getFullYear(), d.getMonth(), d.getDate() + n);
}

/**
 * Days since Monday. `(getDay() + 6) % 7` converts JS's Sunday-first weekday
 * to Monday-first — the same math /calendar's buildMonthGrid uses. A tile
 * whose week starts on Sunday sitting one click away from a month grid that
 * starts on Monday is drift that is invisible in review and jarring in use.
 */
export function mondayOffset(d: Date): number {
  return (d.getDay() + 6) % 7;
}

/**
 * The one window that feeds all three slides and the panel.
 *
 * The max() matters on Sunday, when "tomorrow" is next Monday and falls
 * OUTSIDE the current week — a window of just the week would render an empty
 * Tomorrow slide one day in seven.
 */
export function requestWindow(now: Date): { startDate: string; endDate: string } {
  const today = startOfLocalDay(now);
  const tomorrow = addDays(today, 1);
  const weekStart = addDays(today, -mondayOffset(today));
  const weekEnd = addDays(weekStart, 6);
  const start = weekStart < today ? weekStart : today;
  const end = weekEnd > tomorrow ? weekEnd : tomorrow;
  return { startDate: localDateKey(start), endDate: localDateKey(end) };
}

/** An event is upcoming until it has ENDED — a meeting you are currently in
 * is the most relevant row on the tile, not a past one. */
function isUpcoming(e: CalendarEvent, now: Date): boolean {
  return new Date(e.end).getTime() > now.getTime();
}

/**
 * Which rows of a day's events to show.
 *
 * All-day events pin to the top and spend row budget; the anchor/backfill rule
 * runs over the timed remainder. With `now` before every event the anchor is
 * index 0, so Tomorrow — which has no past — needs no special case.
 */
export function visibleEvents(
  events: CalendarEvent[],
  now: Date,
  limit: number = DAY_ROW_LIMIT,
): { visible: CalendarEvent[]; earlierCount: number; laterCount: number } {
  const pinned = events.filter((e) => e.all_day);
  const timed = events.filter((e) => !e.all_day);

  const shownPins = pinned.slice(0, limit);
  let laterCount = pinned.length - shownPins.length;
  const budget = limit - shownPins.length;
  if (budget <= 0) {
    return { visible: shownPins, earlierCount: 0, laterCount: laterCount + timed.length };
  }

  const anchor = timed.findIndex((e) => isUpcoming(e, now));
  let end = anchor === -1 ? timed.length : Math.min(timed.length, anchor + budget);
  const start = Math.max(0, end - budget);
  end = Math.min(timed.length, start + budget);

  laterCount += timed.length - end;
  return {
    visible: [...shownPins, ...timed.slice(start, end)],
    earlierCount: start,
    laterCount,
  };
}

/**
 * Bar heights (0..1), normalised against the week's busiest day rather than a
 * fixed ceiling. The strip answers "which days are heavy RELATIVE to my week" —
 * an absolute scale makes an ordinary week look empty and a conference week
 * look identical to a normal one.
 *
 * An all-zero week yields all-zero heights, not NaN. This is the guard the
 * division needs and it is a real state, not a defensive flourish: it is
 * exactly what a new user's first Sunday looks like.
 */
export function stripHeights(counts: number[]): number[] {
  const max = counts.reduce((a, b) => (b > a ? b : a), 0);
  if (max <= 0) return counts.map(() => 0);
  return counts.map((c) => c / max);
}

/** One column of the week strip. */
export type WeekColumn = {
  date: string;
  initial: string;
  name: string;
  /** events.length + truncated — the cap is never silent. */
  count: number;
  /** Whether the day carries a Prog Strength-authored event. */
  hasOurs: boolean;
  isToday: boolean;
};

/** The seven Monday-first columns for the week containing `now`. */
export function weekColumns(days: CalendarDay[], now: Date): WeekColumn[] {
  const today = startOfLocalDay(now);
  const weekStart = addDays(today, -mondayOffset(today));
  const todayKey = localDateKey(today);
  const byDate = new Map(days.map((d) => [d.date, d]));

  return WEEKDAY_INITIALS.map((initial, i) => {
    const date = localDateKey(addDays(weekStart, i));
    const day = byDate.get(date);
    return {
      date,
      initial,
      name: WEEKDAY_NAMES[i],
      count: (day?.events.length ?? 0) + (day?.truncated ?? 0),
      hasOurs: (day?.events ?? []).some((e) => e.source === "prog_strength"),
      isToday: date === todayKey,
    };
  });
}

/** The next event that has not started yet, anywhere in `days`. */
export function nextUpcoming(days: CalendarDay[], now: Date): CalendarEvent | null {
  const candidates = days
    .flatMap((d) => d.events)
    .filter((e) => !e.all_day && new Date(e.start).getTime() > now.getTime())
    .sort((a, b) => new Date(a.start).getTime() - new Date(b.start).getTime());
  return candidates[0] ?? null;
}

/**
 * A time in the browser's locale. toLocaleTimeString, not a hand-rolled
 * `h % 12` — that is precisely the formatter that prints "0:00p" for noon.
 */
export function formatEventTime(iso: string): string {
  return new Date(iso).toLocaleTimeString(undefined, { hour: "numeric", minute: "2-digit" });
}

/** "Wed Aug 12" — the day slide's header date. */
export function formatDayHeading(date: string): string {
  const [y, m, d] = date.split("-").map(Number);
  return new Date(y, m - 1, d).toLocaleDateString(undefined, {
    weekday: "short",
    month: "short",
    day: "numeric",
  });
}

/** The composed accessible label for the density strip. */
export function stripLabel(columns: WeekColumn[]): string {
  return columns
    .map((c) => `${c.name} ${c.count} ${c.count === 1 ? "event" : "events"}`)
    .join(", ");
}
```

- [ ] **Step 5: Run the tests**

```
npx vitest run app/\(app\)/dashboard/_components/calendar/shared.test.ts
npm run typecheck
```
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add lib/api.ts "app/(app)/dashboard/_components/calendar/shared.ts" "app/(app)/dashboard/_components/calendar/shared.test.ts"
git commit -m "feat(calendar): add the events API client and the tile's pure module"
```

---

## Task 9: Fixtures, the day slide, and the week strip

**Files:**
- Create: `app/(app)/dashboard/_components/calendar/fixtures.ts`
- Create: `app/(app)/dashboard/_components/calendar/day-slide.tsx`
- Create: `app/(app)/dashboard/_components/calendar/week-strip.tsx`

**Context:** Read `app/(app)/dashboard/_components/recovery/fixtures.ts` for the fixture idiom this codebase uses. Design-system tokens only — `--foreground`, `--muted`, `--faint`, `--border`, `--surface-2`, `--accent`. No `--warning`, no `--danger`: a busy day is not an alarm. `tabular-nums` on every time and count.

- [ ] **Step 1: Write `fixtures.ts`**

Export one builder per state named in the SOW, each returning a `CalendarEventsResponse` plus the `now` it is meant to be rendered at, so the tile test and any future story share one source of truth:

```ts
/**
 * The calendar tile's states, as wire payloads. Each is paired with the `now`
 * it is meant to be read at, because half of them are only distinguishable by
 * where the clock sits inside the day.
 */
export type CalendarFixture = { now: Date; response: CalendarEventsResponse };

export function defaultDay(): CalendarFixture
// A mixed weekday: a marked planned workout (source prog_strength, a
// planned_workout link), two meetings, one all-day event. Reads calm.

export function emptyDay(): CalendarFixture
// Today is free; other days in the week carry events so the strip still renders.

export function busyDay(): CalendarFixture
// Eleven events, six of them past.

export function lateEvening(): CalendarFixture
// `now` is after every event today; eight events.

export function sunday(): CalendarFixture
// `now` is a Sunday; the window runs Monday..next Monday and the Tomorrow
// slide (next Monday) is populated.

export function allDayOnly(): CalendarFixture
// A holiday: one all-day event and nothing else.

export function emptyWeek(): CalendarFixture
// Every day zero.
```

Build them from a small `day(date, events)` helper and an `event({...})` helper so the payloads stay readable. Dates must be consistent with the `now` each fixture carries — build them relative to a fixed anchor (`new Date(2026, 7, 12, 9, 0)` for the Wednesday cases) rather than `Date.now()`, so the tests are deterministic.

- [ ] **Step 2: Write `day-slide.tsx`**

```tsx
/**
 * One day's rows: a heading, up to five `time · title` rows, and honest
 * overflow lines. All-day events pin to the top with no time column.
 *
 * The tile must not lie about a day it has clipped, so `+2 earlier` and
 * `+3 more` are both rendered — as muted lines that open the week panel on
 * this day. An empty day renders "Nothing scheduled." and NOTHING ELSE: no
 * CTA, no suggestion. The tile does not nag a user for having a free
 * afternoon, and it must not become a surface that sells planning a workout.
 */
export function DaySlide({
  label,        // "Today" | "Tomorrow"
  date,         // YYYY-MM-DD
  day,          // CalendarDay | undefined
  now,
  onOpenDay,    // (date: string) => void — opens the panel on this day
}: {
  label: string;
  date: string;
  day: CalendarDay | undefined;
  now: Date;
  onOpenDay: (date: string) => void;
})
```

Rules:
- Heading: `{label} · {formatDayHeading(date)}` — `text-xs text-[var(--muted)]`.
- No events → a single `text-sm text-[var(--muted)]` line reading `Nothing scheduled.`
- Rows come from `visibleEvents(day.events, now)`.
- A row is a flex line: the time in `text-xs tabular-nums text-[var(--faint)]` (`w-[3.5rem] shrink-0`), then the title `truncate text-sm`.
- All-day rows render no time column — an `aria-hidden` spacer of the same width keeps the titles aligned.
- A row whose event has ended is `text-[var(--muted)]` rather than hidden, so the day still reads as a whole.
- A `source === "prog_strength"` row takes the accent: a 1.5×1.5 `rounded-full bg-[var(--accent)]` dot before the title, and — when `link.kind === "planned_workout"` — the title is a `next/link` to `/planned-workouts/{link.id}`. **`--accent` is a PROVENANCE signal, not a status**: a planned workout is not painted warm for being soon or green for being done. Verify the planned-workout route's real path in the app before wiring the href; if `/planned-workouts/{id}` does not exist, link to the calendar page for that date instead and note it in the PR.
- Overflow: when `earlierCount > 0`, a `+{earlierCount} earlier` button above the rows; when `laterCount > 0`, a `+{laterCount} more` button below. Both are `text-[11px] text-[var(--faint)]` buttons calling `onOpenDay(date)`.
- `day.truncated > 0` folds into `laterCount` for display: pass `laterCount + day.truncated` to the `+N more` line, so a capped day still reports its true remainder.

- [ ] **Step 3: Write `week-strip.tsx`**

```tsx
/**
 * The seven-column density strip, Monday-first.
 *
 * THE STRIP IS DECORATION; THE MEANING IS THE CAPTION AND THE COUNTS.
 * Following morning-ledger's railLabel precedent the container takes
 * role="img" with a composed label and the bars are aria-hidden, so a screen
 * reader gets "Monday 1 event, Tuesday 3 events, …" rather than seven
 * unlabelled divs.
 */
export function WeekStrip({
  columns,   // WeekColumn[] from weekColumns()
  next,      // CalendarEvent | null from nextUpcoming()
  onOpenDay, // (date: string) => void
}: { … })
```

Rules:
- `stripHeights(columns.map((c) => c.count))` gives 0..1 fractions; a bar's height is `Math.round(h * 100)%` of a fixed ~28px rail, with a 2px floor so a zero day still draws a hairline rail rather than nothing.
- Bar fill: `bg-[var(--surface-2)]`, or `bg-[var(--accent)]` when `column.hasOurs`.
- Today's column: the initial takes `text-[var(--foreground)]` and the others `text-[var(--faint)]`.
- Under each bar: the count in `text-[10px] tabular-nums`.
- Each column is a `<button>` calling `onOpenDay(column.date)` so the strip is a way into the panel.
- Beneath the strip, one line: `Next: {formatEventTime(next.start)} {next.title}` or `Nothing left this week.`, in `text-xs text-[var(--muted)]`, truncated.
- The strip container: `role="img"` with `aria-label={stripLabel(columns)}`; the bars `aria-hidden`.

- [ ] **Step 4: Verify it compiles and lints**

```
npm run typecheck && npm run lint
```
Expected: clean. (These two components are exercised by Task 11's tile tests; they get no test file of their own — the tile test renders them for real, which is what the SOW's per-state tests describe.)

- [ ] **Step 5: Commit**

```bash
git add "app/(app)/dashboard/_components/calendar/"
git commit -m "feat(calendar): add the day slide, week strip, and fixtures"
```

---

## Task 10: The week agenda panel

**Files:**
- Create: `app/(app)/dashboard/_components/calendar/calendar-week-modal.tsx`

**Context:** Read `app/(app)/dashboard/_components/weather-forecast-modal.tsx` and copy its idiom exactly: `role="dialog"`, `aria-modal="true"`, `aria-labelledby`, an absolutely-positioned `bg-black/60` scrim that closes on click, a `max-h-[85vh] w-full max-w-2xl` panel, the close button focused on mount via a ref, and a document-level `keydown` listener for Escape. Focus RETURN to the ⤢ button is the tile's job, not the modal's.

- [ ] **Step 1: Write the component**

```tsx
/**
 * The week agenda panel — every day, every event, untruncated.
 *
 * This is what makes the density strip useful: the strip says Wednesday is
 * packed, the panel says with what. It FETCHES NOTHING; the tile already holds
 * the week from the single request that feeds all three slides.
 */
export function CalendarWeekModal({
  days,          // CalendarDay[] — the whole window
  initialDate,   // the date the user opened it on; scrolled into view
  onClose,
}: { days: CalendarDay[]; initialDate: string; onClose: () => void })
```

Rules:
- Header: `<h2>` reading `This week`, a one-line `text-xs text-[var(--muted)]` subtitle, and the close `✕` button (`aria-label="Close"`, focused on mount).
- Body: one section per day in `days`, each headed `formatDayHeading(day.date)` (today's heading takes `text-[var(--foreground)]`, the rest `text-[var(--muted)]`), then every event as a `time · title` row — **no windowing, no truncation, and no `+N` lines**; that is the whole point of the panel. All-day rows render an `All day` label in the time column.
- `day.truncated > 0` renders a single `text-[11px] text-[var(--faint)]` line reading `+{truncated} more not shown` at the end of that day, so the server-side cap stays visible here too.
- A day with no events reads `Nothing scheduled.` in `--muted`.
- `prog_strength` rows carry the same accent dot and planned-workout link as the day slide.
- The day matching `initialDate` gets a ref and `scrollIntoView({ block: "start" })` on mount.
- Escape closes; clicking the scrim closes. No focus trap beyond focusing the close button on mount — that matches the existing forecast modal, and diverging here would be a bigger a11y decision than this SOW's remit.

- [ ] **Step 2: Verify**

```
npm run typecheck && npm run lint
```

- [ ] **Step 3: Commit**

```bash
git add "app/(app)/dashboard/_components/calendar/calendar-week-modal.tsx"
git commit -m "feat(calendar): add the week agenda panel"
```

---

## Task 11: `CalendarCard` — the tile

**Files:**
- Create: `app/(app)/dashboard/_components/calendar/calendar-tile.tsx`
- Test: `app/(app)/dashboard/_components/calendar/calendar-tile.test.tsx`

**Context:** Read `app/(app)/dashboard/_components/weather-tile.tsx` end to end first. The paging idiom is lifted from it verbatim: ‹ › buttons with `aria-label`s, dot indicators, `ArrowLeft`/`ArrowRight` on a focused `role="group"`, touch swipe past a 40px threshold, and page state that is ephemeral by design. Read `mini-card.tsx` for `MINI_CARD_PANEL`, `MINI_CARD_HOVER`, and `MiniCardSkeleton`.

**This tile is NOT a `MiniCard`.** It composes `MINI_CARD_PANEL` by hand, because `mini-card.tsx` exports those constants for precisely this case and says so: a tile with its own buttons inside cannot be wrapped in a single `<a>`, since a button inside an anchor is invalid markup that swallows its own clicks. This tile has pager buttons, an expand button, and per-row links.

**There is no gear icon.** `weather`'s gear manages saved locations; `primary`-only means this tile has nothing to manage. The header carries the ⤢ affordance and nothing else.

- [ ] **Step 1: Write the failing test file**

`calendar-tile.test.tsx`. Mock the API and auth exactly as `weather-tile.test.tsx` does:

```tsx
const getCalendarEventsMock = vi.hoisted(() => vi.fn());
vi.mock("@/lib/api", async (orig) => ({
  ...(await orig<typeof import("@/lib/api")>()),
  getCalendarEvents: getCalendarEventsMock,
}));
vi.mock("@/lib/auth", () => ({ getToken: () => "test-token", clearToken: vi.fn() }));
```

Pin the clock per test with `vi.setSystemTime(fixture.now)` inside `vi.useFakeTimers()` (restore in `afterEach`), and drive the fixtures from Task 9's module.

One test per state plus the interaction tests:

```
default          — the marked planned workout renders with its accent dot and
                   links to its planned workout; two meetings and the all-day
                   event render; nothing shouts.
empty-day        — "Nothing scheduled." renders and NO CTA/button text does.
busy-day         — "+6 earlier" renders and "+0 more" does NOT (a zero
                   overflow line is noise, not honesty).
late-evening     — "+3 earlier" renders, no "+N more", and the visible rows
                   carry the muted class.
sunday           — paging to Tomorrow shows next Monday's events (the union
                   window's edge case), not an empty slide.
all-day-only     — one pinned row, no time column, and the week strip counts
                   it as one event.
empty-week       — the strip renders seven columns and the caption reads
                   "Nothing left this week."
not-connected    — the CTA reads "Connect Google Calendar" and its href is
                   `${apiUrl}/auth/google/calendar/connect?return_to=<the
                   dashboard URL>` — the same call the Settings page makes.
reconnect-needed — identical layout, "Reconnect Google Calendar".
unavailable      — "Calendar is unavailable." and no error role in the tree.
disabled         — "Calendar is off."
loading          — the house skeleton is present before the fetch resolves.

paging by button   — Next/Previous move Today → Tomorrow → Week and the dots
                     track; Previous is disabled on the first slide and Next on
                     the last.
paging by keyboard — ArrowRight/ArrowLeft on the role="group" body page.
paging by swipe    — touchStart/touchEnd with dx < -40 pages forward; dx of
                     -20 does not.
opens on Today     — after paging to Week, a remount opens on Today (page state
                     is ephemeral by design).
midnight rollover  — after the tile renders, advance the fake clock past local
                     midnight, dispatch `visibilitychange` with
                     document.visibilityState "visible", and assert
                     getCalendarEvents was called a SECOND time. Then assert a
                     visibilitychange on the SAME date does NOT refetch.
panel              — clicking ⤢ opens the dialog; Escape closes it; focus
                     returns to the ⤢ button; the panel shows an event the day
                     slide had clipped.
one request        — exactly one getCalendarEvents call covers the initial
                     render, all three slides, and opening the panel.
```

- [ ] **Step 2: Run to verify they fail**

```
npx vitest run "app/(app)/dashboard/_components/calendar/calendar-tile.test.tsx"
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `calendar-tile.tsx`**

File header:

```tsx
/**
 * CalendarCard — the self-fetching calendar tile.
 *
 * THREE SLIDES: Today, Tomorrow, and a week-at-a-glance density strip, behind
 * the paging idiom weather established. The ⤢ affordance opens the week agenda
 * panel — every day, every event, untruncated — which is what makes the
 * density strip useful: the strip says Wednesday is packed, the panel says
 * with what.
 *
 * WHY IT SELF-FETCHES. Like weather, this is third-party data with its own
 * auth and failure modes, and it has no business inside the single
 * /dashboard/summary round-trip where a slow Google would delay every other
 * tile on the grid.
 *
 * WHY ONE REQUEST COVERS EVERYTHING. The server returns the window grouped by
 * LOCAL DATE, so the client never re-derives a day boundary, and the window is
 * the union of what the three slides need (see requestWindow). Today,
 * Tomorrow, the strip, and the panel all read the same payload; the panel
 * fetches nothing.
 *
 * WHY THERE IS NO GEAR. weather's gear manages saved locations. This tile
 * reads `primary` and only `primary` — which is exactly why it ships with no
 * new OAuth scope and no re-consent — so there is nothing to manage.
 *
 * NOT A MiniCard: this tile has pager buttons, an expand button, and per-row
 * links, and a button inside MiniCard's anchor is invalid markup that swallows
 * its own clicks. mini-card.tsx exports MINI_CARD_PANEL for precisely this.
 *
 * Every degradation is a calm muted line or a CTA — never an error banner.
 */
"use client";
```

Shape:

```tsx
const SWIPE_THRESHOLD_PX = 40;
const SLIDES = ["today", "tomorrow", "week"] as const;

export function CalendarCard() {
  const [data, setData] = useState<CalendarEventsResponse | null>(null);
  const [failed, setFailed] = useState(false);
  const [loading, setLoading] = useState(true);
  // Ephemeral by design — the tile always opens on Today. A user who left it
  // on the week strip yesterday wants today's events today.
  const [page, setPage] = useState(0);
  const [panelDate, setPanelDate] = useState<string | null>(null);
  // The local date this render is about. The refetch trigger compares against
  // it; it is state, not a ref, because the slides read it too.
  const [renderedDate, setRenderedDate] = useState(() => localDateKey(new Date()));

  const expandRef = useRef<HTMLButtonElement | null>(null);
  const touchStartX = useRef<number | null>(null);
  …
}
```

Behaviour:
- `load()` is a `useCallback` that reads `getToken()`, computes `requestWindow(new Date())`, calls `getCalendarEvents(token, Intl.DateTimeFormat().resolvedOptions().timeZone, startDate, endDate)`, and sets `data`/`failed`/`loading`. A throw or a null payload sets `failed` — which renders the same calm `Calendar is unavailable.` line the `unavailable` status does.
- **Midnight rollover.** One effect subscribing to `visibilitychange` and `focus`:
  ```tsx
  // A dashboard left open past local midnight would otherwise show yesterday
  // as "Today" indefinitely. No interval timer — a tile that polls a clock
  // every minute to catch one transition a day is a background cost for a
  // foreground problem.
  useEffect(() => {
    function check() {
      if (document.visibilityState === "hidden") return;
      const today = localDateKey(new Date());
      if (today === renderedDate) return;
      setRenderedDate(today);
      setPage(0);
      void load();
    }
    document.addEventListener("visibilitychange", check);
    window.addEventListener("focus", check);
    return () => {
      document.removeEventListener("visibilitychange", check);
      window.removeEventListener("focus", check);
    };
  }, [renderedDate, load]);
  ```
- Status rendering, before the slides:
  | status | body |
  | --- | --- |
  | `not_connected` | a `<a>` CTA: `Connect Google Calendar →`, `href` = `` `${config.apiUrl}/auth/google/calendar/connect?return_to=${encodeURIComponent(window.location.href)}` ``, styled like `MiniCardEmpty` (`text-sm text-[var(--muted)]` with the arrow in `--accent`). Check `app/(app)/settings/page.tsx:545` for the exact call the Settings page makes and match it. |
  | `reconnect_needed` | identical, `Reconnect Google Calendar →` |
  | `disabled` | `Calendar is off.` |
  | `unavailable` / a failed fetch | `Calendar is unavailable.` |
  | still loading | `<MiniCardSkeleton />` |
- Slides: `today` and `tomorrow` render `<DaySlide>` for `localDateKey(new Date())` and `localDateKey(addDays(new Date(), 1))`, looking the day up in `data.days`. `week` renders `<WeekStrip>` from `weekColumns(days, now)` and `nextUpcoming(days, now)`.
- The pager: `count = 3`, always rendered (there are always three slides). Copy weather's markup verbatim — ‹ with `aria-label="Previous slide"`, the three dots, › with `aria-label="Next slide"`, plus `onBodyKeyDown` / `onTouchStart` / `onTouchEnd`. The body wrapper is `role="group" aria-label="Calendar" tabIndex={0}` with the same `focus-visible:ring-1 focus-visible:ring-[var(--accent)]`.
- The header: a hand-composed panel

  ```tsx
  <div className={`${MINI_CARD_PANEL} relative flex flex-col gap-3`}>
    <div className="flex items-center justify-between gap-1.5">
      <MiniCardTitle title="Calendar" />
      {canExpand && (
        <button ref={expandRef} type="button" aria-label="Open the week agenda" …>
          <ExpandIcon />
        </button>
      )}
    </div>
    {body}
  </div>
  ```
  Reuse `MiniCardTitle` from `./mini-card`. Copy `ExpandIcon` from `weather-tile.tsx` — it is a 12-line local SVG in that file; **do not** import it from there (it is not exported) and do not export it from there either; a second copy of a local icon is the smaller cost. If it is exported, import it.
- `canExpand` is true only when the tile is showing slides (status `ok`).
- The panel: `panelDate !== null` renders `<CalendarWeekModal days={data.days} initialDate={panelDate} onClose={closePanel} />`, where `closePanel` sets `panelDate` to null **and returns focus to `expandRef`** — every close path funnels through it, matching weather's `closePopover` contract.
- `onOpenDay(date)` (passed to both `DaySlide` and `WeekStrip`) sets `panelDate` to that date.

- [ ] **Step 4: Run the tests**

```
npx vitest run "app/(app)/dashboard/_components/calendar/"
npm run typecheck && npm run lint
```
Expected: PASS.

- [ ] **Step 5: Check both breakpoints by eye**

The SOW requires the card to hold five rows plus a header plus the pager in a one-third-width cell without the grid row growing, and to render full-width single-column on mobile. Verify with `npm run dev` and the browser's device toolbar, or — if the environment has no browser — confirm by construction: the panel uses a fixed row height (`h-6` rows), the header and pager are fixed, and no element sets a width that would force overflow. State which of the two you did in the PR body.

- [ ] **Step 6: Commit**

```bash
git add "app/(app)/dashboard/_components/calendar/"
git commit -m "feat(calendar): add the self-fetching calendar dashboard tile"
```

---

## Task 12: The TypeScript catalog and the renderer

**Files:**
- Modify: `lib/dashboard-tiles.ts`
- Test: `lib/dashboard-tiles.test.ts`
- Modify: `app/(app)/dashboard/_components/tile-renderer.tsx`
- Test: `app/(app)/dashboard/_components/tile-renderer.test.tsx`

**Context:** The TS catalog mirrors the Go one and a contract test pins that they match. The entry has **no `href`** — and this is the third such entry. `quote` has no page; `weather` has no page; `calendar` has no page either, because there is no single destination "the calendar tile" deep-links to: its rows link to *different* places (a planned workout) or to nowhere (a lunch). `TileCatalogEntry.href` is already optional for exactly this.

- [ ] **Step 1: Write the failing test changes**

In `lib/dashboard-tiles.test.ts`:
- `expect(TILE_CATALOG.length).toBe(21)` → `22`.
- Append `"calendar"` to the order list, **after `"weather"`**.
- Append `"calendar"` to `PAGELESS_TILE_IDS`.
- Add `calendar: true` to the `ALL_TILE_IDS` exhaustiveness record.

In `app/(app)/dashboard/_components/tile-renderer.test.tsx`, add a case asserting `<TileCard id="calendar" …>` renders the calendar card (mock `getCalendarEvents` to resolve `{ status: "not_connected" }` so the assertion is on a stable, fetch-free body). Read the file's existing weather/quote cases and mirror them.

- [ ] **Step 2: Run to verify they fail**

```
npx vitest run lib/dashboard-tiles.test.ts
```
Expected: FAIL — count mismatch and a TS error on the unknown `calendar` key.

- [ ] **Step 3: Implement**

`lib/dashboard-tiles.ts` — append to the `TileId` union after `"weather"`:

```ts
  | "calendar";
```

Append to `TILE_CATALOG` after the `weather` entry:

```ts
  {
    id: "calendar",
    title: "Calendar",
    description: "Today and tomorrow from your Google Calendar, plus the week ahead.",
  },
```

(No `href` — see the task context.)

`tile-renderer.tsx` — add, immediately after the existing `weather` branch and **before** the `const href = tileEntry(id).href as string;` line:

```tsx
  // Calendar self-fetches, like weather: third-party data with its own auth and
  // failure modes has no business inside the single /dashboard/summary
  // round-trip, where a slow Google would delay every other tile on the grid.
  // Like quote and weather it has no page behind it, so it resolves before href.
  if (id === "calendar") {
    return <CalendarCard />;
  }
```

with `import { CalendarCard } from "./calendar/calendar-tile";` at the top.

- [ ] **Step 4: Run the full web suite**

```
npm run typecheck && npm run lint && npm run format:check && npm run test
```
Expected: PASS. **`app/(app)/dashboard/page.test.tsx` must pass UNMODIFIED** — if it fails, the tile has leaked into the default layout or the summary round-trip, which is a bug in this work, not in that test.

- [ ] **Step 5: Commit**

```bash
git add lib/dashboard-tiles.ts lib/dashboard-tiles.test.ts "app/(app)/dashboard/_components/tile-renderer.tsx" "app/(app)/dashboard/_components/tile-renderer.test.tsx"
git commit -m "feat(calendar): add the calendar tile to the catalog and renderer"
```

---

## Task 13: Amend the prior SOW

**Files:**
- Modify: `sows/planned-workouts-google-calendar-sync.md` (in `prog-strength-docs`)

- [ ] **Step 1: Add the cross-reference**

Find the section of `sows/planned-workouts-google-calendar-sync.md` that describes the grant / the scope (search for `calendar.events` or `CalendarEventsScope`). Add one sentence noting that the grant now has a reader, and link here:

```markdown
> **The grant now has a reader.** Since
> [`google-calendar-dashboard-tile`](google-calendar-dashboard-tile.md), the
> same `calendar.events` grant is also read — `events.list` on `primary`, for
> the dashboard's calendar tile. No new scope was needed and no user
> re-consented; the write path described here is untouched.
```

- [ ] **Step 2: Commit**

```bash
git add sows/planned-workouts-google-calendar-sync.md
git commit -m "docs: note that the calendar grant now has a reader"
```

---

## Final gate (before opening any PR)

**`prog-strength-api`** — CI runs golangci-lint at the version pinned in `.github/workflows/ci.yml`, `go vet`, a `go mod tidy` drift check, and `go test`. Run all of it locally:

```bash
go build ./...
go vet ./...
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@<CI-pinned version> run
go mod tidy && git diff --exit-code go.mod go.sum
go test ./...
```

**`prog-strength-web`**:

```bash
npm run typecheck && npm run lint && npm run format:check && npm run test && npm run build
```

**Never** `--no-verify`, never a `//nolint`, never a skipped test. If a check fails, the code is what is wrong.

