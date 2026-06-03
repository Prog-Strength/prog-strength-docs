# Shared TypeScript Client SDK (prog-strength-sdk)

**Status**: Draft · **Last updated**: 2026-06-02

## Introduction

`prog-strength-web` and `prog-strength-mobile` each maintain their own ~1100-line `lib/api.ts` with identical TypeScript types and `fetch`-based functions against the **Prog Strength** Go API. Every backend change — a new endpoint, a tweaked response shape, an added field — requires two parallel edits in two repos, with no compile-time guarantee that the two clients stay in sync.

The original mobile rollout (`initial-mobile-app-implementation.md`) chose this duplication deliberately, on the rationale that "two consumers, contracts change slowly, single shared package introduces tooling friction that doesn't pay for itself for a single-developer beta." That trade-off has reversed: backend velocity has grown enough that the "edit twice" discipline now costs more per change than a shared package would. Two clients with redundant API code is already friction worth removing — no third consumer is needed to justify the change.

This SOW introduces `prog-strength-sdk`, a new TypeScript-only repo under the Prog-Strength organization that owns the shared client surface. Both web and mobile consume it as a single source of truth, and the duplicated `lib/api.ts` files are deleted at the end of the rollout.

## Proposed Solution

`prog-strength-sdk` is a TypeScript-only npm package, living in its own repo at `github.com/Prog-Strength/prog-strength-sdk`. It exports a `ProgStrengthClient` class and the full set of API types (`Workout`, `Exercise`, `PantryItem`, etc.). The class is instantiated once per client with a `baseUrl` and an async `getToken()` callback; each method drops the explicit `token` parameter that today's free-function fetchers carry.

Distribution is git-tag installs — no registry. Consumers reference the SDK as `"prog-strength-sdk": "github:Prog-Strength/prog-strength-sdk#v0.1.0"`. Versioning is managed by `semantic-release` on push to `main`, matching the pattern already in use across `prog-strength-api`, `prog-strength-mcp`, and `prog-strength-agent`. Releases stay in the 0.x series for the duration of pre-launch.

Build output is ESM-only `dist/` produced by `tsup` during the release workflow and committed to the release tag, so consumer installs are hermetic — no install-time build, no dev deps pulled into client `node_modules`. The `files` array limits the package surface to `dist/`, `README.md`, and `CHANGELOG.md`.

Migration runs as three independent PRs: one to stand up the SDK (PR 1), then one per client to swap import sites and delete `lib/api.ts` (PR 2 and PR 3). PR 2 and PR 3 are independent of each other and can land in either order once PR 1 has tagged `v0.1.0`.

What stays out of scope: `auth.ts`, `agent.ts`, `stream.ts`, `config.ts`, and `speech.ts` remain per-client. They have genuine platform differences (Keychain vs cookies, `expo/fetch` vs native `ReadableStream`, Web Speech API vs `expo-speech-recognition`) that would require an adapter layer not worth the effort at this stage.

## Goals and Non-Goals

### Goals

- Stand up `prog-strength-sdk` with `ProgStrengthClient` and the full type surface ported from `prog-strength-web/lib/api.ts`.
- Replace every API call site in `prog-strength-web` with SDK calls; delete `prog-strength-web/lib/api.ts`.
- Replace every API call site in `prog-strength-mobile` with SDK calls; delete `prog-strength-mobile/lib/api.ts`.
- Reach the state where adding or changing an API endpoint requires exactly one place to edit (the SDK), plus a dep ref bump per client that wants to pick it up.
- Configure `semantic-release` so the SDK stays in the 0.x series until manually bumped to 1.0 at launch.
- Ship SDK unit tests (`vitest`, mocked `fetch`) covering every public client method's URL, method, auth header, query string, and envelope unwrap.
- Match the existing release pattern from `prog-strength-api`, `prog-strength-mcp`, `prog-strength-agent` (conventional-commits, `semantic-release`, GitHub Actions on push to `main`).

