---
status: shipped
repos:
  - prog-strength-developer
  - prog-strength-infra
  - prog-strength-docs
---

# Fleet Dispatch Gating — One Worker Per SOW

**Status**: Shipped · **Last updated**: 2026-06-15

## Introduction

The autonomous developer platform can now run several workers at once — the manager instance and per-worker metrics made concurrent dispatch safe and observable, and the single-instance Terraform gate was deliberately removed so multiple SOWs can build in parallel. That was the right move, but it left a sharp edge: **nothing stops two workers from being dispatched against the same SOW.** The dispatch workflow takes a `sow_path`, tags the instance with it, and launches — with no check for "is someone already working this one?" An operator who double-clicks the dispatch button, re-runs a workflow they thought failed, or simply forgets a worker is already mid-run will end up with two (or five) instances cloning the same repos, opening duplicate PRs against the same SOW, and burning compute and Claude tokens in parallel for zero added value. The only signal today is an EC2 tag, which nothing reads back before launching.

This is both a correctness problem and a cost problem. Two workers on one SOW race each other: they branch from the same base, make divergent edits, and open conflicting or duplicate PRs that then have to be untangled by hand. And every redundant worker is real money — an EC2 instance running for up to six hours plus the Claude API spend of a full SOW build, multiplied by however many duplicates slipped through. The failure mode is mundane (a mis-click, an over-eager re-run) but the blast radius is not.

This SOW adds a **dispatch gate**: a central record of which SOW each worker is currently building, and a check at dispatch time that **refuses to launch a second worker for a SOW that is already in progress.** The record lives in a DynamoDB table; the check is a single atomic conditional write, so the guarantee — *at most one active worker per SOW* — holds even if two dispatches fire at the same instant.

Crucially, the read/write logic for this does **not** go into the GitHub workflow as more inline bash. The dispatch workflow is already carrying too much logic (fleet-cap math, infra resolution, userdata rendering, instance launch), and the platform is going to keep getting smarter — richer lifecycle management, scheduling, retries, worker-to-SOW assignment policy. Stuffing each new capability into YAML shell steps does not scale and is miserable to test. Instead, this SOW introduces a small, properly-structured **`fleet` control-plane package** in Python — the language the rest of this repo already uses (`bootstrap/`, the worker exporter, the test suite). The workflow and the worker become thin callers of that package. The gate is its first capability; the package is built so the next ten are additive, testable, and out of the workflow.

After this ships, dispatching a worker for an in-progress SOW fails fast with a clear message and launches nothing. Workers register themselves as they start and release as they finish. An operator can list what is currently building and force-release a stuck lock. And the platform has a real, extensible home for control-plane logic to grow into.

## Proposed Solution

A new top-level Python package — proposed name **`fleet`** (the term the dispatch workflow and Grafana dashboard already use for the worker pool) — becomes the control plane. It owns a single responsibility in v1: the **run registry**, a record of which SOW each worker is building, backed by DynamoDB. The package is layered the way the Go API's domains are, so its logic is unit-testable without touching AWS:

- A `RunRegistry` **interface** with the operations the platform needs (`try_acquire`, `attach_instance`, `release`, `get`, `list_active`).
- A `DynamoRunRegistry` **implementation** over `boto3`.
- A `FakeRunRegistry` **in-memory implementation** for tests — the same dual-implementation pattern the API uses with its `sqlite_repository` / `memory_repository` split.
- A thin **CLI** (`python -m fleet …`) that the dispatch workflow and the worker shell out to. The CLI is an adapter over the package, not where logic lives.

The DynamoDB table holds **one item per SOW**, keyed by the SOW path. Acquiring the lock is a **conditional `PutItem`/`UpdateItem`** that succeeds only when no worker currently holds that SOW (no item, a terminal item, or an expired one). Because the condition is evaluated atomically by DynamoDB, two simultaneous dispatches cannot both win — exactly one acquires, the other is refused. The dispatch workflow acquires **before** launching an instance, so a refused dispatch costs nothing. The worker releases the lock when it finishes (success, error, or timeout), in the same finalize step that already pushes summary metrics to the manager's Pushgateway. Stale locks (a worker that died without releasing) self-heal via an `expires_at` derived from the max runtime, and an operator can force-release one by hand.

## Goals and Non-Goals

### Goals

