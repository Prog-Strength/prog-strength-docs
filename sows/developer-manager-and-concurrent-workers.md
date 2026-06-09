---
status: proposed
repos:
  - prog-strength-developer
  - prog-strength-docs
---

# Developer Manager and Concurrent Workers

**Status**: Proposed · **Last updated**: 2026-06-08

## Introduction

The autonomous developer ships SOWs end-to-end, but the v1 design left two friction points that now bite at every run. First, the platform is operationally invisible: once a worker is dispatched, the only ways to know what it is doing are tailing CloudWatch logs by instance ID or opening the EC2 console. There is no "is anything running right now?" surface, no "how long has this been going?", no shape to how the fleet is behaving across runs. Second, the single-instance concurrency gate has become the bottleneck — most weeks there are two or three independent SOWs ready to run, all of them stuck behind whichever one the dispatcher picked first. Running them serially turns a multi-hour day into a multi-day wait.

Both problems share a missing piece: there is no always-on component of the developer platform. Workers are ephemeral by design — good for cost, hostile to observability. Every monitoring approach considered so far (CloudWatch dashboards, ad-hoc Grafana exports, scraping the EC2 API on demand) compromises either the data model or the operator experience. The cleaner move is to introduce a small, permanent **developer manager** instance whose sole job is to host the operator surface for the platform: a Prometheus TSDB for ephemeral-worker time series and a Grafana dashboard reachable at a public HTTPS URL.

The manager is also the right place to relax the concurrency cap from. Once an operator can see how many workers are alive, what each is doing, and what each is costing, dispatching three in parallel stops feeling reckless. With a dashboard saying "3 workers, 14 minutes in, $0.08 spent" the cap stops earning its keep — it becomes the thing slowing the project down, not the thing protecting it.

This SOW stands up the manager (Prometheus + Grafana + Pushgateway + Caddy on a t3.small EC2 instance, sitting in the existing developer VPC), removes the single-instance gate from the dispatch workflow, restructures the worker so that the EC2 instance itself is no longer terraform-managed (so concurrent dispatches do not fight over state), instruments workers with a small metrics exporter and a final-state push-on-termination, and routes `developers.progstrength.fitness` to the manager's Grafana over HTTPS via a Caddy vhost on the manager itself. After this work ships, every dispatched worker shows up as a live tile in Grafana; every completed run shows up as a row in the run history panel; the operator can answer "what is the developer platform doing right now?" from a bookmark.

## Proposed Solution

A new permanent instance — `prog-strength-developer-manager` — runs in the existing developer VPC alongside the ephemeral workers. It is a `t4g.small` (Graviton, 2 GB RAM) — the same instance class the application backend uses to run Prometheus + Grafana, so the sizing is already proven against this exact workload. Arm64 AMI (Amazon Linux 2023). Elastic IP, a small EBS data volume for Prometheus TSDB and Grafana SQLite state, and a docker-compose stack of six services (seven with the stretch log-tail goal below):

- **Prometheus** scrapes worker metrics via `ec2_sd_config` (tag-based service discovery on `Name=prog-strength-developer-worker`). New workers automatically appear as targets the moment they boot; terminated workers automatically disappear. 15-day retention matches the application monitoring stack.
- **Pushgateway** receives a final summary push from each worker just before self-termination — duration, outcome, PRs opened, tool-call counters — so those facts survive the worker's death.
- **Grafana** serves the dashboards. Provisioning auto-creates the Prometheus datasource and registers dashboards from disk; no clickthrough needed.
- **Caddy** terminates TLS at `developers.progstrength.fitness`, reverse-proxying Grafana.
- **node_exporter** monitors the manager's host: CPU, memory, disk, network, filesystem.
- **cAdvisor** monitors *each container's* CPU, memory, and disk I/O on the manager. node_exporter answers "is the box healthy?"; cAdvisor answers "which container is using the box?" — both are needed to decide whether the manager is right-sized or which service is the headroom-eater if it isn't.

