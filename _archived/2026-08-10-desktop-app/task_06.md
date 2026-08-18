---
status: completed
title: "qa-report: plan the desktop QA cycle in docs/qa"
type: qa-report
complexity: high
---

# Task 6: qa-report: plan the desktop QA cycle in docs/qa

## Overview

Plans the desktop QA cycle as living repo docs in the committed `docs/qa/` tree: content-addressed scenarios for every user-visible behavior this workstream shipped, desktop journeys, per-platform session charters (including the macOS no-WebDriver smoke shape), and bug-registry readiness. Planning only — no session execution, no browser/app runs.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST operate on the living `docs/qa/` tree per the `qa-report` skill contract (scenarios as content-addressed files, journeys, charters, global bug registry; `state.csv` is generated, never hand-edited).
2. MUST create `untested` content-addressed scenario files — one per shipped user-visible behavior — each mapped to its PRD story + `_tests.md` IDs with per-OS evidence requirements (N-004): install/first-run provisioning, attach-to-running, start-installed, app auto-update, runtime update (app-owned and managed branches), update recovery state, deep link (running + cold start), single-instance focus, quit contract, agent CLI journey (`compozy app`), brand-consistency spot-check.
3. MUST define desktop journeys and 3-platform charters covering the E2E catalog assigned to task_07 (E2E-001–018, 020–025), including: the SSE-load release gate (E2E-021) methodology, the update rehearsal with forced failure (app intact, perms ≠ `0700`), and the macOS platform-smoke charter (no WebDriver — scripted-manual with recorded evidence).
4. Charters MUST mandate isolated labs (`eng-qa-bootstrap` manifest, unique `COMPOZY_HOME`, no default ports) and clean teardown evidence (L-029) for execution in task_07.
5. Dedup rule: same-behavior scenarios reuse existing ids; no shared-counter coordination.
</requirements>

## Subtasks

- [x] 6.1 Inventory shipped behaviors from tasks 01–05 completion notes + TechSpec surfaces; map to US/test IDs
- [x] 6.2 Author the content-addressed scenario files (`untested`) with per-OS evidence requirements
- [x] 6.3 Author desktop journeys (newcomer, CLI power user, update moment, link-driven, agent-headless)
- [x] 6.4 Author per-platform charters incl. SSE-gate methodology + update-rehearsal + macOS smoke shape
- [x] 6.5 Wire the tracker views (generated `state.csv`) and validate the tree against the qa-report contract

## Implementation Details

Follow the `qa-report` skill + `docs/qa/` existing structure (remote-gateway cycle is the precedent — see `docs/qa/` journeys/charters from that workstream). Evidence requirements reference the E2E case definitions in `_tests.md` verbatim; charters bind to the bootstrap-manifest teardown contract.

### Relevant Files

- `docs/qa/scenarios/`, `docs/qa/journeys/`, `docs/qa/charters/`, `docs/qa/bugs/` — the living tree
- `.compozy/tasks/desktop-app/_tests.md` — E2E definitions the charters operationalize
- `docs/qa/` remote-gateway artifacts — structural precedent

### Related ADRs

- [ADR-002](adrs/adr-002.md), [ADR-003](adrs/adr-003.md), [ADR-011](adrs/adr-011.md) — the contracts the journeys validate

## Deliverables

- Complete planned QA cycle in `docs/qa/`: scenarios (`untested`), journeys, charters, tracker materialized
- No sessions executed; no lab spawned

## Tests

None assigned — planning task; the E2E catalog executes in task_07.

## Web/Docs Impact

`docs/qa/` tree only. **QA impact:** this task IS the planning half of the QA contract.

## Extensibility / Agent Manageability / Config Lifecycle

None — checked: planning artifacts only; no runtime surface.

## References

- `qa-report` skill contract; `docs/qa/` remote-gateway precedent; TechSpec §Web/Docs Impact (QA tracker list)

## Success Criteria

- Every shipped user-visible behavior has exactly one scenario file mapped to US + test IDs with per-OS evidence requirements
- Charters cover the full task_07 E2E catalog incl. the SSE release gate; tracker materializes cleanly
