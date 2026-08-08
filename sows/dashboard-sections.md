---
status: proposed
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Dashboard Sections

**Status**: Proposed · **Last updated**: 2026-08-08

> Frontend-led SOW with a supporting API change. `scope: in-system` — it conforms
> to design-system **v0.4** (dark near-black, periwinkle `#9aa6d6`, Manrope, 14px
> radius) and introduces **no** palette, accent, type, or token change. It builds
> directly on the shipped `customizable-dashboard-tiles.md` layout contract and
> replaces that contract's flat tile list.

## Introduction

`customizable-dashboard-tiles.md` gave the dashboard a user-owned layout: an
ordered `TileId[]`, persisted per user, drag-reorderable in an edit mode. That
solved *which* tiles and *in what order*. It did not solve **grouping**.

With nineteen tiles in the catalog and a three-column grid, a user who enables a
dozen of them gets one long undifferentiated run of cards. Order alone cannot
express "these four are my running tiles, these three are recovery" — the only
signal is adjacency, and adjacency is invisible once a row wraps. The tiles a
user thinks of as a set do not *read* as a set.

This SOW adds **sections**: named groups the user creates, each holding tiles,
each rendered under a section header and a hairline rule. A user with no
sections sees exactly today's dashboard.

## Proposed Solution

The layout stops being a list of tiles and becomes a **list of sections, each
owning its tiles**. One uniform container — there is no "loose tile" concept and
no second drop-target kind.

A section with an empty title renders as a bare grid: no header, no rule. Every
existing layout migrates to a single untitled section, so an untouched dashboard
is byte-identical to today's and the feature is invisible until the user creates
their second section.

Edit mode grows from one sortable axis to two: sections reorder vertically by
their headers, tiles reorder within and drag **between** sections.

## Goals and Non-Goals

### Goals

1. A user can create, rename, reorder, and delete sections, with no cap that a
   real dashboard would hit (12).
2. A user can move any tile into any section by dragging it there, including
   into an empty section.
3. A user can collapse a section in view mode to get it out of the way, and that
   state survives a reload.
4. Sections persist server-side and follow the user across devices, the same as
   tile order already does.
5. An existing user sees no visual change until they create a section.

### Non-Goals

1. **Per-section layout options** — column count, tile size, section color. One
   grid, one density, everywhere.
2. **Preset/suggested sections** — no "we noticed you have 4 running tiles"
   nudge. The user names their own groups or has none.
3. **Sharing or templating sections** between users.
4. **Mobile** — mobile has no dashboard surface yet. When it grows one it adopts
   this contract unchanged; nothing in the model is viewport-specific.
5. **Nested sections.** One level.

## Implementation Details

### Data Model

Migration `055_dashboard_layout_sections.sql` restructures the column added by
`049`:

| Column | Type | Description |
| --- | --- | --- |
| `user_id` | text | Primary key. `REFERENCES users(id) ON DELETE CASCADE`. Unchanged. |
| `sections` | text | JSON array of section objects, in display order. **Replaces `tile_ids`.** |
| `updated_at` | timestamp | Last write. Unchanged. |

Each section object:

```json
{ "id": "s_a1b2", "title": "Recovery", "collapsed": false, "tile_ids": ["hrv_balance", "recovery"] }
```

The migration adds `sections`, backfills every row by wrapping its existing
`tile_ids` array into one untitled section, then drops `tile_ids`. As with
`049`, there is **no CHECK constraint** on tile ids: the Go catalog stays the
source of truth, the write path validates, and the read path filters.

In Go (`internal/dashboard/layout.go`):

```go
type Section struct {
    ID        string   `json:"id"`
    Title     string   `json:"title"`
    Collapsed bool     `json:"collapsed"`
    TileIDs   []TileID `json:"tile_ids"`
}
type Layout struct { Sections []Section }
```

### Invariants

Enforced on write, re-enforced on read (a stored layout can predate a rule):

- **A tile id appears at most once across the entire layout**, not merely within
  one section. `049`'s per-list duplicate check widens to a global one — the
  same tile in two sections would render twice and desync on remove.
- Section ids are non-empty and unique within the layout. They are opaque and
  client-generated; the server never interprets them.
