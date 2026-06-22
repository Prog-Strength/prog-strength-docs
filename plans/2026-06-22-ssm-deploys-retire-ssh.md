# SSM-Based Deploys: Retire the SSH Deploy Key — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move all seven production deploy workflows off `appleboy/ssh-action` onto AWS SSM Run Command authenticated by the existing OIDC role, move backend app secrets into infra-owned AWS Secrets Manager (seeded from GitHub, never in TF state, never on a deploy), close inbound port 22, and delete/scrub the `EC2_SSH_KEY` / `EC2_HOST` secrets.

**Architecture:** Infra (`prog-strength-infra`) gains a `secrets` module (one JSON secret container per service under `prog-strength-backend/<env>/<service>`), instance-role grants (`AmazonSSMManagedInstanceCore` + scoped `GetSecretValue`), OIDC-role grants to create/seed those secrets, versioned on-host deploy scripts (`deploy/*.sh`) that render `.env` from Secrets Manager via the instance role, a `seed-secrets.yml` workflow, and the SSH ingress rule removed. Each service repo's `release.yml`/`manual-deploy.yml` deploy step is rewritten to assume the OIDC role and `aws ssm send-command` the on-host script, polling to completion. Docs are scrubbed of the SSH secrets.

**Tech Stack:** Terraform 1.13.0 (AWS provider), GitHub Actions YAML, Bash, AWS CLI v2 (SSM, Secrets Manager, ECR), jq, Docker Compose v2.

---

## Conventions used throughout

