---
status: shipped
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Recovery Page & Sidebar Tab

**Status**: Shipped · **Last updated**: 2026-07-23

## Introduction

The Whoop integration (see `whoop-integration.md`) ships daily recovery data — recovery score, resting heart rate, and HRV — into **Prog Strength**, but the only surface it has is a dashboard mini-card showing today's resting HR and a 7-day spark. The card's deep-link still points at Settings → Integrations, a placeholder chosen when there was nowhere better to send a click. There is now a month of per-day data (and growing) with no page to browse it on.

This SoW gives recovery a first-class home: a **Recovery** tab in the sidebar and a `/recovery` page showing today's readiness at a glance and the trends behind it. The dashboard card's deep-link flips to the new page. It is deliberately web-only and read-only — the data is Whoop-owned, arrives on its own each morning, and needs no logging affordances.

One placement decision worth recording: workouts, running, and steps consolidated into `/activities`, but recovery does **not** follow them there. Activities are things the user *does* and logs; recovery is passively-ingested body signal, like bodyweight. It gets a top-level route the way `/bodyweight` does.

## Proposed Solution

A new `app/(app)/recovery/page.tsx` renders three sections in the steps-view's hero → chart → log grammar: a color-banded **score ring** for today (with resting HR and HRV beside it), **trend charts** for all three metrics over a 7/30/90-day toggle, and a reverse-chronological **day log**. All of it is fed by one call per range selection to the existing `GET /whoop/recovery` endpoint — the API needs no changes.

The sidebar gains a static **Recovery** item, always visible. Users without a Whoop connection land on an empty state that advertises the integration and links to Settings → Integrations — for a pre-launch product, the tab doubles as discovery for the feature it fronts.

## Goals and Non-Goals

### Goals

- A **Recovery** sidebar item (after Bodyweight) routes to `/recovery`, with a house-style inline SVG icon and standard active-state handling.
- The page shows today's recovery score in a ring color-banded by Whoop's familiar zones, with today's resting HR and HRV beside it, and a clear "no data yet today" state before the morning's webhook lands.
- Trend charts for recovery score, resting HR, and HRV over a selectable 7/30/90-day window (default 30), plus a browsable day log of the same rows.
- Users without a usable connection (absent, revoked, or error) see an explanatory empty state with a CTA into Settings → Integrations; a connected user with zero rows sees "your first recovery lands after tonight's sleep."
- The dashboard recovery card deep-links to `/recovery` instead of Settings.
- All data flows through the existing `GET /whoop/recovery` read endpoint using the house timezone+local-date convention; zero API-side changes.

### Non-Goals

- **Sleep, strain, or any new Whoop resource.** Recovery-only, per the integration SoW's v1 scope; each new resource is its own ingestion + surface work.
- **Connection-gated navigation.** The sidebar's `NAV` array stays static; the gate lives in the page as an empty state, not in the nav.
- **Write or edit affordances.** Recovery data is Whoop-owned; corrections arrive via `recovery.updated` webhooks, never from the user.
- **Deeper historical backfill.** The 90-day range will be sparse until history accumulates naturally; no API work to fetch older data from Whoop.
- **Pagination in the day log.** At most ~90 lightweight rows per range — the log renders what the charts already fetched.
- **Mobile.** Same parity-phase deferral as the rest of the integration.
- **Agent/MCP changes.** The `get_whoop_recovery` tool already serves the agent.

## Implementation Details

### Sidebar (`components/sidebar.tsx`)

One new entry in the static `NAV` array between Bodyweight and Progress: `{ href: "/recovery", label: "Recovery", icon: <RecoveryIcon /> }`. `RecoveryIcon` is a new inline stroke-only SVG at the bottom of the file matching the existing set (24 viewBox, 16px, `stroke="currentColor"`, `strokeWidth="1.75"`) — a heart-with-pulse glyph, visually distinct from `ActivityIcon`'s bare waveform. No `exact` flag (no sibling routes). Active state comes from the existing `startsWith` logic untouched.

### API Client (`lib/api.ts`)

One new type + function following the steps pattern:

- `WhoopRecoveryDay` — `{ date: string; recovery_score: number | null; resting_heart_rate: number | null; hrv_rmssd_milli: number | null }` (fields mirror the API's day rows; all three metrics nullable).
- `listWhoopRecovery(token, { timezone, since, until })` — `GET /whoop/recovery` with the timezone+local-date query convention (`internal/daterange` on the API side), `unwrap()` to `[]`. The client never builds UTC instants — it passes local dates and the IANA timezone, same as every date-windowed surface.

The existing `getWhoopConnection` is reused for the empty-state gate; nothing else changes in the client.

### Page (`app/(app)/recovery/page.tsx` + `_components/`)

Client component in the authed shell. On mount (and on range change): resolve the browser timezone, compute `since`/`until` local dates for the selected range, and `Promise.all([getWhoopConnection(token), listWhoopRecovery(token, …)])`. One fetch feeds all three sections; hero and log derive from the same rows client-side (steps-view pattern — no per-section endpoints).

**Render gate**, in order:

1. Connection `absent` or `revoked` → empty state: what the integration does, **Connect Whoop** button linking `/settings?tab=integrations`.
2. Connection `error` → same layout, reconnect phrasing ("Whoop connection needs attention — Reconnect in Settings").
3. Connected, zero rows in range → "your first recovery lands after tonight's sleep" (or, for larger ranges with older data elsewhere, the standard empty-chart treatment).
4. Data → the three sections below.

**Hero (`RecoveryHero`)**: an SVG ring in the steps `HeroRing` idiom, filled to today's `recovery_score` / 100 and colored by band — **success ≥ 67, warning 34–66, danger ≤ 33** — using the design system's semantic tokens (`--success` / `--warning` / `--danger`), never the periwinkle accent, so the banding reads as state, not brand. Score number centered; two `MiniStat`s beside it for today's resting HR (bpm) and HRV (ms). If today's row is absent, the ring renders unfilled with "no data yet today" and the MiniStats show em-dashes; the most recent day's values are NOT promoted into the hero (yesterday's readiness is not today's).

**Range toggle**: the settings `SegmentedToggle` primitive with 7d / 30d / 90d, default 30d. Selection is page state, not a URL param.

**Trend charts (`RecoveryTrends`)**: three recharts panels (the metrics share no axis — 0–100 score, bpm, ms), stacked on mobile widths, side-by-side where room allows, each in a `ResponsiveContainer` per the steps/running chart idiom:

- **Recovery score** — line chart, y fixed 0–100, with translucent horizontal band zones at the three color bands (recharts `ReferenceArea`) so a glance shows which zone the line lives in.
- **Resting HR** — line chart, y auto-fit, dashed `ReferenceLine` at the range average (lower is better; the average line gives "versus my normal" without pretending there's a goal).
- **HRV** — same treatment as resting HR (higher is better; same average reference line).

Days with a null metric are gaps, not zeros. New `CHART_RECOVERY_SCORE`, `CHART_RECOVERY_RHR`, `CHART_RECOVERY_HRV` (plus band fills) in `lib/chart-colors.ts`, hex-mirroring design-system tokens like the existing entries (recharts cannot resolve CSS vars).

**Day log (`RecoveryLog`)**: reverse-chronological list of the fetched rows — date, score as a small color-banded chip, resting HR, HRV; em-dash for nulls. Plain list, no accordion, no pagination (≤ ~90 rows, already fetched).

### Dashboard (`app/(app)/dashboard/page.tsx`)

`DEEP_LINKS.recovery` changes from `"/settings?tab=integrations"` to `"/recovery"`. The card itself is untouched — it already renders only when the summary carries the recovery block.

### Tests

- Sidebar renders the Recovery item and marks it active on `/recovery`.
- Page render gate: absent → connect CTA; error → reconnect phrasing; connected+empty → first-night copy; data → all three sections.
- Band mapping unit-tested at the boundaries (33/34, 66/67).
- Hero handles missing-today correctly (no promotion of yesterday's row).
- `listWhoopRecovery` passes timezone + local dates (no UTC construction) — mirrors the existing steps client tests.
- Chart data mapping: null metric days produce gaps (recharts `connectNulls` off).

### Design conformance

Everything on the page uses existing primitives (`SegmentedToggle`, `StatTile`/`MiniStat`, `MiniCard` idioms, chart-colors) and the design system (`prog-strength-docs/design-system.md`): dark near-black surfaces, Manrope, 14px radius, semantic tokens for the banding, accent reserved for interactive affordances (toggle, links).

## Open Questions

- **Band thresholds.** 67/34 mirrors Whoop's green/yellow/red convention so numbers agree with the user's Whoop app. If Whoop's published banding differs in detail, match Whoop — agreement with the source device beats internal preference.
- **`user_calibrating`.** Whoop marks new users' scores as calibrating for the first days; we don't currently ingest that flag. If early-days scores look odd, ingesting and badging it is a small follow-up to the integration, not this page.
