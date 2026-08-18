---
status: completed
title: "QA Plan and Session Charters"
type: qa-report
complexity: high
---

# Task 5: QA Plan and Session Charters

## Overview

Plans the real-user QA cycle for the bundles-removal program over the living `docs/qa/` tree: journeys and scenario rows for the extension-only kit truth (enable/secrets/confirm/inventory/preview), the removal surfaces (no `compozy bundle`, no `/api/bundles`, marketplace 3-kind), and the homonym keep-set — expressed as journey-derived scenarios and persona charters for task_06 to execute.

<critical>ALWAYS READ _techspec.md, every ADR, and every per-task memory file before planning.</critical>

<requirements>
- MUST activate the `qa-report` skill with `qa-docs-path=docs/qa` (tree exists — extend, never fork a per-round tree).
- MUST update journey flowcharts in `docs/qa/journeys/` (marketplace/extension journeys lose bundle legs; a kit-lifecycle journey covers install → secrets → enable+confirm → inventory → automation → disable), mint/update scenario files in `docs/qa/scenarios/` (including verifying the six `untested` scenarios task_04 added, and that deletions/strips landed coherently), and write this cycle's session charters in `docs/qa/charters/`.
- MUST cover every public surface touched by tasks 01–04 as scenario `entry_points`: new CLI verbs (`extension inventory|preview|secrets *`, confirm flags), new HTTP/UDS routes, native tools (±), removed surfaces (negative checks), web marketplace/extension detail, docs pages, and the homonym keep-set (`compozy support bundle`, `--source bundled`) — not as standalone test cases.
- MUST map regression hot spots from `_techspec.md` Safety Invariants (1–17) and ADR-001..008 into the charter selection: targeted tier (kit lifecycle, secrets isolation, consent gates, hard-cut negatives) + one adjacent canary journey (marketplace/skills acquisition).
- MUST plan worktree isolation parameters for execution (unique `COMPOZY_HOME`, ports, tmux socket) per `eng-worktree-isolation`.
</requirements>

## Subtasks

- [x] 5.1 Journey updates (bundle legs removed; kit-lifecycle journey added)
- [x] 5.2 Scenario rows reconciled (new untested set verified; strips/deletions coherent; negative removal checks minted content-addressed)
- [x] 5.3 Charters for the cycle (targeted tier + canary) with personas and evidence contracts
- [x] 5.4 Isolation/bootstrap parameters recorded for task_06

## Implementation Details

Operate exclusively on the committed `docs/qa/` tree; `state.csv` is generated output. The scenario inventory shipped by task_04 is the input state — this task plans the cycle, it does not retest.

### Relevant Files

- `docs/qa/{journeys,scenarios,charters,bugs,reports}/**` — the living tree
- `.compozy/tasks/bundles-removal/{_techspec.md,adrs/*,_tests.md}` — hot-spot sources

### Related ADRs

- All of [ADR-001..008](adrs/adr-001.md) — charter hot-spot mapping.

## Deliverables

- Updated journeys, reconciled scenario set, and this cycle's charters committed under `docs/qa/`
- Charter selection mapped to invariants/ADRs with an explicit targeted+canary split

## Tests

- No `_tests.md` IDs (planning task). Gate: the plan passes `qa-report` skill validation (journey-derived scenarios, content-addressed ids, no per-round tree).

## Success Criteria

- Every touched public surface appears as an `entry_point` on some scenario row
- Charters are executable as-is by task_06 (personas, environments, isolation parameters, evidence contracts)
