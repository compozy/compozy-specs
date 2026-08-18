---
status: completed
title: QA Plan and Session Charters
type: qa-report
complexity: high
---

# Task 7: QA Plan and Session Charters

## Overview

Plans the real-user verification cycle for the whole feature on the living `docs/qa/` tree: journeys and scenario rows for every public surface tasks 01–06 shipped, session charters for the execution cycle, and the regression hot-spot map derived from the spec's invariants and ADRs — including the three-provider delivery matrix and the 8-item conformance walk that task_09's external claim will cite.

<critical>ALWAYS READ _spec.md, every ADR, and every per-task memory file before planning.</critical>

<requirements>
1. MUST activate the `qa-report` skill with `qa-docs-path=docs/qa` (bootstrap absent parts of the tree, never fork a per-round tree).
2. MUST cover every public surface touched by tasks 01–06 as scenario `entry_points` on journey-derived rows: CLI verbs (install/validate/status/inventory/list/update/remove/secrets), HTTP/UDS routes (extensions + secrets contracts), native tools, web routes (marketplace + settings extensions), doc pages, and the catalog feed — including the scenario files task_06 minted (adopt, don't duplicate).
3. MUST map regression hot spots from `_spec.md` Part II Safety Invariants 1–12 and ADRs 001–006 into the cycle's charter selection (targeted tier + one adjacent canary journey — existing native-extension install flow is the canary).
4. MUST charter the three-provider delivery matrix (Claude Code, OpenClaw, Hermes real sessions consuming one ingested package: skills activate, stdio server runs with the env contract, remote server reachable through the daemon) and the 8-item minimum-conformance walk as first-class scenarios — they are the evidence gate for task_09 (L-026).
5. MUST plan worktree/home isolation for execution (`eng-worktree-isolation`: unique `COMPOZY_HOME`, daemon port, tmux socket) since parallel QA is likely.
</requirements>

## Subtasks

- [x] 7.1 Journey flowcharts updated in `docs/qa/journeys/` (install/lifecycle, authoring/validate, marketplace/web, secrets/remote)
- [x] 7.2 Scenario rows reconciled in `docs/qa/scenarios/` (adopt task_06's files; fill journey/entry-point links; dedup)
- [x] 7.3 Charters in `docs/qa/charters/` for the cycle: targeted tier + canary + provider matrix + conformance walk
- [x] 7.4 Hot-spot map (invariants/ADRs → charter emphasis) recorded in the charter preamble
- [x] 7.5 Isolation plan (homes/ports/sockets) recorded for task_08

## Implementation Details

Operate exclusively on `docs/qa/` per the living-docs contract (SD-005). Inputs: `_spec.md` (invariants, ADR list, Impact Audit), `_user_stories.md` (journeys), `_dx.md` (exact commands), `_uiux.md` (browser flows), task files 01–06 (surfaces shipped + memory files if present).

### Relevant Files

- `docs/qa/` tree (journeys/scenarios/charters/bugs/reports) — the only output home
- `.compozy/tasks/agent-plugins/_dx.md` — command-level entry points
- `docs/_memory/standing_directives.md` SD-005 — the QA posture

### Dependent Files

- `docs/qa/state.csv` — regenerated view (gitignored; never hand-edited)

### Related ADRs

- All of [ADR-001..006](adrs/) — hot-spot sources

### Web/Docs Impact

- `web/` / `packages/site`: none — QA planning artifacts only (checked: docs/qa tree exclusively).
- QA impact: this task organizes the flags; verdicts stay with task_08.

### Extensibility / Agent Manageability / Config Lifecycle

- none — checked surfaces: planning artifacts only; no runtime, contract, or config change.

## Deliverables

- Updated journeys, reconciled scenarios, cycle charters (incl. provider matrix + conformance walk), hot-spot map, isolation plan
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED — none assigned; QA planning task)**

## Tests

Cases assigned from `_tests.md`: none — this task plans the human/agent QA cycle; automated cases are owned by tasks 01–05.

## Success Criteria

- Every public surface from tasks 01–06 appears as an entry point on exactly one journey-derived scenario row
- The provider-matrix and conformance-walk charters exist with explicit evidence-recording instructions for task_09's citation
- No duplicated scenario ids; `docs/qa` tree internally consistent
