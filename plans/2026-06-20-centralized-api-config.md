# Plan: Centralized API Configuration

Implements `sows/centralized-api-config.md`. Source of truth for every
`prog-strength-api` configuration value becomes a single committed,
publicly-viewable `config.toml`, owned by the service that uses it.
Secrets, Terraform-owned resource identities, and PII stay env-sourced
and appear in the file only as `${VAR}` labels.

## Repos touched

- `prog-strength-api` — the manifest, the loader rewrite, the stray
  `os.Getenv` fold-in, and the deploy-workflow `.env` trim.
- `prog-strength-infra` — trim the api compose `environment:` block.
- `prog-strength-docs` — flip the SOW status (workflow step 4).

## Key grounding facts (verified against the repos)

- CI pins `golangci-lint v2.12.2` (`.github/workflows/ci.yml`); go 1.25.11.
- `gosec` is enabled and **G304 is not excluded** — reading the optional
  `CONFIG_FILE` path with a bare `os.ReadFile(path)` trips G304. Verified
  fix without any `//nolint`: `os.ReadFile(filepath.Clean(path))` passes.
- The real Vercel preview scope is `jimmy-wallaces-projects`
  (`internal/originmatch/originmatch_test.go`), not the SOW's
  `<vercel-scope>` placeholder. Use the real scope in the manifest.
- The deploy workflows already write `AWS_REGION=us-east-2` as an `.env`
  literal (not from a secret) and do **not** currently set
  `DAILY_USAGE_CAP_USD` / `LOG_LEVEL` / `USAGE_PRICE_TABLE_JSON` (those
  come from compose defaults). `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` are
  not yet wired anywhere.
- `ADMIN_EMAILS` and `BETA_ALLOWED_EMAILS` are PII → stay env-sourced
  (`${VAR}`), so they remain forwarded by the workflows and compose.

## Task A — `prog-strength-api`

### A1. Manifest + dependency

- Add `github.com/pelletier/go-toml/v2` (SOW-mandated; AGENTS.md's
  "ask before adding deps" is satisfied by the SOW).
- Create `config.toml` at the repo root, matching the SOW's annotated
  example, with these corrections to literals:
  - `cors.allowed_origins` / `cors.return_to_allowed_origins` use the
    real scope `prog-strength-web-*-jimmy-wallaces-projects.vercel.app`.
  - Keep all `${VAR}` labels exactly (secrets, TF-owned, PII).

### A2. Loader rewrite (`internal/config`)

Resolution order: decode embedded TOML → interpolate `${VAR}` from env
(empty = unset) → explicit env override for known literal keys (env wins)
→ defaults → validate → map to `config.Config`.

- `go:embed config.toml` (the embed lives in the `config` package; copy
  or embed the root file — embed path must be within the package dir, so
  embed a package-local copy is not allowed; instead read the root file
  via embed from `main`? **Decision:** place the `//go:embed config.toml`
  directive in `package config` and keep `config.toml` at repo root —
  `go:embed` cannot reach a parent dir. Therefore embed from the repo
  root requires the embed directive in a root-level package. The repo's
  root package is `cmd/api` (package main) — not `config`. **Resolution:**
  put the embed in a new tiny file at repo root is impossible (no root Go
  package). So: keep the canonical `config.toml` at repo root for operator
  visibility AND embed it. `go:embed` only sees files in/under the
  directive's package dir. Cleanest: the embed directive goes in the
  `config` package and the embedded file must live under `internal/config/`.
  **Final decision:** commit `config.toml` at the repo root as the
  operator-facing manifest, and have `internal/config` embed it via a
  build step is overkill. Instead embed from `cmd/api` (package main,
  repo root) where `//go:embed config.toml` is legal, and pass the bytes
  into `config.Load(embeddedTOML)`. `Load` takes the raw bytes (or a
  default) so tests can pass in-memory TOML. This keeps the file at the
  repo root, embeds it from the one root-level package, and makes the
  loader trivially testable.)
- `config.Load(defaultTOML []byte) (Config, error)` — signature changes
  to accept the embedded bytes; `cmd/api/main.go` passes the embed var.
  If `CONFIG_FILE` is set, read `os.ReadFile(filepath.Clean(path))`
  instead of `defaultTOML`.
- Intermediate `fileConfig` struct mirroring the TOML sections (toml
  tags), then map to the flat `Config`.
- `${VAR}` interpolation: a value matching `^\$\{[A-Z0-9_]+\}$` resolves
  to `os.Getenv` of that name (empty → "").
- Explicit env-override map (preserve today's exact env names — SOW Open
  Q2): `DATABASE_URL`, `TELEMETRY_DATABASE_URL`, `SERVER_ADDR`,
  `DEV_AUTH`, `LOG_LEVEL`, `DAILY_USAGE_CAP_USD`, `USAGE_PRICE_TABLE_JSON`,
  `GOOGLE_REDIRECT_URL` → `login_redirect_url`,
  `GOOGLE_CALENDAR_REDIRECT_URL` → `calendar_redirect_url`,
  `CORS_ALLOWED_ORIGIN` → `allowed_origins`,
  `RETURN_TO_ALLOWED_ORIGINS` → `return_to_allowed_origins`. Env wins over
  the file literal when set (non-empty).
