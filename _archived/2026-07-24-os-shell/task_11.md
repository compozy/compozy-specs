---
status: completed
title: QA Plan and Session Charters
type: qa-report
complexity: high
---

# Task 11: QA Plan and Session Charters

## Overview

Plan the os-shell QA cycle on the living `docs/qa/` tree: mint/reset scenario files for every user-visible behavior the program changed, update journey flowcharts, and write the cycle's session charters. This is planning only — execution belongs to task_12.

<critical>ALWAYS READ _techspec.md, every ADR, and every per-task memory file before planning.</critical>

<requirements>
1. MUST activate the `qa-report` skill with `qa-docs-path=docs/qa` (bootstrap the tree if absent) and operate only on the living tree (`scenarios/`, `journeys/`, `charters/`, `bugs/`, `reports/`) — never per-round `qa/` trees; `state.csv` is generated output only.
2. MUST cover every public surface tasks 01-10 touched, expressed as scenario `entry_points` on journey-derived rows: the desktop shell (windows/dock/menubar/palette/rail/bell), the window snap layer (drag zones, keyboard/palette, agent-written fractions — ADR-009), multi-session observation, spaces + appearance + compact, window-scoped dialogs, `agh desktop-state` CLI verbs, HTTP/UDS desktop-state routes + stream, config `[desktop_state]` keys, and the generated docs pages.
3. MUST mint content-addressed `untested` scenarios for new behavior and reset superseded chrome scenarios to `untested` (e.g. `ET-web-route-chrome-topbar`, `ET-web-tasks-mode-url`, network navigation scenarios) — per the QA-impact flags recorded in tasks 02, 04, 05, 06, 07, 08, 09, 10.
4. MUST map regression hot spots from `_techspec.md` §Safety Invariants (1-19) and ADR-001..009 into the cycle's charter selection: targeted tier (sync convergence, degraded recovery, URL semantics, attention truth, modal scoping) + one adjacent canary journey (an existing non-shell flow, e.g. a task publish, to catch collateral).
5. MUST update `docs/qa/journeys/` flowcharts for the desktop-first navigation model.
</requirements>

## Subtasks

- [ ] 11.1 Inventory the QA-impact flags from tasks 01-10 completion notes
- [ ] 11.2 Mint new `untested` scenarios (desktop, windows, snap, multi-session, palette, spaces, compact, dialogs, desktop-state CLI/API)
- [ ] 11.3 Reset superseded scenarios to `untested` (chrome/topbar/navigation families)
- [ ] 11.4 Update journeys; author the cycle's charters (targeted tier + canary)
- [ ] 11.5 Cross-check: every public surface has an entry_point row

## Implementation Details

### Relevant Files

- `docs/qa/scenarios/`, `docs/qa/journeys/`, `docs/qa/charters/`, `docs/qa/bugs/` — living tree
- `_techspec.md` §Safety Invariants + §Impact Analysis — hot-spot source

### Related ADRs

- All of [ADR-001..009](adrs/) — charter hot-spot mapping input

## Deliverables

- Scenario files minted/reset with content-addressed ids; journeys updated; charters authored for this cycle
- Coverage cross-check note proving every touched surface has an entry point

## Tests

No implementation test IDs — this is the QA planning task; its output is the plan the execution task consumes.

## Success Criteria

- Every surface touched by tasks 01-10 appears as a scenario entry point; no superseded scenario left claiming a pass against the old chrome
- Charters name the invariant hot spots they probe and the canary journey
