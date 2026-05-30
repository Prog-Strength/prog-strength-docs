# Prog Strength Initial Mobile App Implementation

**Status**: Shipped · **Last updated**: 2026-05-30

## Introduction

The primary way a user interacts with **Prog Strength** is from their phone — most likely while they're working out or eating a meal. The product is web-only today, and the experience of using a desktop-shaped web app in a mobile Safari tab is not great: tab chrome, no native gestures, no home-screen icon, awkward keyboard handling for chat. A native iOS app removes all of that. This SOW kicks off building the mobile client in `prog-strength-mobile` so the product can be used the way most users will actually use it.

The mobile app should ship with near-parity to the web app at launch. The process is iterative — the mobile client doesn't need to match the web client pixel-for-pixel, and client-side rendering will diverge in places where iOS conventions differ from desktop web. The API calls and the backend logic underneath are shared verbatim between the two clients; nothing new ships server-side as part of this SOW. The features the initial implementation needs are:

1. Chat with the **Prog Strength** AI agent.
2. Workouts and calendar views for browsing logged sessions.
3. Nutrition tracking with pantry items and recipes.
4. Progress tracking to answer the core "am I actually getting stronger?" question.

A small scaffold already exists in `prog-strength-mobile` from earlier groundwork (Expo + React Native, with login, a streaming chat screen, and the workouts list and detail). There is substantial work left to bring the rest of the auth-gated routes to working state on a phone and to wire up TestFlight + over-the-air updates so iteration after this lands is fast. iOS-only at v1 — Android is not in scope.

## Proposed Solution

`prog-strength-mobile` is an **Expo + React Native** app written in **TypeScript**, using **Expo Router** for file-based routing (the same mental model as the Next.js App Router that powers `prog-strength-web`) and **NativeWind** for styling (Tailwind utility classes via `className`, again mirroring web). Auth is handled by **expo-auth-session** for Google OAuth and **expo-secure-store** (Keychain-backed) for the resulting JWT. Streaming chat consumes the agent's SSE response via `expo/fetch`'s spec-compliant `ReadableStream`. The bottom of the app is a tab navigator; each tab is its own stack so list/detail flows push naturally over their parent tab.

Every screen calls the same `prog-strength-api` and `prog-strength-agent` backends the web client already hits. Type definitions and fetcher functions in `lib/api.ts` mirror the web client's `lib/api.ts` one-for-one — when the API grows an endpoint, web and mobile each add identical lines to keep types in sync. Authentication and JWT propagation work identically: the user signs in once, the token sits in the Keychain, every request carries `Authorization: Bearer <token>`.

The remaining work is two-track:

1. **Build the missing screens.** `/login`, `/workouts`, `/workouts/[id]`, and `/chat` already exist. `/calendar`, `/progress`, `/personal-records`, `/nutrition`, `/pantry`, and `/bodyweight` do not. Each screen ports the web client's data shape and behavior, with mobile-native layout where it diverges meaningfully from desktop (agenda-style date views, bottom sheets in place of centered modals, native pickers).
2. **Set up the release pipeline.** EAS Build for TestFlight Internal distribution (no App Review for internal testers, ~30 minute build cadence), EAS Update for OTA JS bundle updates on every push to main (changes reach the phone within ~30 seconds without a new TestFlight build). The GitHub Action stub for this is already in place; activation is gated on Apple Developer Program enrollment going active.

## Goals and Non-Goals

### Goals

- Ship six new working screens against the existing scaffold: `/calendar`, `/progress`, `/personal-records`, `/nutrition`, `/pantry`, `/bodyweight`. Each at parity with its web counterpart's read paths, plus any write affordances the web client has (Customize modal, recipe builder, log-a-meal form, log-a-weight form, etc.).
- Mirror `lib/api.ts` types + fetchers exactly between web and mobile so a new API endpoint costs identical work on both clients.
- Get a `preview`-profile EAS Build into TestFlight Internal Testing so the app installs as a real home-screen icon on the developer's iPhone.
- Wire EAS Update against the `production` channel so every push to `main` ships a JS bundle update to the phone on next launch — no TestFlight rebuild for JS-only changes.
- Auth: keep the Google OAuth flow working (the dev-sign-in panel from local testing stays gated on `EXPO_PUBLIC_API_URL` being loopback so prod builds never see it).
- All screens hit the existing API + agent — no new backend endpoints.

