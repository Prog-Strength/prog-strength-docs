# Recovery History Endpoint — Implementation Plan

SOW: [`sows/recovery-history-endpoint.md`](../sows/recovery-history-endpoint.md)
Repos: `prog-strength-api`, `prog-strength-docs`
Branch: `feat/recovery-history-endpoint`

## Shape of the work

One new authed route, `GET /recovery/history`, serving the **same**
`RecoverySection` payload `GET /dashboard/summary` already serves under
`sections.recovery`, over a caller-supplied local-date window. No new
derivation: `internal/recoverytrend` is untouched. The only structural change is
that the pure builder in `internal/dashboard/whoop.go` learns an explicit
charted window, and `buildWhoop` becomes the thin wrapper that supplies the
dashboard's particular one.

Everything lands in `prog-strength-api`. `prog-strength-docs` gets the status
flip only.

## Decisions taken here (SOW left them open or under-specified)

1. **Open Question 1 — `since` default.** Ship the flat default
   (`until − page_window_max_days`); do **not** add a `MinDate` repository
   method or a second query. The SOW's lean. The leading-all-null trim is
   *not* implemented either: the SOW frames it as "if the emptiness turns out
   to matter visually", and this endpoint has no consumer yet, so trimming now
   would be speculative and would break the key-for-key identity with
   `/dashboard/summary` for a user whose history starts mid-window.
2. **Open Question 2 — path.** `/recovery/history`, the SOW's lean.
3. **Open Questions 3 & 4** — leave unbuilt / keep both, per the SOW's leans.
4. **`page_window_max_days` reaches the handler via a setter**, not a 15th
   `NewHandler` argument. `SetHRZonesEngine` is the in-repo precedent and its
   comment states the reason: keeping `NewHandler`'s signature stable keeps
   every existing dashboard test untouched — which is exactly the regression
   proof this SOW asks for. `NewHandler` seeds the field with the same 1825
   the manifest ships so a handler built without the setter is still coherent.
5. **Config validation is guarded on `baseline_window_days > 0`.** The
   `[recovery]` block has no defaults and no validation today, and
   `config_test.go`'s `minimalTOML` carries no `[recovery]` block at all — so
   an unconditional `page_window_max_days > 0` check would fail ~30 existing
   config tests. The pair is validated only once a baseline window is
   configured, i.e. once the recovery engine is actually in play. That
   satisfies the SOW's test spec (0 fails, below-baseline fails) without
   rewriting unrelated fixtures.
