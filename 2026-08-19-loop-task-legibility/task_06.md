---
status: pending
title: QA Plan and Session Charters
type: qa-report
complexity: high
---

# Task 6: QA Plan and Session Charters

## Overview

Plans the verification cycle for the loop/task legibility program on the repo's living QA tree (`docs/qa/`): journeys updated, scenario files minted/reset (content-addressed), and session charters selected for this cycle — covering every public surface tasks 01-05 shipped.

<critical>ALWAYS READ `_spec.md`, every ADR, and every per-task memory file before planning.</critical>

<requirements>
1. MUST activate the `qa-report` skill with `qa-docs-path=docs/qa` (bootstrap the tree if absent).
2. MUST express coverage as scenario `entry_points` on journey-derived rows, not standalone test cases, spanning every public surface touched by tasks 01-05:
   - CLI verbs: `compozy task list --include-loop --loop-run`, `compozy loop why`, `compozy loop nodes --run --all`, `compozy loop events --after --follow --view`, `compozy loop runs`, `compozy task timeline` (audit reasons), `compozy config set loops.reconcile_interval`.
   - HTTP/UDS routes: `GET /api/tasks` (+`include_loop`/`loop_run_id`), `GET /api/tasks/:id` (`loop` provenance), `GET /api/workspaces/:ws/loop-runs` (extended items), `…/loop-runs/:id/{nodes,briefing,timeline}`.
   - Native tool: `compozy__task_list` typed args.
   - Web routes: `/tasks` (default/reveal/kanban/detail), `/loop-runs` (re-ranked roster), `/loop-runs/$runId` (two registers, DAG, roster, history).
   - Config key: `loops.reconcile_interval` (default/validation/restart semantics).
   - Docs/skill: generated CLI/API references + `skills/compozy/` task-listing and loops sections.
3. MUST reconcile the scenario ledger: absorb the `untested` scenarios flagged by tasks 01-05, retire `docs/qa/scenarios/TA-web-task-list-loop-subtask-nesting.md` (superseded by the exclusion contract), reset touched ids (`LP-web-detail-inventory-contract`, `LP-web-attention-loop-rows`, `LP-loop-run-deep-link` as applicable), and dedup same-behavior adds by content-addressed id.
4. MUST map regression hot spots from `_spec.md` Part II Safety Invariants (1-14) and ADR-001..006 into the cycle's charter selection: targeted tier = settlement/orphan repair, calm-default parity, timeline resume, two-register briefing test; plus one adjacent canary journey (existing loop authoring→run flow).
5. MUST verify `BUG-20260719-autonomous-progress-unobservable` (P1, re-found 6×) is addressed by the cycle's scenarios and schedule its closure verification.
</requirements>

## Subtasks

- [ ] 6.1 Update journey flowcharts in `docs/qa/journeys/` (supervisor steady-state, operator diagnosis, agent headless — per `_spec.md` User Experience)
- [ ] 6.2 Mint/update scenario files in `docs/qa/scenarios/` (content-addressed; entry_points per surface above)
- [ ] 6.3 Retire the superseded nesting scenario; reset touched scenario ids to `untested`
- [ ] 6.4 Write this cycle's session charters in `docs/qa/charters/` (targeted tier + canary journey)
- [ ] 6.5 Cross-check coverage against every task's QA impact line (no flagged scenario unowned)

## Implementation Details

Operates exclusively on the living `docs/qa/` tree (scenarios/journeys/charters/bugs/reports); `state.csv` is generated output only. Read every `task_NN.md` QA impact line and the walked history in `docs/qa/scenarios/` before minting new ids.

### Relevant Files

- `docs/qa/scenarios/` — scenario ledger (existing loop/tasks ids: `LP-*`, `TA-*`, `ET-web-tasks-mode-url`).
- `docs/qa/journeys/`, `docs/qa/charters/`, `docs/qa/bugs/` — cycle outputs + open registry (`BUG-20260719-autonomous-progress-unobservable.md`).
- `.compozy/tasks/loop-task-legibility/_spec.md` + `adrs/` + `_dx.md` + `_uiux.md` — coverage sources.
- `docs/design/opendesign/loop-legibility/` — landed visual contracts the web scenarios walk against (`loop-legibility-tasks-list.html`, `loop-legibility-run-default.html`, `loop-legibility-needs-you.html`, `loop-legibility-run-dag.html`, `loop-legibility-run-roster.html`, `loop-legibility-runs-roster.html`, plus `DESIGN-NOTES.md`).

### Related ADRs

- [ADR-001](adrs/adr-001.md) · [ADR-002](adrs/adr-002.md) · [ADR-003](adrs/adr-003.md) · [ADR-004](adrs/adr-004.md) · [ADR-005](adrs/adr-005.md) · [ADR-006](adrs/adr-006.md) — hot-spot map for charter selection.

## Deliverables

- Updated `docs/qa/journeys/` + minted/reset `docs/qa/scenarios/` + this cycle's `docs/qa/charters/`
- Supersession recorded (nesting scenario retired, replacements linked)
- Charter selection citing the invariant/ADR hot spots + one canary journey

## Tests

No `_tests.md` IDs are assigned to this task (all 99 belong to tasks 01-05). This task's verification objects are the QA-tree artifacts themselves — every surface in requirement 2 owned by at least one scenario row, every task's flagged scenario absorbed.

## Success Criteria

- Every public surface from tasks 01-05 appears as an `entry_point` on at least one scenario
- Zero orphan QA-impact flags (each task's line maps to a minted/reset scenario id)
- Charters name the safety-invariant hot spots and the canary journey