### Non-Goals

- **Android.** Out of scope. The Expo codebase is platform-agnostic and would mostly work on Android, but no testing, no Play Store presence, no Android-specific UX polish.
- **Backend changes.** No new API endpoints, no schema changes, no new MCP tools. Mobile consumes what web already consumes.
- **Public App Store listing.** TestFlight Internal Testing is the distribution surface for v1. Public App Store submission has its own review cycle, privacy disclosures, and screenshot requirements; separate SOW when it matters.
- **Pixel-perfect visual parity with web.** Mobile layouts will diverge where iOS conventions ask for it (modals → bottom sheets, dropdowns → native pickers, day grids → agenda views on small screens). The data and behaviors are identical; the chrome around them is not.
- **Offline reads / local caching.** The web client doesn't cache either; keeping mobile honest to "fetch on screen mount" avoids drift between the two clients' staleness semantics. Revisit if users start hitting "I'm at the gym with bad signal" friction.
- **Push notifications.** Different feature, different SOW. Would warrant its own design pass for a side-project diet/lifting tracker (which notifications are valuable vs. which are nag-y).
- **iOS-specific flourishes** — haptics, widgets, watchOS companion, Live Activities. Stretch goals if the app is fun enough to use that they become worth adding.
- **Reusing web's React component code directly.** Mobile screens are rewritten against React Native primitives (`View`, `Text`, `Pressable`, `FlatList`). The shared layer is types + fetchers in `lib/api.ts`, not JSX.

## Implementation Details

### Current State

What's already in `prog-strength-mobile`:

- **Stack**: Expo SDK 55, React Native 0.85.3, Expo Router 5.x (file-based, typed routes enabled), NativeWind v4 with Tailwind v3.4, TypeScript strict mode with `@/*` path alias.
- **Auth**: `expo-auth-session` drives Google OAuth against the existing backend's `/auth/google/login?return_to=...` endpoint, with `progstrength://auth/callback` as the redirect URI. The dev-sign-in panel mints a token via `POST /auth/dev/token` when `EXPO_PUBLIC_API_URL` points at loopback / LAN — invisible in production builds. JWT lives in the iOS Keychain via `expo-secure-store`.
- **Shared lib**: `lib/config.ts` (env-driven base URLs), `lib/auth.ts` (Keychain JWT), `lib/api.ts` (slim port of web's: `Workout`, `Exercise`, `WorkoutSet`, `PersonalRecordEvent` types + `listWorkouts`, `listExercises`, `getWorkout` fetchers), `lib/stream.ts` (SSE parser over `expo/fetch`'s `ReadableStream` for chat).
- **Screens**: `app/_layout.tsx` (root with SafeAreaProvider + GestureHandlerRootView), `app/index.tsx` (token-check redirect), `app/login.tsx`, `app/(tabs)/_layout.tsx` (two tabs: Workouts, Chat), `app/(tabs)/workouts/{_layout,index,[id]}.tsx`, `app/(tabs)/chat.tsx`.
- **Distribution config**: `eas.json` with `development` / `preview` / `production` profiles, GitHub Actions workflow `.github/workflows/release.yml` for OTA updates on push and dispatched native builds.

### Routes and Screens to Add

Each new screen mirrors the equivalent web route's behavior. Mobile-specific UX deviations called out per screen.

| Route | Web parity target | Mobile-specific notes |
| --- | --- | --- |
| `/(tabs)/calendar` | Month grid of workouts | Hybrid month-grid-plus-agenda layout instead of a wide grid — see the Calendar Layout section below. |
| `/(tabs)/progress` | Muscle-group filter, timeframe pills, normalized 1RM chart, toggleable 1RM-vs-Sets×Reps×Weight table | Chart uses **Victory Native XL** instead of Recharts (RN-incompatible). Toggle stays as a segmented control. |
| `/(tabs)/personal-records` | Trophy-style card grid, Customize modal | Customize opens as an iOS bottom sheet, not a centered modal — list-style picker reads better on a phone than a checkbox grid. |
| `/(tabs)/nutrition` | Date selector, daily macro tiles, four meal sections with subtotals, quick-add picker | Meal sections each get an inline "+ Add" button that opens the picker pre-set to that meal — denser than the desktop pattern of a single top-of-page form. |
| `/(tabs)/pantry` | Pantry/Recipes tabs, item editor, recipe builder | Tab bar stays as a segmented control. Recipe component picker becomes a fullscreen modal flow rather than the inline dropdown the desktop builder uses. |
| `/(tabs)/bodyweight` | Trend chart with raw + 7-day average, entry form, recent entries | Chart via Victory Native XL again. Entry form is a single screen the user pulls up via a "+" floating button. |

