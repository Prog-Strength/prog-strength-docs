# Single GitHub Actions OIDC CI/CD Role Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace long-lived AWS access keys in all CI/CD with one shared GitHub Actions OIDC IAM role (`prog-strength-github-actions`) managed in prog-strength-infra, then delete the static keys.

**Architecture:** A new `github_oidc` module in prog-strength-infra imports the account's existing GitHub OIDC provider and creates one IAM role trusted by five repos (api/agent/mcp from `main`; infra/developer from `main` + `pull_request`). The role's inline policy is the union of infra's Terraform surface, ECR push, and prog-strength-developer's platform operations, with IAM fenced to `prog-strength-*` prefixes. Six workflows swap static keys for `role-to-assume`; developer flips by deleting its repo-level secret (org secret takes over) and its repo-local role is destroyed.

**Tech Stack:** Terraform 1.13 / AWS provider ~> 6.0, GitHub Actions, `aws-actions/configure-aws-credentials@v4`, `gh` CLI.

**Spec:** `prog-strength-docs/sows/github-actions-oidc-role.md`

**Repo paths** (all under `/Users/jimmywallace/Desktop/prog-strength/repos/`): `prog-strength-infra`, `prog-strength-api`, `prog-strength-agent`, `prog-strength-mcp`, `prog-strength-developer`, `prog-strength-docs`.

**Ordering is load-bearing.** Tasks 1–3 create the role using the old keys (bootstrap). Task 4 publishes the org secret. Tasks 5–9 flip repos one at a time with the old keys still valid as rollback. Task 10 deletes the keys only after every workflow has a green OIDC run. Do not reorder.

**Prerequisite check (before Task 1):** local AWS CLI credentials with IAM read access, and `gh` authenticated with org-admin rights (needed for `gh secret set --org` in Task 4). Verify: `aws sts get-caller-identity` and `gh auth status` both succeed.

---

### Task 1: `github_oidc` module in prog-strength-infra

**Files:**
- Create: `prog-strength-infra/modules/github_oidc/variables.tf`
- Create: `prog-strength-infra/modules/github_oidc/main.tf`
- Create: `prog-strength-infra/modules/github_oidc/outputs.tf`

- [ ] **Step 1: Create a feature branch**

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-infra
git checkout main && git pull --ff-only
git checkout -b feat/github-oidc-role
```

- [ ] **Step 2: Fetch the existing OIDC provider's ARN and thumbprints**

The provider already exists in the account (created outside Terraform; prog-strength-developer reads it via a `data` source). The resource definition must match it exactly so the import produces no destructive diff.

```bash
aws iam list-open-id-connect-providers
# Expected: one entry like
#   arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com
aws iam get-open-id-connect-provider \
  --open-id-connect-provider-arn arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com
# Note the ThumbprintList values and confirm ClientIDList contains "sts.amazonaws.com".
```

Record `<ACCOUNT_ID>` and the thumbprint values — they are used in Step 3 (thumbprints) and Task 2 (import block). Every later occurrence of `<ACCOUNT_ID>` in this plan means this value.

- [ ] **Step 3: Write `modules/github_oidc/variables.tf`**

```hcl
variable "aws_region" {
  description = "Region used to build resource ARNs in the role policy."
  type        = string
}

variable "github_org" {
  description = "GitHub organization the trusted repositories live under."
  type        = string
  default     = "Prog-Strength"
}

variable "main_branch_repos" {
  description = "Repositories whose Actions may assume the role from pushes to main (and workflow_dispatch runs on main)."
  type        = list(string)
  default = [
    "prog-strength-api",
    "prog-strength-agent",
    "prog-strength-mcp",
    "prog-strength-infra",
    "prog-strength-developer",
  ]
}

variable "pull_request_repos" {
  description = "Repositories whose Actions may additionally assume the role on pull_request events (terraform plan on PRs). GitHub does not issue OIDC tokens to fork PRs, so this only ever matches same-repo PRs."
  type        = list(string)
  default = [
    "prog-strength-infra",
    "prog-strength-developer",
  ]
}

variable "oidc_thumbprints" {
  description = "Thumbprint list for the GitHub OIDC provider. Must match the existing provider exactly so the import is a no-op; fetch with `aws iam get-open-id-connect-provider`."
  type        = list(string)
}

