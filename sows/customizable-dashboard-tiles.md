---
status: draft
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Customizable Dashboard Tiles

**Status**: Draft · **Last updated**: 2026-07-31

> Frontend-led SOW with a supporting API change. `scope: in-system` — it conforms
> to design-system **v0.4** (dark near-black, periwinkle `#9aa6d6`, Manrope, 14px
> radius) and introduces **no** palette, accent, type, or token change. It builds
> on the shipped `dashboard-command-center.md` surface and does not re-tone it.

## Introduction

The dashboard is the default post-login landing surface — the first thing a Prog
Strength user sees every day. Its content is currently **fixed**: seven
hard-coded cards (Running, Lifting, Steps, Nutrition, Bodyweight, Recovery,
Streak) in a hard-coded order, above a hard-coded six-cell KPI strip.

That fixed set has two problems.

**It doesn't cover the product.** The unified activity model
(`unified-activity-model.md`) established six activity types — running, walking,
cycling, hiking, strength training, and the catch-all `other`. Only two of them
(running, strength) have a dashboard representation. A user whose training is
mostly hiking or cycling logs those sessions, sees them on the Activities page
and the calendar, and then lands on a dashboard that acts as if that training
doesn't exist.

**It isn't theirs.** Every user gets the same seven cards in the same order
whether or not they lift, log meals, or wear a Whoop strap. A user who never
logs nutrition gets a permanent "Log a meal to start tracking" card in a prime
slot; a user who lifts six days a week gets Lifting in position two regardless.
The dashboard's whole job is to answer *"how am I doing?"* in one glance, and
the answer to that is different per person.

After this ships, every activity type worth reviewing daily has a tile, and the
user arranges the dashboard themselves: drag tiles into the order they want,
remove the ones they don't use, and add ones they do. The dashboard becomes the
set of numbers *that user* wants to see on login, in the order they want to see
them.

## Proposed Solution

Five pieces, one API PR followed by one web PR.

1. **A tile catalog.** Ten tiles — one per subject — defined once in Go and
   mirrored in TypeScript: `running`, `walking`, `cycling`, `hiking`, `lifting`,
   `steps`, `nutrition`, `bodyweight`, `recovery`, `streak`. Three of these
   (walking, cycling, hiking) are new and are written by hand, section and card,
   the same way running and lifting are written today.

2. **A per-user layout, persisted server-side.** A new
   `user_dashboard_layouts` table holds one ordered JSON array of enabled tile
   ids per user. No row means "never customized" and yields the default layout,
   which reproduces today's dashboard exactly.

3. **`GET /dashboard/summary` becomes layout-aware.** It returns the resolved
   `layout` alongside the sections, and computes **only the enabled sections**.
   One round trip, and the order the client renders can never disagree with the
   data it received.

4. **`PUT /dashboard/layout`** writes the ordered tile ids.

5. **An edit mode on the dashboard.** A `Customize` control puts the grid into
   edit mode: tiles get a drag handle (`@dnd-kit`) and a remove button, and an
   inline tray below the grid lists the tiles not currently shown. `Done`
   persists and refetches; `Cancel` discards. The fixed KPI strip is **deleted**
   — with a user-ordered grid whose tiles already lead with their headline
   number, a second fixed row of the same numbers can only contradict the user's
   choices.

The command bar is not a tile. It stays pinned above the grid, and is neither
draggable nor removable.

## Goals and Non-Goals

### Goals

- Every activity type except `other` has a dashboard tile: running, walking,
  cycling, hiking, strength (lifting).
- A user can reorder dashboard tiles by dragging them.
- A user can remove any tile from their dashboard.
- A user can add any catalog tile that isn't currently on their dashboard.
- The layout persists per user, server-side, and survives browsers and devices.
- A user who has never customized sees exactly today's dashboard, minus the KPI
  strip.
- `GET /dashboard/summary` computes only the sections the layout enables.
- A tile added to the catalog without a matching card is a **compile error** in
  web, not a runtime blank.

### Non-Goals