- A **central record** (DynamoDB) of which SOW each active worker is building.
- A **dispatch gate**: refuse to launch a worker for a SOW that already has an active worker — atomically, so concurrent dispatches can't both slip through.
- **At most one active worker per SOW**, guaranteed by a conditional write, not by a best-effort read-then-launch.
- Refused dispatches launch **no instance** (the cost saving).
- Control-plane read/write logic lives in an **extensible Python package**, not inline workflow bash — built so future management/worker logic is additive and testable.
- Workers **register on start** and **release on finish**; stale locks **self-heal** via expiry; an operator can **list** active runs and **force-release** a stuck one.
- Update the `prog-strength-developer` **README** to document the gate, the table, and the CLI.

### Non-Goals

- A **long-running manager service / HTTP API** for the control plane. v1 invokes the package as a CLI from the workflow and worker; the package is structured so a service can wrap it later without rewriting the core. That service is a future SOW.
- Changing the **fleet cap** or the concurrent-dispatch model. This gates *per SOW*, orthogonally to the *total worker* cap, which stays as-is.
- **Queueing / auto-retry** of refused dispatches. A refused dispatch fails; re-dispatching is the operator's call once the in-progress run finishes.
- **Worker health monitoring, scheduling, or assignment policy.** These are the kind of thing the new package will host later, but they are out of scope here.
- **Multi-region.** The table lives in the existing `us-east-2` region.

## Implementation Details

### The `fleet` package (`prog-strength-developer`)

A new top-level package alongside `bootstrap/`, with `boto3` added to `pyproject.toml` dependencies. Structure:

- **`fleet/models.py`** — a `RunRecord` dataclass: `sow` (path, the key), `status` (`working` / `done` / `error` / `timeout`), `instance_id` (optional until attached), `dispatch_id` (a UUID minted per dispatch attempt), `started_at`, `updated_at`, `expires_at`, `dispatched_by` (the GitHub actor, for the audit trail). Plus a `RunStatus` enum and small helpers.
- **`fleet/registry.py`** — the `RunRegistry` protocol/ABC and its operations:
  - `try_acquire(sow, dispatch_id, ttl) -> AcquireResult` — attempt to claim the SOW. Returns success, or a conflict carrying the current holder's `instance_id` / `started_at` so the caller can print a useful message.
  - `attach_instance(sow, dispatch_id, instance_id)` — record the launched instance on the lock this dispatch already holds (guarded on `dispatch_id` so it only patches its own reservation).
  - `release(sow, instance_id, outcome)` — mark the run terminal and free the SOW.
  - `get(sow) -> RunRecord | None` and `list_active() -> list[RunRecord]`.
