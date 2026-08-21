---
status: pending
title: QA Plan and Session Charters
type: qa-report
complexity: high
---

# Task 12: QA Plan and Session Charters

## Overview

Plans the QA cycle for the Profiles program on the living `docs/qa/` tree: journey flowcharts, content-addressed scenario files for every new/changed public surface from tasks 01–11, and the session charters the execution pass (task_13) will walk.

<critical>ALWAYS READ `_spec.md`, every ADR under `adrs/`, and every per-task memory file before planning.</critical>

**Skills**: activate `qa-report` with `qa-docs-path=docs/qa` (bootstrap the tree if absent — it exists and is populated).

<requirements>
- MUST update `docs/qa/journeys/` with the Profiles journeys (lifecycle+selection, scoped work + aggregate, layers/config/credentials, extensions/presets, per-profile state, phase-0 foundation semantics).
- MUST mint/update content-addressed scenario files in `docs/qa/scenarios/` covering every public surface touched by tasks 01–11 — CLI verbs (`compozy profile *`, `--profile`/`--all-profiles`, config `--scope`, secret/provider, extension/preset enablement), HTTP/UDS routes (profiles/selection/plans/ops, scoped+aggregate listings, enablement), web routes (S1–S13), native tools, extension manifest points, and selection env (`COMPOZY_PROFILE`) — expressed as scenario `entry_points` on journey-derived rows, not standalone test cases; dedup against the 938 existing scenario files first (reconcile the per-task QA-impact flags: new files stay `untested`, changed behaviors reset to `untested`).
- MUST map regression hot spots from `_spec.md` Safety Invariants 1–19 and ADR-001/002/012/013/015 into the cycle's charter selection (targeted tier + one adjacent canary journey — e.g., workspace isolation, which shares the enforcement point).
- MUST write session charters in `docs/qa/charters/` for this cycle, sized for the task_13 walk, including the fail-closed leak probes (foreign-profile fixtures) and the archive/permit race scenarios.
</requirements>

## Subtasks

- [ ] 12.1 Journey flowcharts updated in `docs/qa/journeys/` for the six Profiles journey families.
- [ ] 12.2 Scenario sweep: mint/reset content-addressed files in `docs/qa/scenarios/` per the tasks' QA-impact flags; dedup same-behavior conflicts.
- [ ] 12.3 Charter selection: targeted tier from invariants/ADR hot spots + one adjacent canary journey.
- [ ] 12.4 Charters written in `docs/qa/charters/` with evidence expectations and entry points per surface (CLI/HTTP/UDS/web/agent).

## Implementation Details

QA state is the committed `docs/qa/` tree; `state.csv` is generated. Scenario naming follows the existing `<PREFIX>-<kebab-slug>.md` convention — reuse existing prefixes where the surface family matches; content-addressed ids, never a shared counter.

### Relevant Files

- `docs/qa/README.md`, `docs/qa/personas.md`, `docs/qa/templates/` — tree contract.
- `docs/qa/journeys/` (122 files), `docs/qa/scenarios/` (938 files), `docs/qa/charters/`, `docs/qa/bugs/` — the living surfaces this task edits.
- `.compozy/tasks/profiles/_dx.md` + `_uiux.md` — the public-surface inventory the scenarios must cover.
- `.compozy/tasks/profiles/task_01..11` QA-impact lines — the flag ledger to reconcile.

### Related ADRs

- [ADR-001](adrs/adr-001.md), [ADR-002](adrs/adr-002.md), [ADR-015](adrs/adr-015.md) — the highest-risk behaviors the charters target.

## Deliverables

- Updated journeys, minted/reset scenarios (all `untested`), and this cycle's charters — committed under `docs/qa/`.
- Charter coverage traceable to every task's QA-impact flags with zero unreconciled flags.

## Tests

No `_tests.md` IDs — this task produces the QA plan; execution and verdicts belong to task_13.

## Success Criteria

- Every public surface from tasks 01–11 appears as an entry point on at least one scenario row; no orphan QA-impact flag remains.
- Charters name their journeys, personas, evidence expectations, and stop conditions; targeted tier + canary journey selected.