The manager sits in its own public subnet (separate from the worker subnet) with a security group that allows inbound 80/443 from the internet (Caddy) and inbound 9100/9101 *from the worker security group* (Prometheus → workers). Workers reciprocally allow outbound to the manager on 9091 (Pushgateway). DNS is an A record on `developers.progstrength.fitness` pointing at the manager's Elastic IP, added by hand in GoDaddy (which is where `progstrength.fitness` is managed). The EIP is allocated by Terraform and printed as a workflow output; the operator copies it once and pastes it into GoDaddy's DNS panel. It does not change on subsequent applies.

Workers gain two new pieces. First, a `node_exporter` container on port 9100 for host metrics (CPU, memory, disk, network — the standard Prometheus host kit). Second, a tiny Python `worker_exporter` on port 9101 that exposes platform-specific gauges and counters: current SOW path, current state (`booting` / `cloning` / `working` / `opening_prs` / `terminating`), uptime, Claude tool calls grouped by tool name (sourced from the same JSONL renderer pipeline that already feeds the `claude-pretty.log` CloudWatch stream — see `developer_logging_pipeline`), Claude message counts, PRs opened. Both exporters are started by the existing userdata script before Claude Code runs. Both are scraped over the worker's private IP from the manager.

When a worker is about to self-terminate, its userdata pushes a final summary to the manager's Pushgateway under a job named `developer_run` and a grouping key of `instance_id`. The push includes `developer_run_duration_seconds`, `developer_run_outcome` (`success` / `error` / `timeout`), `developer_run_prs_opened`, and a copy of the per-tool counters as a finalized snapshot. After the push, the worker calls `aws ec2 terminate-instances`. Pushgateway holds these series until they're scraped by Prometheus, so they survive into the TSDB and show up in the dashboard's "Run history" panel indefinitely within retention.

Two architectural changes ride along with this. **First, the worker EC2 instance moves out of Terraform.** The dispatch workflow currently runs `terraform apply` with `sow_path` baked in, which serializes every dispatch on the `terraform-apply-prod` state lock — which is exactly why concurrent dispatches are blocked at the IaC layer even before the preflight kicks in. After this work, persistent infrastructure (launch template, IAM, VPC, security groups, log group, secrets, *and the manager*) stays in Terraform. The worker instance itself becomes an `aws ec2 run-instances` call from the dispatch workflow, using the latest version of the persistent launch template with userdata rendered at workflow time. Concurrent dispatches no longer touch shared state.

**Second, the single-instance preflight is deleted.** `scripts/preflight.sh` goes away. The `concurrency` group on `dispatch-sow.yml` is removed. A soft fleet cap (default 10) is enforced in the workflow as a guardrail against runaway dispatches — it counts running workers and refuses to launch if the cap would be exceeded. The cap is a workflow-level constant the operator can raise in a one-line PR; it exists to make a stuck dispatch loop visible rather than to enforce a serial model.

Grafana access uses the same admin credentials as the application monitoring dashboard at `monitoring.progstrength.fitness`. Both `GRAFANA_ADMIN_USER` and `GRAFANA_ADMIN_PASSWORD` are mirrored from `prog-strength-api`'s GitHub repository secrets into `prog-strength-developer`'s. They're seeded into the manager's compose at deploy time. The operator's single muscle-memory password works for both URLs.

Two dashboards, side-by-side in Grafana's nav.

**Developer Platform** — the fleet view. Three sections:

- **Fleet overview** — active worker count (single stat), running workers (table with SOW, instance ID, uptime, state), workers started in the last 24h, completed runs in the last 24h grouped by outcome, average run duration over the last 7d, estimated 24h compute cost (instance-hours × on-demand t3.large price).
- **Per-worker drill-down** — Grafana variable on `instance_id` selects one worker; panels show CPU, memory used / available, network in/out, disk usage, uptime, current state, Claude tool calls per minute by tool name, Claude messages over time, PRs opened so far.
- **Run history** — last 30 completed runs as a table (SOW, started, duration, outcome, PRs opened), pulled from the Pushgateway-sourced final-state series.

**Manager Host Health** — the manager-itself view. Exists to answer "is the manager right-sized, and which container should I worry about if it isn't?" Four sections:

