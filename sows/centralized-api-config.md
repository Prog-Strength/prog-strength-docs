---
status: draft
repos:
  - prog-strength-api
  - prog-strength-infra
  - prog-strength-docs
---

# Centralized API Configuration

**Status**: Draft · **Last updated**: 2026-06-20

## Introduction

`prog-strength-api`'s configuration has outgrown its current shape, and it's scattered across repos. Knobs are environment variables, read one-by-one in `internal/config/config.go`, *set* in three different places — the deploy workflow (`prog-strength-api/.github/workflows/release.yml`, which writes `.env` from `secrets.*`), the infra compose file (`prog-strength-infra/compose/api/docker-compose.yml`), and Terraform — and there is no single place to answer "what is configurable, what is it set to, and which of these is actually a secret?" The seams are already showing:

- **Values are set in more than one repo.** The deploy workflow injects ~20 values into `.env` from GitHub secrets; the compose file independently hardcodes defaults for some of the same concerns (`DAILY_USAGE_CAP_USD=${DAILY_USAGE_CAP_USD:-0.67}`, `LOG_LEVEL=${LOG_LEVEL:-debug}`). Two repos, overlapping responsibility, no single owner.
- **Non-secrets are stored as secrets.** `GOOGLE_REDIRECT_URL`, `GOOGLE_CALENDAR_REDIRECT_URL`, `CORS_ALLOWED_ORIGIN`, `RETURN_TO_ALLOWED_ORIGINS`, and even `DEV_AUTH` (a boolean toggle) are GitHub secrets today. None of them are sensitive. They bloat the secret list and obscure which secrets actually matter.
- **Public tunables are smuggled into compose.** The `DAILY_USAGE_CAP_USD` comment in the compose file literally explains it's "hardcoded here, not forwarded from a GitHub secret, because the value is a public tunable and version control gives us a reviewable history." The instinct is right; the compose file is the wrong home.
- **Config has leaked out of the config package.** `TCX_BUCKET_NAME` is read by a bare `os.Getenv` in `internal/server/server.go`, never passing through `config.Config`.

This work centralizes all of it, owned by the service that uses it. A single, documented, version-controlled `config.toml` in `prog-strength-api` becomes the one place that enumerates every configuration parameter the API has. **The file is 100% publicly viewable**: tuning knobs and non-sensitive config live there as literals, tuned through reviewable commits. Real secrets — API keys, AWS credentials, signing/encryption keys, OAuth client secrets — never appear; they stay GitHub org/repo secrets and surface in the file only as `${VAR}` *labels* (so the manifest is complete) whose values come from the environment at runtime. A small set of values is genuinely owned by Terraform (it provisions the underlying resource); those also stay env-sourced. After this ships, each config value has exactly one owner, the GitHub secret list shrinks to actual secrets, and "what can I configure?" has a single honest answer.

This is also the foundation **Agent Vector Memory** (`sows/agent-vector-memory.md`) builds on: that feature registers a `[vectormemory]` section here rather than introducing its own config file.

## Proposed Solution

### Single owner per value

Every configuration value gets exactly one home, decided by what kind of value it is:

- **Non-secret app / feature config** (tuning knobs, log level, caps, toggles, paths, public app URLs) → **`config.toml` in `prog-strength-api`, as a public literal.** The service that uses the value owns the value. This is the bulk of the migration and the answer to "couple config with the API."
- **True secret** (API keys, AWS credentials, signing keys, AES keys, OAuth client secrets) → **GitHub org/repo secret**, forwarded via env, referenced in `config.toml` as `"${VAR}"` (a label, never a value). Unchanged delivery; the file just documents that the key exists.
- **Infra-provisioned identity** (S3 bucket names, AWS region — Terraform *creates* the resource) → **Terraform-owned**, delivered via env exactly as today, referenced in `config.toml` as `"${VAR}"`. Hardcoding these as literals in the API repo would re-duplicate a value Terraform already owns, so we deliberately don't.

The principle that breaks ties: **a value lives with whoever is its source of truth.** Feature behavior → the API. A provisioned resource's identity → Terraform. A credential → the secret store. No value is *set* in two repos.

### Mechanics

A single committed `config.toml` at the `prog-strength-api` repo root, embedded via `go:embed` (the manifest always ships with the binary; zero-setup common path), with an optional `CONFIG_FILE` env var pointing at an external file to override the embedded default wholesale for emergencies that can't wait for a rebuild.