variable "role_name" {
  description = "Name of the shared CI/CD role. Stays inside the prog-strength-* IAM fence so CI can apply updates to its own policy."
  type        = string
  default     = "prog-strength-github-actions"
}
```

- [ ] **Step 4: Write `modules/github_oidc/main.tf`**

```hcl
# --- Shared GitHub Actions OIDC role for ALL Prog Strength CI/CD ------------
#
# One role, assumed by every repository's workflows via GitHub's OIDC
# provider — the deliberate operational stance is "every repo's GHA uses
# the same CI/CD role" (see prog-strength-docs/sows/github-actions-oidc-role.md).
# Short-lived federated credentials replace the long-lived IAM user keys
# that previously sat in four repos' secrets.
#
# The permission policy is the union of what the workflows do:
#   - terraform plan/apply for this repo's stack
#   - ECR image push from api/agent/mcp releases
#   - prog-strength-developer's platform ops (EC2 workers, SSM, secrets reads)
# Mutating IAM access is fenced to prog-strength-* resource prefixes so a
# stolen CI token can manage this project's infrastructure but cannot
# escalate to account takeover.

data "aws_caller_identity" "current" {}

# The provider predates this module (created manually; also read by
# prog-strength-developer's data source). It is IMPORTED, not recreated —
# see the import block in the root module's imports.tf.
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = var.oidc_thumbprints
}

data "aws_iam_policy_document" "trust" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }
    # main-branch contexts for every trusted repo, plus pull_request
    # contexts only where terraform plan runs on PRs. Fork PRs never get
    # OIDC tokens from GitHub, so the pull_request subjects are only
    # mintable from same-repo branches.
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values = concat(
        [for r in var.main_branch_repos : "repo:${var.github_org}/${r}:ref:refs/heads/main"],
        [for r in var.pull_request_repos : "repo:${var.github_org}/${r}:pull_request"],
      )
    }
  }
}

resource "aws_iam_role" "github_actions" {
  name               = var.role_name
  assume_role_policy = data.aws_iam_policy_document.trust.json
}