- **Headroom at a glance** — current memory used % (single stat, color-coded: green <70%, yellow 70–85%, red >85%), current CPU load average over 5m, free disk % on root volume and on `/var/lib/manager`, current container count vs expected. One screen, one verdict.
- **Resource trends** — host CPU % over time (idle / user / system / iowait stacked), memory used / available / cached over time, swap (expected zero), 1m/5m/15m load averages, network in/out bytes per second, network errors and drops.
- **Storage** — `/var/lib/manager` data-volume usage over time with the volume's total capacity drawn as a line (so the "when will I fill this up?" question is visible), root volume usage over time, Prometheus TSDB on-disk size growth rate, Loki chunk store size (stretch), disk IOPS read/write, EBS gp3 I/O credit balance.
- **Per-container breakdown** — table and time-series of CPU and memory per container (Prometheus, Grafana, Pushgateway, Caddy, cAdvisor, node_exporter, Loki if shipped), container restart counts. The "which container ate the box?" panel.

Dashboard JSON for both lives in `prog-strength-developer/monitoring/grafana/dashboards/`, alongside Prometheus config (`prog-strength-developer/monitoring/prometheus/prometheus.yml`) and Grafana provisioning. The manager's docker-compose lives at `prog-strength-developer/monitoring/docker-compose.yml`. Keeping the entire manager stack in `prog-strength-developer` rather than `prog-strength-infra` preserves the boundary established by the v1 SOW: `prog-strength-infra` holds application infrastructure, `prog-strength-developer` holds developer platform infrastructure. They share a Caddy *pattern*, not a Caddy *instance*.

A new GitHub Actions workflow on `prog-strength-developer` — `deploy-manager.yml` — runs on push to `main` whenever `monitoring/**` or `caddy/**` changes. It connects to the manager via SSM Session Manager (no SSH, no port 22), pulls the latest `prog-strength-developer` working tree, and runs `docker compose up -d --build` to apply the new config. This mirrors the application monitoring deploy pattern.

## Goals and Non-Goals

### Goals