### Non-Goals

- Migrating `auth.ts`, `agent.ts`, `stream.ts`, `config.ts`, or `speech.ts` into the SDK. These have real platform differences and would need an adapter abstraction not worth the effort today.
- Refactoring token storage or refresh in either client. The SDK calls a per-client `getToken()` callback; everything around it stays as-is.
- Publishing to the public npm registry. Git-tag installs only.
- Generating types from the Go API source or an OpenAPI spec. Worth doing eventually; not now.
- Adding any consumer beyond `prog-strength-web` and `prog-strength-mobile`. The SDK exists to deduplicate the two frontend clients; the developer / agent / MCP repos are not target consumers and do not influence the SDK's design.
- Cutting `v1.0.0`. The SDK stays in the 0.x series for the duration of pre-launch.
- Backend API changes. Same Go endpoints, same JWT auth, same `{service, message, data}` envelope.
- Adding integration tests against a live API or any new automated tests in `prog-strength-web` / `prog-strength-mobile`.
- Long deprecation window for the old `lib/api.ts` files. They are deleted in the same PR that swaps their last call site.

## Implementation Details

### Repo Layout

```
prog-strength-sdk/
├── src/
│   ├── index.ts              # public exports: ProgStrengthClient + all types
│   ├── client.ts             # ProgStrengthClient class
│   ├── envelope.ts           # API envelope unwrap helper
│   ├── http.ts               # internal fetch wrapper (auth header injection, base URL)
│   └── types/
│       ├── workout.ts        # Workout, WorkoutExercise, WorkoutSet, PersonalRecordEvent
│       ├── exercise.ts       # Exercise, MuscleGroup, Equipment
│       ├── personal-records.ts
│       ├── nutrition.ts      # PantryItem, Recipe, NutritionLogEntry
│       ├── bodyweight.ts
│       ├── progress.ts       # MuscleGroupProgression, EstimatedOneRepMax
│       └── index.ts          # re-exports
├── tests/
│   └── client.test.ts        # vitest, fetch-mocked unit tests
├── dist/                     # built by tsup; committed by semantic-release at release time
├── .github/workflows/release.yml
├── release.config.js
├── tsup.config.ts
├── tsconfig.json
├── vitest.config.ts
├── package.json
├── package-lock.json
├── CHANGELOG.md              # managed by semantic-release
├── CLAUDE.md
└── README.md
```

Types are split by domain rather than co-located in a single file. `prog-strength-web/lib/api.ts` has grown to ~1100 lines and is starting to be hard to navigate; the split is a one-time gain since individual domain files won't grow as fast as the monolith did. JSDoc comments port verbatim — they explain non-obvious shape decisions (e.g., "dumbbell weight is per-dumbbell, not the pair") that future readers benefit from.

### SDK Surface

Consumer usage:

```ts
import { ProgStrengthClient } from 'prog-strength-sdk';
import type { Workout, Exercise, PantryItem } from 'prog-strength-sdk';

const client = new ProgStrengthClient({
  baseUrl: config.apiUrl,
  getToken: async () => getToken(),  // per-client lib/auth.ts
});

const workouts = await client.listWorkouts({ from, to });
const exercise = await client.getExercise(slug);
await client.createPantryItem({ name, brand, /* ... */ });
```

Shape notes:

- **`ProgStrengthClient` class.** Today's free-function fetchers each take `token` as their first argument, which is repetitive at every call site. The class lets each client instantiate once with `baseUrl` and `getToken`, then call methods without re-passing them.
- **`getToken` is an async function, not a string.** Each request resolves the token afresh via `await getToken()`, so token rotation or refresh in either client is transparent to the SDK. The SDK never stores or manages the token.
- **Method names and shapes match today.** `listWorkouts`, `getWorkout`, `createPantryItem`, etc. Same payload types in, same unwrapped response type out. Only the `token` parameter goes away.
- **Envelope unwrap is internal.** The Go API's `{service, message, data}` envelope is unwrapped inside the Client; public methods return the unwrapped payload type, identical to today.
- **Zero peer dependencies.** No `react`, `next`, or `expo`. The SDK calls global `fetch`, which is available in both Next.js 16 and React Native 0.83 runtimes today.
- **Error handling matches today.** Whatever the current `unwrap` helper does on non-2xx responses ports as-is. No new error types in this SOW.