data "aws_iam_policy_document" "permissions" {
  # EC2/VPC: this repo's terraform owns the whole network + instance stack
  # (VPC, subnets, IGW, route tables, SGs, instance, EIP, volumes);
  # developer CI runs/terminates worker instances and manages launch
  # templates. Service-level grant — most EC2 networking actions don't
  # honor resource-ARN conditions cleanly (same reasoning as the
  # EC2ManagerInfra statement in developer's previous role).
  statement {
    sid       = "EC2"
    actions   = ["ec2:*"]
    resources = ["*"]
  }

  # GetAuthorizationToken only works against "*".
  statement {
    sid       = "ECRAuth"
    actions   = ["ecr:GetAuthorizationToken"]
    resources = ["*"]
  }

  # Image push from release workflows + repository/lifecycle management
  # from this repo's ecr module. Fenced to prog-strength-* repositories
  # (covers prog-strength-prod/* and any future environment prefix).
  statement {
    sid       = "ECRRepositories"
    actions   = ["ecr:*"]
    resources = ["arn:aws:ecr:${var.aws_region}:${data.aws_caller_identity.current.account_id}:repository/prog-strength-*"]
  }

  # Terraform state (prog-strength-terraform-backend — used by both infra
  # and developer stacks) plus the litestream/tcx/avatar data buckets this
  # repo manages. Fenced by bucket name prefix.
  statement {
    sid     = "S3"
    actions = ["s3:*"]
    resources = [
      "arn:aws:s3:::prog-strength-*",
      "arn:aws:s3:::prog-strength-*/*",
    ]
  }

  # Read-only IAM everywhere: terraform refresh of roles, profiles,
  # policies, and the OIDC provider data source in developer's stack.
  statement {
    sid       = "IAMRead"
    actions   = ["iam:Get*", "iam:List*"]
    resources = ["*"]
  }

  # Mutating IAM fenced to prog-strength-* names. Covers this repo's
  # instance role/profile/policies, developer's worker+manager roles, and
  # this role itself (so future policy updates apply from CI rather than
  # requiring an admin re-bootstrap — the trust policy already restricts
  # who can assume the role, so self-PutRolePolicy adds no new principal).
  statement {
    sid = "IAMManageProjectResources"
    actions = [
      "iam:CreateRole",
      "iam:DeleteRole",
      "iam:UpdateAssumeRolePolicy",
      "iam:PutRolePolicy",
      "iam:DeleteRolePolicy",
      "iam:AttachRolePolicy",
      "iam:DetachRolePolicy",
      "iam:TagRole",
      "iam:UntagRole",
      "iam:CreatePolicy",
      "iam:DeletePolicy",
      "iam:CreatePolicyVersion",
      "iam:DeletePolicyVersion",
      "iam:TagPolicy",
      "iam:UntagPolicy",
      "iam:CreateInstanceProfile",
      "iam:DeleteInstanceProfile",
      "iam:AddRoleToInstanceProfile",
      "iam:RemoveRoleFromInstanceProfile",
      "iam:TagInstanceProfile",
      "iam:UntagInstanceProfile",
    ]
    resources = [
      "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/prog-strength-*",
      "arn:aws:iam::${data.aws_caller_identity.current.account_id}:instance-profile/prog-strength-*",
      "arn:aws:iam::${data.aws_caller_identity.current.account_id}:policy/prog-strength-*",
    ]
  }

  # PassRole fenced the same way: the backend instance role and
  # developer's worker/manager roles are passed to EC2 at apply time.
  statement {
    sid       = "IAMPassProjectRoles"
    actions   = ["iam:PassRole"]
    resources = ["arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/prog-strength-*"]
  }

  # Drift management on the imported provider only. Deliberately NO
  # Create/Delete — CI must never be able to delete the provider that
  # its own trust depends on.
  statement {
    sid = "OIDCProviderManage"
    actions = [
      "iam:UpdateOpenIDConnectProviderThumbprint",
      "iam:TagOpenIDConnectProvider",
      "iam:UntagOpenIDConnectProvider",
      "iam:AddClientIDToOpenIDConnectProvider",
      "iam:RemoveClientIDFromOpenIDConnectProvider",
    ]
    resources = [aws_iam_openid_connect_provider.github.arn]
  }

  # Log groups + retention from this repo's logging module; stream reads
  # for developer's terraform refresh.
  statement {
    sid       = "CloudWatchLogs"
    actions   = ["logs:*"]
    resources = ["*"]
  }

  # The logging module's EstimatedCharges alarm.
  statement {
    sid = "CloudWatchAlarms"
    actions = [
      "cloudwatch:DescribeAlarms",
      "cloudwatch:PutMetricAlarm",
      "cloudwatch:DeleteAlarms",
      "cloudwatch:ListTagsForResource",
      "cloudwatch:TagResource",
      "cloudwatch:UntagResource",
    ]
    resources = ["*"]
  }

  # AWS-published AMI parameters (developer's al2023 lookup).
  statement {
    sid       = "SSMParameterRead"
    actions   = ["ssm:GetParameter", "ssm:GetParameters"]
    resources = ["arn:aws:ssm:${var.aws_region}::parameter/aws/service/*"]
  }

  # developer's deploy-manager.yml pushes compose updates to the manager
  # instance via SSM RunCommand and polls the invocation.
  statement {
    sid = "SSMSendCommand"
    actions = [
      "ssm:SendCommand",
      "ssm:ListCommandInvocations",
      "ssm:GetCommandInvocation",
    ]
    resources = [
      "arn:aws:ec2:${var.aws_region}:${data.aws_caller_identity.current.account_id}:instance/*",
      "arn:aws:ssm:${var.aws_region}::document/AWS-RunShellScript",
      "arn:aws:ssm:${var.aws_region}:${data.aws_caller_identity.current.account_id}:*",
    ]
  }

  # developer's terraform refreshes two data "aws_secretsmanager_secret"
  # blocks each plan. Describe only — CI never reads secret values.
  statement {
    sid = "SecretsManagerDescribe"
    actions = [
      "secretsmanager:DescribeSecret",
      "secretsmanager:GetResourcePolicy",
    ]
    resources = [
      "arn:aws:secretsmanager:${var.aws_region}:${data.aws_caller_identity.current.account_id}:secret:prog-strength-developer/*",
    ]
  }
}

resource "aws_iam_role_policy" "github_actions" {
  name   = "${var.role_name}-inline"
  role   = aws_iam_role.github_actions.id
  policy = data.aws_iam_policy_document.permissions.json
}
```

- [ ] **Step 5: Write `modules/github_oidc/outputs.tf`**

```hcl
output "role_arn" {
  description = "ARN of the shared CI/CD role. Set as the org-level AWS_GHA_ROLE_ARN secret on GitHub."
  value       = aws_iam_role.github_actions.arn
}

output "oidc_provider_arn" {
  description = "ARN of the (imported) GitHub OIDC identity provider."
  value       = aws_iam_openid_connect_provider.github.arn
}
```

- [ ] **Step 6: Format check**

```bash
terraform fmt -check -recursive
```
Expected: no output, exit 0. If files are listed, run `terraform fmt -recursive` and re-check.

- [ ] **Step 7: Commit**

```bash
git add modules/github_oidc/
git commit -m "feat: add github_oidc module with the shared CI/CD role

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: Wire the module into the infra root

