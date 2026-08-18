---
status: completed
title: Target health breaker, quarantine and requeue
type: backend
complexity: high
---

# Task 4: Target health breaker, quarantine and requeue

## Overview

Isolates sick dependencies and replaces run death with repair-and-continue: `internal/deadentity`
gains the `loop_target` kind (global `[loops.breaker]` policy, transport-only counting, fail-fast
`target_unavailable`, half-open probes), and the same-node consecutive-failure arithmetic is
redirected from the run-terminal stall into node quarantine with a fully diagnosable entry and a
`requeue` succession path.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- R1: MUST add `DeadEntityKindLoopTarget` (`EntityID = "<family>:<target>"`) and wire
  `BeforeProbe`/`RecordFailure`/`RecordSuccess` at node dispatch via the `TargetHealth` seam
  (`store.DeadEntityKey`); only `transport`-class failures count; handled/semantic failures
  record success (ADR-014; PRD Rule 31). The kind-CHECK rebuild ships in task_01's migration.
- R2: Policy is daemon-global `[loops.breaker]` applied to a SEPARATELY constructed loop-target
  `deadentity.Service` instance sharing the store/event sink — bridge/MCP/sidecar breaker
  thresholds and probe cadence provably unaffected (regression required; round-3 B-003). MUST
  NOT resolve per node/loop/run (round-2 B-003); durability contract asserted: open/half-open
  marks durable, pre-threshold streaks reset on restart (round-2 B-002).
- R3: Open target MUST fail bound attempts fast with class/reason `target_unavailable` entering
  the normal precedence chain — never the run-terminal `circuit_breaker` code (Safety
  Invariant 13); pending retries against an open target end early.
- R4: MUST redirect the same-node-2-consecutive-generations detection to quarantine: control row
  + entry assembled from the attempt ledger (classified chain, timestamps, hints, target, input
  ref — sanitized), run continues on independent branches; the uncapped-watch-loop arm keeps its
  run terminal (ADR-014.3).
- R5: MUST implement `requeue` semantics at the store/succession level: clears quarantine with
  provenance, plans an `OriginRequeue` generation through the succession planner under all
  bounds; repeat episodes append history; rerun sets exclude quarantined producers and a
  required-but-quarantined producer parks the run needs-attention naming the dependency (Safety
  Invariant 14; the public verb surface ships in task_07).
- R6: This task registers + emits its event kinds (`node_quarantined`, `node_requeued`,
  `target_breaker_transition`) in `loopRunEventKindValid`, riding their owning transactions;
  `on_quarantine` effects execute via task_03's relay (dependency edge task_03 → task_04;
  round-3 B-007). Quarantined-state exclusion consumes the single parked-state classifier owned
  by task_02.
</requirements>

## Subtasks

- [x] 4.1 `DeadEntityKindLoopTarget` + kind validation + transition-event emission
- [x] 4.2 `TargetHealth` seam + admission probe at dispatch + failure/success recording from classification
- [x] 4.3 Separate loop-target deadentity instance wired at daemon construction with `[loops.breaker]` policy + isolation regression vs shared instance
- [x] 4.4 Fail-fast `target_unavailable` path into the precedence chain + retry short-circuit
- [x] 4.5 Quarantine redirect (control row, entry assembly, run continues) + watch-arm preservation
- [x] 4.6 Requeue succession path (`OriginRequeue`) + needs-attention for required producers
- [x] 4.7 Restart-behavior suite (streak reset documented; open/half-open durable) + E2E journeys registered for public-surface verification

## Implementation Details

Follow TechSpec "Target health" component, Safety Invariants 13–14, ADR-014. New files:
`target_health.go`, quarantine planning inside `coordinator_lifecycle.go` (task_02 file, grown
within cap) or a new `coordinator_quarantine.go`.

### Relevant Files

- `internal/deadentity/service.go:74-215` — BeforeProbe/RecordFailure/RecordSuccess semantics
- `internal/deadentity/contract.go` — thresholds/options; `internal/store/types_dead_entity.go:12-28` — key/kinds
- `internal/loop/coordinator_terminal_helpers.go:83-140` — same-node arithmetic to redirect
- `internal/loop/coordinator_succession.go:139` — rerun-root computation (exclusion seam)
- `internal/loop/generation_intent.go:10-20` — `OriginRequeue` addition (CHECK ships task_01)
- `internal/tools/mcp_reliability.go:160-187` — consumer precedent + failure-class bridge
- `internal/task/block_types.go:39-43` — `NeedsAttention` vocabulary mirror

### Dependent Files

- `internal/loop/coordinator_generation_reattempt.go` — rerun exclusion of quarantined nodes
- `internal/daemon/boot_dead_entity.go` — service construction with `[loops.breaker]` policy

### Related ADRs

- [ADR-014](adrs/adr-014.md) — breaker reuse + quarantine redirect · PRD [ADR-007](adrs/adr-007.md) — target-health isolation

## Web/Docs Impact

Backend-only here; quarantine entries/inventories surface via task_07 payloads, task_08 UI,
task_11 docs.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: deadentity transition events observable via the existing event store; no
  manifest/hook changes (checked).
- Agent manageability: requeue verb + inventories ship in task_07; this task guarantees durable
  rows + succession semantics.
- Config lifecycle: consumes `[loops.breaker]` from task_01; no new keys.

## QA impact

Flag new `untested` scenarios: sick-target-degrades-one-lane (US-023),
quarantine-diagnose-requeue (US-024).

## Skills

`eng-code-guidelines`, `golang-pro`, `eng-test-conventions`, `testing-boss`,
`eng-consolidate-test-suites`.

## References

- `.resources/temporal/docs/architecture/circuit-breaker.md` — transport-only counting rationale
- `.resources/temporal/service/history/queues/executable.go:367-621` — DLQ routing + the opaque-entry anti-lesson
- `.compozy/tasks/loop-ideas/analysis/behavior-defaults.md` §Temporal DLQ/breaker

## Deliverables

- Per-target breaker live with global policy; quarantine + requeue succession operational
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] UT-044 — rerun set requiring quarantined/rule-paused producer → needs-attention
- [x] UT-133..UT-136 — open/fail-fast/probe/transport-only accounting
- [x] UT-137..UT-145 — quarantine trigger, entry completeness, requeue, watch-arm, sanitization
- [x] IT-017 — quarantine → continue → requeue → generation-count invariant
- [x] IT-022 — shared-state breaker under global policy + restart behavior
- [x] E2E-011 — sick target degrades one lane; E2E-012 — backend journeys and QA scenarios registered; public CLI walks are `blocked-verify` until Task 07

## Success Criteria

- Every assigned test case implemented and passing
- One open target never stops healthy lanes; a quarantine entry is diagnosable from its row alone
- `COUNT(loop_generations) == loop_runs.generation` holds across requeue episodes
