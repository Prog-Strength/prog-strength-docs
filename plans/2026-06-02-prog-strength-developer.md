# Prog Strength Developer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `prog-strength-developer` repository — an autonomous-developer system that spins up an ephemeral EC2 worker on demand, points Claude Code at a designated SOW, opens PRs in each affected Prog-Strength repo, and self-terminates.

**Architecture:** New repo at `repos/prog-strength-developer` holds Terraform (VPC + IAM + EC2 launch template + Secrets Manager refs + CloudWatch), a `workflow_dispatch` GitHub Actions workflow that runs `terraform apply` and exits, a cloud-init userdata script that bootstraps the worker, and a Claude Code prompt template. Worker self-terminates via AWS SDK on completion. Single-instance concurrency enforced by a pre-flight check.

**Tech Stack:** Terraform 1.7+ (AWS provider 5.x), GitHub Actions (workflow_dispatch + OIDC), bash + jq + python3 (for JWT) inside Amazon Linux 2023 userdata, AWS Systems Manager Session Manager for debugging access.

**Source spec:** `prog-strength-docs/sows/prog-strength-developer.md`

**Repo paths used in this plan:**
- New: `/Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-developer`
- Modified: `/Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-docs`

---

## Phase 1 — Repository bootstrap

### Task 1: Initialize the prog-strength-developer repo

**Files:**
- Create: `repos/prog-strength-developer/` (directory)
- Create: `repos/prog-strength-developer/README.md`
- Create: `repos/prog-strength-developer/.gitignore`
- Create: `repos/prog-strength-developer/LICENSE`
- Create: `repos/prog-strength-developer/.terraform-version`

- [ ] **Step 1: Create the repo directory and init git**

```bash
mkdir -p /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-developer
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-developer
git init -b main
```

- [ ] **Step 2: Write `.gitignore`**

```
# Terraform
.terraform/
.terraform.lock.hcl
*.tfstate
*.tfstate.*
*.tfvars
*.tfvars.json
crash.log
crash.*.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Secrets — never commit
*.pem
*.key
.env
.env.*
credentials.json

# macOS
.DS_Store

# Editor
.vscode/
.idea/
*.swp
```

- [ ] **Step 3: Write `.terraform-version`**

```
1.7.5
```

(Pinning matches what AWS-provider 5.x requires as a floor; tfenv reads this file automatically.)

- [ ] **Step 4: Write `LICENSE`** — MIT, owner Jimmy Wallace

```
MIT License

Copyright (c) 2026 Jimmy Wallace

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 5: Write `README.md`** (short — full docs land in Task 14)

```markdown
# prog-strength-developer

