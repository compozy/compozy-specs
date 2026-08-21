---
status: completed
title: Terminal settlement invariant, reconciliation sweep & loops config
type: backend
complexity: critical
---

# Task 1: Terminal settlement invariant, reconciliation sweep & loops config

## Overview

Delivers front 3 of the spec: every terminal transition of a loop run settles its coordinator and cell records inside the same store transaction, a fail-closed boot barrier neutralizes orphans before any claim traffic starts, and an idempotent interval sweep plus a metadata-only provenance backfill converge any remaining divergence. This lands first because it unblocks clean QA on every other front — the two known orphaned coordinators (runs crashed 2026-08-19 ~17:03:58/~17:04:09) disappear on first boot with an audit trail.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST implement `settleLoopRunTerminal(ctx, tx, runID, cause)` in `internal/store/globaldb` as THE single settlement authority per `_spec.md` Core Interfaces: applies the cause→status matrix, settles children before the coordinator, cancels queued/claimed runs of settled records in the same transaction, never records killed/canceled work as "completed", never touches already-terminal cells (Safety Invariant 1).
2. MUST route every transition that sets a loop run terminal through the primitive in-transaction: plan-driven completion, cancel, kill, child-loop stop (`applyCoordinatorRunStopsWithExecutor`), crash-recovery classification, node-cancel finalization, and sweep repair. A transition-path coverage test MUST fail when any terminal mutation bypasses the primitive (IT-030).
3. MUST implement `RunReconciler` in `internal/loop` per the Part II interface: `NeutralizeOrphans` (one transaction removing work-eligibility of every execution record owned by a terminal or missing loop run), `SweepOnce` (idempotent, status-guarded, zero lifecycle mutation on records of non-terminal runs), `BackfillProvenance` (metadata-only, idempotent, relational via `task_runs.loop_run_id → loop_runs`, covers coordinators of ACTIVE runs, returns the repaired count — the single owner of that number, N-002). The reconciler observes and settles via the primitive; it never claims runs (L-005).
4. MUST wire the boot barrier in `internal/daemon`: `NeutralizeOrphans` completes synchronously BEFORE `loopActionRuntime.Recover`, the coordinator backstop, the reconciler ticker, or any claimer starts. The barrier FAILS CLOSED: on error, readiness is never reported and none of those components start — structured startup error; log-and-retry policy applies to interval sweeps only (Safety Invariant 2).
5. MUST run the interval sweep as one daemon-owned goroutine: started by the composition root after the barrier, context-aware store calls, stop-and-join on shutdown, non-overlapping cycles (tick skipped while a cycle runs), failed cycle → structured error event + retry next tick.
6. MUST add `reconcile_interval` (duration, default `"1m"`, positive-only — `0s` rejected with `"reconcile_interval must be positive"`) to the EXISTING `[loops]` config section: struct field + default + validation + merge/overlay + deep clone + agent-settable tool-surface entry + example config + hand-authored config docs. Lifecycle rule `loops.*` (restart-required) already covers it. The boot sweep runs regardless of the interval value.
7. MUST seed coordinator task `metadata_json` with `loop_run_id`/`loop_name`/`workspace_id` at creation (closing the coordinator provenance gap) and repair historical rows via `BackfillProvenance`; rows whose run is gone and unreconstructible keep relational facts only (`run_id` from `task_runs.loop_run_id`, role from `run_kind`) — never id/title parsing.
8. MUST append structured audit reasons on `task_events`: `loop_run_terminal` (inline) ≠ `reconciled_run_terminal` (sweep) ≠ `run_missing` (retention orphan) — machine-readable fields readable via `compozy task timeline` (no cross-task audit query exists, B-007). If the timeline wire payload lacks a structured reason field, extend `internal/api/contract` and run `make codegen` in this task (contract co-ship, L-007).
9. MUST emit the monitoring contract: per-cycle structured slog (`runs_examined`, `records_settled`, `orphans_repaired`, `provenance_backfilled`, `duration_ms`) + per-repair task event with correlation keys (`task_id, run_id, loop_run_id, workspace_id, actor_kind=daemon, release_reason`).
10. MUST keep `internal/loop` free of `internal/api/*`/`internal/daemon` imports, keep `internal/task` loop-noun-free, and respect the 500-line production-file cap — new reconciler/settlement logic lands in single-responsibility files.
11. SHOULD reuse the existing settlement/rollup helpers (`settleCompletedTaskHierarchyWithExecutor`, parent rollup) where their semantics fit (`done` cause) and replace direct use on every other loop terminal path.
</requirements>

