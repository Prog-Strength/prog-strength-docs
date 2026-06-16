---
type: sow
status: draft
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Timeline Redesign — Strava-Social-Dashboard

**Status**: Draft (blocked — clarification needed) · **Last updated**: 2026-06-16

## ⚠️ Blocking clarification needed before implementation (2026-06-16)

Implementation was started and **halted at the API piece** because the SOW's
single, central backend deliverable rests on a premise that does not hold
against the current data model. Raising this as a draft PR rather than
guessing at a scope the owner has not approved.

### The premise that doesn't hold

The SOW states the one real API change is to expose **run route geometry** —
"a simplified (Douglas–Peucker-reduced) polyline plus bounds … **derived from
the run's existing `trackpoints`**" — and frames it as the only gap: "What the
chosen design actually needs that the backend lacks is the route geometry for
run cards."

But **no geographic position (latitude/longitude) is stored or even parsed
anywhere in the stack.** The trackpoint series is a *distance / pace / heart-
rate / elevation* stream with **no coordinates**:

- **TCX parser** — `internal/activity/tcx_parser.go` (`xmlTrackpoint`,
  `parsedTrackpoint`, lines ~32–70) decodes only `Time`, `DistanceMeters`,
  `HeartRateBpm`, `AltitudeMeters`. The TCX `<Position>` element
  (`<LatitudeDegrees>` / `<LongitudeDegrees>`) is **never read** — it is
  dropped on import.
- **Database** — `internal/db/migrations/015_activities_generalize.sql`
  (`activity_trackpoints`, lines 79–89) has columns `elapsed_seconds`,
  `distance_meters`, `heart_rate_bpm`, `pace_sec_per_km`, `elevation_meters`.
  **No `latitude` / `longitude` columns.**
- **Domain model** — `internal/activity/model.go` `Trackpoint` (lines 63–70)
  has `Sequence`, `ElapsedSeconds`, `DistanceMeters`, `HeartRateBpm`,
  `PaceSecPerKm`, `ElevationMeters`. **No coordinates.**

A 2-D map path cannot be reconstructed from cumulative distance alone, so the
"compact route summary" the SOW asks for **cannot be derived from existing
data**. The raw TCX files in S3 (`TCXS3Key`) *may* contain `<Position>` for
GPS-recorded runs, but that geometry is discarded at import and never
persisted, and re-reading S3 per feed post is exactly the per-post S3/N+1 fan-
out the SOW warns against.

### Why this blocks rather than degrades quietly

The route map is not incidental — it is the **signature element** of the
chosen `strava-social-dashboard` design ("a prominent **route map**", "the
Strava-signature run map", "route-forward card"). Two paths forward both need
the owner's call:

1. **Implementing route geometry for real** requires (a) extending the TCX
   parser to capture `<Position>`, (b) a **schema migration** adding
   `latitude` / `longitude` to `activity_trackpoints`, (c) a **backfill** that
   re-parses every existing activity's raw TCX from S3 (stored points have no
   position), and (d) graceful handling of runs whose TCX has no `<Position>`
   at all (treadmill / indoor / non-GPS watches). This directly contradicts
   the SOW's stated scope — "the only backend change", "**additive and
   backward-compatible** (a new optional `content.route` field)", "the feed
   aggregation … untouched", deploy-first-no-migration. It is a materially
   larger backend effort than the SOW authorizes and needs explicit sign-off.

2. **Shipping the dashboard without the route map** (token conformance, the
   three-column layout, ActivityCards with the workout radar / milestone
   banner / big-stat row / kudos + comment footer, the left "your week" rail,
   and the discovery rail — everything that does *not* depend on geometry,
   with `RouteMap` rendering a graceful "no route" state) is fully buildable
   today and needs **no** backend change. But it drops the element the owner
   selected this direction *for*, so it should be an explicit de-scope, not a
   silent omission. (If chosen, the API piece in this SOW becomes a no-op and
   should be struck.)