Autonomous developer for [Prog Strength](https://github.com/Prog-Strength). Spins up
an ephemeral EC2 worker on demand, runs Claude Code against a designated SOW
from [prog-strength-docs](https://github.com/Prog-Strength/prog-strength-docs),
opens PRs in each affected repo, and self-terminates.

See `docs/README.md` for the system overview and `docs/setup.md` for
first-time bootstrap.

## Quick links

- **Dispatch a SOW:** Actions tab → "Dispatch SOW" → Run workflow → paste the SOW path.
- **Watch a run:** CloudWatch log group `/aws/ec2/prog-strength-developer/<instance-id>`.
- **Debug a stuck worker:** `aws ssm start-session --target <instance-id>`.

## SOW

This repo implements `prog-strength-docs/sows/prog-strength-developer.md`.
```

- [ ] **Step 6: Initial commit**

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-developer
git add .gitignore .terraform-version LICENSE README.md
git commit -m "chore: initial repo skeleton"
```

- [ ] **Step 7: Create feature branch for the rest of the work**

```bash
git checkout -b feat/initial-bootstrap
```

(All subsequent tasks commit to this branch. The user will open one big PR at the end, or split into several — that decision is at the wrap-up phase.)

---

## Phase 2 — Terraform persistent infrastructure

All Terraform under `repos/prog-strength-developer/terraform/`.

### Task 2: Terraform skeleton — provider, backend, variables

**Files:**
- Create: `terraform/main.tf`
- Create: `terraform/variables.tf`
- Create: `terraform/backend.tf`

- [ ] **Step 1: Write `terraform/main.tf`**

```hcl
terraform {
  required_version = ">= 1.7.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project   = "prog-strength-developer"
      ManagedBy = "terraform"
    }
  }
}
```

- [ ] **Step 2: Write `terraform/variables.tf`**

```hcl
variable "aws_region" {
  description = "AWS region for the developer VPC and resources."
  type        = string
  default     = "us-east-1"
}

variable "vpc_cidr" {
  description = "CIDR block for the developer VPC. Must not overlap the application VPC."
  type        = string
  default     = "10.20.0.0/16"
}

variable "public_subnet_cidr" {
  description = "CIDR for the single public subnet that hosts the worker."
  type        = string
  default     = "10.20.1.0/24"
}

variable "availability_zone" {
  description = "AZ for the public subnet. Single AZ is fine for ephemeral compute."
  type        = string
  default     = "us-east-1a"
}

variable "instance_type" {
  description = "EC2 instance type for the worker. t3.large = 2 vCPU / 8 GB."
  type        = string
  default     = "t3.large"
}

variable "max_runtime_hours" {
  description = "Hard backstop. The worker terminates after this many hours regardless of Claude's state."
  type        = number
  default     = 6
}

variable "github_org" {
  description = "GitHub organization that owns the repos the worker will clone and open PRs against."
  type        = string
  default     = "Prog-Strength"
}

variable "github_actions_repo" {
  description = "The repo whose Actions workflow is allowed to assume the GHA OIDC role. Restricts the trust policy."
  type        = string
  default     = "Prog-Strength/prog-strength-developer"
}

variable "log_retention_days" {
  description = "CloudWatch Logs retention."
  type        = number
  default     = 30
}

variable "sow_path" {
  description = "Path to the SOW within prog-strength-docs (e.g. sows/foo.md). Templated into userdata."
  type        = string
  default     = ""
}
```

- [ ] **Step 3: Write `terraform/backend.tf`**

```hcl
# S3 + DynamoDB backend for state. Both resources are created MANUALLY
# during initial bootstrap (chicken-and-egg — see docs/setup.md). After
# that, this block tells Terraform where to store state for every
# subsequent apply.
#
# Bucket name and DynamoDB table name are owner-specific and configured
# via `terraform init -backend-config=...` rather than hard-coded here,
# so the same Terraform code is reusable across AWS accounts in the
# unlikely event the project ever moves.

terraform {
  backend "s3" {
    key          = "prog-strength-developer/terraform.tfstate"
    encrypt      = true
    use_lockfile = true
  }
}
```

(`use_lockfile = true` enables Terraform 1.10+'s native S3 lockfile mode, eliminating the need for DynamoDB. If your tfenv resolves to 1.7.x, fall back to `dynamodb_table = "<table>"` instead — call this out for the user.)

- [ ] **Step 4: Validate the skeleton**

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-developer/terraform
terraform fmt -check
terraform init -backend=false   # skips backend init (the bucket doesn't exist yet)
terraform validate
```
Expected: `Success! The configuration is valid.`

- [ ] **Step 5: Commit**

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-developer
git add terraform/main.tf terraform/variables.tf terraform/backend.tf
git commit -m "feat(terraform): provider, backend, and input variables"
```

---

### Task 3: VPC, subnet, IGW, security group

**Files:**
- Create: `terraform/vpc.tf`

- [ ] **Step 1: Write `terraform/vpc.tf`**

```hcl
# Dedicated VPC for the autonomous developer. Explicitly separate from
# the application VPC in prog-strength-infra. No peering, no shared
# route tables — a misbehaving worker cannot route to the prod stack.

resource "aws_vpc" "developer" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "prog-strength-developer-vpc"
  }
}

# One public subnet. Single AZ is fine for ephemeral compute — there is
# no high-availability requirement when each worker exists for hours.
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.developer.id
  cidr_block              = var.public_subnet_cidr
  availability_zone       = var.availability_zone
  map_public_ip_on_launch = true

  tags = {
    Name = "prog-strength-developer-public"
  }
}

resource "aws_internet_gateway" "developer" {
  vpc_id = aws_vpc.developer.id

  tags = {
    Name = "prog-strength-developer-igw"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.developer.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.developer.id
  }

  tags = {
    Name = "prog-strength-developer-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Worker security group: no inbound (SSM Session Manager handles
# debugging), all outbound (long tail of package registries, the
# Anthropic API, GitHub, AWS, arbitrary URLs Claude may fetch).
# Trust boundary is the VPC isolation + IAM role, not egress filtering.
resource "aws_security_group" "worker" {
  name        = "prog-strength-developer-worker-sg"
  description = "Autonomous developer worker — outbound only, no SSH."
  vpc_id      = aws_vpc.developer.id

  egress {
    description = "All outbound — worker needs broad internet access."
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "prog-strength-developer-worker-sg"
  }
}
```

- [ ] **Step 2: Validate**

```bash
cd terraform && terraform fmt -check && terraform validate
```
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add terraform/vpc.tf
git commit -m "feat(terraform): isolated VPC, public subnet, worker security group"
```

---

### Task 4: IAM — worker role + GHA OIDC role

**Files:**
- Create: `terraform/iam.tf`

- [ ] **Step 1: Write `terraform/iam.tf`**

```hcl
# --------------------------------------------------------------------
# Worker role: assumed by the EC2 worker via its instance profile.
# Least-privilege — narrow secret access, self-only termination, SSM,
# CloudWatch Logs.
# --------------------------------------------------------------------

data "aws_caller_identity" "current" {}

data "aws_iam_policy_document" "worker_trust" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "worker" {
  name               = "prog-strength-developer-worker-role"
  assume_role_policy = data.aws_iam_policy_document.worker_trust.json
}

# Managed policy for SSM Session Manager. AWS-curated; cheaper than
# hand-rolling the ssmmessages:* / ec2messages:* statements.
resource "aws_iam_role_policy_attachment" "worker_ssm" {
  role       = aws_iam_role.worker.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

data "aws_iam_policy_document" "worker_inline" {
  # Secrets Manager: only the two developer secrets, by ARN.
  statement {
    sid     = "ReadDeveloperSecrets"
    actions = ["secretsmanager:GetSecretValue"]
    resources = [
      "arn:aws:secretsmanager:${var.aws_region}:${data.aws_caller_identity.current.account_id}:secret:prog-strength-developer/claude-credentials-*",
      "arn:aws:secretsmanager:${var.aws_region}:${data.aws_caller_identity.current.account_id}:secret:prog-strength-developer/github-app-*",
    ]
  }

  # Self-termination only. The Name tag condition restricts the action
  # to instances bearing the developer-worker tag — the worker cannot
  # accidentally terminate an instance in prog-strength-infra.
  statement {
    sid       = "TerminateSelf"
    actions   = ["ec2:TerminateInstances"]
    resources = ["arn:aws:ec2:${var.aws_region}:${data.aws_caller_identity.current.account_id}:instance/*"]
    condition {
      test     = "StringEquals"
      variable = "aws:ResourceTag/Name"
      values   = ["prog-strength-developer-worker"]
    }
  }

  # CloudWatch Logs: worker streams stdout/stderr.
  statement {
    sid = "CloudWatchLogs"
    actions = [
      "logs:CreateLogStream",
      "logs:PutLogEvents",
      "logs:DescribeLogStreams",
    ]
    resources = ["${aws_cloudwatch_log_group.worker.arn}:*"]
  }

  # IMDSv2 token retrieval doesn't need IAM; listed here only as a
  # reminder for reviewers. No statement.
}

resource "aws_iam_role_policy" "worker_inline" {
  name   = "prog-strength-developer-worker-inline"
  role   = aws_iam_role.worker.id
  policy = data.aws_iam_policy_document.worker_inline.json
}

resource "aws_iam_instance_profile" "worker" {
  name = "prog-strength-developer-worker-profile"
  role = aws_iam_role.worker.name
}

# --------------------------------------------------------------------
# GitHub Actions OIDC role: assumed by the dispatch-sow workflow.
# Trust restricted to a single repo + branch so feature-branch runs
# can't accidentally provision resources.
# --------------------------------------------------------------------

data "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
}

data "aws_iam_policy_document" "github_actions_trust" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [data.aws_iam_openid_connect_provider.github.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:${var.github_actions_repo}:ref:refs/heads/main"]
    }
  }
}

resource "aws_iam_role" "github_actions" {
  name               = "prog-strength-developer-github-actions-role"
  assume_role_policy = data.aws_iam_policy_document.github_actions_trust.json
}

data "aws_iam_policy_document" "github_actions_inline" {
  # Terraform read/apply on the developer's resources.
  statement {
    sid = "EC2Manage"
    actions = [
      "ec2:Describe*",
      "ec2:RunInstances",
      "ec2:CreateTags",
      "ec2:TerminateInstances",
      "ec2:CreateLaunchTemplate",
      "ec2:CreateLaunchTemplateVersion",
      "ec2:DeleteLaunchTemplate",
      "ec2:ModifyLaunchTemplate",
    ]
    resources = ["*"]
  }

  statement {
    sid       = "PassWorkerRole"
    actions   = ["iam:PassRole"]
    resources = [aws_iam_role.worker.arn]
  }

  statement {
    sid = "IAMRead"
    actions = [
      "iam:GetRole",
      "iam:GetInstanceProfile",
      "iam:GetRolePolicy",
      "iam:ListRolePolicies",
      "iam:ListAttachedRolePolicies",
    ]
    resources = ["*"]
  }

  # Terraform state in S3 + (optional) DynamoDB lock are configured
  # outside this Terraform via `terraform init -backend-config=...`,
  # but the GHA role still needs the permission to read/write them.
  # Bucket name is owner-specific; permission granted on all S3 here
  # and audited via CloudTrail. Tighten later if cross-tenant.
  statement {
    sid = "S3StateBackend"
    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject",
      "s3:ListBucket",
    ]
    resources = ["*"]
  }

  statement {
    sid = "VPCRead"
    actions = [
      "ec2:DescribeVpcs",
      "ec2:DescribeSubnets",
      "ec2:DescribeSecurityGroups",
      "ec2:DescribeInternetGateways",
      "ec2:DescribeRouteTables",
    ]
    resources = ["*"]
  }
}

