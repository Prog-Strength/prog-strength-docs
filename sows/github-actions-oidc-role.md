---
status: draft
repos:
  - prog-strength-docs
  - prog-strength-infra
  - prog-strength-api
  - prog-strength-agent
  - prog-strength-mcp
  - prog-strength-developer
---

# Single GitHub Actions OIDC Role for All CI/CD

**Status**: Draft · **Last updated**: 2026-06-12

## Introduction

Every Prog Strength repository whose GitHub Actions touch AWS — `prog-strength-api`, `prog-strength-agent`, `prog-strength-mcp`, and `prog-strength-infra` — authenticates today with the same long-lived IAM user credentials, stored as `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` repository secrets. Long-lived keys are the weakest credential form AWS offers: they never expire, they sit in four copies across GitHub, and a leak from any one repo is a leak of full CI/CD access until someone notices and rotates them manually.

`prog-strength-developer` already proved out the better pattern: GitHub's OIDC provider federates directly into an IAM role, every workflow run gets short-lived credentials (minutes, not forever), and there is nothing static to steal. But it did so with its own repo-local role, which leaves the account with a mixed story — one repo on OIDC with its own role, four repos on shared static keys.

This SOW finishes the migration and consolidates deliberately onto **one shared CI/CD role**: after it ships, the operational stance is *"every Prog Strength repository's GitHub Actions assume the same OIDC role"* — no per-repo role bookkeeping, one trust policy to read, one permission policy to audit — and the IAM user's long-lived keys are deleted.

## Proposed Solution

A new `github_oidc` module in `prog-strength-infra` owns three things:

1. **The GitHub OIDC identity provider** (`token.actions.githubusercontent.com`). It already exists in the account (created outside Terraform; `prog-strength-developer` reads it via a `data` source). A Terraform `import` block adopts it into infra's state so it becomes IaC-managed with no manual CLI step. The developer repo's `data` source lookup is unaffected.

2. **One IAM role, `prog-strength-github-actions`**, trusted by the five AWS-using repos via the OIDC provider. App repos (api, agent, mcp) may assume it only from `main`; infra and developer may additionally assume it from `pull_request` events because their `plan.yml` workflows run on PRs.

3. **One scoped permission policy** — the union of what all five repos' workflows need: ECR push, infra's Terraform surface (EC2/VPC, ECR, the S3 buckets, CloudWatch), developer's platform operations (EC2 instance lifecycle, SSM commands, Secrets Manager describe), and IAM restricted to `prog-strength-*`-prefixed resources. Scoped rather than `AdministratorAccess`: a stolen short-lived token can manage Prog Strength infrastructure but cannot escalate to account takeover.

On the GitHub side, the role ARN lives in a single **organization-level secret** `AWS_GHA_ROLE_ARN` (all repos are public, so org secrets are available on the free plan). Six workflow files swap static-key inputs for `role-to-assume`; `prog-strength-developer` already references `secrets.AWS_GHA_ROLE_ARN`, so it migrates by *deleting its repo-level secret* — repo secrets shadow org secrets, so removal flips it to the shared role with zero workflow edits.

Single-role trade-off, accepted deliberately: any repo's CI can exercise the full union of permissions (e.g., an api workflow could in principle perform Terraform-level actions). The trust policy still constrains *when* (main-branch or repo-PR contexts only), credentials still expire in minutes, and the simplicity of one role to reason about is worth more right now than per-repo blast-radius isolation. Splitting into per-purpose roles later is an additive change — new roles, narrowed trust, re-pointed secrets — with no rework of this design.

## Goals and Non-Goals

### Goals

- All six AWS-using workflows (`api/release.yml`, `agent/release.yml`, `mcp/release.yml`, `infra/plan.yml`, `infra/apply.yml`, `infra/replace-instance.yml`) authenticate via OIDC `role-to-assume` with no static-key inputs.
- `prog-strength-developer`'s four workflows assume the same shared role (via the org secret) and its repo-local `prog-strength-developer-github-actions-role` is removed from its Terraform.
- The OIDC provider and the shared role are fully managed in `prog-strength-infra` Terraform; the provider is imported, not recreated.
- The role's permission policy is scoped to the services and `prog-strength-*` resource prefixes the workflows actually use — no `AdministratorAccess`.
- The IAM user's access keys are deactivated and deleted, and `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` secrets are removed from all four repos.
- Old keys remain valid until every workflow is verified on OIDC, so any step of the rollout can be reverted by reverting a workflow file.

### Non-Goals

- **Per-repo or per-purpose roles.** Explicitly deferred; the single shared role is the chosen stance for now and this SOW does not build the split.
- **Changing the deploy mechanism.** SSH-based deploys (`manual-deploy.yml` in api/agent/mcp, `deploy-caddy.yml` in infra) don't use GitHub-side AWS credentials — the `aws` CLI calls in them run on the EC2 host under its instance role — and are untouched.
- **Touching the EC2 instance role** or any runtime (non-CI) IAM.
- **Workflows with no AWS usage** (`ci.yml`, `eval.yml`, `pr-title.yml`, mobile/web releases) are out of scope.

## Implementation Details

### Terraform: `modules/github_oidc` in prog-strength-infra

