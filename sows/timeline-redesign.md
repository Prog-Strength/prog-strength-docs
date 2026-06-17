---
type: sow
status: shipped
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Timeline Redesign — Strava-Social-Dashboard

**Status**: Shipped · **Last updated**: 2026-06-17

## Scope decision (2026-06-16): option 2 — ship the dashboard, defer the route map

Implementation was halted at the API piece (see PR #80) because this SOW's one
backend deliverable — a compact run **route polyline** "derived from the run's
existing `trackpoints`" — has **no data source**. No geographic position
(latitude/longitude) is stored or parsed anywhere in the stack:

- **TCX parser** `internal/activity/tcx_parser.go` decodes only time / distance /
  HR / altitude — the TCX `<Position>` element is never read.
- **DB** `internal/db/migrations/015_activities_generalize.sql`
  (`activity_trackpoints`) has no `latitude` / `longitude` columns.
- **Domain model** `internal/activity/model.go` `Trackpoint` carries no coordinates.

A 2-D path cannot be reconstructed from cumulative distance alone, so the route
summary cannot be derived from existing data; the raw TCX in S3 may hold
`<Position>` for GPS runs, but it is discarded at import and re-reading S3
per-post is the N+1 the SOW warns against.

**Decision: option 2.** Ship the dashboard now with **no backend change** and a
graceful "no route" placeholder where the run map will eventually go. The
real GPS-capture work (TCX-parser `<Position>` support, a coordinates migration,
and an S3 backfill) is **split into its own backend SOW** —
[`sows/run-route-geometry-capture.md`](run-route-geometry-capture.md) — which is
the true prerequisite for the route map and is materially larger than the "one
small API change" this SOW authorized. When that lands, a follow-up flips the
placeholder to a real `RouteMap`. This SOW is accordingly **web + docs only**
(`prog-strength-api` removed from `repos:`); everything else the chosen design
needs is buildable today.

## Introduction

The Timeline (`/timeline`) is Prog Strength's social surface — a multi-author feed of training as it happens, with reactions and comments. The Design Exploration `dx/timeline.md` (PR Prog-Strength/prog-strength-web#59) explored it as a proper social dashboard, and the owner selected **`strava-social-dashboard`**: a three-column layout with a left "your week" rail (profile · streak · a this-week mini-chart), a center following feed of confident activity cards (a prominent route map slot, a labeled big-value stat row, kudos and comments), and a right discovery rail.

One important correction to the scope as it was first framed: **the social following feed already exists.** `GET /timeline` is already the aggregated multi-author home feed — it returns the viewer's own posts *plus* their accepted-followees' non-private posts, hydrated with reactions and comment counts. The timeline only *looks* solo today because this account follows no one yet (no followees → the feed is just your own posts). So no feed-aggregation backend needs building. The one thing the chosen design wants that the backend can't supply is **run route geometry** — and per the scope decision above, there is no coordinate data to draw it from yet, so the route map ships as a graceful placeholder and the geometry work is deferred to its own SOW. **There is no backend change in this SOW.**

So this is a **web-only redesign** (plus the docs bookkeeping). The redesign **conforms to the app's new design tokens** — the slate/violet/Nunito foundation established by the chat & app-shell SOW (`sows/chat-and-app-shell-redesign.md`) — rather than the variant's standalone orange-on-near-black palette, so the timeline matches the rest of the app. The athletic *character* the owner picked (bold condensed display titles, big stat numerals, the route-forward card) is kept; the orange accent yields to the app's violet. The result: a social feed that feels alive and athletic and sits coherently in the redesigned app, with a route-map slot ready to light up the moment geometry exists.

## Proposed Solution

One shippable change: the web dashboard rebuild. No backend work.

**Web (`prog-strength-web`) — the dashboard rebuild.** Redesign `/timeline` to the three-column strava-social-dashboard structure, conforming to the app-wide slate/violet/Nunito tokens:

- **Left "your week" rail** — profile (real name/avatar), a **streak** figure, a **this-week mini-chart**, and run/lift week stats — *derived client-side* from existing activity data (the same kind of weekly aggregation the calendar already does), so no new endpoint is needed.
- **Center following feed** — confident **ActivityCards** for the existing post types (run / workout / pr / best_effort): a milestone banner for PRs and best efforts, the author row (with a "You" badge on your own posts), title/subtitle/notes, a **route-map slot** for runs that renders a graceful "no route yet" placeholder today (and a real map once geometry lands), the existing **muscle-group radar** for workouts, the labeled **big-value stat row**, and a kudos + comment footer. Reactions and comments already exist — they're restyled, not rebuilt.
- **Right discovery rail** — a lightweight "find people to follow" backed by the **existing user search + follow** APIs (not a new suggestion algorithm). A full suggested-athletes ranking and a leaderboard are explicitly deferred.
- **No-followers state, first-class** — since this account follows no one, the feed must feel intentional empty: surface your own training and make discovery the hero, not a dead end.

## Goals and Non-Goals

### Goals

- **Rebuild `/timeline`** to the three-column strava-social-dashboard: left "your week" rail · center following feed · right discovery rail.
- **Conform to the app-wide tokens** (slate/violet/Nunito from `sows/chat-and-app-shell-redesign.md`): keep the athletic *character* (condensed display titles, big stat numerals, route-forward cards) but draw accent and surfaces from the shared tokens — the variant's **orange accent yields to violet**.
- **ActivityCards** for all post types: milestone banner (pr/best_effort), author row + "You" badge, title/subtitle/notes, a **run route-map slot with a graceful placeholder**, **workout radar** (the existing breakdown), the labeled big-stat row, and a kudos + comment footer.
- **A clean, well-isolated `RouteMap` component boundary** — it renders the placeholder today and accepts a route prop, so flipping it to a real map when geometry lands is a localized change, not a card rewrite.
- **Preserve the four reactions** (👍 💪 🔥 🎉) and comment threads — restyled into the kudos/comment footer, not reduced to a single counter.
- **Left rail derived client-side** (streak / this-week / run-lift stats) from existing activity data — no new endpoint.
- **Discovery rail** backed by existing user search + follow; a **first-class no-followers state** that makes discovery the hero.
- Keep web CI green (lint/format/typecheck/test/build), updating tests for the new feed UI.

### Non-Goals

- **Run route geometry / map rendering.** No coordinate data exists; the route map is a placeholder here and the geometry work (parser + migration + backfill) is deferred to [`sows/run-route-geometry-capture.md`](run-route-geometry-capture.md). **No `prog-strength-api` change in this SOW.**
- **Building the following feed.** It already exists (`GET /timeline` aggregates own + accepted-followees' posts). No aggregation work.
- **A leaderboard.** No backend exists; deferred to a social-expansion SOW.
- **A suggested-athletes ranking algorithm.** The discovery rail uses existing search/follow; real suggestions are deferred.
- **Reducing the four reactions to a single "kudos."** The kudos framing is visual; the four reaction types stay.
- **Changing feed, follow, reaction, or comment semantics / data.** Behavior is unchanged.
- **The variant's orange-on-near-black palette / Oswald-as-everything.** Conform to the app tokens; the condensed face is a *scoped display accent*, not the base.
- **Promoting the DX mockup code.** The variant (`app/design-explore/timeline/_variants/StravaSocialDashboard.tsx`, static fixtures, `RouteMap`/`Radar`/`WeekBars` atoms) is the visual spec, not code to copy. The `design-explore` route is untouched and never ships.

## Implementation Details

### Web: the dashboard (`prog-strength-web`)

Rebuild `app/(app)/timeline/page.tsx` and `_components/*` into the three-column dashboard. Token conformance first: this surface uses the **slate/violet/Nunito** tokens from `sows/chat-and-app-shell-redesign.md` — reference those tokens, not new hex. Retain a **condensed athletic display face** (the Oswald-style character the owner picked) as a *scoped* display font for card titles and big stat numerals; that's typography, not a competing accent, so it coexists with the token system. The orange accent maps to violet; near-black surfaces map to the slate ramp.

- **Left "your week" rail** (`_components`) — profile (real `display_name` / avatar / initials), a streak figure, a this-week mini-chart (a `WeekBars`-style sparkline), and run/lift week stats. Derive these client-side from existing activity data (the calendar already computes weekly aggregates the same way); no new endpoint.
- **ActivityCard** (`TimelinePostCard` rework) — milestone banner for `pr`/`best_effort`; author row with a "You" badge when `author` is the viewer; title/subtitle/notes; a **RouteMap** slot for runs; the existing **workout radar** (`WorkoutTimelineSummary`); the labeled big-value stat row (value + small uppercase label, from `content.metrics`); and a footer with the restyled **ReactionBar** (four reactions, kudos styling) + **CommentThread** count.
- **RouteMap component** — render a graceful, on-brand **"no route" placeholder** (e.g. a muted, slate-toned map-frame motif sized to the eventual map's aspect ratio, so the card layout is already correct). It must accept an optional route prop typed for the *future* `content.route` shape (a simplified lat/lng polyline + bounds) and draw a lightweight static SVG polyline when that prop is present — so when `sows/run-route-geometry-capture.md` lands, lighting up real maps is a prop wiring change, not a card redesign. No map tiles, no new deps.
- **Right discovery rail** — "Find people to follow": a small set of results / an entry into the existing user search, each with the existing **Follow** action. No new suggestion backend.
- **No-followers empty state** — when the feed is just your own posts, render an intentional state that foregrounds discovery (the rail's find-people path), not a barren list. This is the state this account sees today, so it must be good.
- Preserve resume/links (`content.href` deep-links to `/running/{id}` etc.), reactions, and comments.

### Ordering & dependency

This SOW **conforms to the design tokens introduced by `sows/chat-and-app-shell-redesign.md`** (slate/violet/Nunito). That foundation should land first; if it hasn't, this SOW references the same token names so the two converge. Flagged so the timeline doesn't reintroduce a standalone palette. The route map's data prerequisite is tracked separately in [`sows/run-route-geometry-capture.md`](run-route-geometry-capture.md) and is **not** a blocker for this SOW.

### Tests

- **Web**: `app/(app)/timeline` + `_components` tests — the three-column layout renders, ActivityCards render the radar for workouts and the **RouteMap placeholder** for runs (and the SVG polyline when a route prop is supplied, from a fixture), the milestone banner shows for pr/best_effort, the four reactions + comments still work, the left rail computes from a known fixture, and the no-followers state foregrounds discovery. lint/format/typecheck/test/build green.

## Rollout

1. **`prog-strength-web`** — the dashboard rebuild (conforming to the app tokens). Vercel preview to verify the feed with the rails, the route-map placeholder, and the no-followers state.
2. **`prog-strength-docs`** — flip `dx/timeline.md` to `status: selected` (idiom `strava-social-dashboard`); mark this SOW shipped on merge.

### Verification after rollout

- `/timeline`: three-column dashboard on the app's slate/violet theme — left "your week" rail (streak, this-week chart, stats), center feed of athletic ActivityCards with the **route-map placeholder** for runs and workout radars, right discovery rail.
- Follow a test account that has runs and confirm their posts appear in the feed (proving the already-existing aggregation); run cards show the placeholder, not a broken slot.
- The four reactions and comments work; milestone PRs/best-efforts get the banner.
- With no one followed, the feed shows your own posts and foregrounds discovery — intentional, not empty.

## Open Questions

- **Condensed display face** — retaining an Oswald-style athletic display for titles/numerals as a scoped accent is the proposed reconciliation. If it clashes with Nunito once both are on screen, fall back to Nunito weights for the athletic emphasis. Visual call at implementation.
- **Route-map placeholder treatment** — what the "no route yet" slot looks like so it reads as intentional, not missing (a muted map-frame motif is proposed). Settle visually at implementation; keep the slot at the eventual map's aspect ratio.
- **Left-rail data source** — client-side derivation is proposed to avoid backend work; if it turns out expensive to assemble client-side, a small `/me/this-week` summary endpoint is the fallback (additive, a later option).
- **Leaderboard & real suggestions** — deferred here; the natural next social SOW once this lands.
- **Route geometry** — tracked in [`sows/run-route-geometry-capture.md`](run-route-geometry-capture.md); when it lands, a small follow-up wires real routes into the `RouteMap` prop this SOW leaves in place.