### Package Configuration

`package.json` essentials:

```json
{
  "name": "prog-strength-sdk",
  "version": "0.0.0-development",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  },
  "files": ["dist", "README.md", "CHANGELOG.md"],
  "scripts": {
    "build": "tsup",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "release": "semantic-release"
  }
}
```

ESM-only output. Both consumers handle ESM cleanly. Skipping CJS halves the build matrix and the published byte count; it can be added if a future consumer needs it.

`tsup.config.ts`:

```ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts'],
  format: ['esm'],
  dts: true,
  sourcemap: true,
  clean: true,
});
```

`tsconfig.json` uses strict mode, ES2022 target, `moduleResolution: "bundler"`.

### Release Pipeline

`.github/workflows/release.yml` mirrors `prog-strength-api/.github/workflows/release.yml` with the following differences:

- Drops the `build_and_push` (ECR) and `deploy` (EC2) jobs — the SDK is not a deployable service.
- The `release` job runs `npm ci`, `npm run typecheck`, `npm test`, and `npm run build` before invoking `semantic-release`. The build step produces `dist/`, which `@semantic-release/git` then commits as part of the release.

`release.config.js`:

```js
module.exports = {
  branches: ['main'],
  plugins: [
    [
      '@semantic-release/commit-analyzer',
      {
        preset: 'conventionalcommits',
        // Stay in 0.x during pre-launch. Remove this rule when cutting v1.0.
        releaseRules: [{ breaking: true, release: 'minor' }],
      },
    ],
    '@semantic-release/release-notes-generator',
    [
      '@semantic-release/changelog',
      { changelogFile: 'CHANGELOG.md' },
    ],
    [
      '@semantic-release/npm',
      { npmPublish: false },
    ],
    [
      '@semantic-release/git',
      {
        assets: ['CHANGELOG.md', 'package.json', 'package-lock.json', 'dist/**'],
        message: 'chore(release): ${nextRelease.version}\n\n${nextRelease.notes}',
      },
    ],
    '@semantic-release/github',
  ],
};
```

`@semantic-release/npm` is included with `npmPublish: false` so the plugin updates `version` in `package.json` without publishing to a registry. The bumped `package.json` is then committed alongside `dist/` by `@semantic-release/git`.

### 0.x Version Pinning

Two configuration choices keep the SDK in the 0.x series during pre-launch:

1. **Bootstrap with a `v0.0.0` tag.** Before the first `feat:` commit lands on `main`, manually create a `v0.0.0` tag on `main` and push it. `semantic-release` uses this as the baseline; the first `feat:` commit bumps to `0.1.0`, the first `fix:` to `0.0.1`. Without this bootstrap, `semantic-release` defaults to `1.0.0` on the first release.
2. **Map breaking changes to `minor` in `releaseRules`.** Without this override, a `feat!:` or `BREAKING CHANGE:` commit would jump to `1.0.0`. Mapping `breaking: true` to `minor` keeps breaking changes inside the 0.x series (e.g., `0.3.0 → 0.4.0`). At v1.0 cut, this rule is removed.

### Migration Plan

Three PRs, executed in order. PR 1 is a hard dependency for the others; PR 2 and PR 3 are independent of each other.

**PR 1 — `prog-strength-sdk` initial commit.**