**Files:**
- Modify: `prog-strength-infra/main.tf` (append module block)
- Modify: `prog-strength-infra/variables.tf` (append variable)
- Modify: `prog-strength-infra/outputs.tf` (append output)
- Modify: `prog-strength-infra/environments/prod.tfvars` (append section)
- Create: `prog-strength-infra/imports.tf`

- [ ] **Step 1: Append to `variables.tf`**

Thumbprints land in tfvars (next steps) so a GitHub rotation is a one-line config change.

```hcl
variable "github_oidc" {
  description = "Shared GitHub Actions OIDC CI/CD role. One role for every Prog Strength repo's workflows — see prog-strength-docs/sows/github-actions-oidc-role.md. oidc_thumbprints must match the existing (imported) provider; fetch with `aws iam get-open-id-connect-provider`."
  type = object({
    oidc_thumbprints = list(string)
  })
}
```

- [ ] **Step 2: Append to `main.tf`**

```hcl
module "github_oidc" {
  source = "./modules/github_oidc"

  aws_region       = var.aws.region
  oidc_thumbprints = var.github_oidc.oidc_thumbprints
}
```

(`github_org`, the repo lists, and `role_name` use the module defaults; override here only when the repo set changes.)

- [ ] **Step 3: Append to `environments/prod.tfvars`**

Use the actual thumbprint values recorded in Task 1 Step 2:

```hcl
# --- GitHub Actions OIDC (shared CI/CD role) ---------------------------------

github_oidc = {
  # Must match the existing provider exactly (import is a no-op then).
  # GitHub's OIDC TLS chain; AWS treats these as advisory for
  # token.actions.githubusercontent.com but the API requires the field.
  oidc_thumbprints = ["<THUMBPRINT_1>", "<THUMBPRINT_2_IF_PRESENT>"]
}
```

- [ ] **Step 4: Create `imports.tf`** (with the real `<ACCOUNT_ID>` from Task 1 Step 2)

```hcl
# Adopts the pre-existing GitHub OIDC identity provider (created manually,
# before this module) into state. After the first successful apply this
# block is inert; it stays as a record that the provider was imported,
# not created, by this stack. prog-strength-developer reads the same
# provider via a data source — unaffected by the import.
import {
  to = module.github_oidc.aws_iam_openid_connect_provider.github
  id = "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
}
```

- [ ] **Step 5: Append to `outputs.tf`**

```hcl
output "github_actions_role_arn" {
  description = "Shared CI/CD role assumed by every repo's GitHub Actions. Set as the org-level AWS_GHA_ROLE_ARN secret."
  value       = module.github_oidc.role_arn
}
```

- [ ] **Step 6: Validate locally**

```bash
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
```
Expected: `Success! The configuration is valid.` (`-backend=false` avoids needing state access locally; the real init runs in CI.)

- [ ] **Step 7 (optional but recommended): Local plan preview**

If local AWS credentials can read the state bucket:

```bash
terraform init
terraform plan -var-file=environments/prod.tfvars
```
Expected summary: `Plan: 1 to import, 2 to add, 0 to change, 0 to destroy.` (import: provider; add: role + inline policy). If the provider shows an in-place update on thumbprints/client IDs, fix the tfvars values to match the real provider before pushing.

- [ ] **Step 8: Commit**

```bash
git add main.tf variables.tf outputs.tf environments/prod.tfvars imports.tf
git commit -m "feat: wire github_oidc module into root and import the existing provider

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: Infra PR — create the role (runs on the OLD keys)

This PR's `plan.yml` still authenticates with the static keys — that is the bootstrap, by design.

- [ ] **Step 1: Push and open the PR**

```bash
git push -u origin feat/github-oidc-role
gh pr create --title "feat: shared GitHub Actions OIDC CI/CD role" --body "Adds the github_oidc module: imports the existing GitHub OIDC provider and creates prog-strength-github-actions, the single CI/CD role all repos' workflows will assume. Implements prog-strength-docs/sows/github-actions-oidc-role.md (bootstrap step — workflows still on static keys until the role exists).

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

- [ ] **Step 2: Verify the plan comment**

```bash
gh pr checks --watch
```
Then read the sticky "Terraform Plan" comment on the PR. Required: `1 to import, 2 to add, 0 to change, 0 to destroy` and the trust policy JSON shows all seven expected `sub` values (5 main-branch + 2 pull_request). **Anything in the destroy column = stop and fix.**

- [ ] **Step 3: Merge and watch the apply**