The loader resolves in a clear precedence order: parse the TOML → substitute `${VAR}` interpolations from the environment → overlay explicit env-var overrides for known keys (env always wins, generalizing today's ad-hoc "emergency `.env` tweak without a rebuild" to *every* knob) → apply defaults for unset optionals → fail startup on any unset required value (the JWT signing key; AWS region when a bucket is set). It produces the **same `config.Config` struct the codebase already consumes**, so this changes the *source* of configuration, not how the rest of the API reads it — blast radius is `internal/config`, the one stray read, and the deploy/compose wiring.

On the delivery side, both the deploy workflow and the compose file stop setting non-secret app config. The workflow's `.env` generation shrinks to **true secrets + Terraform-owned identities only**; the compose file forwards only those same `${VAR}` values plus deploy-orchestration vars (image tag, ECR registry, Litestream replica settings) that were never API app-config to begin with.

Per the owner's call (sole user, pre-launch), the migration is **all-at-once**.

## Configuration Inventory

The complete classification from the infra/workflow audit. "Port" = becomes a public literal in `config.toml`; "Secret" = stays a GitHub secret, `${VAR}` label in the file; "Terraform" = stays env-sourced from Terraform output, `${VAR}` label in the file.

### Port to `config.toml` as public literals (non-secret app config)

| Key | Today | New default in `config.toml` | Notes |
| --- | --- | --- | --- |
| `DATABASE_URL` | compose literal | `/data/app.db` | Container path; override via env for local dev. |
| `TELEMETRY_DATABASE_URL` | derived | *(empty → sibling `telemetry.db`)* | Keep the derive-from-`DATABASE_URL` default. |
| `SERVER_ADDR` | compose literal `:8080` | `:8080` | |
| `DEV_AUTH` | **GitHub secret** | `false` | Boolean toggle — never was a secret. Local dev sets env `DEV_AUTH=true`. |
| `LOG_LEVEL` | compose default `debug` | `info` | Public tunable; was smuggled into compose. |
| `DAILY_USAGE_CAP_USD` | compose default `0.67` | `0.67` | Public tunable; was smuggled into compose. |
| `USAGE_PRICE_TABLE_JSON` | env (optional) | `""` | Empty = built-in price table. Override escape hatch. |
| `GOOGLE_REDIRECT_URL` | **GitHub secret** | prod URL literal | Public OAuth callback URL — not a secret. |
| `GOOGLE_CALENDAR_REDIRECT_URL` | **GitHub secret** | prod URL literal | Public OAuth callback URL — not a secret. |
| `CORS_ALLOWED_ORIGIN` | **GitHub secret** | prod origins (TOML array) | Public domains + Vercel wildcard; not a secret. |
| `RETURN_TO_ALLOWED_ORIGINS` | **GitHub secret** | prod origins (TOML array) | Public domains; not a secret. |

Moving the five **bold** rows out of GitHub secrets is the direct payoff for the "too many secrets" problem.

### Keep as GitHub secrets (true secrets — `${VAR}` label only)

`JWT_SIGNING_KEY` (required), `GOOGLE_CLIENT_SECRET`, `CALENDAR_TOKEN_ENC_KEY`, `FATSECRET_CLIENT_ID`, `FATSECRET_CLIENT_SECRET`, `USDA_FDC_API_KEY`, plus the new `OPENAI_API_KEY` and `ANTHROPIC_API_KEY`. `GOOGLE_CLIENT_ID` is not strictly secret but is a deployment-bound OAuth-app identifier — kept env-sourced (`${VAR}`) rather than committed.

### Keep Terraform-owned (`${VAR}` label only — must NOT be a literal here)

`AVATAR_BUCKET_NAME`, `TCX_BUCKET_NAME` (read by the API), and `AWS_REGION`. Terraform provisions the buckets and defines the region (`variables.tf` / `prod.tfvars`, exposed as outputs); committing the names as literals in the API repo would duplicate Terraform's source of truth. They flow via env as today.

### Explicitly out of scope (not API app-config)

These appear in the audit but are **not** read by `config.Config` — they are deploy-orchestration, the Litestream sidecar, or the monitoring stack, and stay where they are: `ECR_REGISTRY`, `APP_VERSION`, `LITESTREAM_REPLICA_BUCKET`, `LITESTREAM_REPLICA_REGION`, `GRAFANA_ADMIN_USER`, `GRAFANA_ADMIN_PASSWORD`.

### Open judgment call — PII, not committed

`ADMIN_EMAILS` and `BETA_ALLOWED_EMAILS` are real email addresses. They aren't credentials, but committing them to a public repo is a privacy regression, and you asked that the file be publicly viewable — so they are **kept env-sourced (`${VAR}`), not ported to a literal.** (`BETA_ALLOWED_EMAILS` is already on its way out, migrating to a SQLite table per `sows/`'s beta-allowlist decision.) If you'd rather these live in the file, that's a one-line change — flagged in Open Questions.

