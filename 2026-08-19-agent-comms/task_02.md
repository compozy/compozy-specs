---
status: pending
title: "Legacy pipeline adoption: one contract regime"
type: backend
complexity: high
---

# Task 2: Legacy pipeline adoption: one contract regime

## Overview

Reroute the five legacy structured-output pipelines through `internal/contracts` so one regime validates everything: loop capture/settle validation, ask/review `ValidateWaitPayload`, descriptor validation, and the task result path (the blanket 64 KiB ceiling dies). Collapse the three declared-schema resolvers into one and replace the `output_ref` sentinel/failure union with explicit typed result kinds. Behavior parity is the contract — the #438 regression suites are the guardrail, which is why this task carries few new test IDs: the invariants already have owning suites.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Loop capture AND settle validation MUST both flow through `internal/contracts` (single resolver, instrumented so tests can assert one path); the settle-time demotion to `invalid_output` on corrupted payloads (#438 behavior) is preserved byte-for-byte on the golden fixtures.
2. Ask/review `ValidateWaitPayload` MUST become a contracts adapter with byte-identical acceptance on golden fixtures (UT-020).
3. Descriptor validation MUST use the same contracts resolver; two of the three declared-schema resolvers (`linter_reference_schemas.go` and `service_amendments.go` variants) are DELETED — no fallback, no compat shim.
4. `task.MaxResultBytes` as a blanket result ceiling is DELETED; uncontracted task results ride the config-default budget through `contracts.ResolveBudget` (per-contract budgets arrive with task `expect` in task_05).
5. The `output_ref` sentinel/failure union is replaced by explicit result kinds; readers stop string-sniffing and the `outputValue` string-fallback is DELETED; control outcomes (branch-false, route-not-taken, absorbed-failure) keep their exact template-namespace values (IT-055).
6. Every existing loop/task/session suite MUST stay green — a parity break is a production bug in this task, never a test to weaken.
7. No schema changes in this task (migration work is task_03); no new public surface.
</requirements>

## Subtasks

- [ ] 2.1 Adopt contracts in loop capture validation and settle validation; wire the single-resolver instrumentation
- [ ] 2.2 Adopt contracts in ask/review `ValidateWaitPayload` via an adapter with golden-fixture parity
- [ ] 2.3 Adopt contracts in descriptor validation; delete the two redundant resolvers
- [ ] 2.4 Replace the blanket task result ceiling with config-default budgets through `ResolveBudget`
- [ ] 2.5 Replace `output_ref` sentinels with explicit typed result kinds; delete the `outputValue` fallback; migrate every reader
- [ ] 2.6 Sweep for dead code left by the collapse (unused helpers, stale imports) and delete it
- [ ] 2.7 Implement the assigned parity cases; run the full loop/task suites + #438 regression tests; close on `make gate`

## Implementation Details

All edits live in `internal/loop`, `internal/task`, `internal/session` (ask/review path), and `internal/store` readers of result kinds. `_spec.md` Part II › Impact Analysis lists the exact delete targets; Development Sequencing phase 3 is this task verbatim. The loop suites in `internal/loop` (including `action_channel_result_integration_test.go` and the settle-validation tests from #438) are the parity oracle — run them continuously.

### Relevant Files

- `internal/loop/action_schema.go:18-58,102-149,255-294,296-434` — validation/extraction/repair being rerouted through contracts
- `internal/loop/action_result_validation.go:12` + `internal/loop/coordinator_output_validation.go:11` — the #438 checkpoints being generalized
- `internal/task/lease_settlement_authority.go:7` — the settlement-actor fence precedent (context for the settle path)
- `internal/task/coordinator_control.go:11-98` + `internal/task/limits.go:16-17` — the result path + blanket ceiling being replaced
- `internal/session/session_wait.go:12-18,79-127` — the ask/review wait-payload acceptance path
- `internal/store/globaldb/global_db_loop_amendments.go:17-30,179-220` — amendment overlay reading result kinds

### Dependent Files

- `internal/loop/**` — every reader of `output_ref` sentinels and the two deleted resolvers
- `internal/task/**` — result-path budget call sites
- `internal/store/globaldb/**` — projections reading result kinds (compile-time migration to typed kinds)
- `internal/loop/service_integration_test.go` + loop behavior suites — must stay green unmodified in their assertions

### Related ADRs

- [ADR-006: One contract package unifies all five structured-output pipelines](adrs/adr-006.md) — the adoption mandate
- [ADR-012: Loop nodes adopt the contract regime without call records](adrs/adr-012.md) — loops keep task_runs; no call records for loop nodes

### Web/Docs Impact

- `web/`: none — checked surfaces: no API contract or generated-type change (result kinds are internal; public projections unchanged); reason: parity refactor.
- `packages/site`: none in this task — the "Loops" equivalence paragraph in the docs area ships with task_07; reason: docs describe the shipped unified regime once surfaces exist.
- QA impact: user-visible delta is task results between 64 KiB and the 256 KiB default now being accepted (the blanket ceiling is gone) — add a content-addressed `untested` scenario for the task-result size boundary in `docs/qa/scenarios/` (flag only; the walk runs in the loop's QA phase per task_09).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: hooks, Host API, tools/resources, registries, bridge SDKs; reason: internal validation rerouting with no extension-visible contract change.
- Agent manageability: none new — checked surfaces: CLI verbs, HTTP/UDS routes, structured outputs; reason: no surface change; error shapes on loop/task paths keep their existing codes.
- Config lifecycle: consumes `[calls].results.default_budget` (from task_01) as the uncontracted-task budget source; no new keys; `task.MaxResultBytes` const deleted (not a config key). Checked: no `config.toml` key references the deleted const.

## Deliverables

- All five pipelines validating through `internal/contracts`; exactly one declared-schema resolver remains
- `task.MaxResultBytes` blanket ceiling deleted; config-default budget in force
- `output_ref` sentinels replaced by typed result kinds; `outputValue` fallback deleted; zero string-sniffing readers left
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-017 — loop settle adapter parity fixture (identical verdict + `invalid_output` mapping)
- [ ] UT-018 — declared contract on a payload-less node class → lint error from the single resolver
- [ ] UT-019 — the ONE resolver: lint, review, and amendment callers get one schema (three-way parity)
- [ ] UT-020 — ask/review adapter byte-identical acceptance on golden fixtures
- [ ] UT-021 — explicit result kinds; sentinel strings no longer parse; `outputValue` fallback deleted
- [ ] IT-050 — loop run-agent capture + settle both flow through contracts (instrumented single-resolver assertion)
- [ ] IT-051 — stored-payload corruption between capture and settle → settle demotes to `invalid_output` (the #438 regression, now via contracts)
- [ ] IT-055 — control outcomes flow through typed result kinds end to end in the owning loop behavior suite

## Success Criteria

- Every assigned test case implemented and passing
- Full `internal/loop` + `internal/task` suites green, including every #438 regression test, with no assertion weakened
- Delete targets verifiably gone (grep-clean: deleted resolvers, `MaxResultBytes` ceiling usage, `outputValue` fallback)
- `make gate` green at close
