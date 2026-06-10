# User Profile & Preferences — Implementation Plan

> **For agentic workers:** Implement task-by-task. Each task is implemented by a
> subagent, then spec-reviewed and code-quality-reviewed before moving on.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the user profile user-editable and give identity a home in the
web UI. A lifter can pick the name the coach calls them by (`display_name`,
already stored but now editable), record their height once (`height_cm`,
canonical cm, converted at the display edge), and replace the OAuth avatar with
their own picture (`avatar_key` → S3). Profile reads flow through a single
resolved `GET /me` that presigns the S3 avatar or falls back to the OAuth avatar
URL. The agent picks up name + height through the same prompt-prefix bootstrap
that already injects `client_timezone`. The web client gains profile fields on
the Settings page and a sidebar account anchor (avatar + name + a menu holding
Sign out).

**SOW:** `sows/user-profile-and-preferences.md`

**Repos & merge order (rollout):** `prog-strength-infra` (bucket must exist
before the API can write to it) **and** `prog-strength-api` can land in either
order, but the bucket must be **deployed** before the API's avatar endpoints are
exercised → `prog-strength-agent` (additive, no client yet) → `prog-strength-web`
(the only repo that calls the new endpoints) → `prog-strength-docs` (status flip).

**Branch in every repo:** `feat/user-profile-and-preferences`.

---

## Decisions / deviations from the SOW's literal text

- **A third nullable column: `oauth_avatar_url`.** The SOW lists only two new
  columns (`height_cm`, `avatar_key`) and says `GET /me` falls back to "the
  OAuth-provider avatar URL when `avatar_key` is null." But the current OAuth
  flow (`internal/auth/google.go`) only fetches `email` and `name` — Google's
  `picture` claim is never captured and there is **nowhere to read an OAuth
  avatar URL from**. To make the SOW's explicit avatar-fallback goal work we add
  a third nullable column `oauth_avatar_url TEXT`, populate it from the Google
  `picture` claim at signup, and opportunistically refresh it on subsequent
  logins. This is the minimal change that satisfies "removing it reverts to the
  OAuth avatar fallback." Existing users have it null until their next Google
  login (self-healing) or until they upload their own avatar; `GET /me` returns
  `avatar_url: null` in that interim and the web client renders an initials
  placeholder.