**Recommended:** split the SOW — land **option 2** now (the dashboard rebuild
+ token conformance, route map deferred to a graceful placeholder), and spin
**option 1** (GPS position capture: parser + migration + backfill) into its
own backend SOW, since it is the real prerequisite for the route map and is
out of scope for "the one real API change." Once the owner picks a path, this
SOW can be updated and implementation resumed.

No code PRs were opened in `prog-strength-api` or `prog-strength-web`; this is
the only PR for this SOW until the scope is resolved.

## Introduction

The Timeline (`/timeline`) is Prog Strength's social surface — a multi-author feed of training as it happens, with reactions and comments. The Design Exploration `dx/timeline.md` (PR Prog-Strength/prog-strength-web#59) explored it as a proper social dashboard, and the owner selected **`strava-social-dashboard`**: a three-column layout with a left "your week" rail (profile · streak · a this-week mini-chart), a center following feed of confident activity cards (a prominent **route map**, a labeled big-value stat row, kudos and comments), and a right discovery rail.

One important correction to the scope as it was first framed: **the social following feed already exists.** `GET /timeline` is already the aggregated multi-author home feed — it returns the viewer's own posts *plus* their accepted-followees' non-private posts, hydrated with reactions and comment counts. The timeline only *looks* solo today because this account follows no one yet (no followees → the feed is just your own posts). So no feed-aggregation backend needs building. What the chosen design actually needs that the backend lacks is the **route geometry for run cards**: the feed projection carries only `title / subtitle / metrics / href`, with no polyline — so the Strava-signature run map has nothing to draw. That is the one real API change in this SOW.

So this is an api + web redesign with a tightly-scoped backend piece (expose a compact run route in the feed projection) and the visual rebuild on the web. The redesign also **conforms to the app's new design tokens** — the slate/violet/Nunito foundation established by the chat & app-shell SOW (`sows/chat-and-app-shell-redesign.md`) — rather than the variant's standalone orange-on-near-black palette, so the timeline matches the rest of the app. The athletic *character* the owner picked (bold condensed display titles, big stat numerals, the route-forward card) is kept; the orange accent yields to the app's violet. The result: a social feed that feels alive and athletic, shows real route maps, and sits coherently in the redesigned app.

## Proposed Solution

Two pieces, one shippable change.

**API (`prog-strength-api`) — route geometry in the run feed projection.** The timeline feed hydrates each post into a denormalized `content` block so cards need no per-source fetch. For run posts, extend that block with a **compact route summary** — a simplified (Douglas–Peucker-reduced) polyline plus bounds — derived from the run's existing `trackpoints`, present only when the run has them. Kept small (a feed page hydrates many posts; this must not bloat the payload or re-introduce an N+1), consistent with the "keep it small" intent of the TCX-import design. This is the only backend change; the feed aggregation, reactions, comments, and pagination are untouched.

**Web (`prog-strength-web`) — the dashboard rebuild.** Redesign `/timeline` to the three-column strava-social-dashboard structure, conforming to the app-wide slate/violet/Nunito tokens:

- **Left "your week" rail** — profile (real name/avatar), a **streak** figure, a **this-week mini-chart**, and run/lift week stats — *derived client-side* from existing activity data (the same kind of weekly aggregation the calendar already does), so no new endpoint is needed.
- **Center following feed** — confident **ActivityCards** for the existing post types (run / workout / pr / best_effort): a milestone banner for PRs and best efforts, the author row (with a "You" badge on your own posts), title/subtitle/notes, a **route map** for runs (from the new feed geometry), the existing **muscle-group radar** for workouts, the labeled **big-value stat row**, and a kudos + comment footer. Reactions and comments already exist — they're restyled, not rebuilt.
- **Right discovery rail** — a lightweight "find people to follow" backed by the **existing user search + follow** APIs (not a new suggestion algorithm). A full suggested-athletes ranking and a leaderboard are explicitly deferred.
- **No-followers state, first-class** — since this account follows no one, the feed must feel intentional empty: surface your own training and make discovery the hero, not a dead end.

