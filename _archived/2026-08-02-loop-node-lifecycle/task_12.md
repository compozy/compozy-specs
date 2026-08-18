---
status: completed
title: QA Plan and Session Charters
type: qa-report
complexity: high
---

# Task 12: QA Plan and Session Charters

## Overview

Plans the real-user QA cycle for the node lifecycle & failure contract over the committed
`docs/qa/` living tree: maps the new journeys, derives content-addressed scenario files for every
user-visible behavior tasks 02–11 flagged, resets superseded scenarios (`loop stop` semantics),
registers charters for persona-driven sessions, and leaves task_13 an executable plan.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- R1: MUST activate the `qa-report` skill and operate on the living `docs/qa/` tree (scenario
  files, journeys, charters, content-addressed bug registry, dated reports) — never a per-round
  reset tree.
- R2: MUST create `untested` content-addressed scenario files for every QA-impact flag recorded
  in tasks 02–11 (retry-heals, route-fallback, escalation, notifications ×2, sick-target,
  quarantine-requeue, death-resume, cancel-vs-kill, days-long-node, pause-repair,
  approval-link, durable-wait-restart, inventory-escalation, duplicate-suppressed,
  agent-native-tools, operator-lifecycle-ui, **editor-authoring-walk**,
  **catalog-runform-walk**) and reset every scenario touching `loop stop` to `untested`;
  dedup same-behavior adds content-addressed, never counter-coordinated.
- R3: MUST map journeys as flows (author → run → fail → repair → finish; approver; managing
  agent) and write session charters per persona (loop author, operator, approver, managing
  agent) naming surfaces (CLI + Web + API) and evidence expectations.
- R4: MUST plan the isolated lab per `eng-qa-bootstrap` + `eng-worktree-isolation` (unique
  `COMPOZY_HOME`, daemon port, tmux-bridge socket — L-009) and record the planned
  `bootstrap-manifest.json` expectations + teardown contract (L-029) for task_13.
- R5: Charters MUST include the UI-bearing browser plan (Playwright via `browser-use:browser`,
  fallback `agent-browser`) and the CLI/API cross-surface comparison plan
  (`make test-e2e-runtime` + structured CLI vs HTTP state).
</requirements>

## Subtasks

- [x] 12.1 Inventory QA-impact flags from tasks 02–11 completion notes + scenario dedup pass
- [x] 12.2 Author/reset content-addressed scenario files with acceptance walks
- [x] 12.3 Journey maps + persona charters (4 personas × surfaces × evidence)
- [x] 12.4 Lab/bootstrap/teardown plan (isolation, ports, manifest, reap)
- [x] 12.5 Dated plan report under `docs/qa/reports/` linking everything for task_13

## Implementation Details

`docs/qa/` is the durable home; `state.csv` is a generated view. Use scenario front-matter and
ids per the `qa-report` skill contract.

### Relevant Files

- `docs/qa/` — living tree (scenarios/, journeys/, charters/, bugs/, reports/)
- `.compozy/tasks/loop-node-lifecycle/_user_stories.md` — behavior source for walks
- Tasks 02–11 `## QA impact` sections — the flag inventory

### Dependent Files

- `task_13.md` — executes this plan

### Related ADRs

- PRD ADRs 002–009 define the behaviors the scenarios assert; ADR-018 covers editor + hero
  path web walks.

## Web/Docs Impact

None — QA planning artifacts only.

## Extensibility / Agent Manageability / Config Lifecycle

None — planning only (checked: no runtime surfaces touched).

## QA impact

This task IS the flag→scenario materialization step.

## Skills

`qa-report`, `eng-qa-bootstrap` (plan), `eng-worktree-isolation` (plan).

## References

- SD-005 (real-scenario QA posture), L-009 (isolation), L-029 (teardown).

## Deliverables

- Scenario files (new `untested` + resets) committed under `docs/qa/scenarios/`
- Journeys + 4 charters + dated plan report committed
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED — see note)**

## Tests

No `_tests.md` IDs — planning task. Its verification is structural: every task 02–11 QA flag has
exactly one scenario file, every scenario has an executable walk, charters cover all four
personas (including editor authoring + hero path browser evidence), and the plan report links
them (audited in task_13's preflight).

## Success Criteria

- Zero unmaterialized QA-impact flags from tasks 02–11
- Every scenario id content-addressed; no counter collisions; superseded `loop stop` scenarios reset
- task_13 can execute without planning decisions
