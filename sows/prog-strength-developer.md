---
status: shipped
repos:
  - prog-strength-developer
  - prog-strength-docs
---

# Prog Strength Developer

**Status**: Shipped · **Last updated**: 2026-06-02

## Introduction

Prog Strength is a side project, and side projects ship at the pace their owner has hours to spend on them. Recent SOWs (intent-driven context enrichment, persistent chat sessions, streaming TTS) have demonstrated that the brainstorm → spec → plan → subagent-driven execution workflow produces high-quality, multi-repo changes — but every one of those still depends on a human in the loop driving the workflow turn by turn. The owner has a Claude Code Max subscription that, in principle, can run unattended. The bottleneck is no longer Claude's capability; it's the manual scaffolding around it.

This SOW describes an **autonomous developer** that picks up a designated SOW from `prog-strength-docs`, spins up an isolated EC2 instance in a dedicated AWS VPC, executes the full implementation workflow against the relevant repos, opens pull requests against each one, and tears itself down. The owner kicks it off manually via a GitHub Action and then comes back later to review pull requests. No polling, no idle infrastructure, no persistent worker pool.

The system is built on three constraints that lock in the architecture:

- **Isolation is paramount.** A worker that's making code changes autonomously is also a worker that can break things. Network-level isolation (separate VPC, no peering with prod) means a misbehaving or compromised worker cannot reach the application stack. Pull-request boundaries (never push directly to `main`) mean even a successful-but-wrong run lands as a reviewable diff, not a production change.
- **No idle cost.** The worker is provisioned per-task and self-terminates on completion. There is no always-on infrastructure beyond a few cents per month of Secrets Manager and CloudWatch retention. A task that runs in two hours costs ~$0.20 of EC2 time; a project that gets one autonomous task a week costs ~$1/month all-in.
- **The SOW is the source of truth.** Prog Strength is a decentralized application — most features cross 3–4 repositories (api, agent, mcp, web, mobile, infra). GitHub issues are repo-scoped and break down at this boundary. SOWs in `prog-strength-docs` are already the natural unit of cross-cutting work, and adding lightweight YAML frontmatter (`repos:` list, `status:` field) lets the autonomous developer pick them up without further plumbing.

After this work ships, the owner workflow becomes: write a SOW in `prog-strength-docs`, push it, open the `prog-strength-developer` repo on GitHub, click "Run workflow," paste the SOW path, walk away. An hour or two later the worker has opened pull requests in each affected repo. The owner reviews and merges (or sends back for fixes) at their own pace. CloudWatch holds the logs if anything went sideways.

## Proposed Solution

A new repository, **`prog-strength-developer`**, owns the entire system: Terraform code for the AWS resources, the GitHub Actions workflow that dispatches a run, the cloud-init script the EC2 worker runs at boot, and the prompt template Claude Code executes. It does NOT house any application code, and `prog-strength-infra` does NOT house any developer code — the two are kept separate to reflect their genuinely different domains (application infrastructure vs. ephemeral development infrastructure).

When the owner clicks **Run workflow** in `prog-strength-developer` and supplies a `sow_path` input (e.g. `sows/intent-driven-context-enrichment.md`), the GitHub Action:

1. Configures AWS credentials from repository secrets.
2. Calls a lightweight pre-flight script that checks for any existing `prog-strength-developer` EC2 instance in the dedicated VPC. If one is found, the workflow fails immediately with a clear message — only one worker runs at a time in v1.
3. Runs `terraform apply` against the developer VPC to launch a new EC2 instance. The cloud-init userdata is templated with the `sow_path` so the worker knows which SOW to act on.
4. Exits successfully as soon as Terraform finishes. The Action does NOT wait for the worker to complete — running a GitHub Action for hours to babysit an EC2 instance defeats the purpose of having the EC2.

The EC2 worker boots into a userdata script that:

