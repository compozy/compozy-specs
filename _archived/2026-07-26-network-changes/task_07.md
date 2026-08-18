---
status: pending
title: "QA Plan and Session Charters"
type: docs
complexity: high
---

# Task 7: QA Plan and Session Charters

## Overview

Plan the real-user QA cycle for opt-in network participation on the living `docs/qa/` tree: update journey flowcharts, mint/update `state.csv` scenario rows for every public surface touched by tasks 01–06, and author session charters covering local-default, coordinated-run flagship, invitation, live bounds, administration, and agent-manageability hot spots from the TechSpec invariants and ADRs.

<critical>
- ALWAYS READ _techspec.md, every ADR, and every per-task memory file before planning
- ALWAYS READ docs/qa/state.csv and existing journeys/charters before minting rows
- Do not execute QA in this task — planning and charter authorship only
- Flag, don't retest — untested rows ARE the next execution scope
</critical>

<requirements>
1. Activate `qa-report` with `qa-docs-path=docs/qa` (bootstrap the tree if absent).
2. Journey flowcharts in `docs/qa/journeys/` MUST cover solo-builder local path, autonomy-operator coordinated-run+invitation path, network-operator live path, and admin availability toggle.
3. Scenario rows in `docs/qa/state.csv` MUST express entry points for CLI verbs, HTTP/UDS, web routes, doc pages, automation triggers, extension confirmation, agent-operation paths, and `config.toml` keys touched by this program — not standalone unit cases.
4. Session charters in `docs/qa/charters/` MUST select targeted tier + one adjacent canary journey mapped from TechSpec invariants (esp. orchestration independence, no silent degrade, wake budgets, workspace isolation).
5. UI-bearing surfaces (invitation, empties, run conversation, settings) MUST be marked for Playwright/`browser-use` in execution notes.
6. Worktree isolation requirements MUST be stated for concurrent QA (unique `AGH_HOME`, ports, tmux socket).
7. Do not mint BUG-NNNN ids in this task.
8. Mailbox/spend-cap scenarios MUST NOT be planned (Non-Goals).
</requirements>

## Subtasks

- [ ] 7.1 Diff tasks 01–06 public surfaces against current `docs/qa/state.csv` and journeys.
- [ ] 7.2 Update journey flowcharts for local / coordinated / live / admin paths.
- [ ] 7.3 Mint or update scenario rows with correct entry_points and `untested` status.
- [ ] 7.4 Author cycle charters (targeted + canary) referencing ADRs/invariants.
- [ ] 7.5 Hand off execution brief to task_08 (labs, accounts, teardown).

## Implementation Details

Activate `qa-report`. Follow living QA tree conventions in `docs/qa/`. Map hot spots from `_techspec.md` Safety Invariants and ADR-001..011. UI E2E directive: execution will run `make test-e2e-runtime` AND `make test-e2e-web`.

Skills to activate: `qa-report`, `eng-worktree-isolation` (document requirements), `documentation-writer`.

### Relevant Files

- `docs/qa/state.csv`
- `docs/qa/journeys/`
- `docs/qa/charters/`
- `.compozy/tasks/network-changes/_techspec.md` invariants
- `.compozy/tasks/network-changes/_tests.md` E2E catalog (traceability only)

### Dependent Files

- task_08 consumes charters + state.csv rows produced here

### Related ADRs

- [ADR-002](adrs/adr-002.md) — flagship coordinated-run journey
- [ADR-004](adrs/adr-004.md) — one complete release (no partial-mode charters)

## Deliverables

- Updated journeys + charters + state.csv rows for this cycle
- Execution brief for task_08
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)** — none; planning artifact gate

## Tests

No UT/IT/E2E implementation IDs — planning only.

- [ ] Every public surface from tasks 01–06 appears as a scenario entry_point or explicit out-of-scope rationale
- [ ] Charters name teardown via bootstrap `TEARDOWN_COMMAND` / `make qa-reap` (L-029)

## Success Criteria

- Journeys/charters/state.csv ready for execution
- No mailbox/spend-cap scope creep
- Isolation + teardown requirements documented

### Web/Docs Impact

- not applicable — editorial/QA planning only (no runtime behavior change)

### Extensibility / Agent Manageability / Config Lifecycle

- Planning coverage MUST include extension confirmation, CLI/HTTP/UDS parity, and config availability/live bounds keys as scenario entry points.
