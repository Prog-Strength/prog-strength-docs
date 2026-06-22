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
existing OIDC role; app secrets live in **SSM Parameter Store** and are fetched on the host via its
instance role; **port 22 is closed**; and `EC2_SSH_KEY` / `EC2_HOST` are deleted. Operationally:
*"nothing with a shell credential or an open SSH port touches the prod host; CI talks to it only
through SSM, scoped by IAM and logged in CloudTrail."*

## Why this is a clean fit (not a rewrite)

Much of the groundwork already exists, which is why this is hardening rather than re-architecture:

- **The OIDC role already has the SSM permissions.** Per `github-actions-oidc-role.md`, the shared
  `prog-strength-github-actions` role's policy already grants `ssm:SendCommand`,
  `ssm:ListCommandInvocations`, and parameter reads. The GitHub side needs no new IAM.
- **The host already authenticates to AWS by instance role, not static keys.** Inside today's SSH
  session, ECR login already uses the instance metadata service (`aws ecr get-login-password` with no
  static creds). Fetching secrets from Parameter Store the same way is the identical pattern.
- **Infra already owns all on-host orchestration.** The host clones only `prog-strength-infra` on
  first boot (`bootstrap.infra_repo_url`); compose files, the Caddyfile, monitoring, and litestream
  config all live there. The deploy *scripts* belong there too — moving the inline workflow bash into
  versioned scripts in infra is a natural consolidation, not a new concept.
- **The instance role is designed for additive attachments.** `modules/compute/iam.tf` owns the role
  and exposes `instance_role_name`; domain modules hang policies off it (ECR pull, Litestream S3).
  Adding SSM is one more attachment in the same shape — see `modules/ecr/main.tf`'s
  `instance_ecr_pull`.

## Proposed Solution

### Deploy transport: SSM Run Command instead of SSH

Each workflow's deploy step becomes: assume the OIDC role (already wired via
`aws-actions/configure-aws-credentials@v4`), then `aws ssm send-command` targeting the instance and
invoking a **versioned deploy script that already lives on the host** (under the
`prog-strength-infra` checkout). The runner passes only **non-secret** parameters — chiefly the
released version tag. The command is run synchronously (poll `ListCommandInvocations` /
`get-command-invocation` until terminal) so the workflow reflects the real deploy exit status, the
same guarantee `set -e` gives today.

The SSM agent dials *out* to AWS over 443, so SSM needs **no inbound port** — this is what lets us
close 22 entirely at the end.

### Secrets: Parameter Store, fetched on the host

This is the substantive piece. Today ~16 app secrets (in api's `release.yml` alone:
`JWT_SIGNING_KEY`, `GOOGLE_CLIENT_*`, `CALENDAR_TOKEN_ENC_KEY`, the bucket identities, `ADMIN_EMAILS`,
Grafana creds, nutrition + vector-memory provider keys, …) are forwarded over the SSH channel and
written to `.env` on the host.

They **cannot** simply move to SSM command parameters: Run Command records its parameters in command
history and CloudTrail, so passing secrets that way would *leak* them. Instead:

- Each secret becomes an SSM Parameter Store **SecureString** under a `/prog-strength/prod/` prefix.
- The on-host deploy script reads them via the instance role (`aws ssm get-parameters-by-path
  --with-decryption`) and writes `.env` locally — exactly where today's script writes it.
- The runner never sees the secret values at all.

This is **strictly better than today**, where every secret transits the GitHub runner on every
deploy. It also deletes the long `env:`/`envs:` block from each release workflow.

Provider keys that are genuinely optional today (FatSecret, USDA, OpenAI/Anthropic — they deploy as
empty strings when unset) stay optional: a missing parameter resolves to empty, preserving the
current degrade-to-503 behavior.

### Instance role additions (infra)

A new SSM concern (either a small `modules/ssm` or an attachment block mirroring
`modules/ecr/main.tf`) hangs two things off `instance_role_name`:

1. **`AmazonSSMManagedInstanceCore`** (AWS-managed) so the agent registers the instance as a managed
   node and can receive commands.
2. **A scoped inline policy**: `ssm:GetParameter*` on `arn:…:parameter/prog-strength/prod/*`, plus
   `kms:Decrypt` on the key backing the SecureStrings (the AWS-managed `alias/aws/ssm` key needs no
   explicit grant for same-account use; a customer-managed key would).

Scope to the `/prog-strength/prod/*` path rather than account-wide, consistent with the prefix-fence
posture from the OIDC SOW.

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
- App secrets live in SSM Parameter Store SecureStrings under `/prog-strength/prod/` and are fetched
  on the host by its instance role; no secret transits the GitHub runner.
- The EC2 instance role carries `AmazonSSMManagedInstanceCore` + a parameter-read policy scoped to
  `/prog-strength/prod/*`; the instance shows as a managed node in SSM.
- Inbound port 22 is closed in `prod.tfvars`; AWS Session Manager is the verified break-glass shell.
- `EC2_SSH_KEY` and `EC2_HOST` org secrets are deleted and removed from all deploy docs.
- The migration is reversible at every checkpoint until the final cutover (old SSH path stays valid
  until SSM is proven).

### Non-Goals

- **No new GitHub-side IAM.** The OIDC role already grants the needed SSM actions; if a gap surfaces
  it is a one-line policy expansion in infra, not a redesign.
- **Not changing what a deploy *does*** — same `docker compose pull/down/up`, same compose files,
  same image-from-ECR flow. Only the transport and the secret source change.
- **Not migrating non-secret config.** `config.toml` (`centralized-api-config.md`) is unaffected.
- **Not touching `prog-strength-developer`'s deploys** — it does not use the SSH deploy secrets.
- **Not removing the EC2 key pair resource** in this SOW (optional later cleanup; out of scope to
  avoid an instance replacement).