1. Installs dependencies: `claude` CLI, `gh`, `git`, `node`, `go`, `uv`, `python3`, AWS CLI v2, the CloudWatch agent. (Long-term, these are baked into a custom AMI to cut boot time; v1 installs them on demand.)
2. Fetches the cached Claude OAuth credentials from Secrets Manager and writes them to `~/.claude/credentials.json`. Claude Code's normal refresh-token machinery handles renewal during the run.
3. Fetches the Prog Strength Developer GitHub App's private key from Secrets Manager, mints an installation token, and configures `gh auth login --with-token`. The token's hour-long expiry is fine because the developer App can mint a fresh one when needed; for runs longer than an hour, the userdata re-mints periodically.
4. Clones `prog-strength-docs`, reads the SOW at `sow_path`, and parses its frontmatter to discover the `repos:` list.
5. Clones each listed repo into `/workspace`.
6. Renders the Claude Code prompt template with the SOW path baked in, then runs `claude --print --dangerously-skip-permissions < prompt.md`. The prompt instructs Claude to follow the standard workflow: validate the spec with `superpowers:brainstorming` (lightly — the SOW is the spec), produce a plan via `superpowers:writing-plans` if one isn't already in `prog-strength-docs/plans/`, execute it via `superpowers:subagent-driven-development`, and open PRs in each affected repo on a feature branch named after the SOW slug. The prompt ends with explicit non-interactivity instructions: Claude should make reasonable judgment calls rather than asking clarifying questions, since there's no one to ask.
7. On Claude's exit (success, non-zero, or hard timeout), the userdata script flushes CloudWatch logs and calls `aws ec2 terminate-instances` with its own instance ID retrieved from IMDS.

A backstop systemd timer hard-kills the work after 6 hours of wall-clock time. This is the budget for the longest reasonable SOW; if a run hits the limit, the instance still terminates and the owner can investigate via CloudWatch.

Successful completion is signaled by the presence of new PRs in the affected repos. Failed completion is signaled by their absence plus whatever CloudWatch captured. v1 deliberately ships without Slack/email notifications — the GitHub PR list and CloudWatch console are sufficient, and the notification surface can be added in a follow-up once the system has run a few times.

Access to the running worker for live debugging uses AWS Systems Manager Session Manager, not SSH — no port 22 exposed, no keypairs to manage, every session audited via CloudTrail. `aws ssm start-session --target $INSTANCE_ID` is the SSH replacement.

## Goals and Non-Goals

### Goals

- Create a new repository `prog-strength-developer` that houses Terraform for the AWS resources, a GitHub Actions workflow for manual dispatch, a userdata bootstrap script, and the Claude Code prompt template.
- Provision a dedicated AWS VPC for the autonomous developer with no peering or route table connection to the application VPC in `prog-strength-infra`. Worker lives in a public subnet with a public IP for outbound internet; no NAT gateway.
- Implement a `workflow_dispatch` GitHub Actions workflow that takes a `sow_path` input, pre-flights the "no existing instance" check, runs `terraform apply` to launch a worker, and exits without polling.
- Bootstrap the EC2 worker through cloud-init userdata that installs dependencies, pulls secrets, clones the SOW + affected repos, runs Claude Code with `--print --dangerously-skip-permissions`, and self-terminates on completion.
- Introduce a `repos:` + `status:` YAML frontmatter convention in `prog-strength-docs/sows/*.md`. Existing SOWs are retroactively annotated only when they're scheduled for re-runs; new SOWs get frontmatter from the start.
- Configure a GitHub App (`Prog Strength Developer`) installed on the Prog-Strength organization with read+write permissions on the relevant repos. The worker authenticates by minting installation tokens from the App's private key.
- Cache the owner's Claude OAuth credentials in AWS Secrets Manager so the worker can run Claude Code non-interactively. Document the one-time seeding step in a runbook.
- Stream the worker's stdout and stderr to a CloudWatch log group `/aws/ec2/prog-strength-developer/<instance-id>`, retained for 30 days.
- Enforce single-instance concurrency via the GitHub Action pre-flight check, plus a tag-conditioned IAM policy preventing the worker from terminating any instance other than itself.
- Configure access to the worker via AWS Systems Manager Session Manager (no SSH, no port 22 exposed).
- Apply a 6-hour wall-clock backstop via systemd timer that terminates the instance regardless of Claude's state.
- Write a one-time bootstrap runbook in `prog-strength-developer/docs/setup.md` covering: GitHub App creation, Secrets Manager seeding (Claude creds, App private key), AWS credentials for the GitHub Action, initial `terraform apply` of the persistent resources (VPC, IAM role, security group, Secrets Manager entries, GitHub Actions OIDC trust).
- After implementation PRs are opened, open an additional pull request in `prog-strength-docs` (always, regardless of whether the SOW listed it under `repos:`) that flips the SOW's frontmatter `status:` to `shipped`, body `**Status**:` to `Shipped`, and `**Last updated**:` to the run date. Merging that PR is the owner's one-action signal that the work is complete — no manual SOW editing required.