```bash
gh pr merge --squash --delete-branch
gh run watch $(gh run list --workflow=apply.yml --limit 1 --json databaseId --jq '.[0].databaseId')
```
Expected: apply succeeds; output shows `Apply complete! ... 1 imported, 2 added`.

- [ ] **Step 4: Record the role ARN**

```bash
aws iam get-role --role-name prog-strength-github-actions --query 'Role.Arn' --output text
```
Expected: `arn:aws:iam::<ACCOUNT_ID>:role/prog-strength-github-actions`

---

### Task 4: Publish the org-level secret

- [ ] **Step 1: Set the org secret** (all repos are public, so org secrets work on the free plan)

```bash
gh secret set AWS_GHA_ROLE_ARN --org Prog-Strength --visibility all \
  --body "arn:aws:iam::<ACCOUNT_ID>:role/prog-strength-github-actions"
```
If this fails with a scope error: `gh auth refresh -h github.com -s admin:org` and retry.

- [ ] **Step 2: Verify**

```bash
gh secret list --org Prog-Strength
```
Expected: `AWS_GHA_ROLE_ARN` with visibility `all`.

**Note:** prog-strength-developer still has a repo-level `AWS_GHA_ROLE_ARN` that shadows the org secret — that is correct until Task 6.

---

### Task 5: Flip prog-strength-infra workflows to OIDC

**Files:**
- Modify: `prog-strength-infra/.github/workflows/plan.yml:12-32`
- Modify: `prog-strength-infra/.github/workflows/apply.yml:8-25`
- Modify: `prog-strength-infra/.github/workflows/replace-instance.yml:36-68`

- [ ] **Step 1: Branch**

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-infra
git checkout main && git pull --ff-only
git checkout -b ci/oidc-auth
```

- [ ] **Step 2: Edit `plan.yml`**

Permissions block (lines 12–14) becomes:

```yaml
permissions:
  contents: read
  pull-requests: write
  id-token: write   # mint the OIDC token for configure-aws-credentials
```

Credentials step (lines 28–32) becomes:

```yaml
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_GHA_ROLE_ARN }}
          aws-region: us-east-2
```

- [ ] **Step 3: Edit `apply.yml`**

Permissions block (lines 8–9) becomes:

```yaml
permissions:
  contents: read
  id-token: write   # mint the OIDC token for configure-aws-credentials
```

Credentials step (lines 21–25) becomes:

```yaml
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_GHA_ROLE_ARN }}
          aws-region: us-east-2
```

- [ ] **Step 4: Edit `replace-instance.yml`**

Permissions block (lines 36–37) becomes:

```yaml
permissions:
  contents: read
  id-token: write   # mint the OIDC token for configure-aws-credentials
```

Credentials step (lines 64–68) becomes:

```yaml
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_GHA_ROLE_ARN }}
          aws-region: us-east-2
```

- [ ] **Step 5: Commit, push, open PR — the PR is the OIDC test**

`pull_request` runs use the PR branch's workflow file, so this PR's own plan run exercises the new role before merge.

```bash
git add .github/workflows/plan.yml .github/workflows/apply.yml .github/workflows/replace-instance.yml
git commit -m "ci: assume the shared GitHub Actions OIDC role instead of static keys

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
git push -u origin ci/oidc-auth
gh pr create --title "ci: assume the shared OIDC role instead of static keys" --body "Swaps AWS_ACCESS_KEY_ID/AWS_SECRET_ACCESS_KEY for role-to-assume on the shared prog-strength-github-actions role. This PR's own plan run is the OIDC verification (pull_request runs use the PR's workflow file).

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
gh pr checks --watch
```
Expected: plan check green, sticky comment shows `No changes.` If plan fails with `Not authorized to perform sts:AssumeRoleWithWebIdentity` → trust policy `sub` mismatch; if it fails mid-plan with `AccessDenied` on a specific action → add that action to the module policy (new infra PR, still applyable via old-key apply.yml on main).

- [ ] **Step 6: Merge and verify apply on OIDC**

```bash
gh pr merge --squash --delete-branch
gh run watch $(gh run list --workflow=apply.yml --limit 1 --json databaseId --jq '.[0].databaseId')
```
Expected: apply run green (a no-change apply — workflow files aren't Terraform inputs), authenticated via OIDC.

---

### Task 6: Flip prog-strength-developer and decommission its repo-local role

**Files:**
- Modify: `prog-strength-developer/terraform/iam.tf` (delete lines 90–361: everything from the `# GitHub Actions OIDC role` banner to end of file)
- Modify: `prog-strength-developer/terraform/outputs.tf:21-24` (delete the `github_actions_role_arn` output)
- Modify: `prog-strength-developer/terraform/variables.tf:43-47` (delete the `github_actions_repo` variable)
- No workflow edits: all four workflows already use `role-to-assume: ${{ secrets.AWS_GHA_ROLE_ARN }}`.