- **Instance targeting:** `--targets "Key=tag:Name,Values=prog-strength-prod-backend"` (the instance's `Name` tag is `${name_prefix}-backend` where `name_prefix = prog-strength-prod`). A tag filter retires `EC2_HOST` and survives an instance replacement (Open Question 1, leaning tag filter).
- **Secrets run as `ubuntu`:** SSM commands execute as `root`; the infra checkout and Docker creds dir are owned by `ubuntu`. Every `send-command` invokes the on-host script via `sudo -u ubuntu -H bash …` so git ownership and `~/.docker/config.json` writes match the old SSH-as-`ubuntu` semantics exactly (avoids git "dubious ownership" and root-owned files).
- **Version arg:** the runner passes the version tag with a leading `v` (e.g. `v0.22.0`); each script normalizes with `v${1#v}`.
- **Secrets never on the wire:** only the version tag travels with the command. The host reads app secrets from Secrets Manager via its instance role.
- **Region:** `us-east-2` throughout.

## File Structure

`prog-strength-infra`
- Create: `modules/secrets/main.tf`, `modules/secrets/variables.tf`, `modules/secrets/versions.tf` — secret containers + instance-role `GetSecretValue`.
- Modify: `main.tf` (wire `secrets` module), `variables.tf` (`secrets` variable), `environments/prod.tfvars` (`secrets` block; remove SSH ingress), `modules/compute/iam.tf` (SSM core attachment), `modules/github_oidc/main.tf` (backend secrets statement), `modules/compute/bootstrap.sh` (jq + enable SSM agent), `README.md` (scrub EC2 secrets).
- Create: `deploy/api.sh`, `deploy/mcp.sh`, `deploy/agent.sh`, `deploy/caddy-reload.sh` — on-host deploy scripts.
- Create: `.github/workflows/seed-secrets.yml`.
- Modify: `.github/workflows/deploy-caddy.yml` (SSH → SSM).

`prog-strength-api` — Modify `.github/workflows/release.yml`, `.github/workflows/manual-deploy.yml` (SSH → SSM), `DEPLOYMENT.md` (scrub).
`prog-strength-mcp` — Modify `.github/workflows/release.yml`, `.github/workflows/manual-deploy.yml` (SSH → SSM), `README.md` (scrub).
`prog-strength-agent` — Modify `.github/workflows/release.yml`, `.github/workflows/manual-deploy.yml` (SSH → SSM); README deployment note.
`prog-strength-docs` — this plan + SOW status flip.

## Local gate per repo (run before commit/push — no `--no-verify`, no rule suppression)

- **infra:** `terraform fmt -check -recursive`, `terraform init -backend=false && terraform validate` (root + each touched module), `tflint --recursive --config="$PWD/.tflint.hcl"`, `shellcheck` on `modules/compute/bootstrap.sh` and `deploy/*.sh`, `actionlint` on changed workflows.
- **api/mcp/agent:** `actionlint` on changed workflows; markdown changes need no build. (No Go/Python source touched, so the language test suites are unaffected — but if the repo's `AGENTS.md`/`CONTRIBUTING.md` names a gate, run it to confirm green.)

---

## Task 1 — Infra Terraform + bootstrap (Checkpoint 1 IAM/secrets + Checkpoint 3 port close)

**Files:**
- Create: `modules/secrets/main.tf`, `modules/secrets/variables.tf`, `modules/secrets/versions.tf`
- Modify: `main.tf`, `variables.tf`, `environments/prod.tfvars`, `modules/compute/iam.tf`, `modules/github_oidc/main.tf`, `modules/compute/bootstrap.sh`

Work from: `/workspace/prog-strength-infra` on branch `feat/ssm-deploys-retire-ssh`.

- [ ] **Step 1: Create `modules/secrets/versions.tf`** (every module carries one — tflint enforces it)

```hcl
terraform {
  required_version = ">= 1.13.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}
```

(Match the provider version constraint used by the sibling modules — check `modules/ecr/versions.tf` and copy its exact `required_version`/`version` values rather than the placeholders above if they differ.)

- [ ] **Step 2: Create `modules/secrets/variables.tf`**

```hcl
variable "env" {
  description = "Environment segment in the secret name (e.g. prod). Encoded up front so a future environment is prog-strength-backend/staging/* with no rename of the prod secrets."
  type        = string
}

variable "services" {
  description = "Map of backend service name => the secret container's self-documenting description. Drives one aws_secretsmanager_secret per service via for_each, so a new service or environment is a one-line addition."
  type        = map(string)
}

variable "instance_role_name" {
  description = "Name of the EC2 instance IAM role to grant GetSecretValue on this environment's backend secrets. Mirrors the additive-attachment pattern used by the ecr/backup modules."
  type        = string
}
```

- [ ] **Step 3: Create `modules/secrets/main.tf`**

```hcl
# --- Backend runtime secrets (AWS Secrets Manager) --------------------------
#
# One JSON secret container per backend service, named
# prog-strength-backend/<env>/<service>. The prog-strength-backend/ prefix
# reads as clearly separate from prog-strength-developer/* (the autonomous
# developer's owner-private credentials); the /<env>/ segment means a future
# environment is prog-strength-backend/staging/* with no rename or IAM rework.
#
# Terraform owns the *containers*, their IAM, and (future) rotation policy —
# every backend resource stays in this repo. It deliberately does NOT own the
# *values*: there is no aws_secretsmanager_secret_version with real content,
# so plaintext never enters Terraform state (the same rule the developer
# repo's secrets.tf states explicitly). The values are seeded out-of-band
# from GitHub secrets by .github/workflows/seed-secrets.yml, and the host
# reads them at deploy time via its instance role. Hand-edits in the console
# are overwritten by the next seed — see each container's description.
#
# See prog-strength-docs/sows/ssm-deploys-retire-ssh.md.

data "aws_caller_identity" "current" {}

data "aws_region" "current" {}

locals {
  secret_name_prefix = "prog-strength-backend/${var.env}"
}

resource "aws_secretsmanager_secret" "backend" {
  for_each = var.services

  name        = "${local.secret_name_prefix}/${each.key}"
  description = each.value
}

# The host reads only its own environment's backend secrets via the instance
# role — never the developer family or a future environment. The default
# aws/secretsmanager KMS key needs no explicit kms:Decrypt for same-account
# use. Scoped to the /<env>/ path; the trailing /* matches the random suffix
# AWS appends to every secret ARN.
data "aws_iam_policy_document" "instance_secrets_read" {
  statement {
    sid       = "ReadBackendSecrets"
    effect    = "Allow"
    actions   = ["secretsmanager:GetSecretValue"]
    resources = ["arn:aws:secretsmanager:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:secret:${local.secret_name_prefix}/*"]
  }
}

resource "aws_iam_role_policy" "instance_secrets_read" {
  name   = "prog-strength-backend-${var.env}-secrets-read"
  role   = var.instance_role_name
  policy = data.aws_iam_policy_document.instance_secrets_read.json
}
```

Note: if `tflint`/`validate` flags `data.aws_region.current.name` as deprecated on the pinned provider, use `data.aws_region.current.region`. Verify against whichever attribute the rest of the repo uses; prefer the non-deprecated one.

- [ ] **Step 4: Add the `secrets` variable to `variables.tf`** (place it alphabetically/after the existing concern variables, matching the file's object-variable style)

```hcl
variable "secrets" {
  description = "Backend runtime secret containers (AWS Secrets Manager). One JSON secret per service under prog-strength-backend/<env>/<service>; values are seeded by seed-secrets.yml, never stored in Terraform state."
  type = object({
    services = map(string)
  })
}
```

- [ ] **Step 5: Wire the `secrets` module in `main.tf`** (append after the `ecr`/`logging` modules, before or after `github_oidc` — anywhere in the root composition; it attaches to the compute instance role like the other domain modules)

```hcl
module "secrets" {
  source = "./modules/secrets"

  env                = var.project.environment
  services           = var.secrets.services
  instance_role_name = module.compute.instance_role_name
}
```

- [ ] **Step 6: Add the `secrets` block to `environments/prod.tfvars`** (after the `ecr` block; the descriptions are the self-documenting strings the SOW prescribes)

```hcl
# --- Backend runtime secrets (Secrets Manager, infra-owned, GitHub-seeded) --

secrets = {
  # One JSON secret container per backend service under
  # prog-strength-backend/prod/<service>. Values are seeded from GitHub
  # secrets by .github/workflows/seed-secrets.yml and are never stored in
  # Terraform state. The description on each container documents its purpose
  # and infra-seeded origin so the console reader knows hand-edits get
  # overwritten by the next seed.
  services = {
    api   = "Backend prod app config for the api service. Values seeded from GitHub secrets by prog-strength-infra/seed-secrets.yml; not stored in Terraform state. Consumed by the api container's .env at deploy time."
    mcp   = "Backend prod app config for the mcp service. Values seeded from GitHub secrets by prog-strength-infra/seed-secrets.yml; not stored in Terraform state. Consumed by the mcp container's .env at deploy time."
    agent = "Backend prod app config for the agent service. Values seeded from GitHub secrets by prog-strength-infra/seed-secrets.yml; not stored in Terraform state. Consumed by the agent container's .env at deploy time."
  }
}
```

- [ ] **Step 7: Remove the SSH ingress rule from `environments/prod.tfvars`** (Checkpoint 3 — close port 22, keep 80/443). In the `compute.security_group.ingress_rules` list, delete the entire `{ description = "SSH" … }` object (the first element). Leave the HTTP (80) and HTTPS (443) objects intact. Add a brief comment above the list noting why 22 is gone:

```hcl
    # Inbound SSH (22) intentionally removed: deploys run via SSM Run Command
    # and operators break-glass via SSM Session Manager, both of which dial
    # OUT to AWS over 443 — no inbound port needed. See
    # prog-strength-docs/sows/ssm-deploys-retire-ssh.md.
    ingress_rules = [
      {
        description = "HTTP (Caddy ACME challenge + redirect)"
        protocol    = "tcp"
        from_port   = 80
        to_port     = 80
        cidr_blocks = ["0.0.0.0/0"]
      },
      {
        description = "HTTPS"
        protocol    = "tcp"
        from_port   = 443
        to_port     = 443
        cidr_blocks = ["0.0.0.0/0"]
      },
    ]
```

- [ ] **Step 8: Attach `AmazonSSMManagedInstanceCore` to the instance role in `modules/compute/iam.tf`** (append after the instance profile resource — this registers the host as an SSM managed node so deploys via Run Command and break-glass via Session Manager work without inbound SSH)

```hcl
# Register the host as an SSM managed node. This is what lets CI deploy via
# SSM Run Command and operators break-glass via Session Manager with NO
# inbound SSH — the agent dials out to AWS over 443. AWS-managed policy;
# mirrors the additive-attachment pattern the ecr/backup modules use against
# this same role. See prog-strength-docs/sows/ssm-deploys-retire-ssh.md.
resource "aws_iam_role_policy_attachment" "instance_ssm_core" {
  role       = aws_iam_role.api_instance.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}
```

- [ ] **Step 9: Add the backend Secrets Manager statement to `modules/github_oidc/main.tf`** (add a new `statement` block inside `data "aws_iam_policy_document" "permissions"`, immediately after the existing `SecretsManagerDescribe` statement, keeping the two secret families on distinct grants)

```hcl
  # Backend runtime secrets (prog-strength-backend/*). The apply pipeline
  # creates the containers (CreateSecret/TagResource) and seed-secrets.yml
  # writes their values (PutSecretValue/DescribeSecret). Scoped to
  # prog-strength-backend/* so the same role manages every environment's
  # backend secrets as they are added. Separate from the
  # prog-strength-developer/* Describe grant above so the two secret families
  # stay independently auditable. CI seeds values from GitHub; it never reads
  # them back (no GetSecretValue here — only the instance role reads values).
  statement {
    sid = "SecretsManagerBackendManage"
    actions = [
      "secretsmanager:CreateSecret",
      "secretsmanager:TagResource",
      "secretsmanager:PutSecretValue",
      "secretsmanager:DescribeSecret",
    ]
    resources = [
      "arn:aws:secretsmanager:${var.aws_region}:${data.aws_caller_identity.current.account_id}:secret:prog-strength-backend/*",
    ]
  }
```

- [ ] **Step 10: Edit `modules/compute/bootstrap.sh`** — two changes, both keeping `shellcheck` clean (do not broaden the existing `# shellcheck disable=SC2154,SC2034`):

  (a) Add `jq` to the operator-tooling install so the deploy scripts can render `.env` from the Secrets Manager JSON blob. Change the line `apt-get install -y sqlite3` to `apt-get install -y sqlite3 jq`, and extend the comment above it to mention jq is for the deploy scripts' Secrets Manager → `.env` render.

  (b) After the docker network creation block (`docker network create prog-strength …`), add an SSM-agent enable block:

```bash
# --- SSM agent (deploy transport + break-glass shell) -----------------------
#
# Deploys run through SSM Run Command and operators break-glass via SSM
# Session Manager — there is NO inbound SSH. Recent Ubuntu AMIs ship
# amazon-ssm-agent preinstalled via snap; enable + start it explicitly so a
# freshly-bootstrapped host registers as a managed node with no manual step.
# The instance role (AmazonSSMManagedInstanceCore, attached in
# modules/compute/iam.tf) is what authorizes registration. Best-effort across
# snap/systemd packagings so this never fails the bootstrap.
snap start amazon-ssm-agent 2>/dev/null \
  || systemctl enable --now amazon-ssm-agent 2>/dev/null \
  || true
```

- [ ] **Step 11: Run the infra gate and fix anything red**

```bash
cd /workspace/prog-strength-infra
terraform fmt -recursive
terraform fmt -check -recursive
terraform init -backend=false && terraform validate
( cd modules/secrets && terraform init -backend=false && terraform validate )
tflint --recursive --config="$PWD/.tflint.hcl"
shellcheck modules/compute/bootstrap.sh
```
Expected: `fmt -check` exits 0 (formatted), `validate` "Success!", `tflint` no issues, `shellcheck` no findings.

- [ ] **Step 12: Commit**

```bash
git add modules/secrets main.tf variables.tf environments/prod.tfvars \
  modules/compute/iam.tf modules/github_oidc/main.tf modules/compute/bootstrap.sh
git commit -m "feat(secrets): add Secrets Manager + SSM instance role, OIDC seed grants, close SSH ingress"
```

---

## Task 2 — Infra on-host deploy scripts

**Files:** Create `deploy/api.sh`, `deploy/mcp.sh`, `deploy/agent.sh`, `deploy/caddy-reload.sh`.

Work from `/workspace/prog-strength-infra`. Each script is `chmod +x`, `#!/usr/bin/env bash`, `set -euo pipefail`, `shellcheck`-clean. They reproduce the exact compose logic from the old SSH scripts; the only change is `.env` is rendered from Secrets Manager instead of forwarded secrets, and the version arrives as `$1`.

- [ ] **Step 1: Create `deploy/api.sh`**

```bash
#!/usr/bin/env bash
#
# On-host deploy for the api service. Invoked by prog-strength-api's
# release.yml / manual-deploy.yml via SSM Run Command (AWS-RunShellScript),
# replacing the old appleboy/ssh-action inline script. The released version
# tag is the only parameter the runner passes; all app secrets are read from
# Secrets Manager via the instance role and never transit the runner.
# Runs as the ubuntu user (the runner invokes `sudo -u ubuntu -H bash …`) so
# the infra checkout and ~/.docker creds keep their existing ownership.
# See prog-strength-docs/sows/ssm-deploys-retire-ssh.md.
set -euo pipefail

RELEASE_VERSION="${1:?usage: api.sh <version>  (e.g. v0.22.0 or 0.22.0)}"
RELEASE_VERSION="v${RELEASE_VERSION#v}" # normalize to a single leading v

AWS_REGION="us-east-2"
SECRET_ID="prog-strength-backend/prod/api"

cd /home/ubuntu/prog-strength-infra
git fetch --prune
git checkout main
git pull --ff-only

cd compose/api

# ECR login via the instance role (no static creds).
ECR_REGISTRY="$(aws sts get-caller-identity --query Account --output text).dkr.ecr.${AWS_REGION}.amazonaws.com"
aws ecr get-login-password --region "${AWS_REGION}" \
  | docker login --username AWS --password-stdin "${ECR_REGISTRY}"

# Render .env from Secrets Manager. The JSON blob's keys become KEY=value
# lines; absent optional providers (FatSecret, USDA, OpenAI/Anthropic) simply
# don't appear, so docker-compose's ${VAR:-} defaults apply and the endpoints
# degrade to 503 exactly as before. Deploy-orchestration values (region,
# version, registry) are non-secret and appended here, not stored in the blob.
umask 077
{
  aws secretsmanager get-secret-value \
    --secret-id "${SECRET_ID}" --region "${AWS_REGION}" \
    --query SecretString --output text \
    | jq -r 'to_entries[] | "\(.key)=\(.value)"'
  echo "AWS_REGION=${AWS_REGION}"
  echo "APP_VERSION=${RELEASE_VERSION}"
  echo "ECR_REGISTRY=${ECR_REGISTRY}"
} >.env

# Merge the monitoring stack so api + agent share the prog-strength network
# and Prometheus can scrape by service hostname (same as the SSH path).
COMPOSE_FILES=(-f docker-compose.yml -f /home/ubuntu/prog-strength-infra/monitoring/docker-compose.monitoring.yml)

# Pull the released image up-front so a missing/broken push surfaces before
# we tear down the running stack.
docker compose "${COMPOSE_FILES[@]}" pull api
docker compose "${COMPOSE_FILES[@]}" down
docker compose "${COMPOSE_FILES[@]}" up -d

echo "Deployed ${RELEASE_VERSION}"
docker compose "${COMPOSE_FILES[@]}" ps
docker compose "${COMPOSE_FILES[@]}" logs --tail=50
```

- [ ] **Step 2: Create `deploy/agent.sh`** (same shape; agent does NOT merge the monitoring compose file and pulls the `agent` service; secret id `prog-strength-backend/prod/agent`)

```bash
#!/usr/bin/env bash
#
# On-host deploy for the agent service. Invoked by prog-strength-agent's
# release.yml / manual-deploy.yml via SSM Run Command. Version tag is the only
# runner parameter; secrets (ANTHROPIC_API_KEY, OPENAI_API_KEY, JWT_SIGNING_KEY,
# CORS_ALLOWED_ORIGINS) come from Secrets Manager via the instance role. Runs
# as the ubuntu user. See prog-strength-docs/sows/ssm-deploys-retire-ssh.md.
set -euo pipefail

RELEASE_VERSION="${1:?usage: agent.sh <version>  (e.g. v0.1.0 or 0.1.0)}"
RELEASE_VERSION="v${RELEASE_VERSION#v}"

AWS_REGION="us-east-2"
SECRET_ID="prog-strength-backend/prod/agent"

cd /home/ubuntu/prog-strength-infra
git fetch --prune
git checkout main
git pull --ff-only

cd compose/agent

ECR_REGISTRY="$(aws sts get-caller-identity --query Account --output text).dkr.ecr.${AWS_REGION}.amazonaws.com"
aws ecr get-login-password --region "${AWS_REGION}" \
  | docker login --username AWS --password-stdin "${ECR_REGISTRY}"

# Optional OPENAI_API_KEY absent from the blob → /speak returns 503; the agent
# still boots and /chat + /title keep working (same degrade as before).
umask 077
{
  aws secretsmanager get-secret-value \
    --secret-id "${SECRET_ID}" --region "${AWS_REGION}" \
    --query SecretString --output text \
    | jq -r 'to_entries[] | "\(.key)=\(.value)"'
  echo "AWS_REGION=${AWS_REGION}"
  echo "APP_VERSION=${RELEASE_VERSION}"
  echo "ECR_REGISTRY=${ECR_REGISTRY}"
} >.env

docker compose pull agent
docker compose down
docker compose up -d

echo "Deployed ${RELEASE_VERSION}"
docker compose ps
docker compose logs --tail=50
```

- [ ] **Step 3: Create `deploy/mcp.sh`** (mcp currently holds no app secrets — its old `.env` was only deploy orchestration. It still reads its container for uniformity and future-proofing; the blob is seeded `{}`, so `jq` renders nothing and `.env` is byte-equivalent to the old one.)

```bash
#!/usr/bin/env bash
#
# On-host deploy for the mcp service. Invoked by prog-strength-mcp's
# release.yml / manual-deploy.yml via SSM Run Command. The mcp server holds no
# app secrets (auth comes from the agent's per-request Authorization header),
# so prog-strength-backend/prod/mcp is currently an empty {} blob — fetched
# here for a uniform deploy path and so a future mcp secret is a seed-list
# addition, not a script change. Runs as the ubuntu user.
# See prog-strength-docs/sows/ssm-deploys-retire-ssh.md.
set -euo pipefail

RELEASE_VERSION="${1:?usage: mcp.sh <version>  (e.g. v0.1.0 or 0.1.0)}"
RELEASE_VERSION="v${RELEASE_VERSION#v}"

AWS_REGION="us-east-2"
SECRET_ID="prog-strength-backend/prod/mcp"

cd /home/ubuntu/prog-strength-infra
git fetch --prune
git checkout main
git pull --ff-only

cd compose/mcp

ECR_REGISTRY="$(aws sts get-caller-identity --query Account --output text).dkr.ecr.${AWS_REGION}.amazonaws.com"
aws ecr get-login-password --region "${AWS_REGION}" \
  | docker login --username AWS --password-stdin "${ECR_REGISTRY}"

umask 077
{
  aws secretsmanager get-secret-value \
    --secret-id "${SECRET_ID}" --region "${AWS_REGION}" \
    --query SecretString --output text \
    | jq -r 'to_entries[] | "\(.key)=\(.value)"'
  echo "AWS_REGION=${AWS_REGION}"
  echo "APP_VERSION=${RELEASE_VERSION}"
  echo "ECR_REGISTRY=${ECR_REGISTRY}"
} >.env

docker compose pull mcp
docker compose down
docker compose up -d

echo "Deployed ${RELEASE_VERSION}"
docker compose ps
docker compose logs --tail=50
```

- [ ] **Step 4: Create `deploy/caddy-reload.sh`** (Caddy reload in place — fixes the stale `/home/ubuntu/prog-strength-api` path the old SSH workflow used; the caddy service lives in the infra `compose/api` project now)

```bash
#!/usr/bin/env bash
#
# On-host Caddy reload. Invoked by prog-strength-infra's deploy-caddy.yml via
# SSM Run Command when only the Caddyfile changes. Pulls the latest infra
# checkout (Caddyfile lives in caddy/) and reloads Caddy in place so the
# Let's Encrypt certs + ACME account key in the caddy_data volume survive and
# live connections aren't dropped. Caddy runs as a service in the api compose
# project (compose/api/docker-compose.yml). Runs as the ubuntu user.
# See prog-strength-docs/sows/ssm-deploys-retire-ssh.md.
set -euo pipefail

cd /home/ubuntu/prog-strength-infra
git fetch --prune
git checkout main
git pull --ff-only

cd compose/api
docker compose exec -T caddy caddy reload --config /etc/caddy/Caddyfile
docker compose ps caddy
```

- [ ] **Step 5: Make scripts executable and run shellcheck**

```bash
cd /workspace/prog-strength-infra
chmod +x deploy/api.sh deploy/mcp.sh deploy/agent.sh deploy/caddy-reload.sh
shellcheck deploy/*.sh
```
Expected: no findings.

- [ ] **Step 6: Commit**

```bash
git add deploy
git commit -m "feat(deploy): add on-host SSM deploy scripts (Secrets Manager .env render)"
```

---

## Task 3 — Infra workflows + README scrub

**Files:** Create `.github/workflows/seed-secrets.yml`; Modify `.github/workflows/deploy-caddy.yml`, `README.md`.

Work from `/workspace/prog-strength-infra`.

- [ ] **Step 1: Create `.github/workflows/seed-secrets.yml`** — `workflow_dispatch` job that assumes the OIDC role and `put-secret-value`s each service's JSON blob assembled from GitHub secrets. This is the ONLY place a secret value transits a runner, and only on an explicit seed. The forwarded-secret list is exactly the `envs:` lists being removed from the deploy workflows. Optional keys that are empty are filtered out so they don't create empty `.env` lines.

```yaml
name: Seed Secrets Manager

# Sync backend app config from GitHub secrets into AWS Secrets Manager. This
# is the ONLY place a secret value transits a runner — and only on an explicit
# manual seed, never on a deploy. Run it once at setup and again whenever a
# secret rotates. The containers themselves are Terraform-owned
# (modules/secrets); this writes their values. GitHub org/repo secrets remain
# the source of truth; this syncs them in.
# See prog-strength-docs/sows/ssm-deploys-retire-ssh.md.

on:
  workflow_dispatch:

permissions:
  id-token: write # mint the OIDC token for configure-aws-credentials
  contents: read

concurrency:
  group: seed-secrets-prod
  cancel-in-progress: false

jobs:
  seed:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_GHA_ROLE_ARN }}
          aws-region: us-east-2

      - name: Seed prog-strength-backend/prod/api
        env:
          JWT_SIGNING_KEY: ${{ secrets.JWT_SIGNING_KEY }}
          GOOGLE_CLIENT_ID: ${{ secrets.GOOGLE_CLIENT_ID }}
          GOOGLE_CLIENT_SECRET: ${{ secrets.GOOGLE_CLIENT_SECRET }}
          CALENDAR_TOKEN_ENC_KEY: ${{ secrets.CALENDAR_TOKEN_ENC_KEY }}
          LITESTREAM_REPLICA_BUCKET: ${{ secrets.LITESTREAM_REPLICA_BUCKET }}
          LITESTREAM_REPLICA_REGION: ${{ secrets.LITESTREAM_REPLICA_REGION }}
          TCX_BUCKET_NAME: ${{ secrets.TCX_BUCKET_NAME }}
          AVATAR_BUCKET_NAME: ${{ secrets.AVATAR_BUCKET_NAME }}
          ADMIN_EMAILS: ${{ secrets.ADMIN_EMAILS }}
          GRAFANA_ADMIN_USER: ${{ secrets.GRAFANA_ADMIN_USER }}
          GRAFANA_ADMIN_PASSWORD: ${{ secrets.GRAFANA_ADMIN_PASSWORD }}
          FATSECRET_CLIENT_ID: ${{ secrets.FATSECRET_CLIENT_ID }}
          FATSECRET_CLIENT_SECRET: ${{ secrets.FATSECRET_CLIENT_SECRET }}
          USDA_FDC_API_KEY: ${{ secrets.USDA_FDC_API_KEY }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          set -euo pipefail
          umask 077
          payload="$(jq -n \
            --arg JWT_SIGNING_KEY "${JWT_SIGNING_KEY}" \
            --arg GOOGLE_CLIENT_ID "${GOOGLE_CLIENT_ID}" \
            --arg GOOGLE_CLIENT_SECRET "${GOOGLE_CLIENT_SECRET}" \
            --arg CALENDAR_TOKEN_ENC_KEY "${CALENDAR_TOKEN_ENC_KEY}" \
            --arg LITESTREAM_REPLICA_BUCKET "${LITESTREAM_REPLICA_BUCKET}" \
            --arg LITESTREAM_REPLICA_REGION "${LITESTREAM_REPLICA_REGION}" \
            --arg TCX_BUCKET_NAME "${TCX_BUCKET_NAME}" \
            --arg AVATAR_BUCKET_NAME "${AVATAR_BUCKET_NAME}" \
            --arg ADMIN_EMAILS "${ADMIN_EMAILS}" \
            --arg GRAFANA_ADMIN_USER "${GRAFANA_ADMIN_USER}" \
            --arg GRAFANA_ADMIN_PASSWORD "${GRAFANA_ADMIN_PASSWORD}" \
            --arg FATSECRET_CLIENT_ID "${FATSECRET_CLIENT_ID}" \
            --arg FATSECRET_CLIENT_SECRET "${FATSECRET_CLIENT_SECRET}" \
            --arg USDA_FDC_API_KEY "${USDA_FDC_API_KEY}" \
            --arg OPENAI_API_KEY "${OPENAI_API_KEY}" \
            --arg ANTHROPIC_API_KEY "${ANTHROPIC_API_KEY}" \
            '{
              JWT_SIGNING_KEY: $JWT_SIGNING_KEY,
              GOOGLE_CLIENT_ID: $GOOGLE_CLIENT_ID,
              GOOGLE_CLIENT_SECRET: $GOOGLE_CLIENT_SECRET,
              CALENDAR_TOKEN_ENC_KEY: $CALENDAR_TOKEN_ENC_KEY,
              LITESTREAM_REPLICA_BUCKET: $LITESTREAM_REPLICA_BUCKET,
              LITESTREAM_REPLICA_REGION: $LITESTREAM_REPLICA_REGION,
              TCX_BUCKET_NAME: $TCX_BUCKET_NAME,
              AVATAR_BUCKET_NAME: $AVATAR_BUCKET_NAME,
              ADMIN_EMAILS: $ADMIN_EMAILS,
              GRAFANA_ADMIN_USER: $GRAFANA_ADMIN_USER,
              GRAFANA_ADMIN_PASSWORD: $GRAFANA_ADMIN_PASSWORD,
              FATSECRET_CLIENT_ID: $FATSECRET_CLIENT_ID,
              FATSECRET_CLIENT_SECRET: $FATSECRET_CLIENT_SECRET,
              USDA_FDC_API_KEY: $USDA_FDC_API_KEY,
              OPENAI_API_KEY: $OPENAI_API_KEY,
              ANTHROPIC_API_KEY: $ANTHROPIC_API_KEY
            }
            | with_entries(select(.value != ""))')"
          aws secretsmanager put-secret-value \
            --secret-id prog-strength-backend/prod/api \
            --secret-string "${payload}"

      - name: Seed prog-strength-backend/prod/agent
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          JWT_SIGNING_KEY: ${{ secrets.JWT_SIGNING_KEY }}
          CORS_ALLOWED_ORIGINS: ${{ secrets.CORS_ALLOWED_ORIGINS }}
        run: |
          set -euo pipefail
          umask 077
          payload="$(jq -n \
            --arg ANTHROPIC_API_KEY "${ANTHROPIC_API_KEY}" \
            --arg OPENAI_API_KEY "${OPENAI_API_KEY}" \
            --arg JWT_SIGNING_KEY "${JWT_SIGNING_KEY}" \
            --arg CORS_ALLOWED_ORIGINS "${CORS_ALLOWED_ORIGINS}" \
            '{
              ANTHROPIC_API_KEY: $ANTHROPIC_API_KEY,
              OPENAI_API_KEY: $OPENAI_API_KEY,
              JWT_SIGNING_KEY: $JWT_SIGNING_KEY,
              CORS_ALLOWED_ORIGINS: $CORS_ALLOWED_ORIGINS
            }
            | with_entries(select(.value != ""))')"
          aws secretsmanager put-secret-value \
            --secret-id prog-strength-backend/prod/agent \
            --secret-string "${payload}"

      - name: Seed prog-strength-backend/prod/mcp
        run: |
          set -euo pipefail
          # mcp holds no app secrets today (auth is per-request from the
          # agent). Seed an empty object so the container has a readable
          # version and mcp.sh's uniform get-secret-value succeeds; a future
          # mcp secret is a one-line addition here.
          aws secretsmanager put-secret-value \
            --secret-id prog-strength-backend/prod/mcp \
            --secret-string '{}'
```

- [ ] **Step 2: Rewrite `.github/workflows/deploy-caddy.yml`** to deploy via SSM (keep the same name, triggers, permissions add `id-token: write`, concurrency). Replace the `appleboy/ssh-action` step with OIDC + `aws ssm send-command` invoking `deploy/caddy-reload.sh`, polling to completion:

```yaml
name: Deploy Caddyfile

# Pushes Caddy config changes to the EC2 host without waiting for an api
# release. The api release workflow also pulls this repo on every deploy, so
# this workflow only matters when Caddy config changes *between* api releases —
# e.g. adding a new vhost for the MCP server. Runs via SSM Run Command (no SSH,
# no inbound port 22). See prog-strength-docs/sows/ssm-deploys-retire-ssh.md.

on:
  push:
    branches: [main]
    paths:
      - 'caddy/**'
      - '.github/workflows/deploy-caddy.yml'
  workflow_dispatch:

permissions:
  id-token: write # mint the OIDC token for configure-aws-credentials
  contents: read

concurrency:
  group: deploy-caddy-prod
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_GHA_ROLE_ARN }}
          aws-region: us-east-2

      - name: Reload Caddy via SSM Run Command
        run: |
          set -euo pipefail

          command_id="$(aws ssm send-command \
            --document-name AWS-RunShellScript \
            --comment "Reload Caddy" \
            --targets "Key=tag:Name,Values=prog-strength-prod-backend" \
            --parameters commands="sudo -u ubuntu -H bash /home/ubuntu/prog-strength-infra/deploy/caddy-reload.sh" \
            --query 'Command.CommandId' --output text)"
          echo "SSM command dispatched: ${command_id}"

          instance_id=""
          for _ in $(seq 1 30); do
            instance_id="$(aws ssm list-command-invocations \
              --command-id "${command_id}" \
              --query 'CommandInvocations[0].InstanceId' --output text)"
            [ -n "${instance_id}" ] && [ "${instance_id}" != "None" ] && break
            sleep 2
          done
          if [ -z "${instance_id}" ] || [ "${instance_id}" = "None" ]; then
            echo "No SSM invocation registered for command ${command_id}" >&2
            exit 1
          fi
          echo "Running on instance: ${instance_id}"

          while true; do
            status="$(aws ssm get-command-invocation \
              --command-id "${command_id}" --instance-id "${instance_id}" \
              --query 'Status' --output text)"
            case "${status}" in
              Success) break ;;
              Pending | InProgress | Delayed) sleep 5 ;;
              *)
                echo "Caddy reload failed with status: ${status}" >&2
                aws ssm get-command-invocation \
                  --command-id "${command_id}" --instance-id "${instance_id}" \
                  --query 'StandardErrorContent' --output text >&2 || true
                aws ssm get-command-invocation \
                  --command-id "${command_id}" --instance-id "${instance_id}" \
                  --query 'StandardOutputContent' --output text || true
                exit 1
                ;;
            esac
          done

          echo "Caddy reloaded."
          aws ssm get-command-invocation \
            --command-id "${command_id}" --instance-id "${instance_id}" \
            --query 'StandardOutputContent' --output text
```

- [ ] **Step 3: Scrub `README.md`** — replace the `EC2_HOST` / `EC2_SSH_KEY` block (the paragraph "The Caddy deploy workflow (`deploy-caddy.yml`) SSHes into the host instead, so it needs:" plus the two bullets) with text that says the Caddy deploy now uses SSM and needs no host-identity secret:

```markdown
The Caddy deploy workflow (`deploy-caddy.yml`) deploys via SSM Run Command
using the same OIDC role — no SSH, no inbound port 22, and no `EC2_HOST` /
`EC2_SSH_KEY` secrets. It targets the host by its `Name` tag, so it also
survives an instance replacement.
```

Also update the `compute` variables table note for `security_group.ingress_rules` from "SSH/80/443 by default." to "80/443 by default (no inbound SSH — deploys + break-glass go through SSM).". Leave the `ssh_key_name` row as-is (the key pair resource is kept this SOW; only the port is closed) but you may add a short note that it is unused now that 22 is closed.

- [ ] **Step 4: Gate**

```bash
cd /workspace/prog-strength-infra
actionlint .github/workflows/seed-secrets.yml .github/workflows/deploy-caddy.yml
```
Expected: no errors. (If `actionlint` shellcheck-warns on the embedded run scripts, fix the shell, do not suppress.)

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/seed-secrets.yml .github/workflows/deploy-caddy.yml README.md
git commit -m "feat(ci): seed-secrets workflow + SSM Caddy deploy; scrub EC2 SSH secrets from README"
```

---

## Task 4 — prog-strength-api: workflows → SSM + DEPLOYMENT.md scrub

**Files:** Modify `.github/workflows/release.yml`, `.github/workflows/manual-deploy.yml`, `DEPLOYMENT.md`.

Work from `/workspace/prog-strength-api` on branch `feat/ssm-deploys-retire-ssh`.

- [ ] **Step 1: Rewrite the `deploy` job in `release.yml`.** Keep the `release` and `build_and_push` jobs unchanged. Replace the entire `deploy` job with the SSM version below. It assumes the OIDC role (needs `id-token: write`), invokes `deploy/api.sh` via SSM as `ubuntu`, and polls to completion. The long `env:`/`envs:` secret block is deleted entirely — secrets now live in Secrets Manager.

```yaml
  deploy:
    needs: [release, build_and_push]
    if: needs.release.outputs.new_release_published == 'true'
    runs-on: ubuntu-latest
    permissions:
      id-token: write # mint the OIDC token for configure-aws-credentials
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_GHA_ROLE_ARN }}
          aws-region: us-east-2

      - name: Deploy tagged version via SSM Run Command
        env:
          RELEASE_VERSION: ${{ needs.release.outputs.new_release_version }}
        run: |
          set -euo pipefail

          # Invoke the versioned on-host deploy script via SSM Run Command —
          # no SSH, no inbound port 22. The host is targeted by its Name tag
          # (survives an instance replacement, so no EC2_HOST). The script
          # reads app secrets from Secrets Manager via the instance role, so
          # only the non-secret version tag travels with the command. Run as
          # the ubuntu user so the infra checkout + docker creds keep their
          # ownership.
          command_id="$(aws ssm send-command \
            --document-name AWS-RunShellScript \
            --comment "Deploy api v${RELEASE_VERSION}" \
            --targets "Key=tag:Name,Values=prog-strength-prod-backend" \
            --parameters commands="sudo -u ubuntu -H bash /home/ubuntu/prog-strength-infra/deploy/api.sh v${RELEASE_VERSION}" \
            --query 'Command.CommandId' --output text)"
          echo "SSM command dispatched: ${command_id}"

          instance_id=""
          for _ in $(seq 1 30); do
            instance_id="$(aws ssm list-command-invocations \
              --command-id "${command_id}" \
              --query 'CommandInvocations[0].InstanceId' --output text)"
            [ -n "${instance_id}" ] && [ "${instance_id}" != "None" ] && break
            sleep 2
          done
          if [ -z "${instance_id}" ] || [ "${instance_id}" = "None" ]; then
            echo "No SSM invocation registered for command ${command_id}" >&2
            exit 1
          fi
          echo "Running on instance: ${instance_id}"

          # Poll to completion so the workflow reflects the real deploy exit
          # status (the guarantee set -e gave over SSH).
          while true; do
            status="$(aws ssm get-command-invocation \
              --command-id "${command_id}" --instance-id "${instance_id}" \
              --query 'Status' --output text)"
            case "${status}" in
              Success) break ;;
              Pending | InProgress | Delayed) sleep 5 ;;
              *)
                echo "Deploy failed with status: ${status}" >&2
                aws ssm get-command-invocation \
                  --command-id "${command_id}" --instance-id "${instance_id}" \
                  --query 'StandardErrorContent' --output text >&2 || true
                aws ssm get-command-invocation \
                  --command-id "${command_id}" --instance-id "${instance_id}" \
                  --query 'StandardOutputContent' --output text || true
                exit 1
                ;;
            esac
          done

          echo "Deploy succeeded."
          aws ssm get-command-invocation \
            --command-id "${command_id}" --instance-id "${instance_id}" \
            --query 'StandardOutputContent' --output text