resource "aws_iam_role_policy" "github_actions_inline" {
  name   = "prog-strength-developer-github-actions-inline"
  role   = aws_iam_role.github_actions.id
  policy = data.aws_iam_policy_document.github_actions_inline.json
}
```

- [ ] **Step 2: Validate**

```bash
cd terraform && terraform fmt -check && terraform validate
```
Expected: PASS. Note: the `aws_iam_openid_connect_provider` data source will error if the provider isn't already registered in the AWS account — that's documented as a manual bootstrap step (`docs/setup.md`), not a Terraform concern.

- [ ] **Step 3: Commit**

```bash
git add terraform/iam.tf
git commit -m "feat(terraform): worker IAM role + GitHub Actions OIDC role"
```

---

### Task 5: CloudWatch log group + Secrets Manager refs

**Files:**
- Create: `terraform/cloudwatch.tf`
- Create: `terraform/secrets.tf`

- [ ] **Step 1: Write `terraform/cloudwatch.tf`**

```hcl
# Single log group for all worker instances. Each instance writes its
# own log stream named after the instance ID (config in userdata).
resource "aws_cloudwatch_log_group" "worker" {
  name              = "/aws/ec2/prog-strength-developer"
  retention_in_days = var.log_retention_days
}
```

- [ ] **Step 2: Write `terraform/secrets.tf`**

```hcl
# Secrets Manager refs. Both secrets are CREATED and SEEDED manually
# during initial bootstrap (see docs/setup.md) — Terraform doesn't
# manage their contents because their values are owner-private (the
# Claude OAuth refresh token and the GitHub App private key) and
# committing them to state would be a leak risk.
#
# These data sources verify the secrets exist before Terraform proceeds
# and surface their ARNs for use in IAM policies + userdata.

data "aws_secretsmanager_secret" "claude_credentials" {
  name = "prog-strength-developer/claude-credentials"
}

data "aws_secretsmanager_secret" "github_app" {
  name = "prog-strength-developer/github-app"
}
```

- [ ] **Step 3: Validate**

```bash
cd terraform && terraform fmt -check && terraform validate
```
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add terraform/cloudwatch.tf terraform/secrets.tf
git commit -m "feat(terraform): CloudWatch log group + Secrets Manager refs"
```

---

### Task 6: AMI lookup + EC2 launch template

**Files:**
- Create: `terraform/ec2.tf`

- [ ] **Step 1: Write `terraform/ec2.tf`**

```hcl
# Latest Amazon Linux 2023 AMI in the region, looked up via SSM
# parameter. SSM gives us the canonical "latest" pointer without
# pinning to an AMI ID that goes stale within weeks.
data "aws_ssm_parameter" "al2023_ami" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"
}

# Render the userdata template with the runtime variables baked in.
# templatefile() substitutes ${var} placeholders inside userdata.sh.tpl.
locals {
  # Raw contents of the Claude prompt template. file() does NOT process
  # template syntax inside the contents, so the prompt's
  # __PLACEHOLDER__ markers pass through verbatim. The prompt then gets
  # embedded into userdata via a single-quoted heredoc, which protects
  # any `$` characters from bash expansion at runtime.
  prompt_template = file("${path.module}/../bootstrap/prompt.md.tpl")

  userdata = templatefile("${path.module}/../bootstrap/userdata.sh.tpl", {
    aws_region             = var.aws_region
    sow_path               = var.sow_path
    github_org             = var.github_org
    log_group_name         = aws_cloudwatch_log_group.worker.name
    max_runtime_hours      = var.max_runtime_hours
    claude_secret_name     = data.aws_secretsmanager_secret.claude_credentials.name
    github_app_secret_name = data.aws_secretsmanager_secret.github_app.name
    prompt_template        = local.prompt_template
  })
}

resource "aws_launch_template" "worker" {
  name_prefix   = "prog-strength-developer-worker-"
  image_id      = data.aws_ssm_parameter.al2023_ami.value
  instance_type = var.instance_type

  iam_instance_profile {
    name = aws_iam_instance_profile.worker.name
  }

  network_interfaces {
    associate_public_ip_address = true
    security_groups             = [aws_security_group.worker.id]
    subnet_id                   = aws_subnet.public.id
  }

  # IMDSv2 only. Required for the worker's self-termination flow (which
  # queries the instance ID via the IMDS token endpoint) and good
  # hygiene regardless.
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 1
  }

  user_data = base64encode(local.userdata)

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "prog-strength-developer-worker"
      SOW  = var.sow_path
    }
  }
}

# The actual instance: created ONLY when sow_path is non-empty. The
# workflow_dispatch wrapper passes sow_path; manual `terraform apply`
# without it produces only the launch template + persistent infra.
#
# count = 1 when dispatching; count = 0 for state-only applies.
resource "aws_instance" "worker" {
  count = var.sow_path != "" ? 1 : 0

  launch_template {
    id      = aws_launch_template.worker.id
    version = "$Latest"
  }

  # Explicit dependency so the IAM role's inline policy is attached
  # BEFORE the instance boots and starts hitting Secrets Manager.
  depends_on = [aws_iam_role_policy.worker_inline]

  tags = {
    Name = "prog-strength-developer-worker"
    SOW  = var.sow_path
  }
}
```

- [ ] **Step 2: Validate**

```bash
cd terraform && terraform fmt -check && terraform validate
```
Expected: PASS. The reference to `../bootstrap/userdata.sh.tpl` will error during `terraform plan` because the file doesn't exist yet — that's expected and addressed in Task 7. For now, `terraform validate` ignores the missing file because it doesn't actually open it.

- [ ] **Step 3: Commit**

```bash
git add terraform/ec2.tf
git commit -m "feat(terraform): AMI lookup, launch template, gated instance resource"
```

---

### Task 7: Terraform outputs

**Files:**
- Create: `terraform/outputs.tf`

- [ ] **Step 1: Write `terraform/outputs.tf`**

```hcl
output "vpc_id" {
  description = "Developer VPC ID."
  value       = aws_vpc.developer.id
}

output "public_subnet_id" {
  description = "Public subnet hosting the worker."
  value       = aws_subnet.public.id
}

output "worker_security_group_id" {
  description = "Security group ID for the worker."
  value       = aws_security_group.worker.id
}

output "worker_role_arn" {
  description = "IAM role assumed by the worker via its instance profile."
  value       = aws_iam_role.worker.arn
}

output "github_actions_role_arn" {
  description = "IAM role the GitHub Actions workflow assumes via OIDC. Paste into the workflow YAML."
  value       = aws_iam_role.github_actions.arn
}

output "log_group_name" {
  description = "CloudWatch log group for worker output."
  value       = aws_cloudwatch_log_group.worker.name
}

output "worker_instance_id" {
  description = "ID of the current worker instance, if one is running. Empty when sow_path is unset."
  value       = length(aws_instance.worker) > 0 ? aws_instance.worker[0].id : ""
}
```

- [ ] **Step 2: Validate**

```bash
cd terraform && terraform fmt -check && terraform validate
```
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add terraform/outputs.tf
git commit -m "feat(terraform): outputs for VPC, IAM, log group, worker"
```

---

## Phase 3 — Bootstrap scripts and Claude prompt

### Task 8: Userdata bootstrap script

**Files:**
- Create: `bootstrap/userdata.sh.tpl`

- [ ] **Step 1: Write the userdata template**

```bash
#!/bin/bash
# prog-strength-developer worker userdata.
# Rendered by Terraform's templatefile() — variables wrapped in
# ${...} are substituted at apply time.
#
# Lifecycle: install deps → fetch secrets → clone repos → run Claude
# Code → self-terminate. Every branch terminates the instance,
# including failures. A systemd timer is set as a hard backstop.

set -euo pipefail

LOG_DIR=/var/log/prog-strength-developer
mkdir -p "$LOG_DIR"
WORKDIR=/workspace
mkdir -p "$WORKDIR"

# Tee all subsequent output so the CloudWatch agent picks it up.
exec > >(tee -a "$LOG_DIR/userdata.log") 2>&1