## Subtasks

- [x] 1.1 Implement the cause-aware settlement primitive + cause→status matrix in `internal/store/globaldb` (children-first ordering, structured reasons, `SettleResult`)
- [x] 1.2 Route every terminal path through the primitive in-transaction (plan-driven, cancel, kill, child-stop, crash classification, node-cancel finalize)
- [x] 1.3 Implement `RunReconciler` in `internal/loop`: `NeutralizeOrphans`, `SweepOnce`, `BackfillProvenance`
- [x] 1.4 Seed coordinator metadata keys at creation in `global_db_loop_coordinator_seed.go`
- [x] 1.5 Wire the fail-closed boot barrier + owned sweep goroutine in `internal/daemon` (ordering vs `Recover`/backstop/claimers; stop-and-join shutdown)
- [x] 1.6 Add `[loops] reconcile_interval` across the full config lifecycle (struct/default/validation/overlay/clone/tool-surface/example/docs)
- [x] 1.7 Emit audit events + monitoring slog per the spec Monitoring section
- [x] 1.8 Implement assigned unit tests (UT-030..033)
- [x] 1.9 Implement assigned integration tests (IT-001..009, IT-024, IT-025, IT-028, IT-030, IT-031)
- [x] 1.10 Implement E2E-002 in the runtime harness (seed the 2026-08-19 incident shape; boot repairs; second boot silent)
- [x] 1.11 Flag QA scenarios per the QA impact line (walk happens in the loop's QA phase)

## Implementation Details

Follow `_spec.md` Part II: Core Interfaces (primitive + `RunReconciler` signatures), Data Models (cause→status matrix table; provenance-backfill row), Development Sequencing phase 1, Monitoring and Observability, Safety Invariants 1-6. Skills to activate: `eng-code-guidelines` + `golang-master` + `eng-cleanup-failure-paths` (goroutine lifecycle, failure paths); tests per `eng-test-conventions` + `testing-boss` + `eng-consolidate-test-suites`; completion per `deslop` + `cy-final-verify`.

Suite placement (from `_tests.md`): settlement/sweep integration EXTENDS the existing settlement/rollup suite in `internal/store/globaldb`; the transition-path coverage test lives beside `settleLoopRunTerminal`; sweep/barrier get one co-located integration file; config extends `internal/config/loops_test.go`.

### Relevant Files

- `internal/store/globaldb/global_db_task_coordinator_settlement.go:9-42` — the happy-path-only settlement this task generalizes (root cause anchor).
- `internal/store/globaldb/global_db_task_parent_rollup.go:20-160` — rollup helpers the primitive reuses where `done` semantics fit.
- `internal/store/globaldb/global_db_task_coordinator_mutations.go:18-51` + `global_db_loop_cancel_finalize.go:13-70` — terminal paths that must gain inline settlement.
- `internal/store/globaldb/global_db_loop_node_terminal.go` — per-cell terminal transition writes.
- `internal/store/globaldb/global_db_loop_reconcile.go:16-61` — existing boot reconcile pattern (`ReconcileLoopCoordinatorsOnBoot`) the barrier sits beside.
- `internal/store/globaldb/global_db_loop_coordinator_seed.go:90-181` — coordinator seed (metadata gap) + id grammar.
- `internal/loop/coordinator_terminal_plan.go` + `coordinator_terminal_helpers.go` — terminal plan shapes + cell identity helpers (`NodeCellTaskID`).
- `internal/loop/coordinator_lifecycle.go` — `applyTerminalNodeLifecycle`/`applyFailedNodeLifecycle` (cell settlement half).
- `internal/loop/service_cancellation_reconcile.go` — the one existing reconcile entry point (`ReconcilePendingCancellations`).
- `internal/daemon/loop_action_runtime.go:28,130` — `ClaimNextRun` + `Recover` (the ordering constraint the barrier enforces).
- `internal/daemon/task_runtime_boot.go:389` + `boot.go` + `boot_runtime_services.go` — composition-root wiring points.
- `internal/scheduler/scheduler.go:69,91` — the interval-sweep goroutine pattern to mirror (`sweepExpiredLeases`).
- `internal/config/loops.go` + `loop_lifecycle.go:108-129` (`parsePositiveLoopDuration`) + `merge_loops.go` + `config_clone_loops.go` + `internal/config/lifecycle/lifecycle.go:114` + `tool_surface_loops.go` — the config-key lifecycle chain.
- `internal/task/lease_claim.go` + `lease_recovery_manager.go` — `ClaimNextRun` exclusivity (L-005; the sweep never claims).

### Dependent Files

- `internal/store/globaldb/global_db_task_test.go` + the settlement/rollup integration suite — extended by this task's cases.
- `internal/config/loops_test.go` + `internal/config/lifecycle/lifecycle_test.go` — config lifecycle coverage.
- `config.toml` (repo root example) + `packages/site/content/docs/configuration/config-toml.mdx` + `packages/site/content/docs/loops/configure.mdx` — hand-authored config docs gain the key.
- `internal/api/contract/` task-timeline payloads — only if the structured `reason` field is missing today (then `make codegen` co-ships).
- `docs/qa/scenarios/` — new settlement scenario (see QA impact).

### Related ADRs

- [ADR-006: Terminal settlement is part of the transition; sweep converges the rest](adrs/adr-006.md) — this task implements it end to end.
- [ADR-004: Classification rides existing provenance columns](adrs/adr-004.md) — the provenance-backfill shape (metadata-only, relational, truthful degrade).

## Deliverables

- `settleLoopRunTerminal` primitive + every terminal path routed through it (zero bypasses, proven by IT-030)
- `RunReconciler` (barrier/sweep/backfill) wired fail-closed in daemon boot with an owned, joinable goroutine
- Coordinator seed metadata + historical backfill
- `[loops] reconcile_interval` across the full config lifecycle incl. docs
- Structured audit reasons + monitoring slog
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-030, UT-031, UT-032, UT-033 — sweep guards: idempotency, `reconciled_run_terminal` vs `loop_run_terminal`, `run_missing`, backfill count single-owner
- [x] IT-001, IT-002, IT-003, IT-004 — natural/cancel/kill/child-stop paths settle per the cause matrix in the same transaction
- [x] IT-030 — transition-path coverage: any terminal mutation bypassing the primitive fails the build
- [x] IT-005, IT-006, IT-007, IT-008, IT-009 — crash repair within one cycle; idempotent re-run; active runs untouched; sweep-vs-inline race single winner (`-race`); deleted-run orphan → `run_missing`
- [x] IT-028 — boot barrier vs claim race on real SQLite: `ClaimNextRun` never returns terminal-run work
- [x] IT-031 — barrier fails closed: injected store error ⇒ no readiness, zero claims, no recovery/backstop/ticker, structured startup error
- [x] IT-024 — `[loops] reconcile_interval` lifecycle (default/override/validation/boot-sweep-unconditional)
- [x] IT-025 — audit reasons machine-readable on a task's timeline (swept vs natural vs run_missing)
- [x] E2E-002 — orphan-repair journey: seed the incident shape, boot settles both with `reconciled_run_terminal`, second boot emits nothing

## Success Criteria

- Every assigned test case implemented and passing
- A run reaching ANY terminal outcome leaves zero live execution records, verified on store reads in integration and on a real daemon in E2E-002
- Daemon boot with seeded orphans completes neutralization before the first claim; injected neutralization failure reports no readiness and starts no claimers
- `make gate` green on the task's diff; zero lint warnings

### Web/Docs Impact

- `web/`: none — checked surfaces: this task changes store/daemon internals and audit-event values; no wire-contract shape changes unless the timeline `reason` field is missing (then generated types regenerate via `make codegen` and `web/src/generated/compozy-openapi.d.ts` updates with no component changes — the web renders existing timeline payloads).
- `packages/site`: `content/docs/configuration/config-toml.mdx` + `content/docs/loops/configure.mdx` gain `loops.reconcile_interval` (hand-authored pages).
- QA impact: user-visible behavior change (orphan repair at boot; settlement guarantees; new config key) → add a content-addressed `untested` scenario for "terminal loop runs never leave live execution records (crash → boot repair, audit reasons)" under `docs/qa/scenarios/`; inside the `cy-loop-tasks` run this task flags only — the walk runs in the loop's QA phase.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: no new hooks or extension points; settlement/audit events are emitted at owning call sites (`settleLoopRunTerminal`, sweep) — never by tailing event tables; Extension Host API method set unaffected; bridge task subscriptions (terminal notifications by task id) benefit passively — settled records now always reach terminal states; MCP sidecars unaffected.
- Agent manageability: settlement audit observable via `compozy task timeline` structured reasons; sweep observable via structured slog keys; config key settable via `compozy config set loops.reconcile_interval` (tool-surface entry required by requirement 6); deterministic validation error on bad values.
- Config lifecycle: `loops.reconcile_interval` — struct field + default `1m` + positive validation + overlay + clone + `loops.*` restart-required lifecycle row (already present) + tool surface + example + docs + IT-024.