- [ ] **Step 1: Delete the repo-level secret so the org secret takes over**

```bash
gh secret delete AWS_GHA_ROLE_ARN -R Prog-Strength/prog-strength-developer
gh secret list -R Prog-Strength/prog-strength-developer
```
Expected: `AWS_GHA_ROLE_ARN` no longer listed at repo level. Rollback at any point before Step 5's merge: `gh secret set AWS_GHA_ROLE_ARN -R Prog-Strength/prog-strength-developer --body "<old developer role ARN>"` (fetch via `aws iam get-role --role-name prog-strength-developer-github-actions-role` — it exists until the apply in Step 5).

- [ ] **Step 2: Branch and remove the old role from Terraform**

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-developer
git checkout main && git pull --ff-only
git checkout -b ci/shared-oidc-role
```

In `terraform/iam.tf`, delete everything from the comment banner at line 90 (`# GitHub Actions OIDC role: assumed by the dispatch-sow workflow.`) through the end of the file — i.e., these blocks:
- `data "aws_iam_openid_connect_provider" "github"`
- `data "aws_iam_policy_document" "github_actions_trust"`
- `resource "aws_iam_role" "github_actions"`
- `data "aws_iam_policy_document" "github_actions_inline"`
- `resource "aws_iam_role_policy" "github_actions_inline"`

The worker/manager sections (lines 1–88) stay untouched.

In `terraform/outputs.tf`, delete:

```hcl
output "github_actions_role_arn" {
  description = "ARN of the role GitHub Actions assumes via OIDC."
  value       = aws_iam_role.github_actions.arn
}
```
(Exact text may differ slightly — delete the whole `github_actions_role_arn` output block at lines 21–24.)

In `terraform/variables.tf`, delete:

```hcl
variable "github_actions_repo" {
  description = "The repo whose Actions workflow is allowed to assume the GHA OIDC role. Restricts the trust policy."
  type        = string
  default     = "Prog-Strength/prog-strength-developer"
}
```

- [ ] **Step 3: Verify nothing else references the deleted symbols**

```bash
grep -rn "github_actions\|github_actions_repo" terraform/ && echo "FAIL: references remain" || echo "OK: no references"
```
Expected: `OK: no references`. (Workflow files still legitimately reference the *secret* `AWS_GHA_ROLE_ARN`; this grep only covers `terraform/`.)

- [ ] **Step 4: Validate and push the PR — its plan run verifies the shared role**

```bash
cd terraform && terraform fmt -check && terraform init -backend=false && terraform validate && cd ..
git add terraform/iam.tf terraform/outputs.tf terraform/variables.tf
git commit -m "ci: adopt the shared prog-strength-github-actions OIDC role

The repo-local GHA role is superseded by the org-wide role defined in
prog-strength-infra (sows/github-actions-oidc-role.md). The repo-level
AWS_GHA_ROLE_ARN secret is deleted so the org-level secret resolves.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
git push -u origin ci/shared-oidc-role
gh pr create --title "ci: adopt the shared OIDC CI/CD role" --body "Removes the repo-local GHA role/trust/policy from terraform; workflows now resolve AWS_GHA_ROLE_ARN from the org-level secret (repo secret deleted). This PR's plan run is the verification that the shared role works for this stack.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
gh pr checks --watch
```
Expected: plan green **while authenticated as the shared role**, showing `0 to add, 0 to change, 2 to destroy` (old role + its inline policy). An `AccessDenied` during refresh means the shared role's policy is missing an action developer's terraform needs — add it to the module in prog-strength-infra and apply before retrying.

- [ ] **Step 5: Merge; the apply (as the shared role) deletes the old role**

```bash
gh pr merge --squash --delete-branch
gh run watch $(gh run list --workflow=apply.yml -R Prog-Strength/prog-strength-developer --limit 1 --json databaseId --jq '.[0].databaseId')
aws iam get-role --role-name prog-strength-developer-github-actions-role 2>&1
```
Expected: apply green with `2 destroyed`; the final command returns `NoSuchEntity`.

---

### Task 7: Flip prog-strength-api release workflow

**Files:**
- Modify: `prog-strength-api/.github/workflows/release.yml:49-66`