### Duplication the audit found (eliminated or noted)

- **Compose default *and* workflow secret for the same concern** (`DAILY_USAGE_CAP_USD`, `LOG_LEVEL`, the mis-filed "secret" URLs/toggle): eliminated — these become single-owner literals in `config.toml`, removed from both compose and the workflow `.env`.
- **`AWS_REGION` / region hardcoded `us-east-2` in ~5 places** (Terraform, `release.yml`, `manual-deploy.yml`, `apply.yml`, compose defaults): mostly infra-internal orchestration, **out of scope** for this SOW, but noted — the API-consumed `AWS_REGION` stays Terraform-owned; collapsing the broader region duplication is a separate infra cleanup.
- **Bucket names in `variables.tf` defaults *and* `prod.tfvars`**: Terraform-internal, out of scope; noted for an infra follow-up.

## Implementation Details

### The `config.toml` Manifest

The committed file is the operator's single reference. Annotated example (proposed defaults shown; `${VAR}` denotes env-sourced secrets / Terraform-owned identities / PII that are never literals in this public repo):

```toml
# prog-strength-api configuration — single source of truth.
#
# This file is PUBLIC. It contains tuning knobs and non-sensitive config
# ONLY. It never contains secrets.
#   • literal value  → public, version-controlled config. Edit + review + deploy.
#   • "${ENV_VAR}"   → supplied by the environment at runtime. Used for
#                      secrets (GitHub secrets), Terraform-owned resource
#                      identities, and PII. A label here, never a value.
#
# Resolution precedence (highest wins):
#   1. explicit env override for a key (emergency tuning without a rebuild)
#   2. ${VAR} interpolation from the environment
#   3. the literal value below
#   4. built-in default / required-missing is a startup error
#
# CONFIG_FILE may point at an external file to replace this embedded default.

[server]
addr = ":8080"
# Mounts POST /auth/dev/token. Local/dev only — MUST be false in production.
# Not a secret; local dev overrides with env DEV_AUTH=true.
dev_auth = false

[database]
# Container path to the app SQLite DB. Local dev overrides via env.
url = "/data/app.db"
# Empty → sibling telemetry.db next to `url` (e.g. /data/telemetry.db).
telemetry_url = ""

[logging]
# debug | info | warn | error. Public tunable (was a compose default).
level = "info"

[auth]
# HMAC secret for signing/verifying JWTs. REQUIRED — secret → env.
jwt_signing_key = "${JWT_SIGNING_KEY}"
# Operator allowlist gating /admin/*. PII → env (NOT committed). Empty =
# admin surface fail-closed (every /admin route 403s).
admin_emails = "${ADMIN_EMAILS}"

[auth.google]
# OAuth client id: deployment-bound identifier (not a hard secret) → env.
client_id = "${GOOGLE_CLIENT_ID}"
# OAuth client secret → secret → env.
client_secret = "${GOOGLE_CLIENT_SECRET}"
# Public OAuth callback URLs — NOT secrets. Prod literals; local dev
# overrides via env. Empty client_id/secret/redirect ⇒ Google login routes
# are not mounted (local-only iteration with dev_auth).
login_redirect_url = "https://api.progstrength.fitness/auth/google/callback"
calendar_redirect_url = "https://api.progstrength.fitness/auth/google/calendar/callback"
# base64 32-byte AES-256-GCM key encrypting stored Google refresh tokens.
# Secret → env. Empty disables Google Calendar sync.
calendar_token_enc_key = "${CALENDAR_TOKEN_ENC_KEY}"

[cors]
# Browser origins permitted credentialed access. Public domains — NOT
# secrets. Each entry may contain a SINGLE "*" wildcard (Vercel previews).
# Empty disables CORS. Local dev appends localhost via env override.
allowed_origins = [
  "https://progstrength.fitness",
  "https://prog-strength-web-*-<vercel-scope>.vercel.app",
]
# Origins /auth/google/login may redirect back to via ?return_to. Public.
return_to_allowed_origins = [
  "https://app.progstrength.fitness",
  "https://prog-strength-web-*-<vercel-scope>.vercel.app",
]

[beta]
# One-time seed for the DB-backed beta allowlist. PII → env (NOT committed).
# Slated for removal once the SQLite-backed allowlist fully lands.
seed_allowed_emails = "${BETA_ALLOWED_EMAILS}"

[storage]
# S3 buckets are provisioned by Terraform — it owns these names. Env-sourced
# from Terraform outputs; NOT committed as literals (would duplicate TF).
# Empty → in-memory fallback (uploads vanish on restart; DB rows survive).
avatar_bucket_name = "${AVATAR_BUCKET_NAME}"
tcx_bucket_name = "${TCX_BUCKET_NAME}"   # folded in from the former stray os.Getenv
# AWS region for the S3 clients. Terraform-owned; REQUIRED when a bucket is set.
aws_region = "${AWS_REGION}"

[usage]
# Per-user daily external-API spend ceiling (USD). 0 = disabled. Public
# tunable — moved out of compose. ~$20/user/month at 0.67.
daily_cap_usd = 0.67
# Optional override of usage.DefaultPriceTable (emergency escape hatch).
# Empty = built-in defaults. Public rates live in source as reviewable diffs.
price_table = ""

[nutrition_lookup]
# Provider credentials → secrets → env. Empty ⇒ provider skipped and
# GET /nutrition/lookup degrades to 503/agent estimation.
fatsecret_client_id = "${FATSECRET_CLIENT_ID}"
fatsecret_client_secret = "${FATSECRET_CLIENT_SECRET}"
usda_fdc_api_key = "${USDA_FDC_API_KEY}"

[vectormemory]
# Seeds the Agent Vector Memory feature (sows/agent-vector-memory.md).
# Master kill-switch: false ⇒ no distillation, no retrieval, no behavior change.
enabled = true
# Secrets → env.
openai_api_key = "${OPENAI_API_KEY}"
anthropic_api_key = "${ANTHROPIC_API_KEY}"
# Tuning knobs (public). distance_threshold / dedup_threshold are TBD — to be
# calibrated empirically via the probe tooling before retrieval goes live.
distance_threshold = 0.0
dedup_threshold = 0.0
top_k = 5
session_idle_minutes = 30
distill_model = "claude-haiku-4-5-20251001"
embed_model = "text-embedding-3-small"
embed_dim = 1536
```