- **Avatar reaping uses object tagging, not naive age expiration.** The SOW's
  Open Question 1 leans "S3 lifecycle rule, 7-day expiry." A plain
  `expiration { days = 7 }` on the bucket/prefix would delete the *current*
  avatar of any user who hasn't changed it in 7 days (its `avatar_key` still
  points at a now-deleted object → broken image). To keep UUID-per-upload keys
  (the SOW's cache-busting rationale) **and** reap only superseded objects, the
  API best-effort tags the previous object `avatar-status=orphaned` on
  upload/delete, and the lifecycle rule expires **only tagged objects** after 7
  days. Tagging is off the hot path (after the row update, failure-tolerant) so
  the upload path stays fast. The current avatar is never tagged → never reaped.
- **Open-question leans adopted:** presigned GET expiry = **1 hour**; lifecycle
  expiry = **7 days**; `display_name` cap = **60 chars**, enforced server-side at
  `PATCH /me`, soft-truncated client-side in the sidebar row.
- **Height range:** `PATCH /me` validates `height_cm` to **50–250 cm** inclusive
  (per the SOW's "e.g. 50–250 cm"); out of range → 400.
- **Avatar upload cap / content types:** **2 MB**; allowlist `image/png`,
  `image/jpeg`, `image/webp` (per SOW).
- **Web shared profile state:** introduce a `ProfileProvider` /
  `useProfile()` context (mirroring the existing `DistanceUnitProvider` /
  `UsageProvider`) mounted in `app/(app)/layout.tsx`, so the sidebar, chat
  payload, and Settings page all read one resolved profile instead of each
  page calling `getMe` independently.

---

# Part A — prog-strength-api (data model + endpoints + S3)

Module: chi router, domain-oriented `internal/<domain>/` layout, raw
`database/sql` with `?` placeholders, `httpresp` response envelope, `authctx`
for the user id. Build/test: `go build ./...`, `go test ./...`, `gofmt -l`,
`golangci-lint run` (if available). S3 via `aws-sdk-go-v2`
(`internal/activity/s3_archiver.go` is the reference). Bucket name is injected
via env var (mirror `TCX_BUCKET_NAME`).

## Task A1 — Migration: add `height_cm`, `avatar_key`, `oauth_avatar_url`

Files: `internal/db/migrations/0NN_user_profile.sql` (next sequential number —
check the directory; the embed glob `migrations/*.sql` auto-picks it up).

- [ ] Determine the next migration number by listing
  `internal/db/migrations/`. Create the migration:
  ```sql
  ALTER TABLE users ADD COLUMN height_cm REAL;
  ALTER TABLE users ADD COLUMN avatar_key TEXT;
  ALTER TABLE users ADD COLUMN oauth_avatar_url TEXT;
  ```
  All three nullable, no backfill (additive; existing rows get NULLs). No
  down-file (repo convention is forward-only embedded migrations — confirm by
  inspecting existing migration files; do not invent a `_down.sql`).
- [ ] Verify the migration applies and advances the schema version:
  `go test ./internal/db/...`.

## Task A2 — User domain: struct fields + validation helpers (+ tests)

Files: `internal/user/user.go`, `internal/user/user_test.go` (create if absent).

- [ ] Add fields to the `User` struct (match existing json/scan ordering
  conventions):
  ```go
  HeightCm       *float64 `json:"height_cm"`        // nullable
  AvatarKey      *string  `json:"-"`                // internal; never serialized raw
  OAuthAvatarURL *string  `json:"-"`                // internal; fallback source
  ```
  `AvatarKey` and `OAuthAvatarURL` are **not** part of the client-facing JSON of
  the bare `User`; the resolved `avatar_url` is produced by the `GET /me`
  handler (A6). Keep `display_name` required in `Validate()` (unchanged).
- [ ] Add validation constants + a helper the handler can reuse:
  ```go
  const (
      DisplayNameMaxLen = 60
      HeightCmMin       = 50.0
      HeightCmMax       = 250.0
  )
  ```
  Extend `Validate()` so an over-long `display_name` (> 60 runes — count runes,
  not bytes) is rejected, and an out-of-range non-nil `height_cm` is rejected.
  Empty `display_name` stays rejected. Return sentinel/`error` values the
  handler can map to 400 with a clear message.
- [ ] Tests: empty name rejected; 61-rune name rejected; 60-rune accepted;
  height 49.9 / 250.1 rejected; 50 / 250 / nil accepted.
- [ ] Verify: `go test ./internal/user/...`, `gofmt -l`.

## Task A3 — Repository: persist new columns; partial update (+ tests)

Files: `internal/user/repository.go` (interface),
`internal/user/sqlite_repository.go`, `internal/user/memory_repository.go`
(if a memory impl exists — the handler tests use `NewMemoryRepository()`),
`internal/user/sqlite_repository_test.go`.

- [ ] Update `Create` and the SELECT in `GetByID` / `GetByEmail` to include the
  three new columns (scan into the pointer fields; handle SQL NULL). On `Create`,
  persist `oauth_avatar_url` when provided (see A8).
- [ ] Update `Update` to persist `height_cm`, `avatar_key`, and
  `oauth_avatar_url` alongside the existing mutable fields, preserving the
  established pattern (fetch existing for immutable fields, validate, bump
  `updated_at`, check `RowsAffected`). `Update` writes whatever is on the struct
  — partial-update *semantics* live in the handler (which loads the row, applies
  only provided fields, then calls `Update`).
- [ ] Mirror all changes in the memory repository so handler tests stay
  hermetic.
- [ ] Tests: create with height/avatar_key/oauth_avatar_url round-trips through
  SQLite; update height then read back; NULLs handled both directions; existing
  Create/Get/Update tests still pass.
- [ ] Verify: `go test ./internal/user/...`.

## Task A4 — Avatar store abstraction: S3 put/presign/tag + delete (+ tests)

Files: `internal/user/avatar_store.go` (interface + S3 impl + an in-memory/fake
for tests), `internal/user/avatar_store_test.go`. Reference:
`internal/activity/s3_archiver.go` for client construction
(`config.LoadDefaultConfig` + `s3.NewFromConfig`).

- [ ] Define the interface:
  ```go
  type AvatarStore interface {
      // Put writes bytes under key with contentType; returns nil on success.
      Put(ctx context.Context, key, contentType string, body []byte) error
      // PresignGet returns a time-limited GET URL for key.
      PresignGet(ctx context.Context, key string) (string, error)
      // TagOrphaned best-effort marks a superseded object for lifecycle reaping.
      TagOrphaned(ctx context.Context, key string) error
  }
  ```
- [ ] S3 impl (`S3AvatarStore`): `Put` via `PutObject` (set `ContentType`);
  `PresignGet` via `s3.NewPresignClient(client).PresignGetObject` with
  `s3.WithPresignExpires(1 * time.Hour)`; `TagOrphaned` via `PutObjectTagging`
  setting tag `avatar-status=orphaned` (this is the tag the lifecycle rule in
  Part B filters on — keep the key/value in sync with the Terraform). Construct
  with `NewS3AvatarStore(ctx, bucket string)`.
- [ ] Key builder: `func AvatarKey(userID, ext string) string` →
  `fmt.Sprintf("user_id=%s/%s.%s", userID, uuid, ext)` using the repo's
  `internal/id` or a UUID lib already in go.mod (check; `google/uuid` may be
  present — otherwise reuse `internal/id`). Map content-type → ext
  (png/jpeg→jpg/webp).
- [ ] Provide a fake store for tests (in-memory map; presign returns a sentinel
  URL like `https://signed.example/<key>`; records tagged keys).
- [ ] Tests (fake-level): key format; content-type→ext mapping; TagOrphaned
  records the key. (S3 wire calls are not unit-tested — mirror how the TCX
  archiver is treated.)
- [ ] Verify: `go test ./internal/user/...`.

## Task A5 — `GET /me`: resolved profile DTO (+ tests)

Files: `internal/user/handler.go`, `internal/user/handler_test.go`.

- [ ] Define the resolved DTO returned by all four endpoints (so the client
  always gets the same shape):
  ```go
  type meResponse struct {
      ID           string   `json:"id"`
      Email        string   `json:"email"`
      DisplayName  string   `json:"display_name"`
      WeightUnit   string   `json:"weight_unit"`
      DistanceUnit string   `json:"distance_unit"`
      HeightCm     *float64 `json:"height_cm"`
      AvatarURL    *string  `json:"avatar_url"`
  }
  ```
- [ ] `getMe`: load the user via `authctx.UserIDFrom` + `repo.GetByID`. Resolve
  `avatar_url`:
  - if `AvatarKey != nil` → `store.PresignGet(ctx, *AvatarKey)`;
  - else if `OAuthAvatarURL != nil` → that URL;
  - else `nil`.
  On presign failure, log and fall back to `OAuthAvatarURL` (or nil) rather than
  500ing the whole profile. `display_name` is the stored value (always non-empty
  per validator). Return via `httpresp.OK`.
- [ ] Build a `resolveMe(ctx, u)` helper reused by PATCH/POST/DELETE so they all
  return the freshly-resolved profile.
- [ ] Inject the `AvatarStore` into the handler constructor (`NewHandler(repo,
  store)` — update existing call sites; pass a fake in tests, nil-guard if a
  deployment has no bucket configured so `GET /me` still serves name/height).
- [ ] Tests: avatar_key set → avatar_url is the presigned sentinel; avatar_key
  nil + oauth url set → avatar_url is the oauth url; both nil → null; height
  passes through.
- [ ] Verify: `go test ./internal/user/...`.

## Task A6 — `PATCH /me`: edit name/height (+ tests)

Files: `internal/user/handler.go`, `internal/user/handler_test.go`.

- [ ] Request DTO with pointer fields for partial update:
  ```go
  type patchMeRequest struct {
      DisplayName *string  `json:"display_name"`
      HeightCm    *float64 `json:"height_cm"` // distinguishes "set to X" from "absent"
  }
  ```
  Note: this endpoint **does not** touch `avatar_key` (avatar mutation goes
  through A7/A8). It also leaves `weight_unit`/`distance_unit` to the existing
  `PATCH /me` behavior if that already lives here — **important:** the repo
  already has a `PATCH /me` (`updateMe`) handling units. **Extend the existing
  handler/DTO** rather than adding a second route: add `display_name` (already
  present) length validation and `height_cm` to the existing `updateMeRequest`,
  keeping `weight_unit`/`distance_unit`. Confirm by reading the current
  `handler.go` before editing.
- [ ] Apply provided fields onto the loaded user, call `Validate()` (catches
  empty/over-long name and out-of-range height → 400 with clear message via
  `httpresp.Error`), `repo.Update`, then return `resolveMe`.
- [ ] Tests: set height to 180 persists + echoes back; empty name → 400;
  61-char name → 400; height 300 → 400; unit-only patch still works (regression);
  setting `height_cm: null`... decide semantics — treat explicit `null` as
  "clear height". (Use `json.RawMessage`/presence detection only if needed;
  simplest: pointer nil = absent, and provide no way to null height from PATCH
  unless trivial. Document the choice in the PR.)
- [ ] Verify: `go test ./internal/user/...`.

## Task A7 — `POST /me/avatar` + `DELETE /me/avatar` (+ tests)

Files: `internal/user/handler.go`, `internal/user/handler_test.go`; mount in
`Mount`.

- [ ] `POST /me/avatar` (multipart, mirror `internal/activity/handler.go`
  `uploadTCX`): `http.MaxBytesReader` + `ParseMultipartForm` at 2 MB; read the
  `file` form field. Reject > 2 MB → 413 (`httpresp.ErrorWithCode`, code
  `file_too_large`). Sniff content type via `http.DetectContentType` on the first
  512 bytes (don't trust the client header) and require it in the allowlist
  (`image/png`, `image/jpeg`, `image/webp`) → else 415
  (`unsupported_media_type`). Build a new key via `AvatarKey(userID, ext)`,
  `store.Put`, then load the user, capture the **previous** `avatar_key`, set the
  new `avatar_key`, `repo.Update`. After the successful update, best-effort
  `store.TagOrphaned(prevKey)` if the previous key was non-nil (log on error, do
  not fail the request). Return `resolveMe` (201 or 200 — match repo convention;
  `httpresp.OK`/`Created`).
- [ ] `DELETE /me/avatar`: load user; capture current `avatar_key`; set it to
  nil; `repo.Update`; best-effort `store.TagOrphaned(prevKey)`; return
  `resolveMe` (now carrying the OAuth fallback or null).
- [ ] Mount: `r.Post("/me/avatar", h.uploadAvatar)`, `r.Delete("/me/avatar",
  h.deleteAvatar)` in the JWT-gated group next to `/me`.
- [ ] Tests (httptest + multipart helper like `activity/handler_test.go`, fake
  store): valid png upload → 200, avatar_url is presigned sentinel, avatar_key
  set on row, previous key tagged orphaned; oversized → 413; text/plain →
  415; delete clears avatar_key, tags old key, returns oauth fallback; upload
  with no previous avatar does not call TagOrphaned.
- [ ] Verify: `go test ./internal/user/...`.

## Task A8 — OAuth: capture Google `picture`; thread `oauth_avatar_url` (+ tests)

Files: `internal/auth/google.go`, `internal/auth/handler.go`,
`internal/auth/*_test.go`.

- [ ] Add `Picture string `json:"picture"`` to the `googleUser` struct
  (`google.go`) — Google's userinfo already returns it under the
  `userinfo.profile` scope the app requests.
- [ ] Thread it through `findOrCreateUser(ctx, email, displayName, avatarURL
  string)`: on **create**, set `OAuthAvatarURL` (nil if empty). On **existing**,
  if `avatarURL != ""` and differs from the stored value, update the row
  (opportunistic refresh so existing users self-heal on next login). Update both
  call sites: `googleCallback` passes `gu.Picture`; `devToken` passes `""`.
- [ ] Tests: new google user persists `oauth_avatar_url`; existing user with a
  changed picture gets it refreshed; empty picture leaves it nil; dev-token path
  passes empty without error.
- [ ] Verify: `go test ./internal/auth/...`.

## Task A9 — Config + server wiring

Files: `internal/config/config.go`, `internal/server/server.go`.

- [ ] Add `AvatarBucketName string` to `Config` from env `AVATAR_BUCKET_NAME`
  (default ""). No new required vars (additive).
- [ ] In `server.New`: if `cfg.AvatarBucketName != ""` construct
  `user.NewS3AvatarStore(ctx, cfg.AvatarBucketName)`; else nil (handlers
  nil-guard: avatar upload/delete → 503/clear error when unconfigured, `GET /me`
  still serves name/height/oauth-fallback). Pass the store into `NewHandler`.
- [ ] Verify: `go build ./...`, `go test ./...`, `gofmt -l`, lint.

---

# Part B — prog-strength-infra (avatar bucket + IAM + wiring)

Terraform, `modules/<name>/` layout. Reference: `modules/tcx_storage/`
(bucket + versioning + encryption + public-access-block + lifecycle + IAM
policy + role attachment), root `main.tf` / `variables.tf` / `outputs.tf`,
`compose/api/docker-compose.yml` for env injection. Region us-east-2.

## Task B1 — `modules/avatar_storage` module

Files: `modules/avatar_storage/{main.tf,variables.tf,outputs.tf,versions.tf}`
(copy the shape of `modules/tcx_storage/`).

- [ ] `aws_s3_bucket "avatars"` (`bucket = var.bucket_name`, tags
  `Purpose = "user-avatars"`), `aws_s3_bucket_server_side_encryption_configuration`
  (AES256), `aws_s3_bucket_public_access_block` (all true). Versioning is **not
  required** for this design (latest-wins via distinct UUID keys) — omit it, or
  enable it to match house style; if enabled, do not rely on noncurrent
  expiration for reaping.
- [ ] `aws_s3_bucket_lifecycle_configuration "avatars"`: a single rule that
  expires **only objects tagged orphaned**:
  ```hcl
  rule {
    id     = "expire-orphaned-avatars"
    status = "Enabled"
    filter {
      tag {
        key   = "avatar-status"
        value = "orphaned"
      }
    }
    expiration {
      days = var.orphan_expiration_days # default 7
    }
  }
  ```
  Keep the tag key/value **identical** to the API's `TagOrphaned` (Part A4).
  Document why: current avatars are untagged and must never be expired.
- [ ] IAM: `data.aws_iam_policy_document` granting `s3:ListBucket` on the bucket
  arn and `s3:GetObject`, `s3:PutObject`, `s3:PutObjectTagging`,
  `s3:DeleteObject` on `${arn}/*`; `aws_iam_policy` (`name =
  "${var.name_prefix}-avatars"`); `aws_iam_role_policy_attachment` to
  `var.instance_role_name`. (`PutObjectTagging` is required for orphan tagging;
  `GetObject` for presign; `DeleteObject` is harmless future-proofing.)
- [ ] `variables.tf`: `bucket_name`, `name_prefix`, `instance_role_name`,
  `orphan_expiration_days` (default 7). `outputs.tf`: `bucket_name`.
- [ ] `versions.tf`: mirror `modules/tcx_storage/versions.tf`.

## Task B2 — Wire module into root + outputs + variables

Files: root `main.tf`, `variables.tf`, `outputs.tf`,
`environments/*.tfvars` if bucket names are set there.

- [ ] Add an `avatar_storage` variable block mirroring the `tcx_storage` one
  (bucket name etc.), instantiate `module "avatar_storage"` passing
  `name_prefix`, `instance_role_name` (same instance role the tcx module uses),
  and bucket name. Default bucket name `prog-strength-avatars`.
- [ ] Add root output `avatar_bucket_name = module.avatar_storage.bucket_name`
  with a description noting it must be set as `AVATAR_BUCKET_NAME` in the backend
  host's `.env` (mirror the `tcx_bucket_name` output wording).
- [ ] Verify: `terraform fmt -recursive`, `terraform validate` (if the backend
  can init offline; otherwise at least `terraform fmt` + `terraform validate`
  within the module via `-backend=false`). Document if validate can't run
  without cloud creds.

## Task B3 — docker-compose env injection

Files: `compose/api/docker-compose.yml`.

- [ ] Add `- AVATAR_BUCKET_NAME=${AVATAR_BUCKET_NAME}` to the api service
  environment (mirror the `TCX_BUCKET_NAME` line + its comment about in-memory
  fallback when unset). `AWS_REGION` is already forwarded for the tcx bucket;
  confirm it's present (the avatar store needs it too).
- [ ] Verify: `docker compose -f compose/api/docker-compose.yml config` parses
  (if docker available; else YAML lint / visual review).

---

# Part C — prog-strength-agent (name + height in the system prompt)

Python 3.12, FastAPI, `uv`. Tests: `uv run pytest tests/`. Lint:
`uv run ruff check src/ tests/`, format `uv run ruff format`. Target:
`src/prog_strength_agent/prompt.py` (`build_chat_system_prompt`) and
`server.py` (`ChatRequest`, `/chat` handler at the call site).

## Task C1 — Extend `build_chat_system_prompt` (+ tests)

Files: `src/prog_strength_agent/prompt.py`, `tests/test_prompt.py`.

- [ ] Add params `display_name: str | None = None` and
  `height_cm: float | None = None` to `build_chat_system_prompt` (keep existing
  `client_timezone`, `now`). After the timezone prefix, build an identity line:
  - When `display_name` is non-empty: `"You are talking to {display_name}."`
  - Append height only when `height_cm is not None`:
    `" They are {height_cm:g} cm tall."` (use `:g` so `180.0` renders `180`).
  - When `display_name` is empty/None: omit the identity line entirely (and the
    height clause with it — height without a name reads oddly; if height is set
    but name somehow empty, still skip — name is always present in practice).
  - Add the SOW's guidance sentence to the prepended context (not the static
    `SYSTEM_PROMPT`): height is conversational context only; the agent must not
    volunteer height-derived body-composition inferences (BMI etc.); answer only
    if explicitly asked, without editorializing.
  Compose order: timezone prefix, then identity line, then guidance, then
  `SYSTEM_PROMPT` (mirror the existing prefix-concat style).
- [ ] Tests: name present → identity line present; height present → "cm tall"
  clause present and `180.0` renders without trailing `.0`; height None →
  no height clause; name None/empty → no identity line; timezone behavior
  unchanged (existing tests stay green).
- [ ] Verify: `uv run pytest tests/test_prompt.py`.

## Task C2 — `ChatRequest` fields + `/chat` call site (+ tests)

Files: `src/prog_strength_agent/server.py`, `tests/` (chat/server tests).

- [ ] Add to `ChatRequest`: `display_name: str | None = None`,
  `height_cm: float | None = None` (defaults so older clients don't break).
- [ ] At the `build_chat_system_prompt(req.client_timezone)` call site, pass
  `display_name=req.display_name, height_cm=req.height_cm`.
- [ ] Tests: a `/chat` request including `display_name`/`height_cm` produces a
  system prompt containing the identity line (assert via a monkeypatched
  harness/captured prompt, mirroring how existing server tests inspect the
  prompt or stream); request without them still works.
- [ ] Verify: `uv run pytest tests/`, `uv run ruff check src/ tests/`.

---

# Part D — prog-strength-web (Settings fields + sidebar account anchor)

Next.js 16 / React 19 / TS, Tailwind + CSS vars, vitest + RTL, npm.
`npm run test`, `npm run lint`, `npm run typecheck`, `npm run build`. Reuse
form primitives (`inputClass`, `Field`, `SegmentedControl`), CSS-var dark theme,
the `unwrap` API pattern, and the context style of
`lib/distance-unit-context.tsx` / `lib/usage-context.tsx`.

## Task D1 — API client: types + profile calls

Files: `lib/api.ts`.

- [ ] Extend the `User`/profile type with `height_cm?: number | null` and
  `avatar_url?: string | null`. Add a `ResolvedProfile` type matching the A5
  DTO if cleaner than overloading `User`.
- [ ] Update/confirm `getMe` returns the resolved shape. Update `updateMe` to
  accept `{ display_name?, height_cm?, weight_unit?, distance_unit? }` (PATCH).
  Add:
  ```ts
  export async function uploadAvatar(token: string, file: File): Promise<ResolvedProfile>
  export async function deleteAvatar(token: string): Promise<ResolvedProfile>
  ```
  `uploadAvatar` posts `multipart/form-data` with field name `file` (do **not**
  set `Content-Type` manually — let the browser set the boundary), Bearer auth,
  to `${config.apiUrl}/me/avatar`; `deleteAvatar` is `DELETE /me/avatar`. Unwrap
  the `{data}` envelope.
- [ ] Verify: `npm run typecheck`.

## Task D2 — `ProfileProvider` + `useProfile()` (+ tests)

Files: `lib/profile-context.tsx`, `lib/profile-context.test.tsx`.

- [ ] Mirror `usage-context.tsx`: fetch `getMe` on mount (when a token exists),
  expose `{ profile, loading, error, refresh, update, uploadAvatar,
  removeAvatar }`. `update`/`uploadAvatar`/`removeAvatar` call the API then set
  the returned resolved profile into state (so sidebar + chat update instantly).
  On 401 → `clearToken()` + route to `/login`.
- [ ] Tests (RTL + mocked api): fetches on mount; `update` reflects the new
  display_name; `uploadAvatar` swaps `avatar_url`; error state set on failure.
- [ ] Verify: `npm run test -- profile-context`.

## Task D3 — Mount `ProfileProvider` in `app/(app)/layout.tsx`

Files: `app/(app)/layout.tsx`.

- [ ] Wrap the authed subtree with `<ProfileProvider>` (alongside the existing
  `UsageProvider`), so the sidebar, chat page, and settings share one profile.
- [ ] Verify: `npm run typecheck`, `npm run build`.

## Task D4 — Settings page: name / height / avatar fields (+ tests)

Files: `app/(app)/settings/page.tsx`, `app/(app)/settings/page.test.tsx`.

- [ ] Add a **Profile** section above the existing Units section, using
  `useProfile()` and existing form primitives:
  - **Display name**: text input, required, non-empty, soft cap 60 (client
    `maxLength`); on save → `update({ display_name })`; inline error on server
    400.
  - **Height**: numeric input shown in the user's familiar unit (`distance_unit`
    `mi`→inches, `km`→cm — confirm the mapping; SOW says "cm or in" keyed off
    `distance_unit`). Convert in↔cm at the input/display edge (1 in = 2.54 cm),
    store `height_cm`; on save → `update({ height_cm })`. Empty input = clear.
    Show "no height set" when null.
  - **Avatar**: image picker with preview (preview shows current `avatar_url`,
    or OAuth/placeholder when none), a **Remove** button when an avatar is set
    (`removeAvatar()`), and on file pick → `uploadAvatar(file)`. Client-side
    guard: reject > 2 MB / non-image with a toast before calling the API
    (server is authoritative; this is just UX).
- [ ] Reuse `SettingRow`/`Field`/`inputClass`. Match the dark theme.
- [ ] Tests (RTL, mocked `useProfile`): renders current name/height/avatar;
  editing name calls `update`; oversized file shows error and skips upload;
  remove calls `removeAvatar`; height displays converted unit.
- [ ] Verify: `npm run test -- settings`.

## Task D5 — Sidebar account anchor + menu (+ tests)

Files: `components/sidebar.tsx`, `components/sidebar.test.tsx` (create if
absent), possibly a small `components/account-menu.tsx`.

- [ ] Replace the bare bottom Sign-out button with an **account row** built from
  `useProfile()`: avatar (resolved `avatar_url` or initials/placeholder) +
  `display_name` (soft-truncate to fit; `title` attr holds the full name).
  Clicking the row opens a small popover menu containing **Sign out** (calls the
  existing `logout()` — `clearToken()` + `router.replace("/login")`). Keep the
  menu open/close accessible (Esc, click-outside, `aria-expanded`).
- [ ] When the sidebar is **collapsed**, the row collapses to just the avatar
  icon; the menu still opens from it.
- [ ] Keep the Settings nav item as the entry point for editing; the anchor is
  identity + session actions only.
- [ ] Tests (RTL): renders display name + avatar; clicking opens menu with Sign
  out; Sign out calls clearToken/router; collapsed state renders avatar only and
  menu still opens. Provide `useProfile` via mock.
- [ ] Verify: `npm run test -- sidebar`.

## Task D6 — Chat payload: send `display_name` + `height_cm`

Files: `app/(app)/chat/page.tsx`, `app/(app)/chat/page.test.tsx`.

- [ ] In the `/chat` fetch body (next to `client_timezone`), add
  `display_name: profile?.display_name` and `height_cm: profile?.height_cm ??
  null`, sourced from `useProfile()`. Omit/null-safe when profile not loaded.
- [ ] Test: chat request body includes the two fields when a profile is present.
- [ ] Verify: `npm run test -- chat`, `npm run lint`, `npm run typecheck`,
  `npm run build`.

---

# Part E — prog-strength-docs (status flip)

- [ ] On `feat/user-profile-and-preferences`, edit
  `sows/user-profile-and-preferences.md`: frontmatter `status: shipped`; body
  `**Status**: Shipped`; body `**Last updated**: 2026-06-10`. Commit
  `docs: mark user-profile-and-preferences as shipped`.

---

## Per-repo verification gates (run before opening each PR)

- API: `go build ./...` && `go test ./...` && `gofmt -l` (clean) && lint.
- Infra: `terraform fmt -recursive` (clean) && `terraform validate` (module-level
  `-backend=false` if no cloud creds; document if skipped).
- Agent: `uv run pytest tests/` && `uv run ruff check src/ tests/`.
- Web: `npm run test` && `npm run lint` && `npm run typecheck` && `npm run build`.

## Rollout / deployment order (for the docs PR)

1. **`prog-strength-infra`** — creates the `prog-strength-avatars` bucket + IAM +
   lifecycle. Apply first: until the bucket exists and the instance role can
   write to it, the API's `POST /me/avatar` would fail. (API is backward-safe
   without it — avatar endpoints just return a clear error when
   `AVATAR_BUCKET_NAME` is unset — so infra and API can technically merge in
   parallel, but the bucket must be **deployed** before avatar upload is used.)
2. **`prog-strength-api`** — migration + endpoints. Additive; no client calls the
   new routes yet, so safe to deploy before web. Set `AVATAR_BUCKET_NAME` in the
   host `.env` (from the infra output) when deploying.
3. **`prog-strength-agent`** — accepts new optional `ChatRequest` fields; ignores
   them if the web client hasn't shipped. Safe any time after API (it doesn't
   depend on API for this change, but logically groups here).
4. **`prog-strength-web`** — the only caller of the new endpoints + the new chat
   fields. Deploy last, after API is live (else Settings save / avatar upload
   4xx) and after agent is live (else the new chat fields are silently dropped —
   non-fatal, but name/height won't reach the prompt).
5. **`prog-strength-docs`** — merging flips the SOW to `shipped`.

### Verification after rollout (hand-test)
- Open Settings → change display name → reload → name persists and shows in the
  sidebar account row; ask the agent "what's my name?" → it uses the new name.
- Set height in your unit → reload → persists and displays in the right unit;
  ask the agent your height → it answers without volunteering BMI.
- Upload an avatar → appears in the sidebar anchor and Settings preview; reload
  (within the hour) still renders. Remove it → reverts to OAuth avatar (or
  placeholder).
- Collapse the sidebar → account row collapses to the avatar; clicking still
  opens the menu; Sign out works.
- Invalid input: 3 MB image / a PDF / a 200-char name → rejected with a clear
  error.
