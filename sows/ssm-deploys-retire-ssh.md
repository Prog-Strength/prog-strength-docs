---
status: draft
repos:
  - prog-strength-docs
  - prog-strength-infra
  - prog-strength-api
  - prog-strength-mcp
  - prog-strength-agent
---

# SSM-Based Deploys: Retire the SSH Deploy Key

**Status**: Draft · **Last updated**: 2026-06-21

## Introduction

Every production deploy still happens over SSH. Seven workflows across four repos use
`appleboy/ssh-action@v1.0.3` to log into the EC2 host as `ubuntu` and run `docker compose`:

- `prog-strength-api` — `release.yml` (deploy job), `manual-deploy.yml`
- `prog-strength-mcp` — `release.yml`, `manual-deploy.yml`
- `prog-strength-agent` — `release.yml`, `manual-deploy.yml`
- `prog-strength-infra` — `deploy-caddy.yml`

The connection authenticates with two organization secrets: `EC2_SSH_KEY` (the private key for the
`prog-strength-backend-prod-keys` key pair) and `EC2_HOST` (the Elastic IP). Both are referenced in
the four repos' READMEs/DEPLOYMENT docs as well.

This is the deploy mechanism that the OIDC SOW (`github-actions-oidc-role.md`) explicitly left in
place — its Non-Goals called out "Changing the deploy mechanism … SSH-based deploys are untouched."
That SOW migrated *AWS API* auth (ECR push, Terraform) off long-lived keys and onto a single
short-lived OIDC role. This SOW finishes the job for the one credential it deliberately deferred.

Two properties make the SSH key the least comfortable credential left in the org:

1. **It is a standing, long-lived key that grants interactive shell as `ubuntu`** — which has docker,
   i.e. effectively root on the prod host. Unlike the OIDC flow that mints minutes-long credentials
   per run, this key sits in org secrets indefinitely; a single leak is "shell on prod" until it is
   noticed and rotated by hand.
2. **It requires standing inbound SSH.** `environments/prod.tfvars` opens port 22 to `0.0.0.0/0`
   (the GitHub-hosted runner IP range is too broad and volatile to scope to). That is open attack
   surface independent of the key itself.

The end state: deploys run through **AWS Systems Manager (SSM) Run Command**, authenticated by the
existing OIDC role; app secrets live in **AWS Secrets Manager** (infra-owned, seeded from the
existing GitHub secrets) and are fetched on the host via its instance role; **port 22 is closed**;
and `EC2_SSH_KEY` / `EC2_HOST` are deleted. Operationally: *"nothing with a shell credential or an
open SSH port touches the prod host; CI talks to it only through SSM, scoped by IAM and logged in
CloudTrail; secrets never ride a deploy."*

## Why this is a clean fit (not a rewrite)

The groundwork already exists in this account, which is why this is hardening rather than
re-architecture:

- **The SSM deploy transport is already proven here.** The OIDC role's policy
  (`modules/github_oidc/main.tf`, `SSMSendCommand` statement) grants `ssm:SendCommand`,
  `ssm:ListCommandInvocations`, and `ssm:GetCommandInvocation` on the account's instances — and
  `prog-strength-developer`'s `deploy-manager.yml` **already uses SSM RunCommand to push compose
  updates to its manager instance and poll the invocation**. We are applying a working pattern to the
  backend host, not inventing one. (The role's `ssm:GetParameter*` grant is AMI-lookup only —
  `parameter/aws/service/*` — and is irrelevant here, since the *instance* role reads secrets, not
  the OIDC role.)
- **Secrets Manager is the established runtime-secret pattern.** `prog-strength-developer` already
  stores `prog-strength-developer/claude-credentials` and `prog-strength-developer/github-app` in
  Secrets Manager (`terraform/secrets.tf`), read at runtime by an instance role via
  `secretsmanager:GetSecretValue`. That is exactly the pattern the backend `.env` fetch needs — one
  secrets story for the whole account, not two.
- **The host already authenticates to AWS by instance role, not static keys.** Inside today's SSH
  session, ECR login already uses the instance metadata service (`aws ecr get-login-password`, no
  static creds). Fetching secrets via the instance role is the identical pattern.
- **Infra already owns all on-host orchestration.** The host clones only `prog-strength-infra` on
  first boot (`bootstrap.infra_repo_url`); compose files, the Caddyfile, monitoring, and litestream
  config all live there. The deploy *scripts* belong there too — moving the inline workflow bash into
  versioned scripts in infra is a natural consolidation. Keeping the secret definitions there too
  means **every backend resource is managed by `prog-strength-infra`**.