### The Loader (`internal/config`)

`config.Load()` is rewritten; `config.Config` (the struct downstream code consumes) keeps its current fields — plus new `TCXBucketName`, `AWSRegion`, and the `vectormemory` settings — so existing consumers are untouched. Flow:

1. **Read the manifest.** Decode the embedded `config.toml` (via `go:embed`) with `github.com/pelletier/go-toml/v2` into an intermediate typed struct mirroring the sections. If `CONFIG_FILE` is set, read that path instead.
2. **Interpolate.** For any string field whose value matches `${VAR}`, substitute `os.Getenv("VAR")`. A `${VAR}` resolving to empty is treated as "unset" (optional features degrade exactly as today; required values trip step 4).
3. **Env override.** For every mapped key, a conventional env var of the corresponding name overrides the file value — generalizing today's `${DAILY_USAGE_CAP_USD:-0.67}` escape hatch to all knobs, and giving local dev a clean override path for `dev_auth`, `database.url`, CORS origins, etc.
4. **Defaults + validation.** Apply derived defaults (`server.addr` → `:8080`; `database.telemetry_url` → sibling path when empty) and error on required-missing (`auth.jwt_signing_key`; `storage.aws_region` when a bucket is set). Validate `logging.level`.
5. **Map to `Config`.** List fields accept either a native TOML array (literals like CORS origins) or a comma-separated string (from `${VAR}` interpolation, e.g. `ADMIN_EMAILS`), so `splitCSV` survives only as the string-list normalizer.

The stray `os.Getenv("TCX_BUCKET_NAME")` in `internal/server/server.go` is removed; `server.New` reads `cfg.TCXBucketName`.

### Delivery: deploy workflow + infra