- **No tile for the `other` activity type.** Its content would be, by
  definition, unspecific.
- **No multiple tiles per subject.** Exactly one tile per subject; no "Running
  pace" tile separate from a "Running volume" tile.
- **No tile resizing, spans, or a free-form grid.** Tiles are uniform cards in a
  reflowing grid; customization is membership and order only.
- **No new tile subjects** — no Upcoming/planned-workout tile, no Recent-PRs
  tile, no social-timeline tile. The catalog is exactly the ten subjects above.
- **No mobile work.** `prog-strength-mobile` has no dashboard screen; the layout
  API is built so it can adopt one later without rework, but that is not this
  SOW.
- **No live data preview in the add-tile tray.** The summary only computes
  enabled sections, so the tray lists names and descriptions.
- **No layout sharing, presets, or per-device layouts.** One layout per user.
- **No change to the command bar, the chat hand-off, or any deep-link target.**

## Implementation Details

### Data Model

Migration `049_dashboard_layout.sql`:

| Column | Type | Description |
| --- | --- | --- |
| `user_id` | text | Primary key. `REFERENCES users(id) ON DELETE CASCADE` — deleting a user drops their layout. |
| `tile_ids` | text | JSON array of enabled tile ids, in display order. |
| `updated_at` | timestamp | Last write. |

No index beyond the primary key: every read and write is by `user_id`.

There is **no CHECK constraint on tile ids**. Consistent with migration 042's
treatment of `activity_type`, the Go catalog is the source of truth for the
closed set; the write path validates, and the read path filters. Retiring a tile
therefore never requires a migration and never breaks a stored layout.

### The Catalog

`internal/dashboard/tiles.go` defines a `TileID` string type, one constant per
tile, and `Catalog` — the ordered slice that also fixes the order tiles appear
in the add-tile tray. `lib/dashboard-tiles.ts` mirrors it as a `TileId` union
plus `TILE_CATALOG` entries carrying the display title, deep-link href, and the
one-line description shown in the tray. A contract test asserts the two lists
are identical, following the existing `internal/activity/contract_test.go`
pattern.

Tile ids are deliberately the **existing summary section keys** (`lifting`, not
`strength_training`), so no field renaming is required anywhere.

### Read Path

`GET /dashboard/summary?timezone=<IANA>` resolves the layout first, then builds
only the enabled sections.

Two reads stay **ungated regardless of layout**, because other sections derive
from them:

- The 53-week `activityRepo.ListInRange` over all types — it feeds running,
  walking, cycling, hiking, the strength hydration, *and* the streak.
- The step entries and step goal — the streak credits goal-meeting step days,
  so disabling the Steps tile must not change the streak.

Gating therefore saves real work on nutrition, bodyweight, recovery, and the
lifting PR/headline reads, and saves builder CPU (not queries) on the endurance
tiles, whose rows are already in memory.

Response shape (additive — the existing section keys and shapes are unchanged):

```json
{
  "layout": ["running", "hiking", "steps", "streak"],
  "running": { "current_week": { … }, … },
  "hiking":  { "current_week": { … }, … },
  "steps":   null,
  "streak":  { … }
}
```

The `layout` array resolves the two meanings of a missing section:

- **id in `layout`, section `null`** → the tile is on the dashboard but has no
  data yet → render the tile's empty-state CTA.
- **id absent from `layout`** → the tile is not on the dashboard → render
  nothing.

**Layout-read failure degrades, it does not 500.** If the layout table read
fails, the handler logs and falls back to the default layout — the same
principle as the existing per-section `defer1` resilience, where one flaky table
can never blank the dashboard.

### Default Layout

For a user with no stored row:

```
running, lifting, steps, nutrition, bodyweight, [recovery,] streak
```

`recovery` is included **only when the user has a Whoop connection**. This
reproduces today's page exactly, where the Recovery card renders only when its
section is present — a non-Whoop user should not land on an empty Recovery card
they never asked for. Once such a user customizes, Recovery is available in the
add-tile tray like any other.

Defaults are resolved server-side on every read while no row exists; the first
`PUT` writes an explicit layout and the default stops applying.

