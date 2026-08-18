---
status: pending
title: "Owner wiring — resolve-and-persist across session/task/loop/automation"
type: backend
complexity: high
---

# Task 2: Owner wiring — resolve-and-persist across session/task/loop/automation

## Overview

Wire the Resolver into every execution owner — session create, task profile/reservation, loop definition/run, automation dispatch — so participation resolves once, persists as an immutable Spec snapshot, and never inherits. Delete auto default-channel fill, task-run coordination channel synthesis, spawn/review/detached inheritance, and bundle effective-default enrollment. Land the workspace coordination settings service (ADR-009) consulted by coordinated-run resolution.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Session create MUST accept optional participation request (no `channel` field); blank intent MUST resolve Local/`built_in_local` with zero network artifacts; `defaultSessionChannel` auto-fill MUST be deleted.
2. Task execution profiles MUST carry typed participation request fields; reservation/start/enqueue/fan-out MUST resolve-and-persist Spec on the run in the same transaction; `coordinationChannelIDForQueuedRun` / `derivedRunCoordinationChannelID` / `ensureQueuedRunCoordinationChannel` auto-INSERT paths MUST be deleted (derivation survives only inside Resolver for explicit `run` strategy).
3. Loop definitions/runs MUST resolve-and-persist Spec on `loop_runs`; Local loops with network-using nodes MUST fail validate/dry-run with `loop_requires_live`; live loops MUST share one loop-run conversation (no per-node channels).
4. Automation job payloads MUST replace `JobTaskConfig.NetworkChannel` with typed participation request; webhook-injected participation MUST be ignored (job definition wins); duplicate fires MUST resolve identically.
5. Children (spawn/review/detached) MUST resolve independently default Local; no parent channel inheritance; authority denials MUST be typed with zero partial state.
6. Workspace coordination `Get`/`Set` MUST persist `workspace_network_coordination` with monotonic `revision` (B-014); Set while availability off → typed unavailable; task-profile intent overrides workspace for that task.
7. Coordinated next-run resolution MUST use source `workspace_coordination` when enabled and no nearer intent exists; in-flight runs keep their snapshot when the setting flips.
8. Bundle `BindPrimaryChannelAsDefault` / effective-default channel resolution MUST be deleted; plain session create after bundle activation MUST stay Local.
9. Compiler MUST enumerate and remove every deleted field consumer in the owner layers; no dual fields or fallback reads.
10. Hook payload flat channel fields on run context MUST be replaced by the resolved participation object at owning call sites (full hook events land with public surfaces, but owner writes must not reintroduce flat fields).
</requirements>

## Subtasks

- [ ] 2.1 Wire session CreateOpts + API/core create path to Resolver; delete `defaultSessionChannel` and `CreateOpts.Channel`.
- [ ] 2.2 Wire task profile + reservation/start/enqueue/fan-out; delete auto coordination channel synthesis and legacy run channel fields.
- [ ] 2.3 Wire loop definition validation + run persistence; enforce `loop_requires_live`.
- [ ] 2.4 Wire automation dispatch + job schema; reject payload injection.
- [ ] 2.5 Delete spawn/review/detached inheritance; enforce independent Local default + authority checks.
- [ ] 2.6 Implement workspace `CoordinationSettings` service + store; wire into coordinated-run ResolveInput.
- [ ] 2.7 Delete bundle effective-default enrollment; update owner-layer suites for new expectations.
- [ ] 2.8 Land assigned UT/IT cases in canonical session/task/loop/automation/store suites.

## Implementation Details

See TechSpec "Participation Ownership Matrix", "API Endpoints" (owner fields), Build Order step 3, Delete Targets for session/task/loop/automation/bundles, ADR-001/006/009.

Skills to activate: `eng-code-guidelines`, `golang-pro`, `eng-test-conventions`, `testing-boss`, `eng-consolidate-test-suites`, `no-workarounds`.

### Relevant Files

- `internal/api/core/handlers.go` — CreateSession
- `internal/api/core/bundles.go` — `defaultSessionChannel` (delete target)
- `internal/session/spawn.go` — channel inheritance
- `internal/session/manager_types.go`, `manager_helpers.go`, `manager_start_env.go` — CreateOpts/join/env
- `internal/store/globaldb/global_db_task_aux.go` — `ensureQueuedRunCoordinationChannel` (delete target)
- `internal/task/types.go`, `profile.go`, `manager_run_*.go` — profile/run channel fields
- `internal/daemon/loop_runtime_adapters.go` — unconditional loop channels
- `internal/automation/model/types.go`, `dispatch.go` — NetworkChannel forwarding
- `internal/bundles/service.go`, `resource_projection.go` — EffectiveDefaultChannel

### Dependent Files

- `internal/coordinator/`, `internal/scheduler/` — still channel-coupled until task_03
- Public OpenAPI/CLI shapes finalize in task_05; owner layers must use participation types now so codegen can swap cleanly
- Web session create still omits channel today — stays Local after this task without UI change until task_05

### Related ADRs

- [ADR-001](adrs/adr-001.md) — no implicit inheritance; smallest-unit scope
- [ADR-006](adrs/adr-006.md) — resolve once, immutable snapshot
- [ADR-009](adrs/adr-009.md) — workspace coordination record + execution profile

## Deliverables

- Resolve-and-persist on session/task_run/loop_run/automation paths
- Coordination settings service
- All owner-layer delete targets removed
- Every assigned test case implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [ ] UT-011, UT-012, UT-013, UT-014, UT-015, UT-016, UT-017, UT-018, UT-019, UT-020, UT-021 — resolver precedence/authority/no-inheritance through owner inputs
- [ ] UT-047, UT-048, UT-049, UT-050 — loop validation local/live
- [ ] UT-051, UT-052, UT-053, UT-054 — workspace coordination service
- [ ] IT-004, IT-005, IT-006 — task run local/retry/override
- [ ] IT-007, IT-008, IT-009, IT-010, IT-011, IT-012 — session lifecycle + coordination setting effects
- [ ] IT-017, IT-018, IT-019, IT-020, IT-021, IT-022 — loop + automation

## Success Criteria

- Every assigned test case implemented and passing
- Plain session/task/loop/automation create zero network_channels mutations
- Children never inherit parent participation
- Workspace coordination enable affects only subsequent coordinated runs

### Web/Docs Impact

- `web/`: none required in this task — checked: session create already omits channel (becomes truthful Local); task editor `network_channel` fields become invalid until task_05 replaces them — do not leave a broken submit path (gate or remove raw channel inputs if they would 400 against the new API).
- `packages/site`: none required here — autonomy/network doctrine rewrite is task_06.
- QA impact: reset/add `untested` rows for session/task/loop/automation local-default behaviors if any CLI/API already exposes the new fields early; otherwise flag in task_05/06.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: bundle default-channel deletion lands here; extension confirmation flow is task_05.
- Agent manageability: owner create paths must accept participation objects on HTTP/UDS once contract fields exist; CLI flag co-ship completes in task_05 — do not leave CLI sending deleted `channel` fields.
- Config lifecycle: none new — checked: coordination is workspace data (ADR-009), not a config key.