- [ ] **Step 1: Branch and edit**

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-api
git checkout main && git pull --ff-only
git checkout -b ci/oidc-auth
```

The `build_and_push` job header (lines 49–55) gains a job-level permissions block — the top-level `contents: write / issues: write / pull-requests: write` stays for the `release` job (semantic-release needs it), while this job needs only checkout + an OIDC token:

```yaml
  build_and_push:
    needs: release
    if: needs.release.outputs.new_release_published == 'true'
    # The EC2 host is Graviton (arm64). Build natively on a GitHub-hosted
    # ARM runner so the image's only manifest is linux/arm64 — same cost
    # as x86, no QEMU emulation overhead.
    runs-on: ubuntu-24.04-arm
    permissions:
      id-token: write   # mint the OIDC token for configure-aws-credentials
      contents: read    # checkout of the released tag
```

The credentials step (lines 62–66) becomes:

```yaml
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_GHA_ROLE_ARN }}
          aws-region: us-east-2
```

- [ ] **Step 2: Commit, push, PR**

The `fix(ci):` type is deliberate: it makes semantic-release cut a patch release on merge, which exercises the full ECR-push-and-deploy pipeline on OIDC immediately instead of waiting for the next feature release.

```bash
git add .github/workflows/release.yml
git commit -m "fix(ci): authenticate to AWS via the shared OIDC role

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
git push -u origin ci/oidc-auth
gh pr create --title "fix(ci): authenticate to AWS via the shared OIDC role" --body "Swaps static keys for role-to-assume on prog-strength-github-actions (org secret AWS_GHA_ROLE_ARN). fix type intentionally cuts a patch release so the merge itself verifies ECR push + deploy on OIDC.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
gh pr checks --watch
```
Expected: `ci.yml` checks green (no AWS involved).

- [ ] **Step 3: Merge and watch the release pipeline end-to-end**

```bash
gh pr merge --squash --delete-branch
gh run watch $(gh run list --workflow=release.yml -R Prog-Strength/prog-strength-api --limit 1 --json databaseId --jq '.[0].databaseId')
```
Expected: all three jobs green — `release` cuts vX.Y.Z+1, `build_and_push` pushes to ECR authenticated via OIDC, `deploy` brings the tag up on EC2. If `build_and_push` fails at credentials with `Not authorized to perform sts:AssumeRoleWithWebIdentity`, the trust policy is wrong for this repo — fix in infra, re-run the failed jobs from the run page (`gh run rerun <id> --failed`).

---

### Task 8: Flip prog-strength-agent release workflow

**Files:**
- Modify: `prog-strength-agent/.github/workflows/release.yml:49-67` (same shape as api; the build job starts at the `build_and_push:` line and the credentials step is at lines 63–67)

- [ ] **Step 1: Branch and edit**

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-agent
git checkout main && git pull --ff-only
git checkout -b ci/oidc-auth
```

Add to the `build_and_push` job header (immediately after its `runs-on:` line):

```yaml
    permissions:
      id-token: write   # mint the OIDC token for configure-aws-credentials
      contents: read    # checkout of the released tag
```

Replace the credentials step (lines 63–67):

```yaml
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_GHA_ROLE_ARN }}
          aws-region: us-east-2
```

- [ ] **Step 2: Commit, push, PR, merge, verify**

```bash
git add .github/workflows/release.yml
git commit -m "fix(ci): authenticate to AWS via the shared OIDC role

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
git push -u origin ci/oidc-auth
gh pr create --title "fix(ci): authenticate to AWS via the shared OIDC role" --body "Swaps static keys for role-to-assume on prog-strength-github-actions (org secret AWS_GHA_ROLE_ARN). fix type intentionally cuts a patch release so the merge itself verifies ECR push + deploy on OIDC.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
gh pr checks --watch
gh pr merge --squash --delete-branch
gh run watch $(gh run list --workflow=release.yml -R Prog-Strength/prog-strength-agent --limit 1 --json databaseId --jq '.[0].databaseId')
```
Expected: full release pipeline green; image pushed to `prog-strength-prod/agent` via OIDC.

---

### Task 9: Flip prog-strength-mcp release workflow

**Files:**
- Modify: `prog-strength-mcp/.github/workflows/release.yml:49-67` (identical shape to agent)

- [ ] **Step 1: Branch and edit**

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-mcp
git checkout main && git pull --ff-only
git checkout -b ci/oidc-auth
```

Add to the `build_and_push` job header (immediately after its `runs-on:` line):

```yaml
    permissions:
      id-token: write   # mint the OIDC token for configure-aws-credentials
      contents: read    # checkout of the released tag
```

Replace the credentials step (lines 63–67):

```yaml
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_GHA_ROLE_ARN }}
          aws-region: us-east-2