- List fields (`allowed_origins`, `return_to_allowed_origins`) decode
  from either a native TOML array OR a CSV string (custom `stringList`
  type with `UnmarshalTOML`). `splitCSV` survives as the CSV normalizer
  for env overrides and the `${VAR}`-string list fields (`ADMIN_EMAILS`,
  `BETA_ALLOWED_EMAILS`).
- New `Config` fields: `TCXBucketName`, `AWSRegion`, and a nested
  `VectorMemory` struct (Enabled, OpenAIAPIKey, AnthropicAPIKey,
  DistanceThreshold, DedupThreshold, TopK, SessionIdleMinutes,
  DistillModel, EmbedModel, EmbedDim). Existing fields unchanged.
- Defaults: `server.addr` → `:8080`; `database.telemetry_url` empty +
  `database.url` set → sibling `telemetry.db` (keep `deriveTelemetryPath`).
- Validation: `jwt_signing_key` required; `aws_region` required when
  `avatar_bucket_name` or `tcx_bucket_name` is set; `logging.level`
  validated via `slog.Level.UnmarshalText`.

### A3. Fold in the stray read (`internal/server/server.go`)

- Remove `os.Getenv("TCX_BUCKET_NAME")`; read `cfg.TCXBucketName`. Drop
  the now-unused `os` import if nothing else uses it.
- S3 wiring stays identical (clients still resolve region via the AWS
  default chain / `AWS_REGION` env). `AWSRegion` in `Config` drives the
  startup validation; it is not threaded into the SDK clients.

### A4. Deploy workflows

`release.yml` and `manual-deploy.yml`: the `.env` heredoc and the env/
`envs:` lists drop `GOOGLE_REDIRECT_URL`, `GOOGLE_CALENDAR_REDIRECT_URL`,
`DEV_AUTH`, `CORS_ALLOWED_ORIGIN`, `RETURN_TO_ALLOWED_ORIGINS`; add
`OPENAI_API_KEY` and `ANTHROPIC_API_KEY` (true secrets). Keep
`JWT_SIGNING_KEY`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`,
`CALENDAR_TOKEN_ENC_KEY`, `FATSECRET_*`, `USDA_FDC_API_KEY` (secrets);
`ADMIN_EMAILS`, `BETA_ALLOWED_EMAILS` (PII); `TCX_BUCKET_NAME`,
`AVATAR_BUCKET_NAME`, `AWS_REGION` (TF-owned); `LITESTREAM_*`,
`GRAFANA_*`, `APP_VERSION`, `ECR_REGISTRY` (orchestration).

### Tests (A2)

Loader unit tests (in-memory TOML via the new `Load([]byte)` signature):
literals decode to typed fields; `${VAR}` interpolation from env, empty =
unset; explicit env override beats a file literal (`t.Setenv`);
required-missing `jwt_signing_key` errors; `aws_region` required when a
bucket is set; derived defaults apply; `logging.level` rejects garbage;
list fields accept both TOML arrays and CSV `${VAR}` strings; `CONFIG_FILE`
external override replaces the embedded default. Golden test: the
committed `config.toml` parses and yields the expected default `Config`
(reference every new field so `unused` stays quiet). Keep/port the
existing `TestCORSAllowedOrigins` behavior under the new loader.

## Task B — `prog-strength-infra`

Trim the api service `environment:` block in
`compose/api/docker-compose.yml`: drop `DATABASE_URL`, `SERVER_ADDR`,
`DEV_AUTH`, `DAILY_USAGE_CAP_USD`, `LOG_LEVEL`, `CORS_ALLOWED_ORIGIN`,
`RETURN_TO_ALLOWED_ORIGINS`, `GOOGLE_REDIRECT_URL`,
`GOOGLE_CALENDAR_REDIRECT_URL` (now owned by `config.toml`). Keep the
secrets, PII (`ADMIN_EMAILS`, `BETA_ALLOWED_EMAILS`), TF-owned
(`TCX_BUCKET_NAME`, `AVATAR_BUCKET_NAME`, `AWS_REGION`) `${VAR}` forwards,
and the Litestream/ECR/`APP_VERSION` orchestration untouched. Update the
explanatory comments accordingly. No Terraform changes. Validate with
`docker compose config` (or document the manual check if the daemon is
unavailable) and `pre-commit run`/`tflint`.

## Task C — `prog-strength-docs` (workflow step 4)

Flip `sows/centralized-api-config.md`: frontmatter `status: shipped`;
body `**Status**: Shipped`; `**Last updated**: 2026-06-20`. Plus this
plan file. Commit `docs: mark centralized-api-config as shipped`.

## Gate before pushing

Per repo `AGENTS.md`/`CONTRIBUTING.md`:
- api: `golangci-lint@v2.12.2 run`, `go vet ./...`, `go mod tidy` (no
  drift), `go test ./...`.
- infra: `pre-commit run --all-files` (fmt/shellcheck/tflint),
  `docker compose config`.
No `--no-verify`, no `//nolint`, no rule disables, no skipped tests.

## Rollout (for the docs PR body)

1. `prog-strength-api` — merge + deploy first: the new loader must own
   the config before infra stops forwarding it (env override makes a
   lingering var harmless, but the API is the source of truth now).
2. `prog-strength-infra` — merge after the API image is live; trims the
   compose env block.
3. After verifying, delete the now-unused GitHub secrets for the ported
   non-secrets.
