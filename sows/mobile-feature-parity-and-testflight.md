---
status: draft
repos:
  - prog-strength-mobile
  - prog-strength-web
  - prog-strength-api
  - prog-strength-docs
---

# Mobile Feature Parity and TestFlight Distribution

**Status**: Draft · **Last updated**: 2026-06-12

## Introduction

The web app has had a heavy feature sprint (running tracking, progress modernization, personal-records redesign, settings/profile, photo meal logging, usage caps) while the mobile app stayed at its v1 scope of chat, workouts, calendar, nutrition, and a basic progress view. The mobile codebase itself is healthy — Expo SDK 55, React Native 0.83, React 19, strict TypeScript, clean `lib/api.ts` mirrored from web — so this is a feature-gap problem, not a modernization problem.

The owner is the app's only user and wants to live on the mobile app daily to dogfood it. That requires two things this SOW delivers together:

1. **TestFlight distribution actually turned on.** The plumbing from the initial mobile SOW exists (EAS project, bundle ID `fitness.progstrength.app`, `eas.json` profiles, `.github/workflows/release.yml` with OTA-on-push and manual native builds), but it has never been activated: no Apple credentials wired into EAS, no app icon assets, no `EXPO_TOKEN` repo secret, no first build. The Apple Developer Program membership is active, so nothing external blocks this.
2. **Feature parity with the web app**, with UI rebuilt for phone-sized screens rather than transplanted from desktop layouts.

Once shipped, every merge to `main` reaches the owner's iPhone via OTA update in ~30 seconds, and the phone can replace the laptop for day-to-day logging, review, and chat.

## Proposed Solution

Activate TestFlight first (Phase 0), then close the parity gap in four independently shippable phases ordered so the most foundational and most daily-used surfaces land first. Each phase merges as its own PR wave and is delivered to the device over the air, so the owner is testing parity work on a real phone from the first week.

The parity target is the current `prog-strength-web` feature set. The web repo is the reference implementation: API types and fetchers continue the established discipline of mirroring `lib/api.ts` between web and mobile, and each web page's data flow ports over — but layouts are redesigned for mobile idioms (segmented controls, bottom sheets, action sheets, stacked cards) rather than copied.

No backend changes are required. Every endpoint the mobile app needs already exists and is consumed by the web app; `prog-strength-api` is in the repo list for reference only (endpoint shapes), and `prog-strength-web` for reference only (parity source). The one API-facing change on mobile is migrating its progression call off the stale `muscle_group` query parameter onto the `movement_pattern` parameter the web app now uses.

**Phase sizing for autonomous dispatch:** this SOW is deliberately combined, but each phase is scoped to fit a single `dispatch-sow.yml` worker run (6-hour cap). When dispatching, point the worker at one phase at a time (state the phase in the dispatch context or implement phases as sequential dispatches); do not attempt Phases 1–4 in one run.

## Goals and Non-Goals

### Goals

- A TestFlight build of the app installed and running on the owner's iPhone, with OTA updates flowing on every merge to `main`.
- A documented, repeatable release runbook (one-time setup steps vs. recurring steps) in `prog-strength-mobile/README.md`.
- Mobile feature parity with the current web app: settings/profile/units, usage cap awareness, activities hub with running (list, detail charts, TCX import, best efforts), modernized progress analytics, personal-records running view + progression charts, calendar day digest + weekly stats + runs, photo meal logging in chat, nutrition custom meals + direct-log, bodyweight goal + entry editing, exercises catalog.
- Every ported surface usable one-handed on an iPhone 15/16/17-class screen — no horizontally scrolled charts, no desktop modals, no sub-44pt touch targets.

### Non-Goals

- Android builds, signing, or testing (workflow stays `--platform ios`).
- Native GPS run recording (runs arrive via TCX import; native recording is its own future SOW).
- Push notifications, haptics, widgets, or other native-only features beyond parity.
- Offline support or background sync.
- Refresh-token flow (401 still re-prompts OAuth, as today).
- Public App Store submission, App Review prep, or Sign in with Apple (required only for public release).
- Automated E2E test infrastructure (Detox/Appium) — manual on-device smoke testing per phase is the bar for now.

## Implementation Details

### Phase 0 — TestFlight Activation

Mostly one-time interactive ops the owner performs, plus small CI verification. The deliverable is a working pipeline and a runbook, not code.

**One-time setup (owner, interactive — cannot be done by the autonomous worker):**

1. `npx eas-cli login` as the `jwallace145` Expo account.
2. `npx eas-cli credentials` → let EAS manage iOS distribution certificate and provisioning profile against the Apple Developer account; create the App Store Connect app record for `fitness.progstrength.app` if EAS does not auto-create it.
3. Create an Expo access token and add it as the `EXPO_TOKEN` secret on `prog-strength-mobile` (the existing `release.yml` already reads it).
4. First build: `eas build --platform ios --profile preview`, install via the EAS link, smoke test. Then `eas build --platform ios --profile production` + `eas submit` to reach TestFlight Internal Testing (no App Review for internal testers).