```

- [ ] **Step 2: Rewrite the `deploy` job in `manual-deploy.yml`.** Keep the `checkout` + `Resolve version` step (it produces `steps.resolve.outputs.version`). Add the OIDC permissions at job or workflow level and replace the `appleboy/ssh-action` step. Because `manual-deploy.yml` has a `resolve` step that emits the version without a leading `v`, the SSM step passes `v${{ steps.resolve.outputs.version }}`:

  - Add to the `deploy` job:
    ```yaml
    permissions:
      id-token: write
      contents: read
    ```
    (The workflow already declares top-level `permissions: contents: read`; either bump that to include `id-token: write` or add a job-level `permissions` block — job-level is cleaner.)
  - After the `Resolve version` step, add the configure-aws-credentials step and the SSM step. The SSM step body is identical to release.yml's above, except:
    - `env: RELEASE_VERSION: ${{ steps.resolve.outputs.version }}`
    - the `--comment` and `commands` use `v${RELEASE_VERSION}` the same way.
  - Delete the `appleboy/ssh-action` step and its entire `env:`/`with:` secret block.

- [ ] **Step 3: Scrub `DEPLOYMENT.md`.** Focus on the SSH/transport + secret references the SOW mandates; keep edits on-concern (don't rewrite unrelated pre-existing staleness beyond what the transport change touches). Specifically:
  - In **Repository secrets → `prog-strength-api`**, delete the `EC2_HOST` and `EC2_SSH_KEY` rows. Add a short line above the table: app secrets are now seeded into AWS Secrets Manager by `prog-strength-infra/seed-secrets.yml` (these GitHub secrets remain the source of truth and the seed source).
  - In **Repository secrets → `prog-strength-infra`**, delete the `EC2_HOST` and `EC2_SSH_KEY` rows (leave/replace the AWS rows; note CI uses the OIDC role `AWS_GHA_ROLE_ARN`, not static keys).
  - In **Deployment flow → On push**, change "**deploy** job … SSHes to the EC2 host as `ubuntu`" to describe the SSM Run Command path: assume the OIDC role, `aws ssm send-command` invoking `prog-strength-infra/deploy/api.sh`, which pulls infra, renders `.env` from Secrets Manager, and runs `docker compose pull/down/up`. Remove the stale "checkout prog-strength-api on the host" / `--build` lines that the ECR move already obsoleted, since you're rewriting this section anyway.
  - In **Deployment flow → deploy-caddy**, change "SSHes to EC2" to "via SSM Run Command (`deploy/caddy-reload.sh`)".
  - In **Manual operations → SSH**, replace the `ssh -i …pem ubuntu@…` break-glass with AWS Session Manager: `aws ssm start-session --target <instance-id> --region us-east-2` (note: requires the Session Manager plugin; the instance is targetable by its `Name` tag via `aws ec2 describe-instances`). Update the **Manual deploy (skip GitHub Actions)** section to "re-run the `Manual Deploy` workflow (`workflow_dispatch`)" rather than SSHing in and `docker compose up --build`.
  - In **Troubleshooting**, replace the `permission denied (publickey)` / `EC2_SSH_KEY` item with an SSM-oriented one (e.g. deploy fails if the host isn't a registered SSM managed node — check the SSM console / `aws ssm describe-instance-information`).
  - Remove the `scp -i …pem` manual-snapshot example or convert it to a Session Manager-based copy note.
  - Leave the Litestream/backup, cost, and DNS sections alone except where they reference the SSH key path.

- [ ] **Step 4: Gate**

```bash
cd /workspace/prog-strength-api
actionlint .github/workflows/release.yml .github/workflows/manual-deploy.yml
```
Expected: no errors. Also confirm the repo's named gate (see `AGENTS.md`/`CONTRIBUTING.md`) is unaffected — only YAML/markdown changed, so the Go test/lint suite should be untouched; spot-check that `golangci-lint`/`go test` config files weren't modified.

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/release.yml .github/workflows/manual-deploy.yml DEPLOYMENT.md
git commit -m "feat(deploy): deploy via SSM Run Command; retire SSH deploy key"
```