Tab bar lands as **Chat · Workouts · Calendar · Nutrition · Progress** — five tabs is the practical maximum for iOS bottom navigation. The web app's nine sidebar entries don't all fit, so the routes that aren't on the tab bar live one level down as top-of-screen segmented controls inside their parent tab:

- **Progress tab** carries a `( Progress | PRs )` segmented control at the top of the screen. The active segment swaps the body between the muscle-group progression view and the trophy-case PR cards.
- **Nutrition tab** carries a `( Today | Pantry | Bodyweight )` segmented control. Today is the per-meal log; Pantry holds items + recipes; Bodyweight holds the trend chart and entry form. Bodyweight pairs with Nutrition per the daily-nutrition-log SOW.

The segmented-control approach (rather than stack pushes from a "Personal Records" link) keeps related views one tap from each other rather than two — the user can flip between Progress and PRs without going back to a parent screen. Each tab is otherwise its own Expo Router stack so list-to-detail navigation pushes naturally inside whichever segment is active.

### Shared Types + API Client

`lib/api.ts` on the mobile side maintains an identical export shape to `prog-strength-web/lib/api.ts`:

- Same TypeScript types (`Workout`, `Exercise`, `PantryItem`, `Recipe`, `NutritionLogEntry`, `BodyweightEntry`, `PersonalRecord`, `MuscleGroupProgression`, etc.) — copied verbatim, including JSDoc comments where they explain non-obvious shape decisions.
- Same fetcher function signatures (`listWorkouts(token, options)`, `createPantryItem(token, payload)`, …).
- Same envelope-unwrap helper.

The duplication is deliberate at this scale: there are two consumers, the contracts change slowly, and a single shared `npm` package or git submodule introduces tooling friction (workspace setup, version bumping, RN-vs-Next.js TS config divergence) that doesn't pay for itself for a single-developer beta. We accept the duplication, write tests that exercise both clients against the same API, and revisit a shared package if a third consumer ever lands.

Drift discipline: when an endpoint changes shape on the API, both `lib/api.ts` files get updated in the same commit (or a coordinated pair of commits, API → both clients). The pattern is well-established from the four-phase nutrition rollout.

### Charts on Mobile