### Non-Goals

- **Notifications (Slack / email / SMS) on run completion or failure.** The PR list and CloudWatch console are the v1 signal. Notifications are a clean follow-up once the system has run a few times and we know what content is useful in them.
- **Auto-discovery of SOWs from `prog-strength-docs`.** The worker does not scan for status-flagged SOWs; the owner explicitly dispatches by path. This avoids ambiguity ("which one should I pick?") and makes the kickoff fully deterministic.
- **Concurrent workers.** v1 strictly serializes runs. If the owner dispatches while one is in flight, the workflow fails with a clear message. Concurrency is a defer-until-actually-needed item — the failure mode of "branch name collision in 3 repos at once" is annoying enough that getting it right deserves a separate, focused round.
- **A queue for dispatch attempts during in-flight runs.** Closely related to concurrency. v1 returns "busy, try again" rather than queuing.
- **A web UI or dashboard for the autonomous developer.** The GitHub Actions UI is the dispatch surface; the GitHub PR list is the output surface; the AWS console (CloudWatch + EC2) is the debugging surface. No bespoke UI.
- **Pre-baked custom AMI for fast boot.** v1 uses Amazon Linux 2023 + an on-demand install in userdata. Boot-to-Claude is ~5 minutes. If that becomes a real friction point, a packer-built AMI in a follow-up cuts it to under a minute.
- **Sharing the application VPC.** Explicitly rejected — see the introduction. The autonomous developer runs in its own VPC with no peering to prod.
- **Application-stack integration tests on the worker.** The worker writes code and runs unit tests; it does NOT spin up a copy of the prod stack to integration-test against. End-to-end validation remains a human responsibility on the resulting PRs.
- **A second "review" worker that critiques the implementer's PRs.** The owner is the reviewer. Adding an auto-reviewer is an interesting future direction but explicitly out of scope here.
- **Cross-organization SOW execution.** The GitHub App is scoped to the Prog-Strength org. The worker cannot operate on repositories outside it.
- **Persistent learning across runs.** Each worker boots fresh. There is no memory of past runs beyond what's already in the repos themselves (commit history, PR conversations, existing SOWs).
- **A "Status: in_progress" lock on the SOW itself.** v1 does not write a marker back to `prog-strength-docs` saying "this SOW is being worked on right now." The GitHub Action's pre-flight check is the lock; the SOW status field is owner-curated.
- **Implementing this SOW via the autonomous developer itself.** Chicken-and-egg — the system bootstraps manually the first time. Future SOWs that touch only `prog-strength-developer` and `prog-strength-docs` are fair game.

## Implementation Details

### Repository Layout

A new repository `prog-strength-developer` at `https://github.com/Prog-Strength/prog-strength-developer`, structured as:

```
.github/
  workflows/
    dispatch-sow.yml       # workflow_dispatch entry point
terraform/
  main.tf                  # provider, backend (S3 + DynamoDB lock)
  vpc.tf                   # VPC, subnet, IGW, route table, security group
  iam.tf                   # worker role + instance profile, OIDC trust for the workflow
  ec2.tf                   # launch template, AMI lookup, userdata templating
  secrets.tf               # Secrets Manager resource references (not contents)
  cloudwatch.tf            # log group, retention
  variables.tf             # AWS region, instance type, max runtime, etc.
  outputs.tf               # log group name, role ARN
bootstrap/
  userdata.sh.tpl          # cloud-init script, templated with sow_path
  prompt.md.tpl            # Claude Code prompt, templated with sow_path
docs/
  README.md                # what this repo is, how to use it
  setup.md                 # one-time bootstrap runbook
  troubleshooting.md       # common failure modes + where to look
```

Terraform state lives in an S3 bucket distinct from any used by `prog-strength-infra`. State locking via DynamoDB. Both are created during initial bootstrap (manually, before any `terraform apply` runs).

### SOW Frontmatter Convention

Add YAML frontmatter to the top of every SOW in `prog-strength-docs/sows/*.md`:

```yaml
---
status: ready_for_implementation
repos:
  - prog-strength-api
  - prog-strength-agent
  - prog-strength-infra
  - prog-strength-docs        # always include if the SOW itself will be edited
---
```

Valid `status` values:

- `draft` — under design; not ready for autonomous implementation.
- `ready_for_implementation` — the owner is satisfied with the spec; the autonomous developer may be dispatched against it.
- `in_review` — PRs are open, awaiting human review.
- `shipped` — merged and deployed.

The worker does NOT gate on status. The status field is informational and editorial — the owner reads it to decide whether a SOW is dispatch-worthy before clicking "Run workflow." Gating on it would force a status-edit PR before every dispatch, which is friction without protection (the owner is the only person who dispatches, and the dispatch action is itself the trust signal).

`repos:` is read by the worker and is load-bearing. Each entry is a repository name in the `Prog-Strength/<name>` namespace. The worker clones each one into `/workspace/<name>`. Inclusion in `repos:` does NOT obligate the worker to actually modify the repo; it's a clone-list, not a touch-list. (Claude may decide a referenced repo doesn't need changes.)

Backfill of existing SOWs: only the SOWs slated for actual re-runs need retroactive frontmatter. Most existing SOWs are `status: shipped` and won't be dispatched.

### Network Architecture

A dedicated VPC `prog-strength-developer-vpc` with:

- **CIDR**: `10.20.0.0/16`. Chosen to be distinct from any range used by `prog-strength-infra` (which uses `10.10.0.0/16`) so that a future VPC peer could in theory be added without collision — though no peering is configured today.
- **One public subnet** `10.20.1.0/24` in a single availability zone. Single AZ is fine for ephemeral compute; there is no high-availability requirement when each worker exists for at most a few hours.
- **Internet Gateway** attached, with the public subnet's route table sending `0.0.0.0/0` to the IGW.
- **No NAT Gateway.** The worker has a public IP and reaches GitHub, Anthropic, AWS APIs, and package registries directly. Saves ~$35/month.
- **Security group** `prog-strength-developer-worker-sg`:
  - **Inbound: none.** No SSH, no anything. Access for debugging is via SSM Session Manager.
  - **Outbound: all (0.0.0.0/0, all protocols).** The worker needs to reach a long tail of package registries (npm, PyPI, Go module proxy, GitHub releases, etc.) plus GitHub, the Anthropic API, AWS endpoints, and arbitrary URLs Claude may fetch during a run. Restricting outbound here would generate constant breakage with little security benefit (the trust boundary is already the VPC isolation and the IAM role).
- **No VPC peering** to the application VPC. The two networks cannot route to each other. A misbehaving worker cannot reach the prod API even if it had credentials.

### IAM and Permissions

Two IAM roles:

**`prog-strength-developer-worker-role`** — assumed by the EC2 worker via its instance profile. Permissions:

- `secretsmanager:GetSecretValue` on exactly two secret ARNs: `prog-strength-developer/claude-credentials` and `prog-strength-developer/github-app`. Least-privilege; the worker cannot read any other secret.
- `ec2:TerminateInstances` conditioned on `aws:ResourceTag/Name = prog-strength-developer-worker`. The worker can terminate ONLY instances bearing the developer worker tag (in practice, itself). It cannot accidentally terminate instances in `prog-strength-infra`.
- `logs:CreateLogStream` and `logs:PutLogEvents` on the developer log group.
- `ssm:UpdateInstanceInformation`, `ssmmessages:*`, `ec2messages:*` — the standard set required for SSM Session Manager to function. Most easily granted via the AWS-managed policy `AmazonSSMManagedInstanceCore`.

**`prog-strength-developer-github-actions-role`** — assumed by the GitHub Actions workflow via OIDC federation (no static IAM user keys). Permissions:

- `ec2:DescribeInstances`, `ec2:RunInstances`, `ec2:CreateTags`, `iam:PassRole` (to pass the worker role into the launch), `ec2:Describe*` for Terraform.
- `s3:GetObject`/`PutObject` on the Terraform state bucket.
- `dynamodb:GetItem`/`PutItem`/`DeleteItem` on the state lock table.

The OIDC trust policy restricts the role to be assumed only from the `Prog-Strength/prog-strength-developer` repository, on the `main` branch. This means a workflow run from a feature branch of `prog-strength-developer` cannot assume the role — useful so that draft Terraform changes don't accidentally provision resources.

### Secrets Manager

Two secrets created manually during bootstrap (not in Terraform — Terraform manages references but not contents):

**`prog-strength-developer/claude-credentials`** — JSON blob whose value is the literal contents of the owner's local `~/.claude/credentials.json` after running `claude login`. The worker writes this file to disk at boot. Claude Code's refresh-token logic handles renewal during a run.