## Goals and Non-Goals

### Goals

- **Run route geometry in the feed projection** (`prog-strength-api`): extend the run post's `content` with a compact, simplified route polyline + bounds, only when `trackpoints` exist, sized for a multi-post feed page.
- **Rebuild `/timeline`** to the three-column strava-social-dashboard: left "your week" rail · center following feed · right discovery rail.
- **Conform to the app-wide tokens** (slate/violet/Nunito from `sows/chat-and-app-shell-redesign.md`): keep the athletic *character* (condensed display titles, big stat numerals, route-forward cards) but draw accent and surfaces from the shared tokens — the variant's **orange accent yields to violet**.
- **ActivityCards** for all post types: milestone banner (pr/best_effort), author row + "You" badge, title/subtitle/notes, **run route map**, **workout radar** (the existing breakdown), the labeled big-stat row, and a kudos + comment footer.
- **Preserve the four reactions** (👍 💪 🔥 🎉) and comment threads — restyled into the kudos/comment footer, not reduced to a single counter.
- **Left rail derived client-side** (streak / this-week / run-lift stats) from existing activity data — no new endpoint.
- **Discovery rail** backed by existing user search + follow; a **first-class no-followers state** that makes discovery the hero.
- Keep CI green in both repos (Go: golangci-lint/vet/test/tidy; web: lint/format/typecheck/test/build), updating tests for the new projection field and the new feed UI.

### Non-Goals