- **`fleet/dynamo.py`** — `DynamoRunRegistry`, the `boto3` implementation. `try_acquire` is the conditional write described below.
- **`fleet/memory.py`** — `FakeRunRegistry`, an in-memory dict-backed implementation with the same conditional semantics, so the gating logic and CLI are tested without AWS (parity with the API's memory repository).
- **`fleet/service.py`** — thin orchestration over the registry (e.g. `gate_dispatch(...)` composing `try_acquire` + message formatting). Keeps the CLI dumb.
- **`fleet/config.py`** — table name and region from env (`FLEET_TABLE_NAME`, `AWS_REGION`), with the deployed defaults.
- **`fleet/cli.py`** + **`fleet/__main__.py`** — subcommands: `acquire`, `attach`, `release`, `list`, and `release --force`. Exit codes are meaningful (non-zero on a refused acquire) so the workflow can branch on them. Output is human-readable for the workflow log and `--json` for machine use.
- **`tests/`** — unit tests against `FakeRunRegistry` covering acquire success, **acquire conflict** (the core guarantee), takeover of an expired lock, `attach` guarded by `dispatch_id`, release, and the CLI exit codes.

### DynamoDB table (`prog-strength-developer` Terraform)

A new table in `terraform/` (e.g. `dynamodb.tf`): `prog-strength-developer-runs`, **`PAY_PER_REQUEST`** billing (dispatch volume is tiny; no capacity to manage), partition key **`sow`** (string) and no sort key — one item per SOW *is* the lock. TTL enabled on **`expires_at`** for automatic cleanup of old terminal/abandoned rows. Item shape mirrors `RunRecord`.

**Acquire = one atomic conditional write.** `try_acquire` issues a `PutItem` (or `UpdateItem`) with:

```
ConditionExpression:
  attribute_not_exists(#sow)
  OR #status <> :working
  OR #expires_at < :now
```

setting `status = working`, `dispatch_id`, `started_at`, `expires_at = now + max_runtime + buffer`. If the condition fails, DynamoDB raises `ConditionalCheckFailedException` → the SOW is actively held → the dispatch is refused. **Correctness note:** the `expires_at < :now` term is what makes a stale lock reclaimable; DynamoDB TTL *deletion* is best-effort and can lag by hours, so expiry is enforced in the condition, not by relying on the row having been deleted.

### IAM

- **Worker inline policy** (`terraform/iam.tf`, this repo): add a statement granting `dynamodb:GetItem`, `dynamodb:PutItem`, `dynamodb:UpdateItem` (and `dynamodb:Query` for `list`) on the new table's ARN, so the worker can `release` at finalize. Scoped to the single table ARN, consistent with the least-privilege style already in that file.
- **Shared GitHub Actions OIDC role** (`prog-strength-infra`): the dispatch workflow assumes the org-wide CI/CD role, which is managed in infra — not in this repo. Add the same DynamoDB actions on the table ARN to that role's policy so the workflow can `acquire` / `attach` / `list`. This is the one cross-repo change and the reason `prog-strength-infra` is in scope.

### Dispatch workflow (`dispatch-sow.yml`)

The workflow becomes a thin caller of the package. Concretely:

1. After **Configure AWS credentials**, add a step that installs the package (`uv sync`) and runs **`python -m fleet acquire --sow "$SOW_PATH" --dispatch-id "$RUN_ID"`** — placed *before* "Run instance," after the existing fleet-cap check. On a non-zero exit (SOW already in progress), the step fails with a clear `::error::` naming the instance already building it, and **no instance is launched**.
2. The existing **Run instance** step is unchanged.
3. After launch, **`python -m fleet attach --sow "$SOW_PATH" --dispatch-id "$RUN_ID" --instance-id "$IID"`** records the instance on the reservation.

The fleet-cap check (total workers) and this per-SOW gate are complementary guardrails and both stay.

### Worker release (`bootstrap/userdata.sh.tpl`)

The worker already has a finalize path that pushes summary metrics to the manager's Pushgateway just before self-termination. Add a **`python -m fleet release --sow "$SOW_PATH" --instance-id "$INSTANCE_ID" --outcome "$OUTCOME"`** call there, so the SOW is freed the moment the run ends (any outcome). This requires the `fleet` package to be present on the worker — delivered by the same bootstrap mechanism that already puts `worker_exporter.py` on the instance (install the repo package during userdata). If release is ever missed (hard crash), the lock's `expires_at` reclaims the SOW automatically.

### README (`prog-strength-developer`)

Add a **"Fleet control / dispatch gating"** section documenting: the one-worker-per-SOW guarantee and why it exists (correctness + cost); the `prog-strength-developer-runs` table and its lock semantics; the `fleet` package as the home for control-plane logic; and the operator CLI (`python -m fleet list` to see what's building, `python -m fleet release --sow … --force` to clear a stuck lock). Note the package is the intended landing spot for future management/worker logic, not the workflow.

### Tests

- **Package unit tests** (against `FakeRunRegistry`): acquire success; **acquire conflict refuses** (the guarantee); expired-lock takeover; `attach` only patches its own `dispatch_id`; release frees the SOW; CLI exit codes / `--json`.
- **DynamoDB conditional semantics** covered via a `moto`-mocked (or equivalent) test of `DynamoRunRegistry.try_acquire`, asserting the second concurrent acquire raises and is surfaced as a conflict.
- Terraform `plan` validates the table + IAM additions.

### Rollout

1. **`prog-strength-infra`** — add DynamoDB perms to the shared OIDC role and apply, so the workflow role can use the table once it exists.
2. **`prog-strength-developer`** — Terraform (table + worker IAM), the `fleet` package, the workflow gate, the worker release call, and the README. Apply Terraform, then merge the workflow change.
3. No data migration; the table starts empty and fills as workers dispatch. The gate is live on the first dispatch after merge.

## Open Questions

- **Package name** — `fleet` is proposed (consistent with existing "fleet" language). `control`, `controlplane`, or `platform` are alternatives if a broader-than-the-worker-pool name reads better as the control plane grows. Cheap to settle at implementation.
- **History vs. lock-only** — v1 keeps one item per SOW (the lock doubles as the last-run record, overwritten on the next run). If a durable run history is wanted later, add a sort key (`started_at`) or a separate history table; called out so the key choice is deliberate.
- **Heartbeat** — workers self-terminate at max runtime, so a static `expires_at` is sufficient. If runs ever vary widely in length, a periodic heartbeat that extends `expires_at` would tighten stale-lock reclamation; deferred.
- **Worker package delivery** — confirm the cleanest way to put `fleet` on the worker (install the repo package in userdata vs. vendoring just the registry module), consistent with how `worker_exporter.py` is delivered today.
