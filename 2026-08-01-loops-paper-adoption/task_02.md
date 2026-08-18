---
status: completed
title: Succession semantics, namespace history and fenced state
type: backend
complexity: critical
---

# Task 2: Succession semantics, namespace history and fenced state

## Overview

Rewires the coordinator: generation history (`previous.*`/`best.*`) crosses the boundary into
templates and CEL, gate rejection re-runs the deterministic producer union with repair context,
both `next_generation` surfaces start real generations, ratchet rejections seed from best, and
every new mutation (verdict, best, provenance, event) rides the token-fenced coordinator
completion plan. This is the core-semantics slice — highest regression surface of the program.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST activate skills: `eng-code-guidelines`, `golang-pro`, `eng-cleanup-failure-paths`, `systematic-debugging` (livelock regression), `eng-test-conventions`, `testing-boss`.
- MUST build the `GenerationHistory` projection (previous nodes/verdicts-map/route-causes, best via `BestNodeProjection`) and thread it through all 8 namespace call sites (TechSpec Architectural Boundaries; ADR-001).
- MUST carry verdict/best/provenance/event mutations as typed intents on `GenerationSnapshotPayload`/`CoordinatorCompletionPlan`, applied by the store finalizer inside `CompleteCoordinatorAndEnqueueNext`; generation-1 rows written by the loop-start transaction; SSE broadcast only post-commit (Safety Invariant 4/14; B-001).
- MUST implement rejection rerun-roots as the deterministic union of transitive ancestors across ALL route-causing gates (`route_cause_rank` persisted in the same plan; Safety Invariant 8), and delete the blank-output revise seeding.
- MUST make `RouteNextGeneration` start a fresh full-body generation on BOTH surfaces (in-body → origin `gate_next_generation`; DoD → `dod_retry`), deleting the `Terminal{failed, contract}` branch; `iterationCapTerminal` fires when the cap is already reached (ADR-002).
- MUST implement ratchet seeding: baseline present → carry-forward from best (`ratchet_restore`, parent = best); no baseline → seed from last (Safety Invariant 3); best advances only per the eligibility rule inside the plan transaction.
- MUST keep stall/breaker/cap arithmetic untouched (Safety Invariant 9) and all bounds enforced on the new paths (Safety Invariant 10).
- MUST extend the observability coverage-matrix so every new lifecycle path fails the build without its canonical event (enriched `generation_started`, migrated `gate_verdict`).
- MUST respect the 500-line cap: succession logic lands in new `coordinator_succession.go` + `control_namespace_history.go`; `control_plan.go` (494), `coordinator_generation.go` (443), `linter_references.go` (493) must not grow.
</requirements>

## Subtasks

- [x] 2.1 Implement `control_namespace_history.go` projection + reader wiring (`GenerationOutputReader`, `VerdictReader`, `GenerationLineageReader`).
- [x] 2.2 Thread history through the 8 `buildRuntimeNamespace` call sites and the CEL environment.
- [x] 2.3 Extend `GenerationSnapshotPayload`/`CoordinatorCompletionPlan` with intents; extend the store finalizer to apply them atomically; add the loop-start generation-1 write.
- [x] 2.4 Implement rerun-root union + `route_cause_rank` persistence in `coordinator_succession.go`; delete blank-output seeding.
- [x] 2.5 Implement both `next_generation` fresh-plan paths + origins; delete the DoD terminal branch.
- [x] 2.6 Implement ratchet seed-from-best + no-baseline branch + best-update intent.
- [x] 2.7 Emit enriched `generation_started` + migrated `gate_verdict` events post-commit; extend the coverage-matrix test.
- [x] 2.8 Implement all assigned UT/IT cases (rollback matrix, stale claim, livelock regression red-before/green-after); run `make gate`.

## Implementation Details

TechSpec sections: Component Overview (data flow), Core Interfaces, Safety Invariants 3-10/14,
Delete Targets (terminal branch, blank-output seeding). Semantics: ADR-001/002/003/004.

### Relevant Files

- `internal/loop/control_namespace.go:15-55` (197) — namespace builder + 8 call sites listed in ADR-001
- `internal/loop/coordinator.go:41-59,160-218` (244) — reader seams, terminal handling, succession entry
- `internal/loop/coordinator_generation_reattempt.go:65-142,220-253` (257) — rerun set + carry-forward
- `internal/loop/coordinator_generation.go:234-322` (443, frozen) — DoD terminal + stop_when plan builders
- `internal/loop/control_contract_gate.go:113-133` (141) — DoD verdict → terminal branch (delete target)
- `internal/loop/control_gate.go:134,173-252` (277) — revision counter, verdict materialization
- `internal/task/coordinator.go:35-38` + `internal/store/globaldb/global_db_task_coordinator.go` — completion plan + fenced finalizer
- `internal/loop/generation_snapshot.go:14-19,51-125` (228) — snapshot payload extension point