- Initialize the repo with the layout above.
- Port `prog-strength-web/lib/api.ts` into `src/`, refactoring from free functions to `ProgStrengthClient` methods. Web is treated as the source of truth because mobile was already a port of it; any small divergences between the two files are resolved in favor of web's shape.
- Split types by domain into `src/types/*.ts`.
- Add `vitest` tests against a mocked global `fetch`. Coverage: every public method gets at least one test asserting URL, HTTP method, `Authorization` header, query-param serialization, and envelope unwrap. ~30 tests total; not exhaustive, but enough to catch typos and accidental signature drift.
- Manually create the `v0.0.0` tag on `main` *before* PR 1 is merged.
- On merge to `main`, `semantic-release` publishes **`v0.1.0`** (initial `feat:` commit under 0.x rules).

**PR 2 — `prog-strength-web` migration.**

- Add `"prog-strength-sdk": "github:Prog-Strength/prog-strength-sdk#v0.1.0"` to `package.json`.
- Add a thin `lib/api-client.ts` that instantiates `ProgStrengthClient` once, wired to `config.apiUrl` and the existing `getToken()` from `lib/auth.ts`. This instance is imported by call sites.
- Replace every `import { ... } from '@/lib/api'` (both value and type imports) with imports from `prog-strength-sdk` and method calls on the shared client.
- Delete `lib/api.ts` entirely.
- `npm run typecheck` passes. Manual smoke of the golden paths (workout list/detail, log a workout, view PRs, log nutrition, log bodyweight, view progress). Merge.

**PR 3 — `prog-strength-mobile` migration.**

- Same shape as PR 2, against `prog-strength-mobile`.
- Delete `lib/api.ts` and the comment block in its header explaining the "duplication is deliberate" decision.
- `npm run typecheck` passes. Manual smoke on TestFlight build of the golden paths. Merge.

### Testing

- **SDK repo.** `vitest` with `fetch` mocked. Every public client method tested for URL, method, `Authorization` header, query-string serialization, and envelope unwrap. CI runs `typecheck` and `test` on every PR.
- **Web and mobile migrations.** `tsc --noEmit` is the safety net. If a swap doesn't fit the new method signature, type-check fails before merge. No new automated tests added in either client.
- **Manual smoke.** Before merging each migration PR, exercise the golden paths in the affected client. Same pre-merge bar as today.
- **No integration tests** against a live API in this SOW.

### Rollback Plan

- **Per-client revert.** `git revert` the migration PR in the affected client restores its `lib/api.ts` and removes the SDK dep. The other client is unaffected — web and mobile migrations are fully independent post-merge.
- **SDK-level fix.** Land a `fix:` commit on `prog-strength-sdk` `main`; `semantic-release` cuts a patch (e.g., `v0.1.1`). Consumers bump their dep ref when ready — there is no forced upgrade.

### Documentation Updates

After PR 3 merges, append a short note to `prog-strength-docs/sows/initial-mobile-app-implementation.md` under the "Shared Types + API Client" section: *"Superseded by `prog-strength-sdk-shared-client.md`. The duplication trade-off was reversed once API velocity made the 'edit twice' discipline more expensive than a shared package."*

A `CLAUDE.md` lands in the new repo at PR 1 capturing the architectural decisions above (ESM-only, 0.x pinning rules, `dist/` committed at release time, zero peer deps) so future sessions don't waste context re-deriving them.

## Open Questions

1. **Should the Client expose a generic escape hatch** (e.g., `client.request('/some/path', { method, body })`) for endpoints not yet wrapped by a typed method? Options: (a) yes — adds flexibility when a one-off API call is needed before an SDK release; (b) no — every endpoint becomes a typed method, and the commit → tag → bump loop is fast enough to absorb. Tentative lean: (b). A generic escape hatch erodes the type safety the SDK exists to provide.
2. **Where does generated-from-Go type emission eventually live?** Options: (a) the API repo emits TS types as a release artifact and the SDK consumes them as a build input; (b) the SDK owns codegen as a script that pulls from the API repo; (c) `prog-strength-data` becomes the canonical shared-type source. Tentative lean: (a) — keeps the Go contract authoritative and the SDK passive. Out of scope for this SOW; tracked separately.
