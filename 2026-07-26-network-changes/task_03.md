---
status: pending
title: "Autonomy decoupling & network_wake substrate"
type: backend
complexity: critical
---

# Task 3: Autonomy decoupling & network_wake substrate

## Overview

Make kanban orchestration network-independent and generalize `task_runs` so durable network wakes can ride ClaimNextRun. Delete coordinator channel bootstrap gates, scheduler run-channel eligibility matching, task-role fictional default channels, conversational status observer say-paths (replace with typed projection), and `task.Service.ClaimRun` — converging manual exact claims on existing `ClaimCriteria.RunID` through ClaimNextRun with kind-fenced `network_wake` semantics (ADR-011).

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Coordinator MUST bootstrap and operate for Local runs with no channel anywhere; allowlist MUST contain task tools without the network trio unless participation is live; overlay MUST omit channel guidance for Local.
2. Scheduler MUST delete run-channel eligibility match (`coordinationChannelMatches`); wake/claim selection for ordinary task runs MUST not require session channel equality.
3. Typed task-status projection MUST replace conversational `network_task_status_observer` say path (C-06): real task transitions produce typed projection with zero PromptNetwork calls and zero conversational messages.
4. Task-role / starvation session identity MUST bind the owning run's snapshot (projection) without inventing a fictional `default` channel.
5. `task.Service.ClaimRun` and every API/CLI/native-tool caller MUST be deleted; manual exact claims MUST use `ClaimCriteria.RunID` via `ClaimNextRun` with identical token/lease/hooks/events (B-001/B-008).
6. `task_runs` MUST support `run_kind='network_wake'` with nullable `task_id` CHECKs, correlation columns (`network_wake_id`, `network_target_session_id`, `network_owner_key`), and kind-fenced claim/heartbeat/complete/fail/recovery that never touch `tasks.current_run_id` or task status for wakes (B-009/B-017).
7. Network failure mid-participating-run MUST leave task orchestration able to complete; conversation marked unavailable truthfully (IT-015).
8. Synthetic wake enqueue/execution wiring is owned by task_04; this task lands substrate + claim path only so task_04 can Admit into `task_runs`.
9. No parallel wake queue or network-owned claimer may be introduced (L-003/L-005/ADR-011).
10. List/read projections MUST filter/badge wake runs truthfully without breaking task-anchored kind behavior.
</requirements>

## Subtasks

- [ ] 3.1 Remove `DecisionMissingChannel` gate and unconditional network tools/guidance from coordinator bootstrap/overlay/session create.
- [ ] 3.2 Delete scheduler coordination-channel match; update channel suites to Local-capable expectations.
- [ ] 3.3 Replace conversational status observer with typed projection; assert zero PromptNetwork via spy.
- [ ] 3.4 Fix task-role session identity to bind run snapshot without fictional channels.
- [ ] 3.5 Delete ClaimRun surface; route exact claims through ClaimNextRun + RunID criterion across API/CLI/native tools.
- [ ] 3.6 Kind-fence claim/lease/recovery paths for `network_wake` (nullable task_id, skip task reconciliation).
- [ ] 3.7 Land assigned UT/IT cases in coordinator/scheduler/task/daemon suites.

## Implementation Details

See TechSpec Core Interfaces (`ClaimCriteria`, wake generalization), Build Order step 4, Delete Targets for coordinator/scheduler/ClaimRun/task-anchored assumptions, ADR-011, invariants 17–19.

Skills to activate: `eng-code-guidelines`, `golang-pro`, `eng-test-conventions`, `testing-boss`, `eng-consolidate-test-suites`, `systematic-debugging`, `no-workarounds`.

### Relevant Files

- `internal/coordinator/coordinator.go` — bootstrap gate + allowlist + overlay
- `internal/daemon/coordinator_runtime.go` — channel-bound session create
- `internal/daemon/task_role_sessions.go` — fictional default channel
- `internal/daemon/network_task_status_observer.go` — conversational say path (delete/replace)
- `internal/scheduler/scheduler.go` — `coordinationChannelMatches`
- `internal/task/manager_run_claim.go` — `ClaimRun` delete target
- `internal/store/globaldb/global_db_task_claim*.go`, `global_db_task_claim_select.go` — kind fencing
- `internal/task/lease_manager.go` — task reconciliation branches
- `internal/api/core/tasks.go` — ClaimRun handler caller

### Dependent Files

- `internal/network/` delivery/admission (task_04) will enqueue `network_wake` runs into this substrate
- CLI/native-tool claim descriptors regenerate fully in task_05 if contract fields move; behavior change must compile now
- Web run lists may show wake kinds — badge copy/filter completes with task_05 if exposed

### Related ADRs

- [ADR-002](adrs/adr-002.md) — orchestration independence from network
- [ADR-011](adrs/adr-011.md) — durable wakes ride task_runs through ClaimNextRun

## Deliverables

- Local-capable coordinator/scheduler/task-role paths
- Typed status projection (C-06)
- ClaimRun deleted; exact claims via ClaimNextRun
- Kind-fenced network_wake substrate ready for admission
- Every assigned test case implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [ ] UT-061, UT-062 — exact RunID claim via ClaimNextRun
- [ ] IT-013 — typed status projection zero PromptNetwork
- [ ] IT-014 — coordinator bootstraps Local run
- [ ] IT-015 — network failure mid-run; orchestration completes
- [ ] IT-036 — manual exact-run claim journey across public surfaces

## Success Criteria

- Every assigned test case implemented and passing
- Local coordinated task run completes without any network channel row
- No `ClaimRun` symbol remains in production surfaces
- Wake kind CHECKs enforced; task-anchored claims unchanged in behavior

### Web/Docs Impact

- `web/`: none required — checked: run detail claim UX if any must call ClaimNextRun-equivalent API; no invitation/conversation UI in this task.
- `packages/site`: autonomy pages still claim "bind always" until task_06 — do not leave new CLI help text teaching ClaimRun.
- QA impact: flag claim/orchestration scenarios touching exact-run claim as `untested`.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none new — checked: hooks still dispatch at call sites; wake events arrive in task_04.
- Agent manageability: CLI/API/native-tool exact claim MUST converge on ClaimNextRun; structured errors unchanged aside from deleted ClaimRun path.
- Config lifecycle: none — checked: no new keys.