### The Three New Sections

`walking.go`, `cycling.go`, and `hiking.go` in `internal/dashboard` are pure
builders over the already-fetched endurance slice, filtered by type, mirroring
`running.go`'s structure (including its `nil`-on-empty contract: no session of
that type, ever → `nil` section). Each has its own DTO and its own card; the
tile contract is hand-written per type, not generated. What they share is
`endurance_tiles.go` — two value types and one weekly-bucketing helper — so the
DST- and timezone-correct arithmetic exists once rather than three times.

| Tile | Headline | Body | Meta row | Deep link |
| --- | --- | --- | --- | --- |
| Walking | distance this week | 8-week weekly-distance spark | walks · time · last | `/activities` |
| Cycling | distance this week | 8-week weekly-distance spark | rides · time · last | `/activities` |
| Hiking | distance this week | 8-week weekly-distance spark | hikes · elevation gain · time · last | `/hiking` |

Hiking's elevation gain comes from `Activity.ElevationGainMeters`, already on
the base row — no detail-table read. Walking and Cycling have no dedicated view,
so they deep-link to the Activities overview. Distances render in the user's
`distance_unit`, like the Running card.

None of the three carries a `delta_pct_vs_prior_week`: only the deleted KPI
strip consumed a delta. `RunningSection.DeltaPctVsPriorWeek` **stays** on the
API (removing it would be a needless breaking change); the web adapter simply
stops reading it, and the `pctDelta` helper is deleted with the strip.

### Write Path

`PUT /dashboard/layout`, auth required, body `{ "tile_ids": ["running", …] }`.

- **Valid ids, no duplicates** → upsert the row, `204 No Content`.
- **Unknown id** → `422`, listing the valid ids.
- **Duplicate id** → `422`.
- **Empty array** → accepted. An empty dashboard is a legitimate preference; the
  page renders an empty-state CTA that opens the add-tile tray.
- **Read-back filtering** → ids no longer in the catalog are dropped on read, so
  a retired tile degrades silently instead of erroring.

### Web Surface

| File | Role |
| --- | --- |
| `lib/dashboard-tiles.ts` | `TileId` union, `TILE_CATALOG` (id, title, href, tray description) |
| `lib/dashboard.ts` | `adaptDashboard` returns `layout: TileId[]` plus walking/cycling/hiking views |
| `lib/api.ts` | `putDashboardLayout(token, tileIds)` |
| `app/(app)/dashboard/_components/tile-grid.tsx` | `DndContext` + `SortableContext`; sortable wrappers only in edit mode |
| `app/(app)/dashboard/_components/tile-renderer.tsx` | one exhaustive `switch (id)` mapping a tile id to its card |
| `app/(app)/dashboard/_components/add-tile-tray.tsx` | inline tray of catalog tiles not in the draft |
| `app/(app)/dashboard/_components/edit-bar.tsx` | `Customize` / `Cancel` · `Done` |
| `app/(app)/dashboard/_components/walking-card.tsx`, `cycling-card.tsx`, `hiking-card.tsx` | the three new cards |
| *deleted* | `KpiStrip`, `KpiStripSkeleton`, `kpi.tsx` |

New dependency: `@dnd-kit/core` + `@dnd-kit/sortable` (~12 kB gzipped). The repo
has no drag-and-drop library today; hand-rolling pointer-based sorting with
keyboard and touch support is strictly more code and more risk than the library.

Page state is `data` (summary + layout), `mode: "view" | "edit"`,
`draft: TileId[]`, `saving`, `saveError`. Edit mode is ephemeral — never
restored across a refresh, never encoded in the URL.

The tray is inline below the grid rather than a modal: against a ten-tile
catalog, a modal hides the grid the user is editing, while an inline tray
lets them watch the tile appear in place — and needs no focus trap or scroll
lock.

### Interaction and Accessibility

- `Customize` copies `layout` into `draft`. Drag, remove, and add mutate `draft`
  only.