6. **Clamping arithmetic.** `since` is clamped forward to
   `until − page_window_max_days` (so the widest served window is
   `page_window_max_days + 1` inclusive local dates — the same off-by-one the
   dashboard's own `2×win + 1` window carries). Clamp, never 400.

## Task 1 — `buildRecoveryView`: the explicit-window builder

**File**: `internal/dashboard/whoop.go`

Extract the body of `buildWhoop` into:

```go
func buildRecoveryView(
    entries []whooprecovery.Entry,
    eng *recoverytrend.Engine,
    from, to string,   // local YYYY-MM-DD, inclusive, charted window
    now time.Time,     // for Today and the 7-day spark; unchanged semantics
    loc *time.Location,
) *RecoverySection
```

- `buildWhoop(entries, eng, now, loc)` stays, becomes a thin wrapper: computes
  `from = today − eng.BaselineWindowDays()`, `to = today` (the last `win + 1`
  local dates ending today) and delegates. Its doc comment keeps the "one
  particular window" fact in one place.
- `Today` and `RestingHRSpark` are built from `now`/`loc` exactly as today —
  they are not window-relative, and keeping them so is what makes the two
  doors' payloads identical for the same user.
- Date materialization: emit **every** date in `[from − win, to]`, oldest→
  newest, absent days as `recoverytrend.Day{Date: ds}` with null metrics.
  Anchor the walk on `from` and index off it with
  `time.Date(y, mo, d+i, 0, 0, 0, 0, loc)` — the existing idiom, and drift-free
  across DST because every date is computed from the anchor rather than
  accumulated. Derive the inclusive day count between `from` and `to` from
  UTC-normalized midnights so a 23h/25h local day cannot round it wrong.
- `charted := engineDays[win:]`; `eng.ComputeSeries(engineDays)`;
  `eng.Compute(charted)`. Unchanged below the window computation.
- **Degenerate windows must not panic.** If `from`/`to` fail to parse, or
  `to < from`, materialize no dates at all: `engineDays` empty, `charted`
  empty, `Days` a non-nil empty slice, and the three derived blocks come back
  in their unknown/zero state from the engine (`Compute(nil)` /
  `ComputeSeries(nil)` are both safe). Never slice `engineDays[win:]` when
  `len(engineDays) < win`. This path is also what the not-connected handler
  branch uses, so the empty payload's unknown states are engine-derived rather
  than hand-written literals.
- Doc comment on `RecoverySection` (in `dto.go`) notes it is now served by two
  routes and that `Baseline` / `HRV` / `BaselineTrend` describe the window's
  **final** day.

**Tests** — `internal/dashboard/whoop_test.go`:

- Every existing `buildWhoop` test passes **unmodified**. Do not touch them.
- Explicit narrower-than-dashboard window → exactly its own charted dates, with
  per-day bands present on the **first** charted day (the lead-in proof).
- Window whose lead-in reaches before the oldest stored row → every charted
  date still emitted, `status: "unknown"` and null band fields where the
  sample is short.
- Window spanning a month boundary **and** a DST transition
  (`America/Denver`, March) materializes the right local dates.
- Missing mornings inside a wide window are present with null metrics, in
  order.
- A `to < from` window yields an empty `Days` and does not panic.

## Task 2 — `page_window_max_days` config knob

**Files**: `config.toml`, `internal/config/config.go`

- `config.toml` `[recovery]`: add the `page_window_max_days` paragraph to the
  existing comment block, in its neighbours' voice, and
  `page_window_max_days  = 1825`.
- `RecoveryConfig.PageWindowMaxDays int` + the raw TOML struct field
  (`toml:"page_window_max_days"`) + the mapping in `Load`. Extend
  `RecoveryConfig`'s doc comment.
- Validation in `Load`, alongside the JWT/AWS checks, guarded per decision 5:

  ```go
  if cfg.Recovery.BaselineWindowDays > 0 {
      if cfg.Recovery.PageWindowMaxDays <= 0 { ... }
      if cfg.Recovery.PageWindowMaxDays <= cfg.Recovery.BaselineWindowDays { ... }
  }
  ```

  with a short comment explaining that a manifest with no `[recovery]` block
  has no engine to size a window for.

**Tests** — `internal/config/config_test.go`:

- The golden-manifest recovery assertions gain `PageWindowMaxDays == 1825`
  (both the struct literal comparison and the field-by-field test, wherever
  they exist).
- `page_window_max_days = 0` alongside `baseline_window_days = 30` → `Load`
  errors.
- `page_window_max_days = 20` alongside `baseline_window_days = 30` → errors.
- A manifest with a valid pair loads.

## Task 3 — the read path

**Files**: `internal/dashboard/recovery_history_handler.go` (new),
`internal/dashboard/handler.go`, `internal/server/server.go`

- `Handler` gains `pageWindowMaxDays int`, seeded in `NewHandler` to a package
  const `defaultPageWindowMaxDays = 1825`, and
  `SetPageWindowMaxDays(days int)` which ignores a non-positive value.
- `Mount` gains `r.Get("/recovery/history", h.recoveryHistory)`.
- `server.go` calls `dashboardHandler.SetPageWindowMaxDays(cfg.Recovery.PageWindowMaxDays)`
  beside the existing `SetHRZonesEngine` call.
- `recoveryHistory`, in order, first-error-wins:
  1. `auth.UserIDFrom` — missing → `401` (`httpresp.ServerError` is what
     `summary` does for a missing context user; match the *route's* semantics
     by returning `401 missing authenticated user` as `/whoop/recovery` does,
     since the SOW names 401 explicitly).
  2. `timezone` required → `daterange.LoadTimezone`; empty or unloadable →
     `400`.
  3. `since` / `until` optional `YYYY-MM-DD`, malformed → `400`;
     `until` defaults to today in `loc`; `since` defaults to
     `until − pageWindowMaxDays`; `since > until` → `400`.
  4. Clamp `since` forward to `until − pageWindowMaxDays` when the span is
     wider. Never reject.
  5. Connection gate: `whoopConns.Get`; `ErrNotFound` or a non-connected
     status or a read error (logged) → `200` with the empty section.
  6. One `ListRange(ctx, userID, from − win, to)` through `defer1`, so a repo
     error degrades to an empty window rather than a 500.
  7. `buildRecoveryView(entries, h.recovery, from, to, now, loc)`.
  8. `httpresp.OK(w, "read recovery history", recoveryHistoryDTO{Recovery: section})`.
- `recoveryHistoryDTO` in the same file: `Recovery *RecoverySection
  \`json:"recovery"\`` — always non-nil, so the key is always present.
- The handler carries the *why a separate endpoint rather than an enriched
  `/whoop/recovery`* reasoning as a doc comment (MCP token cost), plus the
  `until`-is-today invariant for the three derived blocks.

**Tests** — `internal/dashboard/recovery_history_handler_test.go`:

- `401` without a user.
- `400` for: missing `timezone`, unloadable `timezone`, malformed `since`,
  malformed `until`, `since > until`.
- Not connected → `200`, `days: []`, key present, all three derived blocks in
  their unknown state.
- Connected with rows → `days` covers `[since, until]` inclusive,
  oldest→newest, derived blocks populated.
- A request wider than `page_window_max_days` is clamped → `200`, and the
  window actually fetched/charted starts at the clamp.
- A recovery repository error → `200` with the empty window.
- **Drift guard**: the `recovery` object from `GET /recovery/history` over the
  dashboard's own window is byte-identical to `/dashboard/summary`'s
  `sections.recovery` for the same user — marshal both and compare.

**Tests** — `internal/server/server_test.go`: `GET /recovery/history` joins the
mounted-route list.

## Task 4 — docs + status flip

- `prog-strength-api`: no README/CHANGELOG edit (semantic-release owns the
  changelog); the config comment and the doc comments in tasks 1–3 are the
  documentation deliverable.
- `prog-strength-docs`: `sows/recovery-history-endpoint.md` frontmatter
  `status: shipped`, body `**Status**: Shipped`, `**Last updated**: 2026-08-13`.

## Gate before pushing

From `prog-strength-api`, all green:

```
gofmt -l .
go build ./...
go vet ./...
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run
go mod tidy && git diff --exit-code go.mod go.sum
go test ./...
```

Pre-commit hooks (`pre-commit`, `commit-msg`, `pre-push`) are armed and must
not be bypassed.