- **Caps**: 12 sections, 40-character titles. Titles are stored trimmed;
  whitespace-only normalizes to `""`.
- **The layout always holds at least one section.** An empty `sections: []`
  normalizes to a single untitled empty section, so the read path and the web
  surface never handle a sectionless layout.
- Unknown tile ids are rejected on write and filtered on read, unchanged.

### Read Path

`GET /dashboard/summary?timezone=<IANA>` returns `sections` where it previously
returned `layout`. The per-tile data keys are untouched: which tiles need
building is the flattened union of every section's `tile_ids`, so section
structure never reaches the builders.

`resolveLayout` keeps its degrade-to-default behavior — a layout read failure is
logged and falls back rather than blanking the dashboard. `defaultLayout`
returns the same Whoop-conditional tile set as today, wrapped in one untitled
section.

### Write Path

`PUT /dashboard/layout`, auth required, body `{ "sections": [ … ] }`.

- **Valid** → upsert, `204 No Content`.
- **Unknown tile id** → `422`, listing the valid ids.
- **Duplicate tile id anywhere in the layout** → `422`, naming the tile.
- **Duplicate or empty section id** → `422`.
- **Over 12 sections, or a title over 40 chars** → `422`.
- **Empty `sections`** → accepted, normalized to one untitled empty section.
- **A bare `{ "tile_ids": [...] }` body** → accepted, wrapped into one untitled
  section. This is the compatibility path for a browser tab left open across the
  deploy; without it that tab's save 422s. It is ~10 lines and may be dropped
  once no old clients remain.

### Web Surface

| File | Role |
| --- | --- |
| `lib/api.ts` | `DashboardSection` type; `getDashboardSummary`/`putDashboardLayout` on the sectioned contract. |
| `lib/dashboard.ts` | Adapter carries `sections` through in place of `layout`. |
| `_components/layout-ops.ts` | Pure `Section[]` operations (below). No React, no dnd-kit. |
| `_components/section-list.tsx` | **New.** Owns the single `DndContext` and both sortable axes. |
| `_components/section-header.tsx` | **New.** View: title, rule, collapse chevron. Edit: drag handle, inline title input, delete, `+ Add tile`. |
| `_components/tile-grid.tsx` | Narrows to "render one section's tiles"; `SortableTile` chrome unchanged. |
| `_components/add-tile-tray.tsx` | Gains an "Add to: [section ▾]" target picker. |
| `page.tsx` | Draft/commit machine over `Section[]`; view-mode collapse write; delete confirm. |

`lib/dashboard-tiles.ts` and `internal/dashboard/tiles.go` are **untouched** —
sections are orthogonal to the tile catalog, and the existing contract test
between them keeps passing unchanged.

### Layout Operations

All pure, all directly unit-tested, per `049`'s established split between layout
math and the drag library:

`createSection` · `renameSection` · `deleteSection` · `moveSection` ·
`toggleCollapsed` · `moveTile(sections, tileId, toSectionId, toIndex)` ·
`addTile(sections, tileId, sectionId)` · `removeTile` · `availableTiles`

`moveTile` covers both within-section reorder and cross-section move — one
function, because the drag layer cannot always tell which one a drop is until it
resolves the target container.

`deleteSection` removes the section **and its tiles**. The tiles are not
rehomed. The confirm dialog lives in the page; the op stays dumb.

`availableTiles` flattens across sections before diffing against the catalog.

### Interaction and Accessibility

One `DndContext` wraps everything, with two sortable axes distinguished by
`data: { type: "section" | "tile", sectionId }` on each `useSortable`:

- **Sections** — vertical sort, keyed by section id, dragged by the header handle.
- **Tiles** — `rectSortingStrategy` within each section's `SortableContext`.

