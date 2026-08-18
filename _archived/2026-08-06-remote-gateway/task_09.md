---
status: completed
title: QA execution for remote gateway
type: qa-execution
complexity: critical
---

# Task 9: QA execution for remote gateway

## Overview

Walks the Remote Gateway the way a real operator would: an isolated lab, a persona driving the planned journeys through the product's own interfaces, edge probing per the charters, and a recorded verdict with evidence for every scenario this feature added or reset. A remote-access feature is exactly the class where unit and integration coverage passes while the lived experience breaks, so this walk is the release gate.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- The lab MUST be freshly bootstrapped with a unique home directory, unique daemon and tier ports, and a unique multiplexer socket; the default home and port are forbidden.
- The web proxy target MUST be derived from the bootstrap manifest and MUST NOT be hardcoded.
- Configuration writes against a single isolated home MUST run sequentially, never in parallel.
- Long-lived lab processes MUST be registered by PID on spawn so teardown can reap them.
- Every scenario this feature added or reset MUST be walked to a recorded verdict with evidence; a scenario left untested or failing at completion is a blocking failure.
- A failing walk MUST be fixed in production code and re-walked until it passes; the scenario MUST NOT be weakened to make it pass.
- Defects MUST be filed into the durable registry with reproduction steps and evidence, using content-addressed identities.
- Provider-dependent journeys MUST use the operator-authorized account path the product actually ships; a stubbed provider is not acceptable evidence for a reachability journey.
- Teardown MUST run on every terminal path — pass, fail, blocked, or abort — and MUST be evidenced by a clean teardown record. Completing this task with lab daemons, multiplexer servers, dev servers, browsers, or watchers still alive is a blocking failure.
- The dated report MUST record what was walked, what was found, and what remains blocked, with paths to the evidence.
</requirements>

## Subtasks

- [x] 9.1 Bootstrap an isolated lab with unique home, ports, and socket, and record its manifest
- [x] 9.2 Register every long-lived lab process by PID on spawn
- [x] 9.3 Walk the overlay-and-pairing journey from a second device through the product's own interfaces
- [x] 9.4 Walk the public-ingress journey end to end, wiring a real signed delivery to a Loop
- [x] 9.5 Walk the remote CLI journey, including the local-only refusal and reconnect behavior
- [x] 9.6 Walk the SSH connect journey, including reuse and scoped teardown
- [x] 9.7 Walk the audit-and-teardown journey, clearing a finding by its own remediation
- [x] 9.8 Execute the session charters probing revocation during live work, mid-delivery exposure change, provider degradation, and no-paired-device recovery
- [x] 9.9 Record a verdict with evidence for every scenario added or reset by this feature
- [x] 9.10 File defects into the durable registry and re-walk after each production fix
- [x] 9.11 Write the dated report and execute teardown, citing the clean teardown record

## Implementation Details

Follow the QA Lifecycle section of the TechSpec and the QA execution contract in the committed QA tree. The plan authored in task 08 is the input; this task produces verdicts, evidence, defects, and the dated report.

### Relevant Files

- `docs/qa/scenarios/` — the entries whose verdicts this task records
- `docs/qa/journeys/`, `docs/qa/charters/` — the plan this task executes
- `docs/qa/bugs/` — the durable registry for defects found during the walk
- `docs/qa/reports/` — the dated report this task writes
- `.compozy/tasks/remote-gateway/_prd.md` — the expected experience journeys are judged against
- `.compozy/tasks/remote-gateway/_techspec.md` — the QA Lifecycle requirements for isolation and teardown

### Dependent Files

- Production source across `internal/gateway`, `internal/api/*`, `internal/cli`, and `web/` — defects found here are fixed at their root cause and re-walked

### Related ADRs

- [ADR-006: Fail-closed exposure with agent-operable self-audit](adrs/adr-006.md) — the posture guarantees the walk must confirm
- [ADR-005: Remote client methods and the bounded operation matrix](adrs/adr-005.md) — the matrix the CLI journey validates
- [ADR-001: First-party managed connectivity](adrs/adr-001.md) — the operator-account path provider journeys must use

## Deliverables

- Isolated lab bootstrapped from a fresh manifest with registered process PIDs
- Recorded verdict with evidence for every scenario this feature added or reset
- Executed charters with findings for each probed edge
- Defects filed into the durable registry with reproduction steps, each re-walked after its production fix
- Dated report naming what was walked, what was found, and what remains blocked
- Clean teardown record proving no lab process survived the run
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] E2E-007 — the QA pair runs in an isolated lab with a derived web proxy target, walks every scenario added or reset by this feature to a recorded verdict, and ends with a clean teardown record

## Success Criteria

- Every assigned test case implemented and passing
- No scenario added or reset by this feature remains untested or failing at completion
- Every defect found is filed with reproduction steps and re-walked green after its production fix
- The dated report cites evidence paths for every verdict
- Teardown is evidenced as clean on the terminal path actually taken
- No lab daemon, multiplexer server, dev server, browser, or watcher survives the run