log() { echo "[userdata $(date -u +%FT%TZ)] $*"; }

trap 'log "FATAL: userdata exited at line $LINENO with status $?"; terminate_self' ERR

INSTANCE_ID=$(curl -fsS -H "X-aws-ec2-metadata-token: $(curl -fsS -X PUT \
  -H 'X-aws-ec2-metadata-token-ttl-seconds: 60' \
  http://169.254.169.254/latest/api/token)" \
  http://169.254.169.254/latest/meta-data/instance-id)
export INSTANCE_ID

terminate_self() {
  log "Terminating instance $INSTANCE_ID"
  # Flush CloudWatch agent buffer before disappearing.
  sleep 10
  aws ec2 terminate-instances \
    --region "${aws_region}" \
    --instance-ids "$INSTANCE_ID" || true
}

# --------------------------------------------------------------------
# Install dependencies.
# --------------------------------------------------------------------
log "Installing system packages"
dnf install -y -q \
  git gcc make jq python3 python3-pip \
  amazon-cloudwatch-agent

# AWS CLI v2 is pre-installed on Amazon Linux 2023; just verify.
aws --version

# Install pyjwt (for GitHub App JWT minting) and PyYAML (for SOW
# frontmatter parsing later in the script). Both via pip on the system
# python3 — no venv needed; this is a single-purpose ephemeral host.
pip3 install --quiet 'pyjwt[crypto]>=2.8' 'pyyaml>=6'

# Install GitHub CLI.
log "Installing gh CLI"
dnf install -y -q dnf-plugins-core
dnf config-manager --add-repo https://cli.github.com/packages/rpm/gh-cli.repo
dnf install -y -q gh

# Install Node 20 + npm via NodeSource.
log "Installing Node 20"
curl -fsSL https://rpm.nodesource.com/setup_20.x | bash -
dnf install -y -q nodejs

# Install Go (latest stable).
log "Installing Go"
GO_VERSION=1.22.5
curl -fsSL -o /tmp/go.tgz "https://go.dev/dl/go$${GO_VERSION}.linux-amd64.tar.gz"
tar -C /usr/local -xzf /tmp/go.tgz
export PATH="$PATH:/usr/local/go/bin"
echo 'export PATH="$PATH:/usr/local/go/bin"' >> /etc/profile.d/go.sh

# Install uv (Python project + tool manager).
log "Installing uv"
curl -fsSL https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:/root/.local/bin:$PATH"

# Install Claude Code CLI via the official npm package.
log "Installing Claude Code"
npm install -g --silent @anthropic-ai/claude-code

# --------------------------------------------------------------------
# Configure CloudWatch agent to ship /var/log/prog-strength-developer
# to the developer log group, one stream per instance ID.
# --------------------------------------------------------------------
log "Configuring CloudWatch agent"
cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json <<EOF
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "$LOG_DIR/*.log",
            "log_group_name": "${log_group_name}",
            "log_stream_name": "$INSTANCE_ID",
            "timestamp_format": "%Y-%m-%dT%H:%M:%SZ",
            "retention_in_days": -1
          },
          {
            "file_path": "/var/log/cloud-init-output.log",
            "log_group_name": "${log_group_name}",
            "log_stream_name": "$INSTANCE_ID-cloud-init",
            "retention_in_days": -1
          }
        ]
      }
    }
  }
}
EOF
systemctl enable --now amazon-cloudwatch-agent
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
  -s

# --------------------------------------------------------------------
# Fetch secrets.
# --------------------------------------------------------------------
log "Fetching Claude credentials from Secrets Manager"
mkdir -p /root/.claude
aws secretsmanager get-secret-value \
  --region "${aws_region}" \
  --secret-id "${claude_secret_name}" \
  --query SecretString --output text > /root/.claude/credentials.json
chmod 600 /root/.claude/credentials.json

log "Fetching GitHub App credentials"
aws secretsmanager get-secret-value \
  --region "${aws_region}" \
  --secret-id "${github_app_secret_name}" \
  --query SecretString --output text > /root/.github-app.json
chmod 600 /root/.github-app.json

# Mint an installation token. mint-gh-token.sh ships in this repo; the
# launch template embedded it via cloud-init's write_files mechanism in
# an earlier revision — for v1 we inline the minimal Python here to
# avoid the second-asset coordination problem.
log "Minting GitHub App installation token"
GH_TOKEN=$(python3 - <<'PY'
import json, os, time, base64, urllib.request, sys
import jwt

with open('/root/.github-app.json') as f:
    cfg = json.load(f)

now = int(time.time())
payload = {
    "iat": now - 60,
    "exp": now + 540,
    "iss": str(cfg["app_id"]),
}
encoded = jwt.encode(payload, cfg["private_key"], algorithm="RS256")

req = urllib.request.Request(
    f"https://api.github.com/app/installations/{cfg['installation_id']}/access_tokens",
    method="POST",
    headers={
        "Authorization": f"Bearer {encoded}",
        "Accept": "application/vnd.github+json",
        "X-GitHub-Api-Version": "2022-11-28",
    },
)
with urllib.request.urlopen(req) as resp:
    body = json.loads(resp.read())
print(body["token"])
PY
)
echo "$GH_TOKEN" | gh auth login --with-token

# --------------------------------------------------------------------
# Set the 6h hard backstop. systemd-run --on-active is the simplest
# transient unit for "fire once after N hours."
# --------------------------------------------------------------------
log "Arming ${max_runtime_hours}h backstop"
systemd-run \
  --on-active="${max_runtime_hours}h" \
  --unit=worker-timeout \
  /bin/bash -c "aws ec2 terminate-instances --region ${aws_region} --instance-ids $INSTANCE_ID"

# --------------------------------------------------------------------
# Clone prog-strength-docs, parse the SOW frontmatter for the repo
# list, clone each affected repo.
# --------------------------------------------------------------------
log "Cloning prog-strength-docs"
cd "$WORKDIR"
gh repo clone "${github_org}/prog-strength-docs"

SOW_FILE="$WORKDIR/prog-strength-docs/${sow_path}"
if [ ! -f "$SOW_FILE" ]; then
  log "FATAL: SOW not found at $SOW_FILE"
  terminate_self
  exit 1
fi

log "Reading repo list from SOW frontmatter"
# Extract the YAML frontmatter block (between the first pair of '---')
# and pull repos[] via python (always available on AL2023 + pip).
REPOS=$(python3 - <<PY
import re, sys, yaml
with open("$SOW_FILE") as f:
    text = f.read()
m = re.match(r"^---\n(.*?)\n---\n", text, re.DOTALL)
if not m:
    print("FATAL: SOW has no YAML frontmatter", file=sys.stderr)
    sys.exit(1)
meta = yaml.safe_load(m.group(1)) or {}
for r in meta.get("repos") or []:
    print(r)
PY
)

if [ -z "$REPOS" ]; then
  log "FATAL: SOW frontmatter has empty repos:[]"
  terminate_self
  exit 1
fi

log "Repos to clone: $REPOS"
for repo in $REPOS; do
  if [ "$repo" = "prog-strength-docs" ]; then
    continue   # already cloned
  fi
  log "Cloning $repo"
  gh repo clone "${github_org}/$repo" "$WORKDIR/$repo"
done