---

## Task 5 — prog-strength-mcp: workflows → SSM + README scrub

**Files:** Modify `.github/workflows/release.yml`, `.github/workflows/manual-deploy.yml`, `README.md`.

Work from `/workspace/prog-strength-mcp`.

- [ ] **Step 1: Rewrite the `deploy` job in `release.yml`** exactly like Task 4 Step 1, but invoking `deploy/mcp.sh` and commenting "Deploy mcp v…". The old mcp deploy forwarded no secrets (only `RELEASE_VERSION`), so there is no `env:` secret block to remove beyond `RELEASE_VERSION` plumbing — keep `RELEASE_VERSION` in the SSM step's `env:` as the version source. Add `permissions: id-token: write, contents: read` to the job.

- [ ] **Step 2: Rewrite the `deploy` job in `manual-deploy.yml`** like Task 4 Step 2, invoking `deploy/mcp.sh`, version from `steps.resolve.outputs.version`.

- [ ] **Step 3: Scrub `README.md`.** Replace the **Required GitHub secrets** table (the `EC2_HOST` / `EC2_SSH_KEY` rows) — mcp's deploy now needs no host-identity or SSH secrets at all (it deploys via SSM with the shared OIDC role). Either remove the table and state "Deploys run via SSM Run Command using the shared OIDC role — no repo-level deploy secrets required", or replace it with a one-row note about `AWS_GHA_ROLE_ARN` being an org/shared secret. Update the **Pipeline** paragraph: change "tags and SSH-deploys … SSHes into the EC2 host, `git pull`s `/home/ubuntu/prog-strength-mcp`, writes a minimal `.env`…" to the SSM reality: semantic-release tags, then `aws ssm send-command` invokes `prog-strength-infra/deploy/mcp.sh`, which pulls the infra checkout and runs `docker compose pull/down/up`. Fix the stale `git clone .../prog-strength-mcp` host-prereq line (the mcp repo isn't checked out on the host anymore — compose lives in infra) if it's in the touched section; otherwise leave unrelated content.