**`prog-strength-api/.github/workflows/release.yml` (and `manual-deploy.yml`):** the `.env` generation drops every value now owned by `config.toml` — `DEV_AUTH`, `GOOGLE_REDIRECT_URL`, `GOOGLE_CALENDAR_REDIRECT_URL`, `CORS_ALLOWED_ORIGIN`, `RETURN_TO_ALLOWED_ORIGINS`, and the compose-defaulted tunables — and writes only true secrets (`JWT_SIGNING_KEY`, `GOOGLE_CLIENT_SECRET`, `CALENDAR_TOKEN_ENC_KEY`, `FATSECRET_*`, `USDA_FDC_API_KEY`, `GOOGLE_CLIENT_ID`, plus new `OPENAI_API_KEY` / `ANTHROPIC_API_KEY`) and Terraform-owned identities (`AVATAR_BUCKET_NAME`, `TCX_BUCKET_NAME`, `AWS_REGION`, plus orchestration vars). The corresponding GitHub secrets for the now-ported non-secrets can be deleted after the change lands.

**`prog-strength-infra/compose/api/docker-compose.yml`:** the api service's `environment:` block drops `DATABASE_URL`, `SERVER_ADDR`, `DEV_AUTH`, `DAILY_USAGE_CAP_USD`, `LOG_LEVEL`, `CORS_ALLOWED_ORIGIN`, `RETURN_TO_ALLOWED_ORIGINS`, `GOOGLE_REDIRECT_URL`, `GOOGLE_CALENDAR_REDIRECT_URL` (all now in `config.toml`), keeps forwarding the secrets + Terraform-owned `${VAR}` values, and keeps the Litestream/ECR/`APP_VERSION` orchestration untouched. No Terraform changes.

### Tests

| Repo | Coverage |
| --- | --- |
| `prog-strength-api` | Loader unit tests against in-memory TOML: literals decode to typed fields; `${VAR}` interpolation pulls from env and treats empty as unset; explicit env override beats a file literal (`t.Setenv`); required-missing (`jwt_signing_key`) errors; derived defaults apply; `logging.level` validation rejects garbage; list fields accept both TOML arrays and comma-separated `${VAR}` strings; `CONFIG_FILE` external override replaces the embedded default. Golden test: the committed `config.toml` parses and yields the expected default `Config`. Regression: `TCXBucketName`/`AWSRegion` arrive via `Config` and the server wires S3 storage identically. |
| `prog-strength-infra` | Compose lints/parses; assertion (or documented manual check) that the api service no longer sets the ported keys and still forwards all secrets + Terraform-owned vars. |
| `prog-strength-docs` | This SOW; status transitions. |

### Rollout

Coordinated, all-at-once:

1. **API**: add `config.toml`, rewrite `internal/config`, remove the stray `os.Getenv`, add `go-toml/v2`, update `release.yml`/`manual-deploy.yml` `.env` generation.
2. **Infra**: trim the api compose `environment:` block.
3. Deploy API + infra together. Because the loader honors env overrides, a lingering ported var in `.env`/compose during the transition would harmlessly override the identical file default — no hard ordering hazard.
4. After verification, **delete the now-unused GitHub secrets** for the ported non-secrets (`DEV_AUTH`, `GOOGLE_REDIRECT_URL`, `GOOGLE_CALENDAR_REDIRECT_URL`, `CORS_ALLOWED_ORIGIN`, `RETURN_TO_ALLOWED_ORIGINS`).

Verify after deploy: `/health` green; usage cap reads through to behavior; admin surface still gates on `admin_emails`; Google login + CORS still work from the web app; nutrition lookup still configured. Roll back by reverting both PRs (immutable image tags).

## Open Questions

1. **Admin / beta emails — public or not?** Default in this SOW: keep `ADMIN_EMAILS` / `BETA_ALLOWED_EMAILS` env-sourced (PII, not committed to a public repo). If you'd accept them in the public file, they become literals — one-line change. (Beta emails are migrating to SQLite regardless.)
2. **Env-override key naming.** Preserve today's exact env var names (e.g. `DAILY_USAGE_CAP_USD` ↔ `[usage].daily_cap_usd`) so nothing in infra/secrets has to change — the override map is explicit, not derived. Confirm during implementation.
3. **`${VAR}` markers vs. pure env-overlay.** The design uses both (self-documenting markers + a general override layer); they overlap slightly. Acceptable to unify if it stays self-documenting.
4. **TOML library.** `github.com/pelletier/go-toml/v2` (leaning) vs. `BurntSushi/toml`. Minor; one small dep either way.
5. **Region-duplication cleanup.** `us-east-2` is hardcoded in ~5 infra places. Out of scope here; worth a dedicated infra follow-up to collapse to one source.
6. **Should `agent` / `mcp` adopt the same pattern later?** Out of scope; obvious follow-up template if this lands well.