**Repo work (worker or owner):**

- App icon (1024×1024 PNG) and splash assets in `assets/`, referenced from `app.json`. Use existing branding assets from `prog-strength/branding/` as the source.
- Verify `release.yml` end-to-end: push to `main` triggers `eas update --branch production`; manual `workflow_dispatch` with `build=true` produces a native build and (for `production`) submits to TestFlight.
- Write the release runbook into `prog-strength-mobile/README.md`: one-time steps above, plus the recurring model — JS-only changes ship via OTA automatically; changes that add native modules require a native rebuild + TestFlight submission (see Rollout below).

**Acceptance:** app installed from TestFlight on the owner's phone; a trivial JS change merged to `main` appears on the device on next app launch without a new build.

### Navigation and Information Architecture

The five bottom tabs stay (iOS comfortable maximum): **Chat · Activities · Calendar · Nutrition · Progress**.

- **Workouts tab → Activities tab**, with an `Overview | Workouts | Running` segmented control mirroring the web's `/activities` views and the existing mobile segmented-control pattern (Nutrition and Progress already use one).
- **Settings** gets no tab. The owner's avatar (or initials placeholder) renders in the screen header and pushes a Settings stack screen — standard iOS pattern, and it mirrors the web sidebar's account anchor.
- **Exercises catalog** is a stack screen, reachable from a Settings row ("Exercise catalog") and by tapping an exercise row in workout detail.
- All existing deep routes (`/workouts/[id]`, chat history) keep working; add `/running/[id]` for run detail.

### Phase 1 — Settings, Units, and Usage Foundations

Ordered first because units preferences affect every later display.

- **Settings screen** (parity with web `/settings`): display name (`PATCH /me`), height with in/cm display toggle, avatar upload/remove (`POST /me/avatar` multipart via `expo-image-picker`, `DELETE /me/avatar`), distance unit (mi/km) and weight unit (lb/kg) radios.
- **Profile context**: port the web `ProfileProvider` pattern — fetch `GET /me` once post-auth, share across header avatar, settings, and chat.
- **Units contexts**: port `DistanceUnitProvider` semantics (server-synced via `PATCH /me`, applied to pace/distance/height rendering) and weight-unit handling for workout and bodyweight displays. All later phases consume these.
- **Usage**: `GET /me/usage?tz=…` powering a usage bar in Settings and a capped-aware chat composer (input disabled with reset countdown when capped), matching web behavior.

Note: `expo-image-picker` enters here (avatar) — it is a native module, so Phase 1 ends with a native rebuild. Phase 4's photo meal logging reuses it with no further rebuild.

### Phase 2 — Activities and Running

The largest phase; the web `/activities`, `/running/[id]`, and running-metrics surfaces are the reference.

- **Activities Overview**: combined recent workouts + runs, running metrics summary (`GET /activities/running-metrics`: week distance/count with delta, month, recent avg pace, all-time). Timeframe filter pills (7d/30d/90d/All) shared across the three segments.
- **Workouts segment**: the existing workouts list moves here unchanged.
- **Running segment**: run list (`GET /activities` with cursor pagination) — each row date, name, distance, pace, duration, avg HR, rendered in the owner's distance unit.
- **Run detail** (`/running/[id]`, `GET /activities/{id}`): stats header + trackpoint charts (pace, heart rate, elevation over distance/time) using the chart stack already in the app, sized for ~360pt width. Rename via `PATCH /activities/{id}` (action sheet), delete via `DELETE /activities/{id}` with confirm.
- **TCX import**: `expo-document-picker` → multipart `POST /activities/tcx` (10 MB cap). Map `409` to the web's duplicate-run handling ("already in your log", link to existing activity). `expo-document-picker` is a native module → native rebuild at the start of this phase.
- **Calendar runs**: run pills on the month grid alongside workout pills (distinct color), runs included in the selected-day digest (full digest lands in Phase 3; this phase just adds runs to what exists).

### Phase 3 — Analytics Parity (Progress, PRs, Calendar)

- **Progress segment**: migrate `GET /workouts/progression` from `muscle_group` to `movement_pattern` (push/pull/legs/core/all filter pills), render per-exercise trends (slope %/month with trendline endpoints, "not enough data" below session threshold), normalized chart (each exercise vs. its own recency-weighted baseline), and aggregate stat tiles (lifts tracked / progressing, median slope). Timeframe 30/60/90d backed by URL params where Expo Router allows.
- **Personal Records**: add the `Lifts | Running` toggle inside the PRs segment. Running view: standard-distance best efforts (`GET /running/best-efforts`) as cards. Both views gain expandable per-card progression charts (`GET /personal-records/{exercise_id}/history`, `GET /running/best-efforts/{distance_key}/history`). Customize-headline-lifts flow already exists; keep it.
- **Calendar modernization**: selected-day digest panel (workouts + runs with full detail, expandable tiles) and weekly stat chips (volume, run count) aligned to grid weeks, per web PRs #25–26. Month grid cells stay ≥44pt touch targets; digest scrolls, grid does not.

