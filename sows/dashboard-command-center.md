---
type: sow
status: ready_for_implementation
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Dashboard — Command-Center

**Status**: Ready for implementation · **Last updated**: 2026-06-19

> Frontend-led SOW with a supporting API change. It implements a chosen DX variant
> and therefore inherits that DX's `scope` — here **`in-system`**. The visual
> foundation (design-system **v0.4**, oura-calm) is already decided, so this SOW
> **conforms** to it; it does **not** re-tone the system or touch shared tokens.
> The dashboard surface lands in `prog-strength-web`; a server-side aggregate
> endpoint is added in `prog-strength-api`; `prog-strength-docs` flips the DX to
> `selected` and marks this SOW shipped. This is a **bigger** surface than a
> single-component rebuild: a new route, a sidebar entry, the default-landing
> redirect, a chat hand-off, and a new aggregate endpoint.

## Introduction

Prog Strength has **no top-level home** — after login the app drops the user into
`/chat`, and every domain (training, running, steps, nutrition, bodyweight,
records) lives behind its own deep tab. There's no one place that answers *"how am
I doing across everything?"* on open. The Design Exploration `dx/dashboard.md`
(PR Prog-Strength/prog-strength-web#91) explored a brand-new **Dashboard** — the
default post-login landing, a new sidebar selection, a glanceable v1 roll-up of
running · lifting · steps · nutrition · bodyweight · the activity streak, and a
top-level chat bar that hands a prompt into the chat view.

The DX explored five `in-system` variants and the owner selected **`command-center`**
(reference: **Linear**'s earned density / a trading console): **the dense
power-user console**. A compact **KPI strip** across the top puts every domain's
headline number in one scannable row; the **chat bar is a `›` command bar**;
beneath, tight **mini-cards with sparklines** pack maximum signal into minimum
space — the smallest type scale of the five, tabular alignment, mono numerals,
status-encoded (success/danger) deltas, single-accent restraint. It shows the most
without scrolling, and the brief is to keep it **calm and on-brand** at that
density rather than reading as a trading terminal.

This SOW builds that dashboard **production-quality**. Because `scope: in-system`,
**no palette, accent, type, or design-system change** — it conforms to v0.4. And
per the DX's central honesty constraint, **every metric traces to a real Prog
Strength metric** — there is **no readiness / HRV / sleep / stress** (Prog Strength
collects no wearable biometrics; Garmin/Strava were compositional references only).

**The one substantive backend addition is a server-side aggregate endpoint.** The
DX (and the first cut of this SOW) assumed a client-side fan-out of ~10 existing
endpoints. But the dashboard is the **default landing — the highest-traffic surface
in the app** — and the sparklines + streak require **weeks of raw history pulled to
the client just to reduce to ~8 sparkline points and a streak count**, which is
wire-wasteful and worsens as a user accumulates history. So the dashboard is fed by
a single **`GET /dashboard/summary`** that does the fan-out and aggregation
server-side (timezone-correct, one DB, small payload) and the web client becomes a
thin typed fetch. This collapses ~10 round-trips to **one** and moves the
spark-bucketing + streak derivation to Go.

## Proposed Solution

Six coordinated pieces, one API PR + one web PR:

1. **`GET /dashboard/summary` (API)** — a new aggregate endpoint that composes the
   existing domain repositories and returns the compact, dashboard-shaped payload
   (per-domain headline + sparkline series + the derived streak). Resilient
   internally: a failing/empty domain yields a `null` section, never a 500.

2. **The web data adapter** — a thin typed fetch of `/dashboard/summary` →
   `DashboardData`, with the unit/format conversion the client owns. (Replaces the
   ~10-call client fan-out the first cut proposed.)

3. **The command-center surface** — a new `app/(app)/dashboard/page.tsx` rendering
   the KPI strip, command bar, and dense mini-card grid, reimplementing the mockup
   `app/design-explore/dashboard/_variants/command-center.tsx` (the **visual spec,
   not code to promote**) with real per-card **empty / loading** states and **deep
   links** into each domain page.

4. **The chat hand-off** — the command bar routes to **`/chat?prompt=<encoded>`**;
   the existing chat page reads `?prompt=` and continues the session there (a small,
   additive change to the chat page — the dashboard never calls the agent itself).

5. **The sidebar entry** — a `Dashboard` item added to `components/sidebar.tsx`,
   positioned first as the primary overview.

6. **The default landing** — authenticated users land on `/dashboard` instead of
   `/chat` (the redirect is in `app/page.tsx`, `app/login/page.tsx`,
   `app/auth/callback/page.tsx`).

## Goals and Non-Goals

### Goals

**API (`prog-strength-api`)**

- **Add `GET /dashboard/summary?timezone=<IANA>`** — a new handler (a new
  `internal/dashboard` package, mounted under the authed group like the other
  domains; the cross-domain `internal/activity` overview is the precedent for
  composing repositories server-side) returning a compact JSON payload mirroring the
  web `DashboardData`:
  - **running** — current-week distance + run count + **Δ% vs prior week** + recent
    avg pace (reuse the existing running-metrics computation) + latest run + an
    **~8-week weekly-distance spark**.
  - **lifting** — current-week duration + sessions + sets + **PRs this week**, a
    headline estimated-1RM, and an **~8-week weekly-volume spark**.
  - **steps** — avg + today + goal + a **last-7-day spark**.
  - **nutrition** — today's calories + protein/carbs/fat vs goals.
  - **bodyweight** — current + **rate/week** + goal + a recent **trend spark**.
  - **streak** — derived server-side from the **union of activity dates** (workouts
    `performed_at`, runs `start_time`, steps `date`) de-duped by the user's local
    calendar day: `weeks`, `active_days_this_week`, the Mon–Sun `week[]` flags
    (streak definition is Open Question #2).
  - **Timezone-correct**: all week/day bucketing uses the request `timezone` via the
    existing `internal/daterange` convention; the model never builds UTC windows.
  - **Resilient**: each section is computed independently — a domain with no data (or
    a recoverable read miss) returns a `null` section, so the endpoint degrades
    per-domain rather than failing the whole dashboard.
  - **Reuse, don't fork**: build on the existing running-metrics, workout-volume /
    PR-event, steps, macro, and bodyweight-trend logic already in those packages
    rather than re-deriving; keep `current_estimated_1rm` semantics consistent with
    `/personal-records`.
- **Return native/metric values** (distances in meters, weights in their stored
  unit + a `unit`) so the **web converts to the user's display unit** — consistent
  with how the other endpoints return metric and the client converts.
- **Test** the handler: the full multi-sport user, the partial user (some sections
  `null`), the brand-new user (all sections `null` except a zeroed streak), the
  streak derivation across the date union, and timezone bucketing. Go tests + the
  api CI pipeline green; ships via the api repo's semantic-release.

**Web (`prog-strength-web`)**

- **Add `getDashboardSummary(token, timezone)`** to `lib/api.ts` + the
  `DashboardData` types (promoted from the mockup's `_fixtures/data.ts`), and a thin
  adapter that maps the payload, **converting to the user's display unit + format**
  (the mockup hardcodes `mi`/`lb`; honor `getMe()` unit/timezone — Open Question #5).
- **Create the `/dashboard` route** (`app/(app)/dashboard/page.tsx`, client
  component) inheriting the app shell, rendering the command-center composition:
  - **KPI strip** — Streak · Run · Lift · Steps · Fuel · Weight as a single
    hairline-divided row (6 cols desktop, reflowing on mobile), mono tabular
    numerals, the run/lift **discipline dots**, **status-encoded deltas**
    (success/danger — never the accent).
  - **Command bar** — the `›` chat input (periwinkle as command/active chrome) that
    hands off on submit.
  - **Mini-card grid** — dense per-domain cards (Running, Lifting, Steps, Nutrition,
    Bodyweight, Streak) with a big number, a **sparkline**, a meta row, and the
    nutrition macro bars; each card a **link into its deep page**
    (`/activities?view=running`, `/workouts`, `/activities?view=steps`,
    `/nutrition`, `/bodyweight`, `/activities`).
  - **In-system color**: periwinkle as command/active chrome only; the
    `--discipline-run-*` / `--discipline-lift-*` dots tag disciplines; desaturated
    `--success`/`--danger` carry deltas; `--macro-*` tints the macro bars; mono is
    incidental (Geist_Mono), Manrope elsewhere.
- **Wire the chat hand-off** — the command bar `router.push("/chat?prompt=" +
  encodeURIComponent(value))`; **extend the chat page** (`app/(app)/chat/page.tsx`)
  to read `?prompt=` on mount and start/continue the session with it, mirroring its
  `?session=` handling and lazy session creation (auto-send vs pre-fill is Open
  Question #1).
- **Add the sidebar entry** — a `DashboardIcon` (inline SVG, matching the existing
  nav-icon convention) + a `{ href: "/dashboard", label: "Dashboard", … }` entry at
  the **top** of the `NAV` array in `components/sidebar.tsx`, with the existing
  active-state highlighting.
- **Change the default landing to `/dashboard`** — update the three redirect
  touch-points (`app/page.tsx`, `app/login/page.tsx`, `app/auth/callback/page.tsx`)
  from `/chat` to `/dashboard`.
- **Handle every state the DX required**: the **brand-new / empty user** (the first
  thing they ever see — every card an inviting empty CTA, the streak reads "start
  your streak," never a `0`/`NaN%`); the **partial user** (each card degrades from
  its `null` section); **no goals set** (steps/macro/bodyweight attainment → "set a
  goal"); **loading** (the single fetch in-flight → the KPI strip + cards paint
  skeletons); **both breakpoints** (the KPI strip and grid reflow to a clean mobile
  single column with the command bar reachable).
- **Tests** — the adapter (payload → `DashboardData`, unit/format conversion, the
  null-section paths); component tests that the KPI strip + mini-cards render from a
  known payload, empty/loading render, deep links point at the right routes, and the
  command bar routes to `/chat?prompt=…`. **CI green** (lint/format/typecheck/test/build).

### Non-Goals

- **No second aggregate / no GraphQL.** One `GET /dashboard/summary` read endpoint;
  no write paths, no per-widget endpoints.
- **Any design-system or shared-token change.** `scope: in-system` against v0.4 —
  conform only; no token/accent/type edit, no `design-system.md` change.
- **No invented metrics.** No readiness / HRV / sleep / stress / training-status —
  only the real metrics above.
- **Not reimplementing the deep pages.** The dashboard is a glanceable **overview
  that links into** Activities / Workouts / Nutrition / Bodyweight — it does not
  replace their charts, logs, or editing. No drill-down, no editing on the dashboard.
- **Not building the other four variants** (focus-grid, hero-summary,
  discipline-columns, feed-digest); the DX PR is **closed, never merged**; the
  `design-explore/dashboard` route stays gated and never ships.
- **MCP / SDK changes.** The web calls the Go API directly (`lib/api.ts` `fetch`);
  the agent/MCP doesn't consume the dashboard summary. Neither is in `repos:`.
- **No new chat capability / no agent, voice, or session-model change.** The
  hand-off only passes a prompt in via the URL.
- **No new aggregation tables or denormalized cache.** `/dashboard/summary` computes
  on read from the existing tables; a materialized/cached summary is a later
  optimization if read latency demands it, not v1.

## Implementation Details

### API — `GET /dashboard/summary` (`prog-strength-api`)

A new `internal/dashboard` package: a `Handler` that takes the (already-constructed)
workout, running, steps, nutrition, and bodyweight repositories/services and a
`Mount(r)` registering the authed route — the same shape as the other domain
handlers, with `internal/activity`'s cross-domain overview as the composition
precedent. The handler resolves the user's timezone (request param, validated as the
other tz-aware endpoints do), computes each section concurrently or sequentially
(single DB, cheap), and assembles the DTO. Sparks are weekly/daily buckets over the
windows below; the streak walks the de-duped activity-date union backward from
today. Each section is wrapped so a recoverable error/empty → `null` section, logged,
not fatal. New handler-level tests cover full/partial/empty/streak/timezone.

### Web — adapter + route + presentation (`prog-strength-web`)

`getDashboardSummary` in `lib/api.ts`; a thin adapter (unit/format conversion only —
the heavy lifting is server-side now); `app/(app)/dashboard/page.tsx` calling it
(an effect or a TanStack query per app convention) and rendering the command-center
components ported from the mockup — `Kpi`, `CommandBar`, `MiniCard`, `BigNum`,
`MetaRow`, `MacroBar`, `Spark`, `compact()` — wired to the typed payload and the
design-system tokens. The `Spark` SVG and the dense layout port directly.

### Web — chat hand-off, sidebar, default landing

The mockup's `CommandBar` already does `router.push("/chat?prompt=" + …)`. The
production change is on the **chat page**: read `searchParams.get("prompt")` on
mount, feed it into the composer / send flow the way `?session=` is handled, then
strip the param so refresh/back doesn't re-fire (Open Question #1). Add the
`Dashboard` nav entry first in `components/sidebar.tsx`; flip the three redirects to
`/dashboard` (keep `/chat` reachable from the nav).

### Tests

- **API** — `/dashboard/summary` over full / partial / empty users, the streak
  derivation, timezone bucketing, and per-section `null` degradation.
- **Web** — adapter mapping + unit conversion; KPI strip + mini-cards from a known
  payload; empty/loading; deep-link hrefs; command bar → `/chat?prompt=`.

## Rollout

The web dashboard **depends** on the endpoint (unlike an additive nullable field),
so the API ships first or they ship in lockstep.

1. **`prog-strength-api`** — `GET /dashboard/summary` + tests; ship via the api
   semantic-release pipeline. Verify the payload over a full / partial / brand-new
   account with a real timezone.
2. **`prog-strength-web`** — the adapter + `/dashboard` route + sidebar entry +
   default-landing redirect + chat-page `?prompt=` + tests, in one PR. Vercel
   preview to verify across a full / partial / brand-new account and both
   breakpoints, and that login now lands on `/dashboard`.
3. **`prog-strength-docs`** — flip `dx/dashboard.md` to `status: selected` (winning
   idiom `command-center`); mark this SOW `shipped`.

### Verification after rollout

- `GET /dashboard/summary` returns the compact payload in **one** request
  (timezone-correct sparks + streak), degrading per-section to `null` for a partial
  or brand-new account, with every existing endpoint untouched.
- After login the user **lands on `/dashboard`** (not `/chat`); a `Dashboard` entry
  sits first in the sidebar and is active.
- The page reads as a **calm dense console**: the six-metric KPI strip
  (Streak · Run · Lift · Steps · Fuel · Weight), the `›` command bar, and a grid of
  sparkline mini-cards — every number tracing to real data, **no readiness/HRV/sleep**,
  fed by the **single** summary call (Network shows one `/dashboard/summary`).
- Each card **links into its deep page**; the **streak** reads intentionally (active
  streak, or "start your streak" at zero).
- The **command bar** sends the user to `/chat` with the prompt carried over and the
  session continuing there.
- A **brand-new account** sees an inviting board of empty-state CTAs (no `0`/`NaN%`);
  a **partial** account degrades per-card; **no-goal** domains show "set a goal";
  **skeletons** appear while the summary loads; **mobile** reflows cleanly.
- `design-system.md` unchanged; the `design-explore/dashboard` route stays gated /
  404 in production; **no DX mockup code shipped**; the DX PR is closed (never
  merged).

## Open Questions

1. **Chat hand-off: auto-send vs pre-fill.** The user typed a question on the
   dashboard expecting an answer. **Lean:** on `/chat?prompt=…`, **auto-send once**
   on mount (lazy-creating the session as the chat page already does), then strip the
   param so refresh/back doesn't re-send — "continue that chat session" reads as the
   answer already starting. Alternative: **pre-fill** the composer and let the user
   hit enter. Confirm at review.
2. **Streak definition (now server-side).** What counts as an **active day** (any
   logged workout *or* run *or* steps entry? steps only if `> 0` / `≥ goal`?) and an
   **active week** (≥1 active day?), and is `weeks` consecutive active *weeks* or
   *days*? **Lean:** active day = any workout/run/steps-logged day; active week = ≥1
   active day; `weeks` = consecutive prior active weeks (matches the Strava-style "25
   weeks" framing). User-visible and easy to get subtly wrong — confirm.
3. **Endpoint shape: one summary vs sectioned.** **Lean:** a single
   `GET /dashboard/summary` returning all sections (one round-trip, simplest client).
   If one domain is materially slower server-side, the handler can still compute
   sections concurrently; only split into separate endpoints if a section needs
   independent caching later. Confirm the single-endpoint shape.
4. **Sparkline windows.** **Lean:** ~8 weeks (running distance, lifting volume),
   last 7 days (steps), the bodyweight trend points. These set the server's read
   windows and the payload size — confirm.
5. **Units & display preferences.** The server returns native/metric; the **web
   converts** to the user's display unit (and uses the IANA timezone from `getMe()`
   for the request). **Lean:** keep conversion client-side, consistent with the rest
   of the app (`useDistanceUnit`). Confirm the unit source.
6. **Read-time compute vs cache.** `/dashboard/summary` computes on read from
   existing tables. **Lean:** fine for v1 at current data sizes; if the landing
   feels slow on large histories, add a short-TTL cache or a materialized summary as
   a **follow-up**, not now.