The web client uses [Recharts](https://recharts.org), a React-DOM-only library. React Native needs a different choice. Two pages on the mobile app need charts:

- `/progress` — multi-series scatter + trendline + reference line, X axis is dates, Y axis is normalized 1RM, ~50–200 points typical.
- `/bodyweight` — single-series line + 7-day rolling average overlay, both lines share an axis, ~30–365 points typical.

[Victory Native XL](https://commerce.nearform.com/open-source/victory-native/) is the closest API to Recharts in the React Native ecosystem: declarative components for axes, lines, scatters, custom tooltips. It's Skia-backed, performant, and actively maintained. The data shapes the web charts already compute can be passed through with minimal massaging. **Lean**: Victory Native XL for both pages.

### Calendar Layout on Mobile

Direct port from web — a wide month grid with workout summaries inside each cell — doesn't survive translation to a phone screen. At 7 columns wide on a ~390pt iPhone display, each cell is ~50pt wide, which barely fits a date label, let alone exercise names. The proven iOS pattern for this kind of view is what Apple's own Calendar app does: a compact month grid at the top with dots indicating days that have content, and an agenda-style list below showing the selected day's events in full.

Applied here: the top third of the screen is a swipeable month grid (previous / next month), with a colored dot under any day that has at least one logged workout. Tapping a day selects it and scrolls the bottom two-thirds — an agenda list — to show that day's workouts in detail (workout name, exercise names, set counts). The active day highlights in the grid; the agenda's empty state on an unworked day reads "No workouts on <date>" with a link into the workouts log.

This is the v1 layout. It's the answer to the question the intro pointed at.

### Distribution + Release Pipeline

Two-tier distribution, mirroring what's already scaffolded in the repo:

- **Tier 1 — Expo Go on the developer's phone (free, day-one).** Scan the QR from `npm run start`, app loads in Expo Go over LAN. Used for iterative dev. Requires the phone and the Mac on the same network.
- **Tier 2 — TestFlight Internal Testing ($99/yr Apple, ~2-4 hours of setup).** Real home-screen icon, no Expo Go required. EAS Build runs in CI on dispatch, EAS Submit hands the build to App Store Connect, and the app lands in TestFlight Internal within ~30 minutes. JS-only changes go OTA via EAS Update on every push to `main`, reaching the phone in ~30 seconds.

The GitHub Actions workflow `.github/workflows/release.yml` already does both: `push` to `main` fires an `eas update`; manual `workflow_dispatch` with `build=true` fires a full `eas build`. The remaining setup is administrative — Apple Developer Program activation, app icon at `assets/icon.png` (1024×1024, no alpha), an `EXPO_TOKEN` GitHub repo secret, the one-time `eas init` + `eas credentials` interactive flow from the local checkout. These are documented in `prog-strength-mobile/README.md`.

Once Tier 2 is live, the loop the user described in the scaffolding conversation — "commit + push → on phone in ~30 seconds for JS changes" — is the day-to-day. Native rebuilds happen only when SDK upgrades, new native modules, or app icon / config plugin changes ship.

### Backend Compatibility

This SOW assumes no backend work. The two coordinating changes that already shipped to make the mobile flow practical:

- `RETURN_TO_ALLOWED_ORIGINS` on the API includes `progstrength://auth/callback` so the OAuth flow can redirect back into the app's custom URL scheme.
- ECR + the deploy pipeline are set up so any backend change ships independently of mobile pushes.

If either of those regresses, mobile breaks in predictable ways: OAuth bounces with `cancel`, or the agent's chat SSE fails the same way it does for web. Both surfaces should be exercised post-deploy when the API moves.

## Open Questions

1. **Personal Records and Bodyweight inclusion at v1.** The intro lists four feature buckets (chat, workouts, nutrition, progress) and doesn't explicitly call out Personal Records or Bodyweight. Both pair tightly with the listed buckets: PRs are the "current strength" anchor that pairs with Progress; Bodyweight pairs with Nutrition for the diet-inference signal the agent reads. **Lean**: include both in v1. The screens are small (single chart + form for bodyweight; trophy grid + Customize modal for PRs), and the user-facing story is much weaker without them.

2. **Exercises catalog screen.** Web has `/exercises` — a browseable list of every catalog exercise. The agent already surfaces catalog entries via chat (`list_exercises` MCP tool), which covers the "what slug do I want?" use case. **Lean**: defer for v1. Easy to add later as a Pantry-style list page if usage shows a need.

3. **Chart library: Victory Native XL.** Closest API to Recharts, Skia-backed, performant on RN. Alternatives are `react-native-svg-charts` (lower-level, less featured) and `react-native-skia` directly (highest control, most code). **Lean**: Victory Native XL — minimizes the per-chart code rewrite from web.

4. **Modal pattern: centered modal vs. iOS bottom sheet.** Web uses centered modals (Customize on PRs, item editor on Pantry, recipe builder). iOS users expect interactive lists and pickers to come up as a bottom sheet. **Lean**: bottom sheets via `@gorhom/bottom-sheet` for the list/pick interactions; centered modals only for short confirmations ("Delete this entry? Yes / No"). Mixing both is fine — they signal different intent.

5. **Cross-client type sharing.** `lib/api.ts` is duplicated between web and mobile today. Could be hoisted into a shared npm package, a git submodule, or generated from the API's response shapes via OpenAPI. **Lean**: keep the duplication for v1. The drift cost is small at single-developer pace; the tooling cost of a workspace / submodule is real (especially with React Native's TS config quirks). Revisit if a third consumer ever lands or if drift starts producing bugs.

6. **Sign in with Apple.** Apple's App Store guidelines require Sign in with Apple as a sibling option whenever a third-party identity provider (Google) is offered. Currently irrelevant because the v1 target is TestFlight Internal Testing only — App Review doesn't apply. **Lean**: defer until the public App Store submission SOW. Add `expo-apple-authentication` and a corresponding `/auth/apple/...` backend path then.