### Phase 4 — Chat and Nutrition Polish

- **Photo meal logging**: chat composer gains an attachment button → iOS action sheet with Camera / Photo Library (`expo-image-picker`, already in the binary from Phase 1). Downscale/compress client-side to ≤5 MB JPG/PNG/WebP, send as multimodal content blocks (`{type: "image", source: {type: "base64", …}}`) matching the web composer's shape; persisted turns store the `[image attached]` placeholder string as web does.
- **Nutrition gaps**: custom one-off meals (`POST /nutrition-log/custom` with name + macros, quick-add sheet), direct "Log" action on pantry items and recipes (web PR #31), edit flows for log entries where missing.
- **Bodyweight**: goal (`GET/PUT /me/bodyweight-goal`) with reference line on the trend chart and goal status tile; entry editing (`PUT /bodyweight/{id}`) via the existing action-sheet row pattern.
- **Exercises catalog screen**: searchable list (`GET /exercises`, public), A–Z grouping, expandable rows with description + muscle-group and equipment pills. Entry points per the IA section.

### Mobile UI/UX Standards (cross-cutting, applies to every phase)

These are requirements, not suggestions — screens that violate them are not done:

- **Touch targets ≥44pt**; row actions via iOS action sheets (the established bodyweight pattern), never hover-dependent affordances.
- **Bottom sheets over desktop modals** for forms and pickers (quick-add, goal editing, customize lifts); full-screen modals only for multi-field forms (recipe editor).
- **Safe-area insets** respected on every screen, including behind the tab bar and inside bottom sheets.
- **Charts fit the viewport**: designed for ~360pt width, no horizontal scrolling; prefer fewer axis labels and tap-to-inspect over dense desktop legends.
- **Segmented controls max 3 options**; beyond that, restructure the IA rather than cramming.
- **One-handed reach**: primary actions (quick-add, attach, log) anchored bottom or in thumb range, not top-right.
- **Dark mode stays forced** (`userInterfaceStyle: "dark"`), matching web. No light-mode work in this SOW.
- **Loading and empty states** for every new list/chart (skeleton or spinner + a sentence), since cellular latency is the norm on a phone.

### Rollout

- **Merge order**: Phase 0 → 1 → 2 → 3 → 4. Later phases depend on Phase 1's units/profile contexts; Phases 2–4 are otherwise independent of each other.
- **Build cadence**: native rebuild + TestFlight submission required only when the native module set changes — end of Phase 1 (`expo-image-picker`) and start of Phase 2 (`expo-document-picker`). Everything else ships OTA via the existing `eas update` on merge to `main`. If convenient, add both modules in a single rebuild at the Phase 1/2 boundary.
- **Per-phase verification**: each phase ends with a manual smoke pass on the owner's physical iPhone (not just simulator) before the next phase is dispatched. Voice features (speech recognition, TTS) must be re-verified after every native rebuild, since they depend on native modules pinned to SDK 55.
- **Revert story**: OTA updates can be rolled back by re-publishing the prior update on the `production` branch via `eas update:republish`; native regressions roll back by expiring the bad TestFlight build and re-installing the prior one.

## Open Questions

1. **Workout editing on mobile.** Web is also read-only for workouts (created via chat), so parity does not require edit/delete — but living on the phone may surface the need quickly. Options: keep read-only (parity), or add edit/delete to workout detail. **Lean: keep read-only**; file separately if dogfooding demands it.
2. **TanStack Query adoption on mobile.** Web uses it selectively (PRs, Progress). Mobile currently uses fetch + `useState` everywhere. Options: keep the minimal pattern, or adopt TanStack Query for the new analytics views to match web's caching behavior. **Lean: adopt only for Progress/PR history charts** where view-switching makes caching visibly valuable; keep the minimal pattern elsewhere.
3. **Exercises catalog entry point.** Settings row + workout-detail row links are proposed. Option: also a search affordance on the Activities screen. **Lean: ship the two proposed entry points** and see whether it gets used at all before adding more.
4. **App icon source.** `prog-strength/branding/` assets exist but may not include a 1024×1024 iOS-ready icon. Options: derive from existing branding, or commission/generate a dedicated icon. **Lean: derive from existing branding** for TestFlight; a proper icon pass can ride the public-App-Store SOW.