- [ ] **Step 4: Gate**

```bash
cd /workspace/prog-strength-mcp
actionlint .github/workflows/release.yml .github/workflows/manual-deploy.yml
```

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/release.yml .github/workflows/manual-deploy.yml README.md
git commit -m "feat(deploy): deploy via SSM Run Command; retire SSH deploy key"
```

---

## Task 6 — prog-strength-agent: workflows → SSM + README note

**Files:** Modify `.github/workflows/release.yml`, `.github/workflows/manual-deploy.yml`, `README.md`.

Work from `/workspace/prog-strength-agent`.

- [ ] **Step 1: Rewrite the `deploy` job in `release.yml`** like Task 4 Step 1, invoking `deploy/agent.sh`, comment "Deploy agent v…". Delete the `env:` secret block (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `JWT_SIGNING_KEY`, `CORS_ALLOWED_ORIGINS`) — those now live in Secrets Manager. Keep `RELEASE_VERSION` as the version source in the SSM step's `env:`. Add `permissions: id-token: write, contents: read`.

- [ ] **Step 2: Rewrite the `deploy` job in `manual-deploy.yml`** like Task 4 Step 2, invoking `deploy/agent.sh`, version from `steps.resolve.outputs.version`, secret block removed.

- [ ] **Step 3: README.** The agent README's **Deployment** section does not reference `EC2_HOST`/`EC2_SSH_KEY` or SSH, so there is no secret to scrub. Add one sentence to that section noting deploys go through SSM Run Command (invoking `prog-strength-infra/deploy/agent.sh`) with app secrets read from AWS Secrets Manager — keeping the doc accurate post-cutover. Do not invent SSH content to remove.

- [ ] **Step 4: Gate**

```bash
cd /workspace/prog-strength-agent
actionlint .github/workflows/release.yml .github/workflows/manual-deploy.yml
```

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/release.yml .github/workflows/manual-deploy.yml README.md
git commit -m "feat(deploy): deploy via SSM Run Command; retire SSH deploy key"
```