### Dependent Files

- `internal/loop/coordinator_terminal_helpers.go:76-212` — stall/breaker scans must remain semantically identical
- `internal/daemon/loop_managed_binding.go:53-57` — session reuse unchanged (no generation in key)
- `internal/loop/goal/**` — untouched; turn-level feedback remains the intra-node analogue

### Related ADRs

- [ADR-001](adrs/adr-001.md) — namespace roots + call-site threading
- [ADR-002](adrs/adr-002.md) — rerun-root union + both `next_generation` surfaces
- [ADR-003](adrs/adr-003.md) — eligibility, seeding, plan-fenced persistence
- [ADR-004](adrs/adr-004.md) — origins, parent rules, count equivalence

### Web/Docs Impact

- `web/`: none in this task — contract payloads unchanged until task_03; checked: no `internal/api/contract` edits here.
- `packages/site`: none in this task — semantics documented in task_05.
- QA impact: behavior changes (re-attempt semantics, DoD retry, ratchet) become user-visible only through task_03 surfaces; scenario resets/mints recorded in task_06 (`TA-loop-failure-breaker` reset; new `LP-ratchet-*`, `LP-dod-reject-retry`, `LP-revise-repair-context`).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: hook payload fields (`loop.gate.post` score/outcome/best; `loop.generation.pre` origin/parent) are emitted from here but surfaced/asserted in task_03; pre-hook `LoopControlPatch` cannot mutate verdict/best (observe-only) — checked `internal/hooks/dispatch_patch_clones.go`.
- Agent manageability: no direct surface change in this task; deterministic behavior contract (origins, terminals) feeds task_03 outputs.
- Config lifecycle: none — existing `loops.defaults.*` bounds (`iteration_cap`, `gates.max_revisions`, budgets) govern all new paths unchanged; checked `internal/config/tool_surface_loops.go:11-36`.

## Deliverables

- Repair context + ratchet + both `next_generation` surfaces working end-to-end against a real SQLite store.
- All four delete targets of this slice removed (terminal branch, blank-output seeding).
- Rollback matrix + stale-claim proof of plan atomicity; livelock regression red-before/green-after.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-011, UT-012, UT-013, UT-014 — history projection (verdicts map, route causes, best projection)
- [x] UT-028, UT-029 — generation intents + origin mapping + DDL/invariant rejection
- [x] UT-030, UT-031, UT-032, UT-033, UT-034, UT-035 — rerun-root union, repair context, both `next_generation` surfaces, cap boundary, seeding branches, child-run carry matrix
- [x] IT-001, IT-002 — history via real queries; command scorer through evaluator→verdict
- [x] IT-003, IT-006, IT-007 — plan atomicity (fail-after-each-mutation + stale claim), best eligibility in tx, one provenance row per path
- [x] IT-009, IT-010, IT-011, IT-012 — livelock regression (both actions), DoD retry flow, ratchet restore, ablation (no metric → boolean byte-compatible)
- [x] IT-022, IT-023 — coverage-matrix extension; two-parallel-gates determinism

## Success Criteria

- Every assigned test case implemented and passing
- `make gate` green (race-enabled); zero lint warnings; frozen files did not grow
- Stall/breaker/cap suites untouched and green (semantics preserved)
- Event coverage matrix fails on any new path missing its canonical event (proven by mutation)

## Completion Evidence

- Canonical Loop, GlobalDB, and daemon suites pass (2,964 tests); focused race coverage passes for
  history, multi-gate union, both successor surfaces, ratchet, rollback boundaries, observability,
  and real SQLite claim convergence.
- `make gate` escalated to the full monorepo verification and passed with current fingerprint
  `e75cdf81474c702cbca604330810d1d30c19de8c`; Go lint reports zero issues.
- Production files remain below 500 lines. Deslop review found no compatibility path, temporary
  marker, ignored error, or duplicated writer.

## Compozy Impact Audit

- **Native tools:** no impact; checked native tool IDs, descriptors, schema digests, risk flags,
  capability gates, and fallbacks. No native-tool contract changed in this task.
- **Extensibility and hooks:** durable generation/verdict events land internally; public hook payload
  surfacing remains assigned to Task 03. Existing patch clones and config lifecycle are unchanged.
- **Workspace data isolation:** history reads propagate `workspace_id`; fenced writes derive from the
  claimed workspace-owned loop run. Real-store rollback/stale-claim tests prove no alternate writer.
- **Official Compozy skill:** no impact; no public CLI path, tool ID, capability, hook contract, or
  config key changed. Public documentation remains assigned to Task 05.