- **The instance role is designed for additive attachments.** `modules/compute/iam.tf` owns the role
  and exposes `instance_role_name`; domain modules hang policies off it (ECR pull, Litestream S3).
  Adding Secrets Manager read is one more attachment in the same shape — see `modules/ecr/main.tf`'s
  `instance_ecr_pull`.

## Proposed Solution

### Deploy transport: SSM Run Command instead of SSH

Each workflow's deploy step becomes: assume the OIDC role (already wired via
`aws-actions/configure-aws-credentials@v4`), then `aws ssm send-command` targeting the instance and
invoking a **versioned deploy script that already lives on the host** (under the
`prog-strength-infra` checkout). The runner passes only **non-secret** parameters — chiefly the
released version tag. The command is run synchronously (poll `get-command-invocation` until terminal)
so the workflow reflects the real deploy exit status, the same guarantee `set -e` gives today.

The SSM agent dials *out* to AWS over 443, so SSM needs **no inbound port** — this is what lets us
close 22 entirely at the end.

### Secrets: Secrets Manager, infra-owned, seeded from GitHub

This is the substantive piece. Today ~16 app secrets (in api's `release.yml` alone:
`JWT_SIGNING_KEY`, `GOOGLE_CLIENT_*`, `CALENDAR_TOKEN_ENC_KEY`, the bucket identities, `ADMIN_EMAILS`,
Grafana creds, nutrition + vector-memory provider keys, …) are forwarded over the SSH channel and
written to `.env` on the host on **every deploy**.

They cannot move to SSM command parameters (Run Command records its parameters in command history and
CloudTrail — passing secrets that way leaks them). Instead, all backend secrets live in Secrets
Manager, with a deliberate split of ownership that keeps plaintext out of Terraform state:

- **Terraform (infra) owns the secret *containers*.** One JSON secret per service —
  `prog-strength-api/app-config`, `prog-strength-mcp/app-config`, `prog-strength-agent/app-config` —
  created via `aws_secretsmanager_secret`, mirroring the `prog-strength-developer/*` naming. Infra
  owns the resource, its IAM, and (future) rotation policy: every backend resource stays in one repo.
- **Terraform does *not* own the secret *values*.** No `aws_secretsmanager_secret_version` carrying
  real content; values never enter TF state. This honors the rule `prog-strength-developer/
  terraform/secrets.tf` already states explicitly ("committing them to state would be a leak risk").
- **A `seed-secrets.yml` workflow in infra seeds the values from GitHub secrets.** A
  `workflow_dispatch` job assumes the OIDC role, reads the app secrets from GitHub org/repo secrets,
  and `aws secretsmanager put-secret-value`s each service's JSON blob. Run it once at setup and again
  whenever a secret rotates. **This is the only place a secret transits a runner — and only on an
  explicit seed, never on a deploy.**
- **The host reads secrets via its instance role at deploy time.** The on-host script runs
  `aws secretsmanager get-secret-value`, parses the JSON with `jq`, and renders `.env` into the same
  target dir as today. Optional provider keys (FatSecret, USDA, OpenAI/Anthropic) absent from the
  blob default to empty, preserving today's degrade-to-503 behavior. The runner never sees a value.

This deletes the long `env:`/`envs:` block from every release workflow and removes secrets from the
deploy path entirely.

**Source-of-truth note (accepted):** GitHub org secrets remain the source of truth for app config
(as today), now *synced* into Secrets Manager rather than streamed over SSH on every deploy. This
differs from the developer repo's owner-private root credentials (the Claude OAuth token, the GitHub
App private key), which are hand-seeded once and never live in GitHub. App config secrets already
live in GitHub; keeping them there as the seed source keeps the process automated and the account
reproducible from `infra + GitHub secrets`, at the cost of GitHub still holding them. Accepted
deliberately; a future move to Secrets-Manager-as-source-of-truth (hand-seeded, GitHub-free) is an
additive change to the seed step, not a redesign.

### IAM changes