# --------------------------------------------------------------------
# Render the Claude prompt and run Claude Code.
# --------------------------------------------------------------------
log "Writing prompt template to disk"
# The prompt template is embedded via Terraform's templatefile(); the
# single-quoted heredoc keeps bash from interpreting any $-prefixed
# tokens inside the prompt itself.
mkdir -p /opt/prog-strength-developer
cat > /opt/prog-strength-developer/prompt.md.tpl <<'PROMPT_TEMPLATE_EOF'
${prompt_template}
PROMPT_TEMPLATE_EOF

log "Rendering Claude prompt for SOW ${sow_path}"
SOW_SLUG=$(basename "${sow_path}" .md)
sed \
  -e "s|__SOW_PATH__|${sow_path}|g" \
  -e "s|__SOW_SLUG__|$SOW_SLUG|g" \
  -e "s|__GITHUB_ORG__|${github_org}|g" \
  /opt/prog-strength-developer/prompt.md.tpl \
  > "$WORKDIR/prompt.md"

cd "$WORKDIR"
log "Starting Claude Code"
# --print is non-interactive batch mode.
# --dangerously-skip-permissions waives the interactive permission
#   prompts that would otherwise block headless operation.
claude --print --dangerously-skip-permissions < prompt.md \
  | tee "$LOG_DIR/claude.log"

CLAUDE_EXIT=$?
log "Claude exited with status $CLAUDE_EXIT"

# --------------------------------------------------------------------
# Self-terminate. terminate_self() is also wired to the ERR trap so
# any earlier failure has already invoked it.
# --------------------------------------------------------------------
terminate_self
```

- [ ] **Step 2: Lint with shellcheck (best-effort)**

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-developer
shellcheck -x bootstrap/userdata.sh.tpl || true
```
Note: `templatefile()` substitution syntax (`${var}`) overlaps with bash variable syntax — shellcheck will emit some false positives. Skim the output; ignore warnings about `${aws_region}` / `${sow_path}` / `${log_group_name}` etc. (those are Terraform variables, not bash). Don't `|| true` away genuine logic errors.

If shellcheck isn't installed: `brew install shellcheck`.

- [ ] **Step 3: Syntax check with bash -n**

This catches gross syntax errors that shellcheck might miss in a template:

```bash
# Substitute placeholder values to make it parse as plain bash
sed 's/\${[^}]*}/DUMMY/g' bootstrap/userdata.sh.tpl | bash -n
```
Expected: no output (syntax valid).

- [ ] **Step 4: Commit**

```bash
git add bootstrap/userdata.sh.tpl
git commit -m "feat(bootstrap): cloud-init userdata for the worker"
```

---

### Task 9: Claude prompt template

**Files:**
- Create: `bootstrap/prompt.md.tpl`

- [ ] **Step 1: Write the prompt template**