When the OAuth refresh token eventually expires (months out, per Anthropic's current refresh-token policy), the owner re-runs `claude login` locally and re-seeds the secret. The runbook documents this as a periodic chore.

**`prog-strength-developer/github-app`** — JSON blob containing:

```json
{
  "app_id": 123456,
  "installation_id": 7890123,
  "private_key": "-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----\n"
}
```

The worker uses these to mint a short-lived installation token via the GitHub App JWT flow.

### GitHub App

A new GitHub App `Prog Strength Developer` registered on the Prog-Strength organization with:

- **Repository permissions**: Contents (read+write), Pull requests (read+write), Issues (read+write), Metadata (read), Workflows (write — required to push files under `.github/workflows/`).
- **Organization permissions**: none.
- **Installed on**: all current and future repositories in the Prog-Strength org. (Convenient for this SOW; can be narrowed later if useful.)
- **Webhook**: disabled. v1 has no use for App webhooks.

Tokens minted from the App's private key are scoped per installation and expire in 1 hour. The worker's bootstrap script re-mints a token if a run exceeds 50 minutes of wall-clock time since the last mint.

### GitHub Actions Workflow

`prog-strength-developer/.github/workflows/dispatch-sow.yml`:

```yaml
name: Dispatch SOW
on:
  workflow_dispatch:
    inputs:
      sow_path:
        description: "Path to the SOW within prog-strength-docs (e.g. sows/intent-driven-context-enrichment.md)"
        required: true
        type: string

permissions:
  id-token: write     # for OIDC federation to AWS
  contents: read

jobs:
  dispatch:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/prog-strength-developer-github-actions-role
          aws-region: us-east-1
      - name: Preflight — no in-flight worker
        run: ./scripts/preflight.sh
      - uses: hashicorp/setup-terraform@v3
      - name: Terraform apply
        working-directory: terraform
        run: |
          terraform init
          terraform apply -auto-approve \
            -var "sow_path=${{ inputs.sow_path }}"
```

The pre-flight script (`scripts/preflight.sh`) calls `aws ec2 describe-instances --filters "Name=tag:Name,Values=prog-strength-developer-worker" "Name=instance-state-name,Values=pending,running"` and exits non-zero if any are returned. Clean failure mode for concurrent-dispatch attempts.

### EC2 Worker Bootstrap

The launch template uses Amazon Linux 2023 (latest AMI looked up via SSM parameter) on a `t3.large` instance (2 vCPU, 8 GB RAM). Instance type chosen based on the worker's job: it spends most of its time waiting on Anthropic API calls, but the build/test phases of Go, Python, and Node projects benefit from the headroom over `t3.medium`. Spot pricing is not used for v1 — a worker interrupted mid-run would leave dangling branches and require manual cleanup. Worth ~$0.06/run to skip that.

The userdata is rendered by Terraform from `bootstrap/userdata.sh.tpl` with `sow_path` substituted in. Top-level structure:

```bash
#!/bin/bash
set -euo pipefail

# Install dependencies
dnf install -y git gcc make jq
# Install Node, Go, uv via their respective bootstrap scripts
# Install AWS CLI v2
# Install CloudWatch agent
# Install Claude Code CLI

# Configure CloudWatch agent to stream /var/log/prog-strength-developer/*.log
# and stdout/stderr of subsequent commands.

# Fetch secrets
aws secretsmanager get-secret-value \
  --secret-id prog-strength-developer/claude-credentials \
  --query SecretString --output text > ~/.claude/credentials.json
chmod 600 ~/.claude/credentials.json

# Mint a GitHub App installation token
GH_APP=$(aws secretsmanager get-secret-value \
  --secret-id prog-strength-developer/github-app \
  --query SecretString --output text)
./mint-gh-token.sh "$GH_APP" > /tmp/gh-token
gh auth login --with-token < /tmp/gh-token

# Clone prog-strength-docs and read the SOW frontmatter
gh repo clone Prog-Strength/prog-strength-docs /workspace/prog-strength-docs
SOW_PATH="${sow_path}"   # injected by Terraform templating
REPOS=$(yq '.repos[]' /workspace/prog-strength-docs/$SOW_PATH)

# Clone each affected repo
for repo in $REPOS; do
  gh repo clone Prog-Strength/$repo /workspace/$repo
done

# Render the Claude prompt
sed "s|__SOW_PATH__|$SOW_PATH|g" /opt/prog-strength-developer/prompt.md.tpl \
  > /workspace/prompt.md

# Set up the 6-hour backstop
systemd-run --on-active=6h --unit=worker-timeout \
  /bin/bash -c "aws ec2 terminate-instances --instance-ids $(ec2-metadata -i | cut -d ' ' -f 2)"

# Run Claude
cd /workspace
claude --print --dangerously-skip-permissions < prompt.md \
  > /var/log/prog-strength-developer/claude.log 2>&1 || true

# Final flush and self-terminate
sleep 10  # let CloudWatch agent push remaining bytes
aws ec2 terminate-instances --instance-ids $(ec2-metadata -i | cut -d ' ' -f 2)
```

The `|| true` after the `claude` call ensures the script proceeds to self-termination even on a non-zero exit. Failures are not retried automatically — the owner reviews CloudWatch and decides what to do.

### Claude Code Prompt Template

`bootstrap/prompt.md.tpl` is rendered with the SOW path substituted in. The template:

```
You are running unattended on an EC2 instance. There is no human to ask
questions of. Make reasonable judgment calls and proceed — if you
genuinely cannot determine the right path forward, exit with a clear
message instead of fabricating an answer.

The SOW you are implementing is at:
    /workspace/prog-strength-docs/__SOW_PATH__

The affected repositories are cloned at /workspace/<repo-name>. Each is
checked out on `main`. You are responsible for creating a feature
branch in each one you modify, named after the SOW slug (the basename
of the SOW path without extension), prefixed with `feat/`.

Workflow:

1. Read the SOW. If a plan already exists at
   /workspace/prog-strength-docs/plans/*-<sow-slug>.md, use it. If not,
   produce one by invoking the `superpowers:writing-plans` skill.
2. Execute the plan via the `superpowers:subagent-driven-development`
   skill. Follow it exactly. Do not skip review stages.
3. After all tasks complete, open a pull request in each repository
   you modified. Push the feature branch first. Use `gh pr create`.
   PR titles and bodies should follow the format you see in recent
   commits and merged PRs in those repos.
4. Exit. The system will terminate the instance.

You are running with `--dangerously-skip-permissions`. Do NOT use this
freedom to take destructive actions (force-pushing, deleting branches,
modifying unrelated repos). Every change you make should be reviewable
as a normal pull request diff.

Do NOT attempt to merge any PR you open. The owner is the reviewer.
```

The prompt is short on purpose. The Prog Strength workflow knowledge lives in the superpowers skills that Claude Code already knows; restating that knowledge inline would couple the prompt to skill-version drift.

### Self-Termination and Abort Criteria

Three independent paths terminate the instance:

1. **Happy path** — userdata reaches the final `aws ec2 terminate-instances` call after Claude exits.
2. **Backstop timer** — the systemd-run unit fires after 6h and terminates regardless of whether userdata is still executing.
3. **Manual** — the owner explicitly terminates via the AWS console or `aws ec2 terminate-instances` if a run is visibly hung before the backstop fires.

There is no path that leaves an instance running indefinitely. The terminate call is idempotent (calling it twice is harmless), so the backstop firing during a successful userdata run is just a no-op.

### Observability

A CloudWatch log group `/aws/ec2/prog-strength-developer` with a 30-day retention. Each instance creates a log stream named after its instance ID. The CloudWatch agent on the worker is configured to stream:

- `/var/log/cloud-init-output.log` — userdata output, useful for debugging boot failures.
- `/var/log/prog-strength-developer/*.log` — Claude's stdout/stderr and the userdata's own structured progress markers.

Per-stream log volume is bounded by the 6h max runtime. At Claude Code's typical token rate, a long run might produce ~50 MB of logs — well under any reasonable budget.

PRs in the affected repositories are the primary success signal. A worker that completes successfully will leave PRs visible in the Prog-Strength org's PR list within the typical run window. A worker that fails will leave the log group as the only artifact.

The owner's debug procedure for a failed run, in order of cheapness:

1. Check the affected repos' PR lists. Sometimes the worker succeeded for some repos and failed for others; the partial state is informative.
2. Open the CloudWatch log group for the worker's instance ID. Skim the userdata progress markers + the last 200 lines of Claude's stdout.
3. If the worker is still alive (rare — usually the backstop will have fired), open an SSM Session Manager session to inspect `/workspace` and `~/.claude/`.

### Cost Model

Per-run, assuming a 2-hour SOW:

- EC2 (t3.large @ ~$0.083/hr × 2): **~$0.17**
- Outbound data transfer (Claude API, GitHub clones, package downloads, ~1 GB at $0.09/GB): **~$0.10**
- CloudWatch Logs ingestion (~50 MB at $0.50/GB): **~$0.03**

Total per run: **~$0.30**.

Always-on costs:

- Secrets Manager (2 secrets at $0.40/each/month): **$0.80/month**
- CloudWatch Logs storage (30-day retention of ~500 MB total): **<$0.05/month**
- S3 state bucket + DynamoDB lock table: **<$0.10/month**

Always-on total: **~$1/month**.

A budget alarm is configured at $50/month — well above expected usage but a tripwire for anything anomalous (a run that fails to terminate, runaway log volume, etc.).

### Initial Bootstrap Runbook

The autonomous developer cannot build itself the first time. The runbook in `prog-strength-developer/docs/setup.md` covers the one-time manual setup, executed by the owner in the listed order:

1. Create the `prog-strength-developer` GitHub repository under the `Prog-Strength` org. Empty.
2. Create the `Prog Strength Developer` GitHub App (UI flow on github.com/settings/apps/new). Generate a private key. Install the App on all org repositories.
3. In AWS, create the Terraform state bucket and the DynamoDB lock table (one-time, hand-crafted, not via Terraform — chicken-and-egg).
4. Push the `prog-strength-developer` repository contents (Terraform, workflow, scripts) to GitHub.
5. Configure the GitHub Actions OIDC federation trust between AWS IAM and the GitHub Actions OIDC provider (one-time IAM step).
6. Run `terraform apply` from the owner's laptop (not yet from the workflow) to create the persistent resources: VPC, IAM roles, CloudWatch log group, Secrets Manager secret slots.
7. Seed the Secrets Manager entries:
   - Run `claude login` locally if not already done. Copy `~/.claude/credentials.json` into the `prog-strength-developer/claude-credentials` secret.
   - Paste the GitHub App's `{app_id, installation_id, private_key}` JSON into `prog-strength-developer/github-app`.
8. Dispatch a test run against a trivial SOW (e.g. a SOW that just updates a typo in `prog-strength-docs/README.md`) to validate the end-to-end flow.

The runbook is referenced by the README; future SOWs that touch this system should keep the runbook in sync.

### Testing the System

There is no traditional test suite for this SOW. The system is integration-level by nature. v1 validation is empirical:

1. **Trivial-SOW smoke test** — see bootstrap step 8 above. A SOW that asks Claude to change one line in a README, on a single repo. End-to-end success means the worker boots, fetches credentials, clones, runs Claude, opens a PR, and self-terminates. If this fails, the system is broken; if it passes, the moving pieces are connected.
2. **Multi-repo SOW** — the next test is a small SOW that touches 2+ repos. Validates that the frontmatter parsing, multi-repo clone, and per-repo PR creation all work.
3. **Abort path** — manually trigger a worker on a SOW the worker will obviously struggle with (e.g. a SOW with a deliberately ambiguous spec) and verify the 6h backstop fires and the instance terminates.
4. **Concurrency pre-flight** — dispatch a second workflow run while the first is in flight; verify the second fails with a clear message and no second instance is created.

A `terraform plan` is run on every workflow execution before `apply`, which catches IaC drift before resources are mutated. There is no separate CI step beyond that.

## Open Questions

1. **Should the worker post a comment on the SOW after PRs are opened?** The natural place would be the GitHub commit that introduced the SOW, since SOWs aren't always PR'd individually. A small "PRs opened by autonomous developer:" comment with links would aid discoverability without adding notification infra. Recommendation: defer to v1.1 — the owner knows what they dispatched and can find the PRs themselves at the start.

2. **What's the right model selection inside the Claude prompt?** The prompt currently leaves this to Claude's judgment (which the subagent-driven-development skill handles per-task). It may be worth pinning a model selection in the prompt template (e.g. "use sonnet by default, opus for architecture decisions") to control costs. Recommendation: ship without pinning and observe the cost/quality tradeoff before adding a constraint.

3. **How should the worker handle SOWs that already have an in-progress PR in some of the affected repos?** The current design always creates a fresh feature branch and opens a new PR. If the SOW was previously partially worked on (manually or by a prior worker run), this produces duplicate PRs. Recommendation: ship with the naïve behavior; the prevention path is "don't dispatch SOWs that have in-flight PRs," which is the owner's responsibility for v1.

4. **Should the SOW frontmatter include a `base_branch:` field for repos that don't ship from `main`?** Currently the worker assumes every affected repo's base branch is `main`. If a SOW needs to land on a release branch (e.g. mobile's `release/X.Y` branches), the SOW will need to communicate that. Recommendation: ship without the field and add it in a follow-up when a real SOW needs it; YAGNI until then.

5. **Should there be a "dry run" mode that skips Claude entirely?** A `dry_run: true` workflow input that runs the worker through bootstrap (fetch creds, clone repos, parse frontmatter) and then exits without invoking Claude would be useful for validating the system after Terraform changes. Recommendation: defer; the trivial-SOW smoke test serves the same purpose with slightly more friction.