- Provision a permanent `prog-strength-developer-manager` EC2 instance (`t4g.small` on Amazon Linux 2023 arm64, Elastic IP, 20 GB gp3 EBS data volume) in a new public subnet of the existing developer VPC, with a dedicated security group allowing inbound 80/443 from the internet and inbound 9100/9101 from the worker SG.
- Stand up a docker-compose stack on the manager containing Prometheus (v3.5.0+), Grafana (v12.2.1+), Pushgateway (v1.10.0+), Caddy (v2.8+), `node_exporter` (v1.9.1+), and `cAdvisor` (v0.49+). Match container versions to the application monitoring stack where they overlap.
- Configure Prometheus to discover workers via `ec2_sd_config` filtering on `Name=prog-strength-developer-worker`, with relabeling that promotes the worker's `SOW` tag and instance ID into metric labels. Scrape `:9100` (node_exporter) and `:9101` (worker_exporter) on each discovered target.
- Bake `node_exporter` and a small Python `worker_exporter` into the worker userdata. The worker_exporter exposes `developer_worker_info`, `developer_worker_state`, `developer_worker_uptime_seconds`, `developer_claude_tool_calls_total{tool}`, `developer_claude_messages_total{role}`, and `developer_prs_opened_total`. State and counter updates come from tailing the existing `~/.claude/projects/<slug>/<uuid>.jsonl` files.
- On worker termination (success, error, or 6-hour timeout), push a final summary to the manager's Pushgateway under job `developer_run` keyed by `instance_id`. Push fields: `developer_run_duration_seconds`, `developer_run_outcome`, `developer_run_prs_opened`, finalized per-tool counters.
- Provision two Grafana dashboards (JSON-defined, auto-loaded by Grafana provisioning): `Developer Platform` (fleet view, three sections as described above) and `Manager Host Health` (manager-itself view, four sections: headroom, resource trends, storage, per-container breakdown).
- Reuse `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASSWORD` from `prog-strength-api`'s GitHub repository secrets by mirroring them into `prog-strength-developer`. Document the one-time mirror in `prog-strength-developer/docs/setup.md`.
- Add a Caddy container on the manager terminating TLS at `developers.progstrength.fitness` and reverse-proxying Grafana. DNS is managed in GoDaddy (where `progstrength.fitness` lives), so the EIP is allocated by Terraform and exposed as a `manager_public_ip` output; the operator adds the A record by hand in GoDaddy's panel as a one-time setup step. The EIP is stable across applies.
- **(Stretch)** Ship a live Claude log tail into the Grafana dashboard via Loki + Promtail: add a Loki container on the manager, add a Promtail sidecar on each worker that tails `/var/log/prog-strength-developer/claude-pretty.log` (the existing JSONL-renderer output — see `developer_logging_pipeline`) with `instance_id` and `sow` labels, register Loki as a second Grafana datasource, and add a "Live Claude output" Logs panel to the per-worker drill-down section with Grafana's Live tail mode enabled. Worker SG allows outbound to the manager on Loki's port 3100; manager SG allows inbound 3100 from worker SG. Cuts to `aws ssm start-session` for live debugging only — full structured logs still go to CloudWatch as today.
- Move the worker EC2 instance out of Terraform: keep the launch template, IAM, VPC, security groups, log group, secrets, and manager in Terraform; switch the dispatch workflow to `aws ec2 run-instances --launch-template ...` with userdata rendered at workflow time.
- Delete `scripts/preflight.sh` and remove the single-instance check from `.github/workflows/dispatch-sow.yml`. Remove the `concurrency` group that gates concurrent dispatches.
- Add a soft fleet cap (default 10) in the dispatch workflow: count running workers via `aws ec2 describe-instances`, refuse to launch if the cap would be exceeded, log a clear message. The cap is a workflow-level constant the operator can raise in a one-line PR.
- Update worker security group to allow outbound to the manager on port 9091 (Pushgateway). Update worker IAM role to remove any assumption of being the only worker (e.g., a self-termination policy conditioned on `instance_id`-matching rather than `tag:Name`-matching, if the current policy is shaped that way).
- Add `deploy-manager.yml` GitHub Actions workflow that SSM-connects to the manager on push to `main` (when `monitoring/**` or `caddy/**` changes) and runs `docker compose up -d --build`.
- Update `prog-strength-developer/docs/README.md` with the new architecture (manager + concurrent workers), the `developers.progstrength.fitness` URL, and the dashboard's purpose. Update `docs/setup.md` with one-time bootstrap steps (manager EIP allocation, DNS record, GRAFANA_* secret mirror). Update `docs/troubleshooting.md` with "no workers showing in Grafana" and "Pushgateway target down" runbooks.

### Non-Goals

- **Alerting.** No Alertmanager, no Slack notifications when a worker fails or stalls. The dashboard is the surface; this matches the v1 monitoring SOW's stance and stays consistent. Easy follow-up once the platform's failure modes are better characterized.
- **Multi-instance manager (HA).** One manager instance, no failover. The cost of an hour of downtime on the developer dashboard is near zero — the workers themselves keep running and the existing CloudWatch logs remain the source of truth for any in-flight run.
- **Manager scaled with worker count.** A t3.small handles the v1 worker volume easily. Revisit only if active worker count routinely exceeds ~20.
- **Replacing CloudWatch logs.** Logs continue to flow to `/aws/ec2/prog-strength-developer/<instance-id>` as today. Grafana adds metrics; it does not absorb log search.
- **Worker-to-worker isolation hardening.** Concurrent workers run in the same VPC and same subnet today. No new lateral-isolation work — each worker still has its own IAM-scoped self-termination policy and clones its own `/workspace`. The trust boundary remains: the developer VPC is isolated from the application VPC; workers within it trust each other no more or less than they did before.
- **Removing the soft fleet cap.** A cap of 10 stays in place as a budget guardrail. The cap is the thing that catches a dispatch-loop bug; without it, a misbehaving workflow could quietly burn dollars. Raising it is one PR.
- **Per-SOW cost attribution.** The cost panel sums fleet-wide compute time. Charging back to individual SOWs would need a real Cost Allocation Tags pipeline; not worth it at single-operator scale.
- **Authentication beyond Grafana's built-in admin login.** Same posture as `monitoring.progstrength.fitness`: a non-default admin username plus a long random password gating a single login form on a single-user beta. SSO is a non-goal until multi-user access lands.
- **Migrating the application monitoring stack to use the manager.** Two separate Grafana instances, each on its own host, federated only by sharing admin credentials. Combining them would re-introduce VPC peering, which is the thing v1 explicitly avoided.
- **Persisting Prometheus TSDB or Grafana SQLite to S3.** Both volumes are local EBS. If the manager is destroyed, dashboards are lost but rebuildable from git; in-flight metrics are lost but every worker re-publishes on its next push or scrape. The data is replaceable.
- **Reworking the SOW frontmatter or dispatch path UX.** The `workflow_dispatch` form still takes a `sow_path` input. No queue, no auto-discovery, no Slack-bot dispatch — same surface as v1, just with the gate lifted.