## Implementation Details

### On-host deploy scripts (prog-strength-infra)

Move the inline bash from each workflow into versioned scripts in infra, e.g.
`deploy/api.sh`, `deploy/mcp.sh`, `deploy/agent.sh`, `deploy/caddy-reload.sh`. Each script:

1. `cd /home/ubuntu/prog-strength-infra && git fetch --prune && git checkout main && git pull --ff-only`
   (unchanged from today).
2. ECR login via instance role (unchanged).
3. **New:** fetch secrets with `aws ssm get-parameters-by-path --path /prog-strength/prod/
   --with-decryption --recursive` and render `.env` (`umask 077`, same target dir as today).
4. `docker compose pull/down/up` with the same `-f` merge of the monitoring compose file; echo +
   `ps` + `logs --tail` as now. Keep `set -euo pipefail`.

The version tag arrives as an environment variable/parameter from the SSM invocation (non-secret).

### SSM invocation (each release/deploy workflow)

Replace the `appleboy/ssh-action` step with:

- `aws-actions/configure-aws-credentials@v4` with `role-to-assume: ${{ secrets.AWS_GHA_ROLE_ARN }}`
  and `id-token: write` (the `build_and_push` job already does exactly this — copy the pattern).
- `aws ssm send-command --document-name AWS-RunShellScript --instance-ids <id>` (resolve the instance
  by tag filter rather than hardcoding, so `EC2_HOST` is fully retired) with parameters running the
  on-host script and passing the version tag. Poll to completion and fail on non-`Success` status.

`deploy-caddy.yml` follows the same shape, invoking `deploy/caddy-reload.sh`.

### Parameter Store seeding

Seeding the SecureStrings is a one-time operation from the current secret values. Options, decide at
implementation: (a) `terraform`-managed `aws_ssm_parameter` resources with values injected at apply
(keeps them in state — acceptable for SecureString in this single-account setup, matches how secrets
already flow through Terraform-owned identities), or (b) a one-shot CLI seed documented in the infra
README, keeping values out of state. Lean (b) for the true secrets to avoid plaintext-in-state, (a)
for non-sensitive identities already in tfvars. Resolve in Open Questions.

### SSM agent presence

Recent Ubuntu EC2 AMIs ship the SSM agent preinstalled via snap. Confirm it is running on the live
host (`snap services amazon-ssm-agent` / managed-node status in the SSM console) and, if `bootstrap.sh`
provisions the host from a base image, ensure the agent is enabled there so a future
`replace-instance` run comes up SSM-ready.

## Rollout Sequence

Three checkpoints, each independently verifiable and reversible. The SSH path stays fully functional
until Checkpoint 3.

### Checkpoint 1 — Parameter Store + instance role (no behavior change yet)

1. Infra PR: SSM attachment on the instance role (`AmazonSSMManagedInstanceCore` + scoped parameter
   read) and confirm/enable the SSM agent. Verify the instance appears as a managed node.
2. Seed `/prog-strength/prod/*` SecureStrings from current secret values.
3. **Verify:** from the host, `aws ssm get-parameters-by-path --with-decryption` returns the expected
   keys. Deploys still run over SSH — nothing user-visible changed. Fully revertible (drop the
   attachment, delete the parameters).

### Checkpoint 2 — SSM deploy path, alongside SSH

1. Infra PR: add the `deploy/*.sh` scripts (secret fetch from Parameter Store + existing compose
   logic).
2. Convert **one** workflow first (suggest `agent/manual-deploy.yml` — lowest blast radius,
   dispatchable on demand) to the SSM transport. Trigger it; confirm a green deploy with the service
   healthy and `.env` correctly rendered from Parameter Store.
3. **Verify before proceeding:** the SSM-deployed service is byte-for-byte equivalent in behavior to
   an SSH deploy. If anything is off, revert the single workflow — SSH is untouched everywhere else.

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
   and the agent docs.

## Failure / Rollback

- Through Checkpoints 1–2, SSH remains the live deploy path; any SSM problem is rolled back by
  reverting the single converted workflow.
- In Checkpoint 3, until 22 is closed, reverting a workflow file restores SSH for that repo. The
  point of no easy return is closing the port + deleting the secrets — gated behind a verified
  Session Manager session and green SSM runs across all seven workflows. If SSM deploys regress after
  the port is closed, reopen 22 via a one-line `prod.tfvars` revert and re-add the secrets to recover
  the SSH path.

## Open Questions

1. **Parameter Store seeding: Terraform-managed vs one-shot CLI?** Lean: CLI seed for true secrets
   (keeps plaintext out of TF state), Terraform `aws_ssm_parameter` for the non-sensitive identities
   already in tfvars. Confirm at implementation.
2. **Instance targeting in `send-command`: tag filter vs instance-id?** A tag filter (e.g.
   `Name=prog-strength-backend`) fully retires `EC2_HOST` and survives an instance replacement; a
   hardcoded id is simpler but reintroduces a host-identity secret. Lean: tag filter.
3. **Customer-managed KMS key for SecureStrings, or the default `aws/ssm` key?** Default key needs no
   extra grant and is fine for a single-account beta (consistent with the ECR module's "managed key
   is fine at this scale" stance). Lean: default key; revisit if multi-account ever lands.
4. **Keep or drop the EC2 key pair after cutover?** Dropping `ssh_key_name` is cleaner but a future
   instance replacement would come up with no key at all (Session Manager only) — acceptable, but
   call it out explicitly before removing. Lean: keep the key pair resource, rely on the closed port;
   separate cleanup later.