- `import` block + `aws_iam_openid_connect_provider` resource for `token.actions.githubusercontent.com` (audience `sts.amazonaws.com`). Import happens through the normal plan/apply pipeline; the plan output for the import must show no destructive diff before merge.
- `aws_iam_role.github_actions` (`prog-strength-github-actions`) with trust policy:
  - Principal: the OIDC provider ARN (Federated).
  - Condition `StringEquals` on `token.actions.githubusercontent.com:aud` = `sts.amazonaws.com`.
  - Condition `StringLike` on `token.actions.githubusercontent.com:sub` matching any of:
    - `repo:Prog-Strength/prog-strength-api:ref:refs/heads/main`
    - `repo:Prog-Strength/prog-strength-agent:ref:refs/heads/main`
    - `repo:Prog-Strength/prog-strength-mcp:ref:refs/heads/main`
    - `repo:Prog-Strength/prog-strength-infra:ref:refs/heads/main`
    - `repo:Prog-Strength/prog-strength-infra:pull_request`
    - `repo:Prog-Strength/prog-strength-developer:ref:refs/heads/main`
    - `repo:Prog-Strength/prog-strength-developer:pull_request`
  - The repo list is a module variable so adding a future repo is a one-line tfvars change.
- Inline scoped policy, derived during implementation from (a) the resources infra's modules manage and (b) `prog-strength-developer/terraform/iam.tf`'s existing role policy. Service-level shape:
  - **ECR**: auth token + push/pull on `prog-strength-prod/*` repositories, plus repository management for infra's `ecr` module.
  - **EC2/VPC**: describe + lifecycle for the instance, launch templates, SGs, subnets, EIP, volumes (covers both infra's compute module and developer's worker instances).
  - **S3**: the Terraform state bucket(s) and the Litestream/TCX/avatar buckets.
  - **IAM**: create/manage roles, policies, instance profiles **only** on `prog-strength-*`-prefixed ARNs, plus read-only `iam:Get*/List*`. The role must *not* be able to edit its own policy without the `prog-strength-*` fence (accepted residual risk of the single-role stance, noted for a future split).
  - **CloudWatch Logs + alarms**, **SSM** (`SendCommand`, `ListCommandInvocations`, parameter reads for AMI lookup), **Secrets Manager** describe on `prog-strength-developer/*`.
- Output: the role ARN.

### Public-repo fork safety

All repos are public, and the trust policy includes `pull_request` subjects for infra and developer. This is safe: on fork PRs, GitHub runs `pull_request` workflows without secrets and without `id-token: write`, so a fork cannot mint an OIDC token with the base repo's `sub` claim. Same-repo PRs (the only ones that get tokens) are author-controlled in a solo org.

### Workflow changes (six files)

In each of `api/release.yml`, `agent/release.yml`, `mcp/release.yml`, `infra/plan.yml`, `infra/apply.yml`, `infra/replace-instance.yml`:

- Add to the AWS-using job: `permissions: { id-token: write, contents: read }` (preserving any extra permissions the job already declares, e.g. for semantic-release or PR comments).
- On `aws-actions/configure-aws-credentials@v4`: delete `aws-access-key-id` / `aws-secret-access-key`, add `role-to-assume: ${{ secrets.AWS_GHA_ROLE_ARN }}`. `aws-region: us-east-2` stays.

`prog-strength-developer`'s four workflows need no edits — see rollout step 4.

### Rollout sequence

Ordered to avoid the chicken-and-egg (the role must exist before anything can assume it) and to keep rollback trivial throughout:

1. **Infra PR** adding the `github_oidc` module. `plan.yml` runs on the old keys; verify the plan shows the provider import as clean and the role as create-only. Merge; `apply.yml` (old keys) creates the role.
2. **Set the org-level secret** `AWS_GHA_ROLE_ARN` to the new role ARN.
3. **Flip infra**: PR converting its three workflows. The PR self-verifies — `pull_request` runs the PR's own `plan.yml`, exercising OIDC before merge.
4. **Flip developer**: delete its repo-level `AWS_GHA_ROLE_ARN` secret; the org secret takes over. Verify with a `plan.yml` run, then a follow-up PR removes the old role + provider data source from `developer/terraform/iam.tf` (its own `apply.yml`, now running as the shared role, deletes the old role).
5. **Flip api, agent, mcp**: one PR each converting `release.yml`. Verify api immediately by re-running release (or a no-op merge); agent/mcp verify on their next release — or force one with `workflow_dispatch` if waiting is undesirable.
6. **Decommission**: after all six workflows have a green OIDC run — delete `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` from all four repos, deactivate the IAM user's access keys, wait a few days for anything unnoticed to break, then delete the keys and the user. **Pre-check before deletion**: confirm the IAM user is not used outside GitHub Actions (e.g., a local AWS CLI profile) via its credential last-used timestamp.

### Failure / rollback

Until step 6, the old keys remain valid: any failed OIDC conversion is rolled back by reverting that repo's workflow file. After step 6, rollback means issuing new keys — which is why step 6 waits for green runs across all six workflows.

## Open Questions

1. **Should infra's `replace-instance.yml` keep `pull_request` trust?** It's `workflow_dispatch`-only from main, so the main-ref subject covers it. Lean: yes, nothing extra needed — listed only to confirm no workflow is dispatch-from-non-main.
2. **Exact IAM action list for the scoped policy.** Deferred to implementation: enumerate from infra's modules and developer's existing policy, then tighten iteratively using the first OIDC plan/apply runs as the test. Lean: start from developer's proven policy + infra's Terraform surface, expand on AccessDenied rather than starting broad.