## Implementation Details

### Manager Terraform

A new file `prog-strength-developer/terraform/manager.tf` provisions:

- `aws_subnet.manager_public` — a second public subnet in the developer VPC, distinct CIDR from the worker subnet. Sharing a route table with the worker subnet is fine; the security group is the trust boundary.
- `aws_eip.manager` — Elastic IP, associated to the manager ENI. Stable target for the DNS A record.
- `aws_security_group.manager` — inbound 80/443 from 0.0.0.0/0 (Caddy), inbound 9090 from manager SG (Prometheus self-scrape — actually self-loopback, but explicit), inbound 22 *closed* (SSM only). All outbound open.
- Update `aws_security_group.worker` — allow inbound 9100/9101 from `manager` SG. Allow outbound to `manager` SG on 9091.
- `aws_iam_role.manager` — instance profile with `AmazonSSMManagedInstanceCore` (for SSM access) and a minimal CloudWatch logs policy for the manager's own log group. Does *not* get EC2 describe rights (the workflow runs those, not the manager).
- `aws_instance.manager` — `t4g.small`, Amazon Linux 2023 arm64 (`/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64`), IMDSv2 only. Userdata installs docker + docker-compose-plugin, mounts the data EBS at `/var/lib/manager`, clones `prog-strength-developer`, and runs `docker compose up -d`. Subsequent updates come through `deploy-manager.yml`.
- `aws_ebs_volume.manager_data` + `aws_volume_attachment.manager_data` — 20 GB gp3, attached as `/dev/sdf`. Mounts to `/var/lib/manager`, holds Prometheus TSDB, Grafana SQLite, and (stretch) Loki's local chunk store.
- **DNS lives in GoDaddy, not Terraform.** No `aws_route53_record`. Terraform emits `manager_public_ip` from `aws_eip.manager.public_ip` as a workflow output; the operator copies it into GoDaddy's DNS panel as a one-time `A developers → <ip>` record. EIP stability keeps this a one-time step.

### Manager docker-compose

`prog-strength-developer/monitoring/docker-compose.yml`. Services:

- `prometheus` — `prom/prometheus:v3.5.0`. Mounts `prometheus.yml` from the working tree, persistent volume on `/prometheus` for TSDB. 15-day retention.
- `pushgateway` — `prom/pushgateway:v1.10.0`. Persistent volume on `/data` to survive restarts. Prometheus scrapes it as a target.
- `grafana` — `grafana/grafana:12.2.1`. Same env-var pattern as the application monitoring compose (`GF_SECURITY_ADMIN_USER`, `GF_SECURITY_ADMIN_PASSWORD` from `${GRAFANA_ADMIN_USER}` / `${GRAFANA_ADMIN_PASSWORD}` injected by the deploy workflow). Provisioning volume mounts the dashboard JSON and datasource config from the working tree. SQLite state in a persistent volume.
- `caddy` — `caddy:2.8`. Mounts `caddy/Caddyfile` from the working tree. Persistent volumes for `/data` (cert storage) and `/config`. Ports 80/443 published.
- `node_exporter` — `prom/node-exporter:v1.9.1`, same configuration as the application monitoring stack. Provides host CPU, memory, disk, network, and filesystem metrics. Mounts `/`, `/proc`, `/sys` read-only and bind-mounts the data volume so `node_filesystem_*` series include `/var/lib/manager`.
- `cadvisor` — `gcr.io/cadvisor/cadvisor:v0.49.1` (arm64 image). Mounts the docker socket and `/var/lib/docker`, `/sys`, `/` read-only. Exposes per-container CPU, memory, network, and filesystem metrics on port 8080. Prometheus scrapes it as a separate job.