```markdown
You are running unattended on an EC2 instance. There is no human to ask
questions of in this session. Make reasonable judgment calls and proceed —
if you genuinely cannot determine the right path forward, exit with a clear
message instead of fabricating an answer.

# Your task

Implement the Statement of Work at:

    /workspace/prog-strength-docs/__SOW_PATH__

The affected repositories have been cloned at `/workspace/<repo-name>`. Each
is checked out on `main`. You are responsible for creating a feature branch
in each repo you modify, named `feat/__SOW_SLUG__`, and opening a pull
request on each one against `main`.

# Workflow

Follow the standard Prog Strength autonomous workflow:

1. **Read the SOW.** Understand the goals, non-goals, and constraints.
2. **Find or write the plan.** If a plan already exists at
   `/workspace/prog-strength-docs/plans/*-__SOW_SLUG__.md`, use it as-is.
   If not, produce one by invoking the `superpowers:writing-plans` skill.
3. **Execute the plan.** Invoke the `superpowers:subagent-driven-development`
   skill and follow it exactly. Do not skip review stages — every task
   should be implemented by a subagent, then spec-reviewed and
   code-quality-reviewed before moving on.
4. **Open PRs.** After all tasks complete, push each feature branch and
   run `gh pr create` in each modified repository. The GitHub App you're
   authenticated as has push access. PR titles and bodies should follow
   the format you'll see in recent merged PRs in those repos.
5. **Exit.** The system will terminate the instance.

# Constraints

- You are running with `--dangerously-skip-permissions`. Do NOT use this
  freedom to take destructive actions: no `git push --force`, no
  `git reset --hard` against `main`, no `rm -rf` outside `/workspace`,
  no modifying repos that aren't in the SOW's `repos:` list.
- Every change you make should be reviewable as a normal pull request
  diff. The owner is the reviewer.
- Do NOT attempt to merge any PR you open. You don't have permission and
  the owner is the gate.
- If you encounter ambiguity in the SOW that genuinely blocks progress,
  open a "draft" PR in `prog-strength-docs` proposing a SOW clarification
  rather than guessing. Exit afterwards.

# Helpful context

- The org is `__GITHUB_ORG__`. All repo references in the SOW resolve
  under that org.
- You have ~6 hours of wall-clock budget. If you're at the 5-hour mark
  and not done, prioritize opening incomplete PRs over hitting the
  backstop with nothing visible.
- CloudWatch is capturing your stdout. Log progress markers liberally
  ("starting task N", "subagent dispatched for X") so the owner can
  follow along when they review the run.
- The `gh` CLI is already authenticated. Use it for clone, push, and
  PR creation. Do not configure git remotes by hand.

# Begin

Start by reading the SOW at the path above, then proceed.
```

- [ ] **Step 2: Verify Markdown is well-formed (no template-step test needed)**

```bash
head -3 bootstrap/prompt.md.tpl
```
Expected: starts with `You are running unattended` — no stray YAML or syntax bleed.

- [ ] **Step 3: Commit**

```bash
git add bootstrap/prompt.md.tpl
git commit -m "feat(bootstrap): Claude Code prompt template"
```

---

### Task 10: Preflight script for the GitHub Action

**Files:**
- Create: `scripts/preflight.sh`

- [ ] **Step 1: Write the script**

```bash
#!/usr/bin/env bash
# preflight.sh — fail the workflow fast if a worker is already running.
# Called from .github/workflows/dispatch-sow.yml before `terraform apply`.
#
# Single-instance concurrency is the v1 model. A second dispatch while a
# worker is alive returns non-zero with a clear message; the workflow
# fails before any AWS state changes.

set -euo pipefail

REGION="${AWS_REGION:-us-east-1}"

running=$(aws ec2 describe-instances \
  --region "$REGION" \
  --filters \
    'Name=tag:Name,Values=prog-strength-developer-worker' \
    'Name=instance-state-name,Values=pending,running' \
  --query 'Reservations[].Instances[].InstanceId' \
  --output text)

if [ -n "$running" ]; then
  echo "::error::A prog-strength-developer worker is already running: $running"
  echo "::error::v1 enforces single-instance concurrency. Wait for the current run to terminate."
  exit 1
fi

echo "preflight: no in-flight worker; OK to proceed"
```

- [ ] **Step 2: Make executable**

```bash
chmod +x scripts/preflight.sh
```

- [ ] **Step 3: Lint with shellcheck**

```bash
shellcheck scripts/preflight.sh
```
Expected: clean.

- [ ] **Step 4: Commit**

```bash
git add scripts/preflight.sh
git commit -m "feat(scripts): preflight check for single-instance concurrency"
```

---

## Phase 4 — GitHub Actions workflow

### Task 11: dispatch-sow.yml

**Files:**
- Create: `.github/workflows/dispatch-sow.yml`

- [ ] **Step 1: Write the workflow**

```yaml
name: Dispatch SOW

on:
  workflow_dispatch:
    inputs:
      sow_path:
        description: "Path to the SOW within prog-strength-docs (e.g. sows/foo.md)"
        required: true
        type: string

permissions:
  id-token: write   # for AWS OIDC federation
  contents: read

env:
  AWS_REGION: us-east-1
  TF_VERSION: 1.7.5
  # Set in repo Variables (not Secrets — these are non-sensitive).
  TF_STATE_BUCKET: ${{ vars.TF_STATE_BUCKET }}
  TF_STATE_LOCK_TABLE: ${{ vars.TF_STATE_LOCK_TABLE }}

jobs:
  dispatch:
    runs-on: ubuntu-latest
    timeout-minutes: 15   # workflow runs Terraform then exits; never long
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_GHA_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Preflight — no in-flight worker
        run: ./scripts/preflight.sh

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform init
        working-directory: terraform
        run: |
          terraform init \
            -backend-config="bucket=${TF_STATE_BUCKET}" \
            -backend-config="region=${AWS_REGION}" \
            -backend-config="dynamodb_table=${TF_STATE_LOCK_TABLE}"

      - name: Terraform apply
        working-directory: terraform
        run: |
          terraform apply -auto-approve \
            -var "sow_path=${{ inputs.sow_path }}"

      - name: Summary
        run: |
          INSTANCE_ID=$(cd terraform && terraform output -raw worker_instance_id)
          echo "## Worker dispatched" >> "$GITHUB_STEP_SUMMARY"
          echo "" >> "$GITHUB_STEP_SUMMARY"
          echo "- **SOW:** \`${{ inputs.sow_path }}\`" >> "$GITHUB_STEP_SUMMARY"
          echo "- **Instance ID:** \`$INSTANCE_ID\`" >> "$GITHUB_STEP_SUMMARY"
          echo "- **CloudWatch:** /aws/ec2/prog-strength-developer/$INSTANCE_ID" >> "$GITHUB_STEP_SUMMARY"
          echo "- **SSM:** \`aws ssm start-session --target $INSTANCE_ID\`" >> "$GITHUB_STEP_SUMMARY"
```

- [ ] **Step 2: Lint with actionlint (best-effort)**

```bash
actionlint .github/workflows/dispatch-sow.yml
```
If actionlint isn't installed: `brew install actionlint`. If you skip it, run `yamllint`:

```bash
yamllint -d "{extends: default, rules: {line-length: disable, document-start: disable, truthy: disable}}" \
  .github/workflows/dispatch-sow.yml
```
Expected: clean (or only `line-too-long` warnings, which we ignore).

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/dispatch-sow.yml
git commit -m "feat(workflow): dispatch-sow workflow_dispatch entry point"
```

---

## Phase 5 — Documentation

### Task 12: docs/README.md (system overview)

**Files:**
- Create: `docs/README.md`

- [ ] **Step 1: Write the system overview**

```markdown
# prog-strength-developer

An autonomous developer for [Prog Strength](https://github.com/Prog-Strength). Picks
up a designated SOW from `prog-strength-docs`, spins up an ephemeral EC2 worker
in a dedicated VPC, runs Claude Code unattended against that SOW, opens PRs in
every affected repo, and self-terminates.

Specification: `prog-strength-docs/sows/prog-strength-developer.md`.

## What this repo contains

- `terraform/` — IaC for the AWS resources (VPC, IAM, EC2 launch template,
  CloudWatch log group, Secrets Manager references).
- `bootstrap/` — the cloud-init userdata template and the Claude Code prompt
  template, both rendered at apply time with the SOW path baked in.
- `scripts/` — small helpers used by the GitHub Actions workflow.
- `.github/workflows/dispatch-sow.yml` — the manual `workflow_dispatch` entry
  point. Takes `sow_path` as input.
- `docs/` — system overview (this file), one-time bootstrap runbook, and
  troubleshooting notes.

## How a run works

1. You write a SOW in `prog-strength-docs/sows/<name>.md` with YAML
   frontmatter listing the affected `repos:`.
2. You open this repo on GitHub → Actions → "Dispatch SOW" → Run workflow →
   paste the SOW path (e.g. `sows/foo.md`).
3. The workflow:
   - Pre-flights for a running worker (single-instance concurrency).
   - Assumes the AWS GHA OIDC role.
   - Runs `terraform apply` with the SOW path templated into the worker's
     userdata.
   - Exits. Does NOT wait for the worker.
4. The EC2 worker boots, installs deps, fetches Claude OAuth credentials
   and a GitHub App installation token from Secrets Manager, clones the
   SOW + every repo listed in `repos:`, runs Claude Code, opens PRs in
   each modified repo, and self-terminates.
5. You review and merge the resulting PRs at your own pace.

## What the worker has access to

- The two Secrets Manager entries (`prog-strength-developer/claude-credentials`,
  `prog-strength-developer/github-app`) and nothing else.
- The ability to terminate EC2 instances tagged `Name=prog-strength-developer-worker`
  (i.e. itself).
- The dedicated CloudWatch log group.
- All outbound internet (GitHub, Anthropic, AWS, package registries).

It is on a separate VPC with no peering to the application VPC. It cannot
reach the prod API/MCP/DB.

## What the worker does NOT do

- Merge PRs (you do).
- Notify on completion (PRs visible in GitHub + CloudWatch are the signal).
- Run in parallel with another worker (v1 enforces single-instance).
- Modify repos outside the `repos:` list in the SOW frontmatter.

## First-time setup

See [`setup.md`](./setup.md) for the bootstrap runbook.

## Debugging

See [`troubleshooting.md`](./troubleshooting.md).

## Cost

Roughly $0.30 per SOW run (mostly EC2 time on `t3.large`), plus ~$1/month of
always-on costs (Secrets Manager + log retention). A $50/month AWS budget alarm
fires before anything anomalous.
```

- [ ] **Step 2: Commit**

```bash
git add docs/README.md
git commit -m "docs: system overview"
```

---

### Task 13: docs/setup.md (bootstrap runbook)

**Files:**
- Create: `docs/setup.md`

- [ ] **Step 1: Write the runbook**

```markdown
# Setup runbook (one-time)

The autonomous developer cannot build itself the first time. Walk through
these steps in order. Estimated time: 30–45 minutes.

## Prerequisites

- An AWS account where you have admin (or close-to-admin) permissions for IAM,
  VPC, EC2, Secrets Manager, CloudWatch, S3, and DynamoDB.
- The AWS CLI installed locally and configured (`aws configure`).
- Terraform 1.7+ installed locally (tfenv recommended; `.terraform-version`
  in this repo will be honored).
- The GitHub CLI (`gh`) installed and authenticated against your account.
- Admin access to the `Prog-Strength` GitHub organization.

## 1. Create the prog-strength-developer repository on GitHub

```bash
gh repo create Prog-Strength/prog-strength-developer \
  --public \
  --description "Autonomous developer for Prog Strength" \
  --confirm
```

(Or skip and create via the GitHub UI.)

## 2. Create the GitHub App

Go to https://github.com/organizations/Prog-Strength/settings/apps/new and
create an app named **Prog Strength Developer** with:

- **Homepage URL:** `https://github.com/Prog-Strength/prog-strength-developer`
- **Webhook:** uncheck "Active" (we don't use webhooks).
- **Repository permissions:**
  - Contents: Read & write
  - Pull requests: Read & write
  - Issues: Read & write
  - Workflows: Read & write
  - Metadata: Read-only
- **Organization permissions:** none.
- **Where can this app be installed?** Only on this account.

After creating:

1. Generate a private key (Settings → General → Private keys → Generate
   a private key). A `.pem` file downloads. Keep it safe; you'll paste
   it into Secrets Manager.
2. Install the App on the org (left sidebar → Install App → Prog-Strength).
   Choose "All repositories" for v1. Note the installation ID from the
   URL after install (it's the last path segment).
3. Note the App ID from the General settings page.

## 3. Set up the AWS GitHub Actions OIDC provider

The provider is account-level and only needs to exist once. If you've used
GHA OIDC against this account before, skip.

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

(The thumbprint is GitHub's documented value; AWS will accept it.)

## 4. Create Terraform state backend resources

The state bucket and lock table are managed OUTSIDE Terraform. Pick names
that include your AWS account ID for uniqueness.

```bash
export TF_BUCKET="prog-strength-tfstate-$(aws sts get-caller-identity --query Account --output text)"
export TF_LOCK_TABLE="prog-strength-tflock"

aws s3api create-bucket --bucket "$TF_BUCKET" --region us-east-1
aws s3api put-bucket-versioning --bucket "$TF_BUCKET" --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption --bucket "$TF_BUCKET" \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

aws dynamodb create-table \
  --table-name "$TF_LOCK_TABLE" \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

## 5. Push this repository

If you haven't already pushed the local repo to GitHub:

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-developer
git remote add origin https://github.com/Prog-Strength/prog-strength-developer.git
git push -u origin main
git push -u origin feat/initial-bootstrap
```

## 6. First terraform apply (local, NOT from CI)

This creates VPC + IAM roles + log group + Secrets Manager references. The
references will fail unless the secrets exist, so we'll create empty secrets
first.

```bash
aws secretsmanager create-secret \
  --name prog-strength-developer/claude-credentials \
  --secret-string '{"placeholder": true}'

aws secretsmanager create-secret \
  --name prog-strength-developer/github-app \
  --secret-string '{"placeholder": true}'

cd terraform
terraform init \
  -backend-config="bucket=$TF_BUCKET" \
  -backend-config="region=us-east-1" \
  -backend-config="dynamodb_table=$TF_LOCK_TABLE"

terraform apply
```

Confirm and apply. Output includes the GitHub Actions role ARN — save it.

## 7. Configure GitHub Actions secrets and variables

In the prog-strength-developer repo on GitHub → Settings → Secrets and
variables → Actions:

**Variables (not Secrets):**
- `TF_STATE_BUCKET` — the bucket name from step 4.
- `TF_STATE_LOCK_TABLE` — the DynamoDB table name from step 4.

**Secrets:**
- `AWS_GHA_ROLE_ARN` — the `github_actions_role_arn` Terraform output.

## 8. Seed the real secrets

```bash
# Claude OAuth credentials — must do `claude login` locally first if not done.
aws secretsmanager put-secret-value \
  --secret-id prog-strength-developer/claude-credentials \
  --secret-string "$(cat ~/.claude/credentials.json)"

# GitHub App. Replace APP_ID, INSTALLATION_ID, and the key path.
APP_ID=123456
INSTALLATION_ID=7890123
PRIVATE_KEY_PATH=~/Downloads/prog-strength-developer.YYYY-MM-DD.private-key.pem

cat > /tmp/gh-app.json <<EOF
{
  "app_id": $APP_ID,
  "installation_id": $INSTALLATION_ID,
  "private_key": $(jq -Rs . < "$PRIVATE_KEY_PATH")
}
EOF

aws secretsmanager put-secret-value \
  --secret-id prog-strength-developer/github-app \
  --secret-string "file:///tmp/gh-app.json"

rm /tmp/gh-app.json
```

## 9. Trivial-SOW smoke test

Add a minimal SOW to `prog-strength-docs/sows/test-developer-bootstrap.md`:

```markdown
---
status: ready_for_implementation
repos:
  - prog-strength-docs
---

# Test: Developer Bootstrap

**Status:** Test only · **Last updated:** TODAY

Adds a single line "Tested by prog-strength-developer on YYYY-MM-DD" to
the bottom of `README.md` in prog-strength-docs.
```

Push it, then in the prog-strength-developer repo on GitHub: Actions →
Dispatch SOW → run with `sows/test-developer-bootstrap.md`.

Within ~15 minutes you should see a PR open on prog-strength-docs and the
EC2 instance terminate. Watch progress in CloudWatch
(`/aws/ec2/prog-strength-developer/<instance-id>`).

## You're done

Subsequent SOWs are dispatched via the same workflow. Re-seed
`claude-credentials` every few months when the OAuth refresh token expires
(see `troubleshooting.md`).
```

- [ ] **Step 2: Commit**

```bash
git add docs/setup.md
git commit -m "docs: first-time setup runbook"
```

---

### Task 14: docs/troubleshooting.md

**Files:**
- Create: `docs/troubleshooting.md`

- [ ] **Step 1: Write the troubleshooting guide**

```markdown
# Troubleshooting

## "Worker dispatched but no PRs ever appeared"

Order of operations:

1. **Check if the EC2 actually terminated.**
   ```bash
   aws ec2 describe-instances \
     --filters 'Name=tag:Name,Values=prog-strength-developer-worker' \
               'Name=instance-state-name,Values=pending,running' \
     --query 'Reservations[].Instances[].[InstanceId,LaunchTime]' \
     --output table
   ```
   If one's still alive, either the work is in progress (check CloudWatch)
   or the backstop hasn't fired yet.

2. **Read CloudWatch logs.** Find the instance ID from the GitHub Actions
   summary, then:
   ```bash
   aws logs tail /aws/ec2/prog-strength-developer \
     --log-stream-names <instance-id> \
     --since 1h
   ```
   Scan the last ~200 lines. Common failure modes are listed below.

3. **SSH (well, SSM) into the box** if it's still alive:
   ```bash
   aws ssm start-session --target <instance-id>
   ```

## Common failure modes

### "Bootstrap dies fetching secrets"

CloudWatch shows `AccessDeniedException` from Secrets Manager. Either:

- The secret doesn't exist yet (run setup step 8).
- The worker IAM role doesn't have permission. Check
  `terraform/iam.tf`'s `ReadDeveloperSecrets` statement; the resource
  ARN pattern must match the secret's actual ARN (Secrets Manager appends
  a random suffix; the pattern uses `*` to match it).

### "GitHub App token mint fails with 401"

CloudWatch shows the Python urllib snippet failing on the
`/access_tokens` call. Causes:

- App ID or installation ID in the secret is wrong. Re-check the App's
  settings page and the install URL.
- Private key in the secret is malformed (missing newlines, wrong PEM
  header, etc.). The `jq -Rs .` step in setup converts a PEM file to a
  JSON-safe string; if you skipped it, the literal newlines will have
  broken the JSON.
- App is installed on the org but lacks the necessary repository
  permissions. Re-check the App configuration: Contents (write), Pull
  Requests (write), Workflows (write).

### "Claude login fails with 401"

The OAuth refresh token in `~/.claude/credentials.json` has expired (or
the file in Secrets Manager is stale). Re-run `claude login` locally,
copy the new credentials.json, and re-seed the secret:

```bash
aws secretsmanager put-secret-value \
  --secret-id prog-strength-developer/claude-credentials \
  --secret-string "$(cat ~/.claude/credentials.json)"
```

Refresh tokens last several months; expect to do this 2–4× per year.

### "PRs opened but worker still running"

Either Claude wrote PRs but didn't exit cleanly (the 6h backstop will
eventually fire), or the script is stuck on a downstream step. SSM in
and check `ps -ef | grep claude` and `tail -f /var/log/prog-strength-developer/*.log`.

### "Workflow fails on preflight check"

Output says "A prog-strength-developer worker is already running:
<instance-id>". This is by design — v1 is single-instance. Either wait
for the current worker to terminate or, if you know it's stuck, manually
terminate:

```bash
aws ec2 terminate-instances --instance-ids <instance-id>
```

Then re-dispatch.

### "Terraform apply fails with 'OIDC provider does not exist'"

You skipped setup step 3. Create the provider:

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

### "Terraform plan wants to destroy and recreate the launch template every run"

Expected. Each `terraform apply` from the workflow renders the userdata
template with the new `sow_path` baked in, which changes the launch
template's user_data, which Terraform considers a forced replacement.
The launch template recreation is cheap; the previously-launched
instances (if any) are not affected.

## "I want to nuke everything and start over"

```bash
cd terraform
terraform destroy
```

Then manually delete the S3 state bucket, DynamoDB lock table, the two
Secrets Manager entries, and the OIDC provider. The GitHub App can stay;
just uninstall it from the org if you want it gone.

## "How do I see what Claude was thinking on a failed run?"

CloudWatch retains 30 days of logs. The instance's log stream is named
exactly the instance ID, viewable in the AWS Console under CloudWatch
Logs → /aws/ec2/prog-strength-developer.

Within the stream:

- `[userdata ...]` lines are progress markers from `userdata.sh.tpl`.
- Plain lines (no prefix) are Claude's stdout — its thoughts, tool calls,
  and subagent dispatches.

The last 50–100 lines are usually where the failure mode is most
diagnostic.
```

- [ ] **Step 2: Commit**

```bash
git add docs/troubleshooting.md
git commit -m "docs: troubleshooting guide"
```

---

## Phase 6 — SOW frontmatter convention in prog-strength-docs

### Task 15: Add frontmatter convention to prog-strength-docs README

**Files:**
- Modify: `repos/prog-strength-docs/README.md` (read first to find the right insertion point)
- Move: the existing `sows/prog-strength-developer.md` SOW (already on disk) into a committed state

- [ ] **Step 1: Read the existing prog-strength-docs README**

```bash
cat /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-docs/README.md
```

(Note the structure — sections, conventions in use. Adapt the insertion below to match.)

- [ ] **Step 2: Add a SOW frontmatter section to the README**

Append (or insert under an existing "Conventions" heading if one exists) a section:

```markdown
## SOW frontmatter

Every SOW in `sows/` SHOULD include YAML frontmatter at the top:

```yaml
---
status: <draft|ready_for_implementation|in_review|shipped>
repos:
  - prog-strength-api
  - prog-strength-agent
  - prog-strength-infra
  - prog-strength-docs   # always include when the SOW itself will be edited
---
```

`status` is informational and editorial — the autonomous developer
(`prog-strength-developer`) does not gate dispatch on it; the owner does
by choosing when to click "Run workflow."

`repos:` is load-bearing for `prog-strength-developer`. Each entry must
be a repository name in the `Prog-Strength/<name>` namespace. The
autonomous developer reads this list to decide which repos to clone.

Existing SOWs are not retroactively annotated — only SOWs that are
actually slated for autonomous re-runs need frontmatter. Anything
already `shipped` can stay as-is.

See `sows/prog-strength-developer.md` for the SOW that introduces this
convention.
```

- [ ] **Step 3: Switch to a feature branch and commit the SOW + plan + README change together**

The SOW and plan files are already on disk in `prog-strength-docs/` from the brainstorming + planning phases. Create the feature branch and stage them alongside the README update.

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-docs
git checkout -b feat/prog-strength-developer
git add README.md sows/prog-strength-developer.md plans/2026-06-02-prog-strength-developer.md
git commit -m "docs: prog-strength-developer SOW, plan, and frontmatter convention"
```

---

## Phase 7 — Wrap-up

### Task 16: Smoke-test the local Terraform configuration

**Files:** none (validation only)

- [ ] **Step 1: From `prog-strength-developer/terraform/`, run a no-backend validate + format check**

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-developer/terraform
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
```
Expected: PASS.

- [ ] **Step 2: From `prog-strength-developer/`, run shellcheck on all scripts**

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-developer
shellcheck scripts/preflight.sh
# userdata.sh.tpl will emit template-variable false positives; eyeball and skip.
shellcheck -x bootstrap/userdata.sh.tpl || true
```

- [ ] **Step 3: yamllint the workflow file**

```bash
yamllint -d "{extends: default, rules: {line-length: disable, document-start: disable, truthy: disable}}" \
  .github/workflows/dispatch-sow.yml
```
Expected: clean.

- [ ] **Step 4: Verify the SOW frontmatter parses**

```bash
python3 - <<'PY'
import re, yaml
with open("/Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-docs/sows/prog-strength-developer.md") as f:
    text = f.read()
m = re.match(r"^---\n(.*?)\n---\n", text, re.DOTALL)
meta = yaml.safe_load(m.group(1))
assert meta["status"] == "ready_for_implementation"
assert "prog-strength-developer" in meta["repos"]
assert "prog-strength-docs" in meta["repos"]
print("ok")
PY
```
Expected: `ok`.

- [ ] **Step 5: Commit a smoke-test marker**

This is purely a "tests pass" marker so the final commit is clearly the end of bootstrap.

```bash
cd /Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-developer
# Nothing to commit if all prior tasks landed cleanly; verify state.
git status
```

(No commit if `git status` is clean. The task is complete on validation pass.)

---

## Final manual handoff

After all 16 tasks land:

1. The owner pushes both repos and opens PRs:
   - `prog-strength-developer/feat/initial-bootstrap` → main (new repo on GitHub will need to be created first; see setup runbook Task 13).
   - `prog-strength-docs/feat/prog-strength-developer` → main.
2. Once PRs merge, follow `docs/setup.md` to bootstrap the AWS resources and run the trivial-SOW smoke test.
3. After the smoke test passes, the system is live.

---

## Spec coverage check

| SOW requirement | Plan task |
|---|---|
| New `prog-strength-developer` repo with skeleton | Task 1 |
| Dedicated VPC with no peering to app VPC | Task 3 |
| Public subnet + IGW + no NAT | Task 3 |
| Worker security group (no inbound, all outbound) | Task 3 |
| Worker IAM role with least-privilege permissions | Task 4 |
| GHA OIDC role restricted to repo+branch | Task 4 |
| CloudWatch log group with 30-day retention | Task 5 |
| Secrets Manager refs (data sources only) | Task 5 |
| AMI lookup + launch template + gated instance | Task 6 |
| Terraform outputs for downstream consumers | Task 7 |
| Userdata: install deps, fetch secrets, clone repos, run Claude, self-terminate | Task 8 |
| 6h hard backstop via systemd-run | Task 8 |
| Claude prompt template with workflow instructions | Task 9 |
| Preflight check for single-instance concurrency | Task 10 |
| `workflow_dispatch` workflow with OIDC + preflight + apply + exit | Task 11 |
| System overview docs | Task 12 |
| First-time bootstrap runbook | Task 13 |
| Troubleshooting guide | Task 14 |
| SOW frontmatter convention documented in prog-strength-docs | Task 15 |
| Local validation of Terraform, shell, YAML | Task 16 |

Every SOW requirement maps to at least one task.