---

## Task 7 — prog-strength-docs: SOW status flip

**Files:** Modify `sows/ssm-deploys-retire-ssh.md` (+ this plan is committed on the same branch).

Work from `/workspace/prog-strength-docs` on branch `feat/ssm-deploys-retire-ssh`.

- [ ] **Step 1: Edit `sows/ssm-deploys-retire-ssh.md`:**
  - YAML frontmatter `status: draft` → `status: shipped`.
  - Body header `**Status**: Draft · **Last updated**: 2026-06-21` → `**Status**: Shipped · **Last updated**: 2026-06-22`.

- [ ] **Step 2: Commit** (the plan file is already added in an earlier controller commit on this branch, or add it here)

```bash
git add sows/ssm-deploys-retire-ssh.md plans/2026-06-22-ssm-deploys-retire-ssh.md
git commit -m "docs: mark ssm-deploys-retire-ssh as shipped"
```

---

## Rollout / deployment order (for the docs PR body)

The infra PR establishes the new transport and secret store; service repos and the Caddy workflow depend on it. Merge order:

1. **`prog-strength-infra`** first — secret containers, instance-role `GetSecretValue` + SSM managed-node registration, OIDC seed grants, the `deploy/*.sh` scripts, `seed-secrets.yml`, the SSM Caddy workflow, and the closed port 22. Apply, **run `seed-secrets.yml`**, confirm the host shows as a managed node and `.env` renders from Secrets Manager — until this lands and is seeded, every other repo's SSM deploy would fail (no script on host, no secrets to read). On the live host, ensure `jq` is installed (the running host predates the bootstrap change) and the SSM agent is registered before closing 22; verify `aws ssm start-session` works (break-glass) before relying on the closed port.
2. **`prog-strength-api`**, **`prog-strength-mcp`**, **`prog-strength-agent`** — independent of each other; can merge in parallel once infra is deployed and seeded. Each repo's next release (or `Manual Deploy`) then runs over SSM.
3. After all seven workflows have a green SSM run and 22 is confirmed closed without breakage: delete the `EC2_SSH_KEY` / `EC2_HOST` org secrets (manual, in GitHub settings — out of band of these PRs).

## Verification after rollout

- Host appears in the SSM console / `aws ssm describe-instance-information` as a managed node.
- `seed-secrets.yml` run is green; `aws secretsmanager get-secret-value --secret-id prog-strength-backend/prod/api` returns the expected keys; the `jq` render produces a `.env` byte-equivalent to the previous SSH-written one.
- One service deploy over SSM (suggest `agent/manual-deploy.yml` first) is green with the service healthy and `.env` correct.
- `aws ssm start-session --target <instance-id>` opens a shell (break-glass verified) **before** relying on the closed port.
- With 22 closed: 80/443 still serve (`https://api.progstrength.fitness/health` 200), and SSM deploys still succeed.
- Caddy reload via `deploy-caddy.yml` (`workflow_dispatch`) is green and certs persist.
```