```

- [ ] **Step 2: Commit, push, PR, merge, verify**

```bash
git add .github/workflows/release.yml
git commit -m "fix(ci): authenticate to AWS via the shared OIDC role

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
git push -u origin ci/oidc-auth
gh pr create --title "fix(ci): authenticate to AWS via the shared OIDC role" --body "Swaps static keys for role-to-assume on prog-strength-github-actions (org secret AWS_GHA_ROLE_ARN). fix type intentionally cuts a patch release so the merge itself verifies ECR push + deploy on OIDC.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
gh pr checks --watch
gh pr merge --squash --delete-branch
gh run watch $(gh run list --workflow=release.yml -R Prog-Strength/prog-strength-mcp --limit 1 --json databaseId --jq '.[0].databaseId')
```
Expected: full release pipeline green; image pushed to `prog-strength-prod/mcp` via OIDC.

---

### Task 10: Decommission the long-lived keys

**Gate: every item below must be true before deleting anything.**

- [ ] **Step 1: Verify green OIDC runs across the board**

```bash
for repo in prog-strength-api prog-strength-agent prog-strength-mcp; do
  gh run list --workflow=release.yml -R Prog-Strength/$repo --limit 1
done
gh run list --workflow=apply.yml -R Prog-Strength/prog-strength-infra --limit 1
gh run list --workflow=plan.yml  -R Prog-Strength/prog-strength-infra --limit 1
gh run list --workflow=apply.yml -R Prog-Strength/prog-strength-developer --limit 1
```
Expected: most recent run of each is `completed success` and postdates its repo's OIDC flip. (`replace-instance.yml` can't be safely test-fired — it destroys the EC2 host. Its diff is identical to apply.yml's; a failed future run falls back to re-running after a fix, with no key dependency.)

- [ ] **Step 2: Confirm the IAM user serves nothing else**

```bash
aws iam list-users --query 'Users[].UserName'
# Identify the CI user (the one whose key matches the GitHub secrets), then:
aws iam list-access-keys --user-name <CI_USER>
aws iam get-access-key-last-used --access-key-id <KEY_ID>
```
**Decision point:** if `LastUsedDate` is recent but later than the last static-key workflow run, or the user is referenced by anything outside GitHub Actions (local `~/.aws/credentials` profile, scripts), stop and report to the operator — in that case only the GitHub secrets are removed and the key deletion is deferred.

- [ ] **Step 3: Remove the static-key secrets from all four repos**

```bash
for repo in prog-strength-api prog-strength-agent prog-strength-mcp prog-strength-infra; do
  gh secret delete AWS_ACCESS_KEY_ID     -R Prog-Strength/$repo
  gh secret delete AWS_SECRET_ACCESS_KEY -R Prog-Strength/$repo
done
```

- [ ] **Step 4: Deactivate (not delete) the access keys**

```bash
aws iam update-access-key --user-name <CI_USER> --access-key-id <KEY_ID> --status Inactive
```
Deactivation is reversible; deletion is not. An inactive key fails exactly like a deleted one, so anything still depending on it surfaces now.

- [ ] **Step 5: Soak for at least 3 days, then delete**

After ≥3 days with no failures attributable to the inactive key:

```bash
aws iam delete-access-key --user-name <CI_USER> --access-key-id <KEY_ID>
# If the user exists solely for CI (confirmed in Step 2), remove it entirely:
aws iam list-attached-user-policies --user-name <CI_USER>
aws iam list-user-policies --user-name <CI_USER>
# detach/delete any listed policies, then:
aws iam delete-user --user-name <CI_USER>
```
Expected: `aws iam get-user --user-name <CI_USER>` returns `NoSuchEntity`.

---

### Task 11: Mark the SOW shipped

**Files:**
- Modify: `prog-strength-docs/sows/github-actions-oidc-role.md:2` (frontmatter `status: draft` → `status: shipped`) and the `**Status**: Draft` body line → `**Status**: Shipped · **Last updated**: <completion date>`

- [ ] **Step 1: Update, commit, PR**

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-docs
git checkout main && git pull --ff-only
git checkout -b docs/oidc-sow-shipped
# edit the two status lines as above
git add sows/github-actions-oidc-role.md
git commit -m "docs: mark GitHub Actions OIDC role SOW as shipped

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
git push -u origin docs/oidc-sow-shipped
gh pr create --title "docs: mark GitHub Actions OIDC role SOW as shipped" --body "All repos' workflows run on the shared prog-strength-github-actions OIDC role; static IAM user keys are deleted.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
gh pr merge --squash --delete-branch
```