Compose network is internal-only by default; only Caddy publishes ports.

### Prometheus scrape config

`prog-strength-developer/monitoring/prometheus/prometheus.yml`. Two scrape jobs of note:

- `developer_worker` — `ec2_sd_config` filtering on `tag:Name=prog-strength-developer-worker` and `instance-state-name=running`. Two `relabel_configs`: one to construct the scrape target from `__meta_ec2_private_ip` + port (9100 or 9101 — duplicate the job, or use port-templating), one to promote `__meta_ec2_tag_SOW` → `sow` label and `__meta_ec2_instance_id` → `instance_id` label.
- `pushgateway` — static target on `pushgateway:9091`. `honor_labels: true` so per-instance push labels survive.

Plus three self-scrapes: `manager_node` on `node_exporter:9100` (host metrics for the Manager Host Health dashboard), `manager_containers` on `cadvisor:8080` (per-container metrics for the same dashboard's per-container panels), and `prometheus` self-scrape. Mirrors the application monitoring pattern, extended with cAdvisor.

### Worker exporter

A small Python script `prog-strength-developer/bootstrap/worker_exporter.py`, started by userdata under systemd before Claude Code runs. Uses `prometheus_client`. Exposes:

- `developer_worker_info{sow,instance_id,started_at}` — gauge always = 1, carries identifying labels.
- `developer_worker_state{state}` — gauge = 1 for the active state, 0 for others. Updated by writing a state file (`/var/run/developer-worker/state`) at each lifecycle transition in the userdata script.
- `developer_worker_uptime_seconds` — gauge, updated each scrape.
- `developer_claude_tool_calls_total{tool}` — counter, incremented by tailing `~/.claude/projects/*/*.jsonl` and pattern-matching `tool_use` events.
- `developer_claude_messages_total{role}` — counter, same JSONL tail.
- `developer_prs_opened_total` — counter, incremented by the userdata script after each successful `gh pr create`.

The JSONL tailer runs in the same process as the exporter for simplicity. The existing `claude-pretty.log` renderer sidecar is unaffected — same data, different consumer.

### Push-on-termination

Userdata adds a trap (`trap finalize EXIT`, on the main work block) that runs before `aws ec2 terminate-instances`. `finalize`:

1. Reads `developer_worker_state` final value and computes duration from `STARTED_AT`.
2. Pushes the final summary to the manager's Pushgateway via `curl -X POST` to `http://manager:9091/metrics/job/developer_run/instance/$INSTANCE_ID` with the metrics in text exposition format. Labels include `sow` and `outcome` (derived from the exit code).
3. Sleeps 2 seconds (so Prometheus, scraping Pushgateway on its 15s interval, has a chance to pick up the push if a scrape just happened — strictly belt-and-braces; Pushgateway persists the series until explicitly deleted).
4. Returns; the existing termination call runs.

The manager's hostname inside the worker resolves via DNS — set as a Terraform output, baked into userdata as the manager's private IP (private IP is stable for the manager's lifetime; if the manager is replaced, workers in flight at that moment fail their final push gracefully and the next worker picks up the new IP).

### Dispatch workflow changes

`.github/workflows/dispatch-sow.yml`:

- Drop the `preflight` job.
- Drop the `concurrency` group so concurrent dispatches don't queue.
- Replace `terraform apply` with a render-and-run step: render `bootstrap/userdata.sh.tpl` with the workflow inputs, base64-gzip it, call `aws ec2 run-instances --launch-template LaunchTemplateName=prog-strength-developer-worker-... --user-data <encoded>` (the launch template carries the IAM role, security group, subnet, instance type, AMI lookup).
- Insert a fleet-cap check step before run-instances: count running workers, fail with a clear error if at or above the cap.

`terraform/ec2.tf`:

- Remove `aws_instance.worker` and the `count = var.sow_path != "" ? 1 : 0` machinery.
- Keep `aws_launch_template.worker` unchanged in structure, but stop rendering `var.sow_path` into its userdata. Instead, the launch template carries the *base* userdata (`bootstrap/userdata.sh.tpl` with sow_path as an empty placeholder), and the workflow's `--user-data` flag overrides at run-instances time with the SOW-specific render.
- Remove `var.sow_path` from `variables.tf`; it's no longer a Terraform input.

`apply.yml` / `plan.yml` concurrency group remains on `terraform-apply-prod` — only `dispatch-sow.yml` opts out.

### Stretch goal: live Claude log tail via Loki + Promtail

The existing JSONL renderer sidecar already writes a human-readable Claude transcript to `/var/log/prog-strength-developer/claude-pretty.log` on each worker (see `developer_logging_pipeline`). That file is the right source for a live tail — it's already structured, already rendered, and the operator is already used to its format from the `claude` CloudWatch stream. Loki + Promtail wire it into Grafana with minimal new code.

On the **manager**:

- Add a `loki` service to `docker-compose.yml` using `grafana/loki:3.2` with a local filesystem chunk store at `/var/lib/manager/loki`. Single-binary mode (no microservices split — that's overkill at single-host scale). 7-day retention; the data is debugging-grade, not audit-grade.
- Manager security group adds inbound 3100 from the worker SG.
- Grafana provisioning adds a second datasource `Loki` pointing at `http://loki:3100`.

On each **worker**:

- Userdata installs a `promtail` (or `grafana-alloy` — see below) systemd unit ahead of Claude Code.
- Promtail config tails `/var/log/prog-strength-developer/claude-pretty.log` and labels every line with `instance_id`, `sow`, and `job=claude`. It ships to `http://<manager_private_ip>:3100/loki/api/v1/push`.
- Worker SG adds outbound 3100 to the manager SG.

In the **Grafana dashboard**:

- Add a "Live Claude output" Logs panel to the per-worker drill-down section. Query: `{instance_id="$instance_id", job="claude"}`. Set the panel to Live tail mode, line wrap on, oldest-first off.
- The same panel works at the fleet level by removing the `instance_id` filter — useful when the operator wants to see *anything happening across the fleet* at a glance.

**Promtail vs Grafana Alloy.** Promtail is in maintenance mode; Alloy is the official successor and bundles Prometheus + Loki + OTel collectors into one binary. For a single tail file the operational difference is nil and Promtail is meaningfully smaller. Defaulting to Promtail; flagging Alloy as the future-proof swap if Loki's stretch lands and a follow-up wants to consolidate.

**Why this is gated as stretch.** Loki adds ~150–300 MB resident on the manager. `t4g.small` has 2 GB; Prometheus + Grafana + Pushgateway + Caddy + node_exporter sit comfortably in well under 1 GB combined, so there is headroom — but it shrinks the margin to where a future dashboard expansion or a heavy ad-hoc query could press it. If memory pressure shows up during implementation, the right move is to ship the v1 of the platform without Loki and bump the manager to `t4g.medium` in a separate PR before adding it.

### Caddyfile on the manager

`prog-strength-developer/caddy/Caddyfile`:

```
developers.progstrength.fitness {
    @internal path /metrics
    respond @internal 404

    reverse_proxy grafana:3000
}
```

Same `(block_internal)` pattern as the application Caddyfile, simplified — only `/metrics` matters here (no `/internal/*` routes on this surface).

### Worker IAM tweak

Current self-termination policy condition (per the v1 SOW): "tag-conditioned IAM policy preventing the worker from terminating any instance other than itself." If the current condition matches by tag `Name=prog-strength-developer-worker` (which now matches *all* concurrent workers), it must tighten to match by `aws:ec2:SourceInstanceArn` or by an `aws:ResourceTag` keyed to a unique per-worker tag. Add a `WorkerInstanceId=$INSTANCE_ID` tag at launch via the `aws ec2 run-instances --tag-specifications ...`, and condition the IAM policy on `aws:ResourceTag/WorkerInstanceId` equalling the policy variable for the assuming role. Verify in IAM Policy Simulator that worker A cannot terminate worker B before merging.

### Operator setup steps

One-time bootstrap, documented in `docs/setup.md`:

1. Mirror `GRAFANA_ADMIN_USER` and `GRAFANA_ADMIN_PASSWORD` from `prog-strength-api`'s repository secrets to `prog-strength-developer`'s.
2. Merge the SOW's PR into `prog-strength-developer/main`. `apply.yml` provisions the manager, EIP, and all SG changes. Copy the `manager_public_ip` output from the Terraform run.
3. In GoDaddy's DNS panel for `progstrength.fitness`, add an `A` record on the `developers` subdomain pointing at the EIP from step 2. TTL 600 is fine; the IP does not change on subsequent applies.
4. After the DNS record propagates (usually under a minute), SSM into the manager (`aws ssm start-session --target $MANAGER_INSTANCE_ID`) and verify `docker compose ps` shows all services healthy. Verify Caddy got a Let's Encrypt cert with `docker compose logs caddy`.
5. Open `https://developers.progstrength.fitness` and log in with the Grafana admin credentials. Dashboard `Developer Platform` should be present and empty.
6. Dispatch a test SOW (or wait for a real one). Worker appears in the Fleet overview within ~30 seconds.

## Risks and Open Questions

- **EC2 SD scrape latency on short runs.** Prometheus' EC2 SD refresh is 60s by default; a worker that runs for two minutes might not appear on the dashboard before it pushes its final state. Mitigation: tighten `refresh_interval` to 15s on the `developer_worker` job. Trade-off: more EC2 describe calls (still well under any rate limit).
- **Pushgateway as a Prometheus anti-pattern.** Pushgateway is documented as appropriate for "service-level batch jobs" — which is exactly what a worker run is — but it does sit somewhat outside the pull model. The alternative (a small HTTP receiver on the manager that the worker POSTs to, writing the final state to a local SQLite that a custom exporter reads) is more code for the same effect. Choosing Pushgateway as the standard tool over custom plumbing.
- **Soft cap of 10 is a guess.** Real fleet ceiling depends on AWS instance limits in the developer VPC's region and on the operator's tolerance for surprise EC2 bills. 10 leaves plenty of headroom for normal use and is easy to revisit once concurrency is real.
- **Manager replacement.** When the manager is replaced (AMI upgrade, instance-type change), its EIP stays but its private IP changes — workers in flight at replacement time fail their final Pushgateway push. They terminate cleanly anyway; the operator just loses the run-history entry for those specific runs. Acceptable. A follow-up could route worker pushes through DNS resolved at push time.
- **No SOW activity in the cost panel for the first 24h.** The cost panel uses Prometheus's own counters of instance-hours, so it has no history at first deploy. Acceptable — the alternative (back-filling from CloudTrail) is disproportionate effort.
- **Loki memory pressure on `t4g.small` (stretch).** Loki at idle is ~150 MB; under a busy multi-worker tail it can push toward 300 MB. The manager's other services sit well under 1 GB combined, so headroom exists — but it shrinks the margin. Mitigation if pressure shows up: ship the v1 platform without Loki, bump the manager to `t4g.medium` (4 GB) in a separate one-line PR, then add Loki. The non-Loki design works fine on `t4g.small`; the application backend has proven that exact configuration.
- **GoDaddy DNS is a manual one-time step.** Not codified in Terraform. If `developers.progstrength.fitness` ever needs to point at a different host, that's a manual GoDaddy edit. Acceptable at single-operator scale; the EIP doesn't move on its own.
