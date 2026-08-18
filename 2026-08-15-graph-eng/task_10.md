---
status: pending
title: QA Plan and Session Charters
type: qa-report
complexity: high
---

# Task 10: QA Plan and Session Charters

## Overview

Plans the QA cycle for the graph-eng program over the repo's living QA tree (`docs/qa/`): journeys, content-addressed scenario files, and session charters covering every public surface tasks 01–09 shipped — the new grammar (`ask`, `route`, `review:`, `strategy:`, `bind_as`/`index_as`, `stop_when` object, `on_eval_error`), the seven request/time-travel operations across CLI/HTTP/UDS/native tools, partiality (`completion_state`), per-lane verbs, windowed width, the web surfaces S1–S11, and the two new capabilities.

<critical>ALWAYS READ _spec.md, every ADR, and every per-task memory file before planning.</critical>

<requirements>
1. Activate the `qa-report` skill with `qa-docs-path=docs/qa` (bootstrap the tree if absent).
2. Coverage MUST span every public surface touched by tasks 01–09 — CLI verbs (`loop requests|request|respond|diff|rerun|fork`, `node amend`, `--item` variants), HTTP/UDS routes, native tools + capabilities (`loops.respond`, `loops.timetravel`), web routes (run page, diff view, editor, bell), doc pages, and the `requests.expire_after` config key — expressed as scenario `entry_points` on journey-derived rows, not standalone test cases.
3. Scenario dedup MUST run against the existing registry and against the `untested` scenarios the implementation tasks already minted (tasks 01–09 QA-impact lines) — reconcile, never duplicate.
4. Regression hot spots from `_spec.md` Part II Safety Invariants 1–13 and ADR-001/002/004/006 MUST map into the cycle's charter selection (targeted tier + one adjacent canary journey — the existing loop approval/quarantine journey is the natural canary).
</requirements>

## Subtasks

- [ ] 10.1 Reconcile the implementation-minted `untested` scenarios into journey-derived rows
- [ ] 10.2 Mint/update scenario files for the request plane, routing, strategies/partial, time travel, and web surfaces
- [ ] 10.3 Update journey flowcharts in `docs/qa/journeys/` (request lifecycle; time-travel lifecycle; partial-run lifecycle)
- [ ] 10.4 Author this cycle's charters in `docs/qa/charters/` with the invariant-driven hot-spot selection
- [ ] 10.5 Verify every entry_point ↔ surface mapping against `_dx.md`

## Deliverables

- Updated `docs/qa/journeys/`, minted/updated `docs/qa/scenarios/` rows, cycle charters in `docs/qa/charters/`

## Tests

- None — planning task; verification happens in task_11 against these artifacts.

## Success Criteria

- Every public surface from tasks 01–09 appears as an entry_point on at least one scenario row
- Charters name the invariant each targeted session stresses