- **Building the following feed.** It already exists (`GET /timeline` aggregates own + accepted-followees' posts). No aggregation work.
- **A leaderboard.** No backend exists; deferred to a social-expansion SOW.
- **A suggested-athletes ranking algorithm.** The discovery rail uses existing search/follow; real suggestions are deferred.
- **Reducing the four reactions to a single "kudos."** The kudos framing is visual; the four reaction types stay.
- **Changing feed, follow, reaction, or comment semantics / data.** Aside from the additive run-route field, behavior is unchanged.
- **The variant's orange-on-near-black palette / Oswald-as-everything.** Conform to the app tokens; the condensed face is a *scoped display accent*, not the base.
- **Promoting the DX mockup code.** The variant (`app/design-explore/timeline/_variants/StravaSocialDashboard.tsx`, static fixtures, `RouteMap`/`Radar`/`WeekBars` atoms) is the visual spec, not code to copy. The `design-explore` route is untouched and never ships.

## Implementation Details

### API: run route in the feed projection (`prog-strength-api`)

In `internal/timeline`, the source hydrator builds each post's `content`. For `SourceRun` posts, add an optional **route** to the content DTO: a simplified polyline (array of lat/lng, Douglas–Peucker-reduced to a small point budget) plus a bounding box, sourced from the run's `trackpoints`. Absent when the run has none (treadmill/manual). Keep the point budget low — a feed page hydrates up to the page limit, so the route must be a thumbnail-grade path, not full fidelity (the detail page keeps full `trackpoints`). Update the content model, the run hydration, and both repositories' tests; the feed query, cursor, reactions, and comment batching are unchanged. Mirror the field in the web `TimelinePost.content` type.

### Web: the dashboard (`prog-strength-web`)

Rebuild `app/(app)/timeline/page.tsx` and `_components/*` into the three-column dashboard. Token conformance first: this surface uses the **slate/violet/Nunito** tokens from `sows/chat-and-app-shell-redesign.md` — reference those tokens, not new hex. Retain a **condensed athletic display face** (the Oswald-style character the owner picked) as a *scoped* display font for card titles and big stat numerals; that's typography, not a competing accent, so it coexists with the token system. The orange accent maps to violet; near-black surfaces map to the slate ramp.

- **Left "your week" rail** (`_components`) — profile (real `display_name` / avatar / initials), a streak figure, a this-week mini-chart (a `WeekBars`-style sparkline), and run/lift week stats. Derive these client-side from existing activity data (the calendar already computes weekly aggregates the same way); no new endpoint.
- **ActivityCard** (`TimelinePostCard` rework) — milestone banner for `pr`/`best_effort`; author row with a "You" badge when `author` is the viewer; title/subtitle/notes; **RouteMap** for runs from the new `content.route` (render as a lightweight static SVG polyline within the route bounds — cheap, no map tiles, good for a feed; a tile map is a later option); the existing **workout radar** (`WorkoutTimelineSummary`); the labeled big-value stat row (value + small uppercase label, from `content.metrics`); and a footer with the restyled **ReactionBar** (four reactions, kudos styling) + **CommentThread** count.
- **Right discovery rail** — "Find people to follow": a small set of results / an entry into the existing user search, each with the existing **Follow** action. No new suggestion backend.
- **No-followers empty state** — when the feed is just your own posts, render an intentional state that foregrounds discovery (the rail's find-people path), not a barren list. This is the state this account sees today, so it must be good.
- Preserve resume/links (`content.href` deep-links to `/running/{id}` etc.), reactions, and comments.

### Ordering & dependency

This SOW **conforms to the design tokens introduced by `sows/chat-and-app-shell-redesign.md`** (slate/violet/Nunito). That foundation should land first; if it hasn't, this SOW references the same token names so the two converge. Flagged so the timeline doesn't reintroduce a standalone palette.

### Tests

- **API**: unit-test the run route projection — present + simplified when trackpoints exist, absent when they don't, point budget respected; feed handler/repository tests updated for the new content field. golangci-lint / vet / tidy / test green.
- **Web**: `app/(app)/timeline` + `_components` tests — the three-column layout renders, ActivityCards render route maps for runs and the radar for workouts, the milestone banner shows for pr/best_effort, the four reactions + comments still work, the left rail computes from a known fixture, and the no-followers state foregrounds discovery. lint/format/typecheck/test/build green.

## Rollout

1. **`prog-strength-api`** — the run route projection. Additive and backward-compatible (a new optional `content.route` field); deploy first so the field is available before the web cards expect it.
2. **`prog-strength-web`** — the dashboard rebuild (conforming to the app tokens). Vercel preview to verify the feed with real route maps, the rails, and the no-followers state.
3. **`prog-strength-docs`** — flip `dx/timeline.md` to `status: selected` (idiom `strava-social-dashboard`); mark this SOW shipped on merge.

### Verification after rollout

- `/timeline`: three-column dashboard on the app's slate/violet theme — left "your week" rail (streak, this-week chart, stats), center feed of athletic ActivityCards with **real run route maps** and workout radars, right discovery rail.
- Follow a test account that has runs and confirm their posts appear in the feed (proving the already-existing aggregation) with route maps from the new projection.
- The four reactions and comments work; milestone PRs/best-efforts get the banner.
- With no one followed, the feed shows your own posts and foregrounds discovery — intentional, not empty.

## Open Questions

- **Condensed display face** — retaining an Oswald-style athletic display for titles/numerals as a scoped accent is the proposed reconciliation. If it clashes with Nunito once both are on screen, fall back to Nunito weights for the athletic emphasis. Visual call at implementation.
- **Route render: static SVG vs. tile map** — a static SVG polyline is proposed for the feed (cheap, no tiles, no extra deps). If a real basemap is wanted later (the Garmin/Strava look), that's an additive enhancement, not this SOW.
- **Route point budget** — the exact simplified point count / bounds encoding for the feed thumbnail; pick the smallest that still reads as the route's shape. Settle in implementation.
- **Left-rail data source** — client-side derivation is proposed to avoid backend work; if it turns out expensive to assemble client-side, a small `/me/this-week` summary endpoint is the fallback (additive, a later option).
- **Leaderboard & real suggestions** — deferred here; the natural next social SOW once this lands.