- **Instance role** (infra, mirroring `modules/ecr`'s attachment): add
  `AmazonSSMManagedInstanceCore` (AWS-managed) so the agent registers the node, plus an inline
  `secretsmanager:GetSecretValue` scoped to the `prog-strength-*/*` secret ARNs. The default
  `aws/secretsmanager` KMS key needs no explicit `kms:Decrypt` grant for same-account use.
- **OIDC role** (infra, `modules/github_oidc`): the SSM transport actions already exist. Add
  `secretsmanager:CreateSecret` / `TagResource` (the apply pipeline creates the containers) and
  `secretsmanager:PutSecretValue` / `DescribeSecret` (the seed workflow writes values), scoped to
  `prog-strength-*`. The existing `SecretsManagerDescribe` statement (developer/* only) is widened or
  joined by a backend statement.

### Closing the door (infra + GitHub)

Once all seven workflows deploy via SSM and Session Manager break-glass is verified:

- Remove the SSH ingress rule from `environments/prod.tfvars` (keep 80/443).
- Delete `EC2_SSH_KEY` and `EC2_HOST` org secrets; scrub them from the four repos' deploy docs.
- The EC2 key pair (`prog-strength-backend-prod-keys`) can stay as a Terraform resource — closing the
  port neutralizes it without forcing an instance replacement — or be removed in a later cleanup.

## Goals and Non-Goals

### Goals

- All seven deploy workflows authenticate via the existing OIDC role and deploy via `aws ssm
  send-command`; none use `appleboy/ssh-action` or the SSH secrets.
- All backend app secrets live in infra-owned Secrets Manager secrets (`prog-strength-<service>/
  app-config`), seeded from GitHub secrets by an infra `seed-secrets.yml` workflow, with no secret
  value ever written to Terraform state and no secret transiting a runner on a deploy.
- The instance role can read those secrets (`GetSecretValue`) and is registered as an SSM managed
  node; the OIDC role can create/seed them and send commands.
- Inbound port 22 is closed in `prod.tfvars`; AWS Session Manager is the verified break-glass shell.
- `EC2_SSH_KEY` and `EC2_HOST` org secrets are deleted and removed from all deploy docs.
- The migration is reversible at every checkpoint until the final cutover (the SSH path stays valid
  until SSM is proven).

### Non-Goals

- **No new deploy *behavior*** — same `docker compose pull/down/up`, same compose files, same
  image-from-ECR flow. Only the transport and the secret source change.
- **Not migrating non-secret config.** `config.toml` (`centralized-api-config.md`) is unaffected.
- **Not changing the source of truth for app secrets** — they remain in GitHub, now synced to Secrets
  Manager. (Secrets-Manager-as-source-of-truth is a deferred, additive change.)
- **Not touching `prog-strength-developer`'s deploys or its existing secrets** — it already runs on
  SSM + Secrets Manager and does not use the SSH deploy secrets.
- **Not removing the EC2 key pair resource** in this SOW (optional later cleanup; out of scope to
  avoid an instance replacement).

## Implementation Details

### Secret containers + seeding (prog-strength-infra)

- A `modules/secrets` (or inline in the compute/backend composition) declaring
  `aws_secretsmanager_secret` for each of `prog-strength-api/app-config`,
  `prog-strength-mcp/app-config`, `prog-strength-agent/app-config`. No `_version` resource with real
  values; if a placeholder version is needed to satisfy consumers, guard it with
  `lifecycle { ignore_changes = [secret_string] }` so the seed workflow's values are never reverted
  by a later apply.
- `seed-secrets.yml` (`workflow_dispatch`): assume OIDC role → for each service, assemble the JSON
  blob from `${{ secrets.* }}` and `aws secretsmanager put-secret-value --secret-id
  prog-strength-<svc>/app-config --secret-string …`. The forwarded-secret list is essentially the
  `envs:` list being removed from the deploy workflows — it moves here, run on demand instead of per
  deploy.

### On-host deploy scripts (prog-strength-infra)

Move the inline bash from each workflow into versioned scripts, e.g. `deploy/api.sh`, `deploy/mcp.sh`,
`deploy/agent.sh`, `deploy/caddy-reload.sh`. Each service script:

1. `cd /home/ubuntu/prog-strength-infra && git fetch --prune && git checkout main && git pull --ff-only`
   (unchanged).
2. ECR login via instance role (unchanged).
3. **New:** `aws secretsmanager get-secret-value --secret-id prog-strength-<svc>/app-config
   --query SecretString --output text | jq -r 'to_entries[] | "\(.key)=\(.value)"' > .env`
   (`umask 077`, same target dir as today; absent optional keys simply don't appear → empty).
4. `docker compose pull/down/up` with the same `-f` merge of the monitoring compose file; echo +
   `ps` + `logs --tail` as now. Keep `set -euo pipefail`. Version tag arrives as a non-secret
   parameter from the SSM invocation.

### SSM invocation (each release/deploy workflow)

Replace the `appleboy/ssh-action` step with:

- `aws-actions/configure-aws-credentials@v4` with `role-to-assume: ${{ secrets.AWS_GHA_ROLE_ARN }}`
  and `id-token: write` (the `build_and_push` job already does this — copy the pattern).
- `aws ssm send-command --document-name AWS-RunShellScript --targets <tag filter>` invoking the
  on-host script and passing the version tag, then poll `get-command-invocation` to completion and
  fail on non-`Success`. `deploy-caddy.yml` invokes `deploy/caddy-reload.sh`. Pattern reference:
  `prog-strength-developer/.github/workflows/deploy-manager.yml`.

### SSM agent presence

Recent Ubuntu EC2 AMIs ship the SSM agent preinstalled via snap. Confirm it is running on the live
host (managed-node status in the SSM console) and, if `bootstrap.sh` provisions the host from a base
image, ensure the agent is enabled so a future `replace-instance` run comes up SSM-ready.

## Rollout Sequence

Three checkpoints, each independently verifiable and reversible. The SSH path stays fully functional
until Checkpoint 3.

### Checkpoint 1 — Secrets Manager + instance role (no deploy change yet)

1. Infra PR: secret containers + instance-role attachments (`AmazonSSMManagedInstanceCore` +
   `GetSecretValue`) + the OIDC-role create/put expansion + confirm/enable the SSM agent. Verify the
   instance appears as a managed node.
2. Run `seed-secrets.yml` to populate the JSON blobs from the current GitHub secrets.
3. **Verify:** from the host, `aws secretsmanager get-secret-value` returns the expected keys and the
   `jq` render produces a `.env` byte-equivalent to today's. Deploys still run over SSH — nothing
   user-visible changed. Fully revertible (drop the attachments, delete the secrets).

### Checkpoint 2 — SSM deploy path, alongside SSH

1. Infra PR: add the `deploy/*.sh` scripts (Secrets Manager fetch + existing compose logic).
2. Convert **one** workflow first (suggest `agent/manual-deploy.yml` — lowest blast radius,
   dispatchable on demand) to the SSM transport. Trigger it; confirm a green deploy with the service
   healthy and `.env` correctly rendered from Secrets Manager.
3. **Verify before proceeding:** the SSM-deployed service is behavior-equivalent to an SSH deploy. If
   anything is off, revert the single workflow — SSH is untouched everywhere else.

### Checkpoint 3 — Cut over, verify break-glass, close the door

1. Convert the remaining six workflows to SSM (one PR per repo). Verify each: api by re-running
   release (or a no-op merge), mcp/agent on next release or via `workflow_dispatch`, infra by a Caddy
   config change or dispatch.
2. **Verify Session Manager break-glass works** (`aws ssm start-session` opens a shell on the host)
   *before* closing 22 — this is the safety gate against locking yourself out.
3. Infra PR: remove the SSH ingress rule from `prod.tfvars`. Confirm 80/443 still serve and SSM
   deploys still succeed with 22 closed.
4. After all seven workflows have a green SSM run **and** 22 is confirmed closed without breakage:
   delete `EC2_SSH_KEY` and `EC2_HOST` org secrets and scrub them from
   `prog-strength-api/DEPLOYMENT.md`, `prog-strength-mcp/README.md`, `prog-strength-infra/README.md`,
   and the agent docs. (The app-config GitHub secrets stay — they seed Secrets Manager.)

## Failure / Rollback

- Through Checkpoints 1–2, SSH remains the live deploy path; any SSM problem is rolled back by
  reverting the single converted workflow.
- In Checkpoint 3, until 22 is closed, reverting a workflow file restores SSH for that repo. The
  point of no easy return is closing the port + deleting the SSH secrets — gated behind a verified
  Session Manager session and green SSM runs across all seven workflows. If SSM deploys regress after
  the port is closed, reopen 22 via a one-line `prod.tfvars` revert and re-add the secrets to recover
  the SSH path.

## Open Questions

1. **Instance targeting in `send-command`: tag filter vs instance-id?** A tag filter (e.g.
   `Name=prog-strength-backend`) fully retires `EC2_HOST` and survives an instance replacement; a
   hardcoded id reintroduces a host-identity value. Lean: tag filter — matches developer's
   `deploy-manager.yml`.
2. **One combined secret per service, or split true-secrets from non-sensitive identities?** Lean: one
   `app-config` JSON blob per service for simplicity (the bucket-name identities are low-sensitivity
   and already in tfvars/config); split later only if a value needs independent rotation.
3. **Keep or drop the EC2 key pair after cutover?** Dropping `ssh_key_name` is cleaner but a future
   instance replacement would come up with Session Manager only — acceptable, but call it out before
   removing. Lean: keep the key pair resource, rely on the closed port; separate cleanup later.

> Resolved during scoping: **seeding method** — infra-owned containers + automated GitHub→Secrets
> Manager seed workflow, values never in TF state (per the user's pattern and the developer-repo
> rule). **Encryption** — default `aws/secretsmanager` KMS key, no customer-managed key at
> single-account scale. **Storage choice** — Secrets Manager over Parameter Store, for consistency
> with the account's existing runtime-secret pattern.