- `Done` → `PUT`, then refetch the summary — newly enabled tiles have no data in
  the current payload — then exit edit mode.
- A failed save keeps the user **in** edit mode with the draft intact and an
  inline error beside `Done`. Nothing is lost, and retry is one click.
- `Cancel` discards the draft.
- `PointerSensor` with an 8px activation distance, so a touch drag on the grid
  still scrolls the page.
- dnd-kit's keyboard sensor plus a labelled drag handle make reordering
  keyboard-operable; each tile carries a `Remove {title}` button with an
  accessible name.
- Reorder, add, and remove are **pure functions** over `TileId[]`, unit-tested
  independently of the DnD library.

### Empty and Error States

- Enabled tile, `null` section → that tile's empty CTA ("Log a hike to start
  tracking").
- **Recovery gains an empty state it never needed before** — *"Connect Whoop to
  see recovery"* — because it is now a tile a user can deliberately enable
  without a connection.
- A per-section repo failure yields `nil` and therefore the same empty CTA
  (existing `defer1` behavior, unchanged).
- A failed summary fetch shows the existing page-level error banner.

### Testing

**Go**

- Catalog/enum test: every `TileID` constant appears exactly once in `Catalog`.
- Layout repository: round-trip, upsert-overwrites, unknown-id filtering on
  read, `ON DELETE CASCADE`.
- Layout handler: `204` on success, `422` on unknown id and on duplicates,
  empty array accepted, auth required.
- Summary handler gating: a disabled section is absent from the response; **the
  streak is unchanged when the Steps tile is disabled** (the shared-read trap);
  a layout-read failure falls back to the default rather than 500-ing.
- Default layout: with and without a Whoop connection.
- `walking`/`cycling`/`hiking` builders: `nil` on no history, weekly bucketing
  across timezones and DST boundaries, mirroring `running_test.go`.

**Web**

- Catalog ↔ `TileId` exhaustiveness.
- Reorder / add / remove pure functions.
- `page.test.tsx`: enter edit mode; `Cancel` discards; `Done` saves and
  refetches; a failed save keeps edit mode and the draft; empty dashboard shows
  the CTA.
- Tile-renderer exhaustiveness is enforced by the compiler (exhaustive `switch`
  over `TileId`), not by a test.

**Contract**

- Go `Catalog` and TS `TILE_CATALOG` assert-equal.

### Backfill or Migration

1. **Mechanism** — none. `user_dashboard_layouts` starts empty and every
   existing user resolves to the default layout on read. Rows appear lazily on
   first customization.
2. **Recoverability** — the migration creates one table and writes no rows;
   there is no partial state to recover. Rolling back is a `DROP TABLE`, after
   which every user resolves to the default.
3. **Scale boundary** — one small row per customizing user, read by primary key
   once per dashboard load. This shape holds well past any volume this product
   will see; it would only need revisiting if layouts became per-device or
   versioned.

**Rollout order:** the api PR ships first (additive `layout` field plus the new
`PUT`), then the web PR. No feature flag — the default layout reproduces today's
dashboard, so the only visible change for an existing user is the KPI strip
going away.

## Open Questions

1. **Should the KPI strip come back as an eleventh, user-orderable "summary"
   tile?** Options: leave it deleted; reintroduce it as a tile that renders the
   headline of the next N tiles; reintroduce it fixed. Tentative lean: leave it
   deleted and see whether the dense scan row is missed once the grid is
   user-ordered — reintroducing it later is additive.
2. **Does the `other` activity type eventually earn a tile?** Options: never;
   add one showing sessions and duration only; fold `other` sessions into a
   generic "Everything else" tile. Tentative lean: never — a type whose defining
   property is having no defining properties has nothing glanceable to show. If
   users accumulate meaningful `other` volume, that is a signal to promote it to
   a real type instead.
3. **Should mobile adopt the same layout when it grows a dashboard?** Options: a
   shared layout; a separate `mobile` layout row; mobile has no customization.
   Tentative lean: share the single layout — the tile catalog is subject-based,
   not viewport-based, and a user who removed Nutrition means it on both.