Cross-section moves resolve in `onDragOver` (dnd-kit's multi-container pattern),
with each section registered as a droppable so an **empty** section is a valid
target. Collision detection moves from bare `closestCenter` to `pointerWithin`
with a `rectIntersection` fallback — `closestCenter` alone resolves ambiguously
once containers nest.

The 8px pointer activation constraint is kept, so a touch drag still scrolls
before it reorders. The `KeyboardSensor` and `sortableKeyboardCoordinates`
carry over; cross-section keyboard movement is explicitly in scope for the
tile axis and is the one interaction most likely to need iteration.

Section headers are real headings (`<h2>`); the collapse control is a
`<button>` carrying `aria-expanded`, naming its section.

### Two Write Paths

Edit mode keeps today's machine: mutate a local `draft`, `PUT` on Done, discard
on Cancel.

**Collapse/expand is the exception** — it happens in view mode, outside that
machine. It applies optimistically to local state and fires an immediate `PUT`;
a failure reverts the toggle and surfaces the page's existing inline error. This
is the only write this feature adds outside edit mode, and the only place the
dashboard mutates without an explicit Done.

### Empty and Error States

- **Empty section, view mode** — hidden entirely. A header over nothing is noise.
- **Empty section, edit mode** — shown as a dashed drop zone, so it can be
  dragged into.
- **Untitled section** — no header, no rule, in both modes; in edit mode the
  title input is present but blank with a placeholder.
- **Empty dashboard** — the existing CTA, shown when the flattened tile count
  across all sections is zero.
- **Failed save** — unchanged: stay in edit mode, keep the draft, inline error.

### Testing

**Go**

- Repository: round-trip with multiple sections, upsert-overwrites, unknown-id
  filtering on read, empty-sections normalization, `ON DELETE CASCADE`.
- Handler: `204` on success; `422` on unknown tile id, on a tile duplicated
  *across* sections, on duplicate/empty section id, on over-cap sections, and on
  an over-length title; empty `sections` accepted; bare `tile_ids` body accepted
  and wrapped; auth required.
- Migration: a row with N tile ids backfills to exactly one untitled section
  holding those ids in order; an empty array backfills to one empty section.
- Summary: sections flatten to the correct build set; a tile enabled in the
  second section builds identically to one in the first; layout-read failure
  falls back to the default.

**Web**

- Every layout op, exhaustively — especially `moveTile` across sections, into an
  empty section, and to a no-op target; and `deleteSection` taking its tiles.
- `section-header`: renders no header when untitled; collapse toggles
  `aria-expanded`.
- `page.test.tsx`: create/rename/delete a section; delete confirm cancels
  cleanly; Cancel discards section edits; Done saves the sectioned body;
  view-mode collapse issues a `PUT` and reverts on failure.
- Tile-renderer exhaustiveness stays compiler-enforced, not tested.

### Backfill or Migration

1. **Mechanism** — a single in-migration `UPDATE` wrapping each existing
   `tile_ids` array into a one-element `sections` array, then `DROP COLUMN
   tile_ids`. Rows are one small JSON blob per customizing user; the table is
   read by primary key.
2. **Recoverability** — the wrap is lossless and deterministic, so the inverse
   (unwrap the first section's `tile_ids`) fully reconstructs the old column.
   Rolling back mid-deploy leaves users on the default layout at worst, which is
   the same state a `049` rollback produced.
3. **Scale boundary** — unchanged from `049`. One row per customizing user, one
   primary-key read per dashboard load. Sections add bytes to a blob that was
   already being read whole.

**Rollout order:** the api PR ships first — its tolerant reader means the
currently-deployed web client keeps saving successfully against the new
contract — then the web PR. No feature flag: an existing layout becomes one
untitled section, which renders exactly as it does today.

## Open Questions

1. **Should `deleteSection` rehome its tiles instead of deleting them?** This
   SOW deletes them behind a confirm, per an explicit product call. The
   alternative — append them to the preceding section — needs no confirm and has
   no regret path, but reads less like "delete this group." Tentative lean: ship
   the confirm, and revisit if the confirm turns out to be the friction rather
   than the safety.
2. **Should collapse state be per-device rather than per-user?** It persists
   with the layout today, so collapsing on a phone collapses on a laptop.
   Options: keep it in the layout; move it to `localStorage`; store both.
   Tentative lean: keep it in the layout until someone notices — a collapsed
   section is a statement about the tiles, not about the screen.
3. **Does an untitled section need a visible affordance in view mode?** Today it
   is completely invisible, which is correct for the migrated single-section
   case but means a user who empties a title loses the section's only handle
   until they re-enter edit mode. Tentative lean: leave it invisible; edit mode
   is one click away and always shows every section.
