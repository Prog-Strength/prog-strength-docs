# Centralize Container Logs in CloudWatch

**Status**: Implemented (pre-deploy) · **Last updated**: 2026-05-31

## Introduction

Today, debugging anything that happens on the Prog Strength EC2 host requires SSH-ing in and running `docker compose logs <service>`. There's no way to search across services, no history beyond what the host's `json-file` driver happens to be retaining, and no view at all from a laptop without network access to the box. When a user reports "I tried to log a workout at 2pm and nothing happened," the only way to investigate is a terminal session that may not survive long enough to see the symptom recur.

The metrics side already lives in Grafana (Prometheus scraping `/metrics` on each service), so request rates, p95 latency, and error counts are visible without touching the host. Logs are the missing peer: when a metric spikes, you want to read the matching log line, not just see a number go up.

This SOW ships a focused, cheap fix — point each container's stdout/stderr at AWS CloudWatch Logs via Docker's built-in `awslogs` driver, pre-create the log groups in Terraform with a sane retention, and call it done. No agent sidecar, no log shipping daemon, no application code changes. After this lands, `aws logs tail /prog-strength/api --follow` from a laptop replaces an SSH session, and Logs Insights queries replace `grep` over a scratch file.

## Proposed Solution

Each compose service (`api`, `agent`, `mcp`) gets a `logging:` stanza pointing at the Docker `awslogs` driver. Three log groups are created in Terraform — `/prog-strength/api`, `/prog-strength/agent`, `/prog-strength/mcp` — with 30-day retention. The existing EC2 instance role gets a CloudWatch Logs write policy scoped to just those three groups (least-privilege; the role still can't read or write anywhere else).

Docker's `awslogs` driver runs inside the Docker daemon process — no sidecar container, no extra memory footprint per service, no separate log file on disk to manage. Each container's stdout/stderr stream is buffered and POSTed to CloudWatch's `PutLogEvents` API directly. The driver auto-creates a log stream per container (using the container ID as the stream name) and rotates streams on restart.

Logs Insights — CloudWatch's built-in query interface — replaces the current "grep on the host" debugging flow. Five GB of scanned data per month is free indefinitely, which more than covers the kind of "show me everything around the time the user said it broke" investigations this is meant to support.

Monitoring stack containers (Grafana, Prometheus) deliberately stay on the default `json-file` driver. Their logs are noisy, low-signal, and we already have the dashboards themselves as the primary inspection surface — shipping them to CloudWatch would double our ingestion bill for output nobody reads.

## Goals and Non-Goals

### Goals

- Each service's stdout/stderr lands in CloudWatch within seconds of being written, queryable from any machine with `aws logs` configured.
- Log groups are pre-created via Terraform with a 30-day retention policy so a forgotten `aws_cloudwatch_log_group` resource doesn't accumulate years of storage cost.
- IAM is least-privilege: the EC2 instance role can write `PutLogEvents` only against the three Prog Strength log groups, can't create groups at runtime, can't read or write anywhere else in CloudWatch.
- Total monthly cost stays under $1 under normal traffic. A CloudWatch billing alarm at $5 fires if ingestion ever crosses out of cents-per-month territory so we hear about it before it becomes a $50 surprise.
- The `awslogs` driver runs alongside the existing `json-file` driver behavior on monitoring containers — no all-or-nothing flip.
- Existing telemetry tables in the API (`telemetry_events`, etc.) are unaffected. They serve a different purpose (per-event structured analytics, retained ~indefinitely) and stay where they are.

### Non-Goals

- **Structured (JSON) logging.** The API uses Go's stdlib `log` and the agent uses Python's `logging` with default formatting — both emit plain text. Migrating to JSON would make Logs Insights queries much sharper (`fields @timestamp, level, msg`) but it's a meaningful per-service code change. File as a follow-up.
- **Log-based alerting.** No metric filters firing PagerDuty on "ERROR" frequency for v1. We already have Prometheus-side error-rate alerting via Grafana; doubling that on the logs side is duplicative until we have evidence one channel misses what the other catches.
- **Distributed tracing.** OpenTelemetry-shaped request tracing is a different concern; it doesn't ride on the log pipeline. If we add it later it'll be a separate SOW.
- **Shipping monitoring-stack logs.** Grafana + Prometheus internal logs stay on `json-file`. Low signal, high volume, and the dashboards themselves are the inspection surface.
- **Multi-environment log groups.** Prod is the only environment that exists today. If a staging environment lands later it'll get its own log group prefix (`/prog-strength-staging/...`) — a one-line variable change in the new Terraform module, not a refactor.
- **Backfilling existing logs.** Whatever's currently in the host's `json-file` driver is fine to leave behind. The flip is forward-only.

## Implementation Details

### Compose changes (`prog-strength-infra/compose/<service>/docker-compose.yml`)

Each of `api`, `agent`, `mcp` gets a `logging:` stanza. Shape:

```yaml
services:
  agent:
    # ...
    logging:
      driver: awslogs
      options:
        awslogs-region: ${AWS_REGION:-us-east-1}
        awslogs-group: /prog-strength/agent
        # `awslogs-stream` left unset on purpose: Docker auto-generates
        # a per-container stream name (container ID), which means each
        # restart gets its own stream and a clean read of "this run."
        # `awslogs-create-group` deliberately NOT set — the group is
        # pre-created via Terraform so the runtime IAM policy can drop
        # the `logs:CreateLogGroup` permission.
        awslogs-create-stream: "true"
```

`AWS_REGION` is forwarded from `.env` like the other environment-derived values. The deploy workflow already writes that file from GitHub secrets on each release; this just consumes an existing var.

### Terraform: new `modules/logging`

Following the existing domain-module pattern (`modules/backup`, `modules/ecr`), a new `modules/logging` owns:

- Three `aws_cloudwatch_log_group` resources (`api`, `agent`, `mcp`) with `retention_in_days = 30`.
- One `aws_iam_policy` document scoping `logs:CreateLogStream` and `logs:PutLogEvents` to the three groups' ARNs and their `:log-stream:*` children.
- One `aws_iam_role_policy_attachment` attaching the policy to the existing instance role (passed in via `instance_role_name`, matching the convention `modules/backup` and `modules/ecr` already use).
- One `aws_cloudwatch_metric_alarm` watching the `EstimatedCharges` metric on the CloudWatch namespace, threshold $5, period monthly. SNS-less for v1 — the alarm just shows red in the console. Wiring a notification destination is a follow-up that can hang off the same alarm resource.

Module call site in `main.tf`:

```hcl
module "logging" {
  source              = "./modules/logging"
  name_prefix         = local.name_prefix
  retention_days      = 30
  instance_role_name  = module.compute.instance_role_name
  monthly_budget_usd  = 5
}
```

The module is fully self-contained: removing the module call from `main.tf` deletes the groups, drops the policy, and the EC2 host stops shipping logs. No coupling outside `modules/compute`'s `instance_role_name` output.

### Sensitive-data audit (before flipping the switch)

Both the API and the agent currently log a few things that *could* be sensitive once they leave the host:

- API: request paths include resource IDs (workout IDs, pantry IDs). Not PII per se, but high-cardinality.
- Agent: the SSE chat handler logs the model name and message length per request — not the message body itself, intentionally. Worth confirming this hasn't drifted.
- Auth: the auth handler logs `email=...` on successful sign-in. Email is PII for some regulatory frames. Logged level is INFO, so it will ship.

Pre-flip pass: grep for `log.Printf`, `log.info`, `logger.info` in api/agent code, list everything that includes a user-derived value, and either redact (hash the email, drop the path body parameter) or downgrade to DEBUG. CloudWatch logs aren't public — they're behind IAM — but the principle of "don't ship what we don't need to keep" applies anyway.

### Cost model

CloudWatch Logs pricing (US regions, current as of 2026):

- Ingestion: **$0.50 / GB**
- Storage: **$0.03 / GB / month**
- Logs Insights queries: **$0.005 / GB scanned** (5 GB scanned/month free, always)
- AWS Free Tier: **5 GB ingestion/month free for the first 12 months** of a new account.

Expected steady-state volume for three services with current traffic:

| Service | Estimated MB/day | Estimated MB/month |
|---|---|---|
| api | 10 | ~300 |
| agent | 30 | ~900 |
| mcp | 5 | ~150 |
| **Total** | **~45 MB/day** | **~1.35 GB/month** |

Ingestion cost: 1.35 GB × $0.50 = **$0.68/month**. Storage on the 30-day rolling window is ~1.35 GB × $0.03 ≈ **$0.04/month**. Free tier (if applicable) absorbs all of it for the first year. Worst case 10× volume estimate (chatty INFO logging): ~$7/month — still cheaper than a single t4g.nano.

### Operational read paths

After this lands, the new ways to investigate:

- **Tail a single service from a laptop**: `aws logs tail /prog-strength/agent --follow --since 5m`
- **Search across services for a request ID**: Logs Insights query `filter @message like /req-xxxxx/` against all three log groups.
- **Find every 5xx from the API in the last hour**: Logs Insights, `filter @message like /\] 5\d\d /`.

SSH is preserved as the fallback — Docker's `json-file` driver doesn't *replace* the `awslogs` driver, but we're not running both at once. If CloudWatch ingestion ever silently breaks (network blip, IAM regression), Docker will buffer in memory up to a few seconds and then drop. There's no on-disk log file fallback — that's an accepted tradeoff for not paying for a sidecar.

## Open Questions

1. **Retention: 7, 30, or 90 days?** 30 is the v1 lean — long enough for "what happened last week" investigations, short enough to keep storage cost trivially small. 7 would save almost no money (storage is already ~$0.04/month) at the cost of meaningfully worse postmortems. 90 is fine cost-wise but feels long for a side-project debugging surface where most useful inspection happens within a few days of an incident. Default: 30.
2. **Should the auth handler's `email=...` log line be redacted before flip?** Privacy lean says yes — emails are recoverable identifiers even when they live behind IAM. Cheapest fix: hash the email (sha256 first 8 chars) in the log line. Downside: now you can't visually grep "did jimmy@... do this" without round-tripping a hash. Tentative lean: hash, accept the lookup friction.
3. **Billing alarm: SNS-wired now, or just-the-alarm-record for v1?** SNS adds a topic resource, a subscription confirmation flow, and a destination (email? Slack?). Just having the alarm sit red in the console is enough for v1 since it'd only fire on an actual surprise; we can wire SNS when we hook other alarms up to it (which will happen eventually). Tentative lean: alarm-only.
4. **Ship monitoring-stack (Grafana, Prometheus) logs too?** They're noisy and low-signal, doubling ingestion volume for output we rarely read since we have the dashboards themselves. The argument for: when the monitoring stack itself misbehaves, having the logs centralized matters more. Tentative lean: skip for v1; revisit if we ever hit a Grafana issue we can't diagnose from the host.
5. **Migrate to JSON-structured logging at the same time, or as a follow-up?** JSON would make Logs Insights queries dramatically sharper (`fields level, msg, request_id` instead of text-grepping). But the API uses Go stdlib `log` and the agent uses Python `logging` with default formatting — both are real code changes across many call sites. Tentative lean: separate SOW after this one ships and we've felt the pain of plain-text grep in Insights.

## Deploy Notes

The pieces have an order. The compose files reference log groups that must exist before any container starts — flipping compose first means containers fail to start with "log group not found" errors from the awslogs driver.

1. **Apply Terraform first.** `terraform apply` from `prog-strength-infra/` creates `/prog-strength/api`, `/prog-strength/agent`, `/prog-strength/mcp` log groups, the IAM policy, and the billing alarm. Verify each group exists with `aws logs describe-log-groups --log-group-name-prefix /prog-strength`.

2. **Confirm `AWS_REGION` is set in the host's `.env`.** The compose files reference `${AWS_REGION:-us-east-1}`; if the EC2 host's `.env` doesn't already have `AWS_REGION`, the driver defaults to us-east-1 even though the rest of the stack lives in us-east-2. Either add `AWS_REGION=us-east-2` to the deploy workflow's secret-derived `.env`, or accept the (working but cross-region) default. The Terraform module creates groups in whatever region the provider points at — confirm regions match.

3. **Apply compose changes.** Push to `prog-strength-infra` main; the host's bootstrap pulls the new compose files on next deploy. `docker compose up -d --force-recreate api agent mcp` from the respective directories restarts each container with the new logging driver. The driver is per-container, so the live process has to restart to pick up the change.

4. **Verify ingestion.** Within ~10 seconds of restart, `aws logs tail /prog-strength/api --since 1m --follow` should show output. If a service's group doesn't appear, the most common cause is the IAM policy hasn't propagated yet (IAM is eventually consistent; usually < 30 seconds, occasionally a few minutes).

5. **Pre-flip code audit landed alongside.** A `prog-strength-agent` commit (`router: drop user input text from INFO log line`) removed the only log line that included raw user content. The audit found the SOW's worry about `email=...` in the API auth handler was unfounded — `internal/auth/` has zero `log.` calls.
