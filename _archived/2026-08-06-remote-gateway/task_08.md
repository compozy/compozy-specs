---
status: completed
title: QA planning for remote gateway
type: qa-report
complexity: high
---

# Task 8: QA planning for remote gateway

## Overview

Plans the real-user QA cycle for the Remote Gateway as living repository documentation: the scenarios this feature adds or invalidates, the journeys a persona walks end to end, and the session charters that probe its edges. Planning is a separate task from execution because the plan is a durable artifact the project keeps, not a byproduct of one run.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- Planning MUST operate on the committed QA documentation tree; per-round throwaway trees are forbidden.
- Every user-visible behavior this feature adds MUST get a content-addressed scenario entry marked untested.
- Every user-visible behavior this feature changes MUST have its existing scenario reset to untested.
- Journeys MUST cover the operator-facing flows end to end: enabling the overlay and pairing a device, enabling public ingress and wiring a repository webhook to a Loop, operating a remote daemon from the CLI, connecting over SSH, and auditing and tearing down exposure.
- Session charters MUST probe the feature's edges rather than restate its happy paths — revocation during live work, exposure changes mid-delivery, provider degradation, and recovery with no paired device.
- Scenario identities MUST be content-addressed; same-behavior duplicates MUST be merged rather than coordinated through a shared counter.
- The plan MUST record the durability boundary (no store-and-forward) as expected behavior so QA does not file it as a defect.
- The plan MUST NOT execute sessions, drive a browser, or produce run evidence — that is the execution task.
</requirements>

## Subtasks

- [x] 8.1 Inventory the user-visible behavior this feature adds or changes across web, CLI, API, and configuration
- [x] 8.2 Add content-addressed scenario entries for new behavior, marked untested
- [x] 8.3 Reset existing scenarios invalidated by changed behavior
- [x] 8.4 Map the operator journeys as flows with their entry points and observable outcomes
- [x] 8.5 Write session charters targeting the feature's edges and failure modes
- [x] 8.6 Record expected-behavior notes for the documented durability and recovery boundaries
- [x] 8.7 Register any known limitation as a tracked entry rather than an open defect

## Implementation Details

Follow the QA documentation contract in the committed QA tree and the QA Lifecycle section of the TechSpec. This task authors plan artifacts only; the execution task consumes them.

### Relevant Files

- `docs/qa/scenarios/` — content-addressed scenario entries this task adds or resets
- `docs/qa/journeys/` — operator journeys mapped as flows
- `docs/qa/charters/` — session charters probing edges
- `docs/qa/bugs/` — the durable bug registry referenced by execution
- `.compozy/tasks/remote-gateway/_user_stories.md` — the behavior catalog scenarios derive from
- `.compozy/tasks/remote-gateway/_prd.md` — user experience and business rules that define expected outcomes

### Dependent Files

- `docs/qa/reports/` — the execution task writes its dated report here against this plan

### Related ADRs

- [ADR-006: Fail-closed exposure with agent-operable self-audit](adrs/adr-006.md) — posture expectations QA validates
- [ADR-004: Public tier surface set](adrs/adr-004.md) — what must and must not be publicly reachable

## Deliverables

- Content-addressed scenario entries for every new behavior, marked untested
- Reset entries for every scenario invalidated by changed behavior
- Operator journeys covering overlay pairing, webhook-to-Loop, remote CLI, SSH connect, and audit-and-teardown
- Session charters targeting revocation during live work, mid-delivery exposure change, provider degradation, and no-paired-device recovery
- Expected-behavior notes for the documented durability and recovery boundaries
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- No automated cases are assigned to this task: its deliverable is the QA plan itself, and the executable coverage for this feature is owned by tasks 01–07 and exercised by task 09.

## Success Criteria

- Every user-visible behavior added or changed by this feature has a scenario entry in the committed tree
- Every journey names its entry point and its observable outcome
- Every charter names the edge it probes rather than a happy path
- No scenario identity collides with an existing one, and same-behavior duplicates are merged
- The plan is executable by the QA execution task without further clarification
