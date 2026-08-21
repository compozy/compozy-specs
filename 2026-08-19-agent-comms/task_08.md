---
status: pending
title: "QA Plan and Session Charters"
type: qa-report
complexity: high
---

# Task 8: QA Plan and Session Charters

## Overview

Turn the shipped agent-comms feature into the living QA contract: journey flowcharts, content-addressed scenario files for every public surface tasks 01-07 touched, and the session charters for this cycle — so task_09 executes against a real plan instead of ad-hoc poking.

<critical>ALWAYS READ _spec.md, every ADR, and every per-task memory file before planning.</critical>

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Activate the `qa-report` skill with `qa-docs-path=docs/qa` (bootstrap the tree if absent).
2. Output: journey flowcharts updated in `docs/qa/journeys/`, scenario files minted/updated in `docs/qa/scenarios/` (content-addressed ids; dedup same-behavior conflicts against the registry), session charters in `docs/qa/charters/` for this cycle.
3. Coverage: every public surface touched by tasks 01-07 — CLI verbs (`compozy call/message/agent list/session stop --subtree/task --expect/config calls.*`), HTTP/UDS `/calls` + `/messages` + session-stop `subtree` + task `expect`, the 7 native tools + `session_stop.subtree` (+ spawn-removal negative), web routes (`/agents/activity`, `/agents/inbox`, `/agents/calls/$callId`, catalog/detail, inspector Calls tab, attention), docs pages, extension points (`call` hooks, Host API reads), and `[calls]` config keys — expressed as scenario `entry_points` on journey-derived rows, not standalone test cases.
4. Fold in every scenario flagged by tasks 02, 05, and 06 (their `QA impact` lines) — reset/added files must appear in this cycle's charter selection.
5. Map regression hot spots from `_spec.md` Part II Safety Invariants (1-14, 3a-3e) and ADRs 001-013 into the charter selection: targeted tier (settlement races, delivery exactly-once, sanitize sweep, depth/caps, publish one-way, operator fence) + one adjacent canary journey (Network conversations or Loops, which share substrate).
</requirements>

## Subtasks

- [ ] 8.1 Read the full spec set, all 13 ADRs, and the per-task QA-impact flags
- [ ] 8.2 Update `docs/qa/journeys/` with the delegation/mailbox/subagents/operator journeys
- [ ] 8.3 Mint/update content-addressed scenario files covering every surface above (dedup against the registry)
- [ ] 8.4 Write this cycle's charters in `docs/qa/charters/` (targeted tier + canary)
- [ ] 8.5 Verify every flagged scenario from tasks 02/05/06 is represented and `untested`

## Implementation Details

Operates on the committed `docs/qa/` tree only (state.csv is a generated view). Scenario `entry_points` reference exact `_dx.md` invocations and `_uiux.md` routes.

### Relevant Files

- `docs/qa/` — the living QA tree (scenarios/journeys/charters/bugs/reports)
- `.compozy/tasks/agent-comms/{_spec.md,_dx.md,_uiux.md,_user_stories.md}` + `adrs/` — coverage sources

### Dependent Files

- `docs/qa/scenarios/*.md` — minted/reset rows this cycle
- `docs/qa/charters/*.md` — the charter set task_09 executes

### Related ADRs

- All of [ADR-001..ADR-013](../agent-comms/adrs/) — the regression hot-spot map for charter selection

## Deliverables

- Updated journeys, content-addressed scenarios for every touched surface, and this cycle's charters
- A coverage statement mapping tasks 01-07 surfaces → scenario ids (no orphan surface)

## Tests

No `_tests.md` IDs are assigned to this task; its deliverable IS the QA plan. Exit check: every surface listed in requirement 3 appears in at least one scenario's `entry_points`.

## Success Criteria

- `docs/qa/` tree updated and internally consistent (ids content-addressed, no duplicates)
- Every task-flagged scenario present and `untested`; charter selection covers the invariant hot spots + one canary
