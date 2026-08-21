---
status: completed
title: P6 windowed fan-out width
type: backend
complexity: high
---

# Task 6: P6 windowed fan-out width

## Overview

Removes the 64-branch logical-width cliff: total lane count becomes unbounded while only a `max_parallel`-bounded window of lanes materializes at a time, with exactly-once lane creation under epoch fencing, truthful settled/active/pending/never-materialized counts throughout, and strategy interaction at width (never-materialized lanes settle canceled-never-started without task runs). The three 64-ceilings (lint, runtime terminal, config validation) are deleted.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. The window engine MUST materialize lanes on demand (lane terminal → next pending index materializes once), bounded by `max_parallel`, with `(generation, node, item)` exactly-once creation under epoch fencing and window re-formation from durable state after restart.
2. The 64 ceilings MUST be deleted as a hard cut: `LoopMaxFanoutWidth` enforcement in lint (`fan_out_ceiling_exceeded` as a cap rule), the runtime `fan_out_width_exceeded` terminal at 64, and the `<= 64` config validation — `max_fan_out` remains required and validates as a positive author bound only.
3. Strategy semantics at width MUST hold: `fail_fast`/`race` cancel never-materialized lanes without ever starting them (cause distinct from canceled-in-flight); join arithmetic and `progress.*` stay truthful over materialized + pending lanes.
4. `branch_pruned`/absence aggregates MUST stay bounded at width (16 KiB event bound) with per-lane causes queryable.
5. No schema change (P6 is schema-free per the ownership table); config validation change ships with its docs and round-trip tests.
</requirements>

## Subtasks

- [x] 6.1 Window engine (`fanout_window.go`): on-demand materialization, exactly-once fencing, restart re-formation
- [x] 6.2 Delete the three ceilings (lint/runtime/config) + update `max_fan_out` validation semantics
- [x] 6.3 Strategy/window integration: never-materialized settlement, truthful counts, bounded aggregates
- [x] 6.4 Docs co-ship (`dsl-reference.mdx` fan-out table ceiling removal, config docs) + skill row
- [x] 6.5 Implement all assigned tests; run focused task verification (the batch gate is deferred by the execution directive)

## Implementation Details

Reference `_spec.md` Part II — Window engine component, Delete Targets (ceilings), Safety Invariant 6, ADR-004.

### Relevant Files

- `internal/loop/fanout_materialization.go:16-149` — up-front chunk materialization to windowize
- `internal/loop/types.go:10-21` — `LoopMaxFanoutWidth` constant home
- `internal/loop/linter.go:215-237`, `internal/loop/control_plan.go:314-365` — lint + runtime ceiling sites
- `internal/config/loops.go` (`loopDefaultsMaxFanoutWidth`, `Validate`) — config ceiling
- `internal/loop/control_readiness.go:103-174` — throttling window to fold into the engine
- `internal/daemon/loop_action_runtime.go:94` — action-runtime semaphore interaction

### Dependent Files

- task_05's `control_join.go` — settlement over materialized+pending lanes
- `internal/loop/coordinator_succession.go:79-85` — rerun re-materialization width
- `packages/site/content/docs/loops/dsl-reference.mdx`, config docs

### Related ADRs

- [ADR-004: Fan-out v2](adrs/adr-004.md) — windowed width kept in scope (grill Q11)

### References

Read before implementing (evidence catalog: `analysis/sim.md` mechanism 6):

- `.resources/sim/apps/sim/executor/orchestrators/parallel.ts:27,113-139,141-183,418-526` — the direct source: `totalBranches` unbounded, only `currentBatchSize` virtual lanes materialize at a time (`branchIndexOffset` + `totalBranches`), sentinel-end aggregates per batch, compacts, `advanceToNextBatch`, re-arms — our window engine's shape
- `.resources/sim/apps/sim/executor/utils/parallel-expansion.ts:41-139` — per-window lane expansion with offsets (the exactly-once-per-index bookkeeping)
- `.resources/sim/apps/sim/executor/execution/snapshot-serializer.ts:17-133,229-231` — batch boundaries as natural checkpoints + snapshot compactness assertion (the durable-window-state discipline our restart re-formation needs)
- `.resources/temporal/service/history/api/respondworkflowtaskcompleted/workflow_size_checker.go:115-185` — pending-limit enforcement at command-validation time (how a bound is enforced without a hard product cliff — the spirit of replacing the 64 ceiling with author bounds + windows)

## Web/Docs Impact

- `web/`: S3 aggregate-count display at width lands in task_08; contract unchanged here beyond counts already shipped in task_05.
- `packages/site`: fan-out table + config page ceiling removal.
- QA impact: behavior change → reset/add an `untested` wide-fan-out scenario in `docs/qa/scenarios/`; walk before completion.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked (no surface change).
- Agent manageability: truthful width counts already flow through `loop status` structured output.
- Config lifecycle: `fan_out_width` validation semantics change (ceiling removed) — docs + validation tests in the same change.

## Deliverables

- Unbounded logical width with a bounded window, ceilings deleted, truthful counts at 100× typical width
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] UT-045, UT-046, UT-047, UT-048, UT-049 — window advance, window≥count, never-materialized settlement, single-cause determinism, bounded aggregates
- [x] UT-106 — config ceiling removal validation
- [x] IT-008 — restart mid-window/mid-cancellation re-forms exactly-once; pending request still answerable
- [x] E2E-013 — 500-lane fan-out with `max_parallel: 8` completes with truthful progress throughout

## Success Criteria

- Every assigned test case implemented and passing
- `grep -rn "64" internal/loop/types.go internal/config/loops.go` shows no width ceiling; a 500-lane definition compiles and runs
- No lane ever runs twice across the restart suite
- `make gate` passes

## Completion Notes

- Durable output rows now reconstruct each fan-out window after restart; terminal lanes open exactly the next available indexes, while strategy exits record never-started lanes without task runs and chunk `branch_pruned` events below the 16 KiB contract.
- Fresh focused verification passed 174 race-enabled Go tests across the Loop and config packages, 99 web tests through Turborepo, web typecheck, codegen-check, and `git diff --check`. Earlier broad Loop/config verification passed 2,439 unaffected tests; the batch gate and live E2E walk remain deferred by the controlling task-loop plan.
- QA tracker: `LP-wide-fanout-window.md` is flagged `untested` for the batch QA phase.

Compozy Impact Audit:

- Native tools: no tool ID, descriptor, schema digest, or capability-gate change; checked `loop status` progress projections, which continue to expose total, running, pending, and settled counts.
- Extensibility and hooks: no extension, hook, capability, tool/resource registry, bridge SDK, or MCP sidecar change. Config lifecycle changed only `loops.defaults.<kind>.fan_out_width` validation and docs from a fixed 64 cap to any nonnegative window.
- Workspace data isolation: no new datum or scope. Window reconstruction uses existing workspace-owned run, generation, node, and item output rows; CLI, HTTP, UDS, SSE, cache, and event paths remain unchanged.
- Official Compozy skill: updated `skills/compozy/references/loops.md` and `references/configuration.md` for the positive authored bound and uncapped active window.
