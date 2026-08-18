---
status: pending
title: "QA Plan and Session Charters"
type: qa-report
complexity: high
---

# Task 11: QA Plan and Session Charters

## Overview

Plans the real-user verification of the command-palette program on the living `docs/qa/` tree: journey flowcharts, scenario files, and session charters covering every public surface tasks 01–10 shipped. Planning only — execution and verdicts belong to task_12.

<critical>ALWAYS READ `_spec.md`, every ADR in `adrs/`, and every per-task memory file before planning.</critical>

<requirements>
1. MUST activate the `qa-report` skill with `qa-docs-path=docs/qa` (bootstrap the tree if absent) — the living-docs contract only: `docs/qa/{scenarios/,journeys/,charters/,bugs/,reports/}`; `state.csv` is generated output.
2. MUST express coverage as scenario `entry_points` on journey-derived rows — never standalone test cases — for every public surface tasks 01–10 touched: CLI (`cmd-palette list|inspect|invoke|clients|personalization|bind|unbind|alias|bindings|pin|unpin` + `--global` + `approvals show|cancel`), HTTP/UDS (`/api/cmd-palette/*`, `/api/tools/approvals/*`, settings sections), web surfaces (S1–S16), native tools (`compozy__cmd_palette_*`), extension surfaces (`resources.cmd_palette`, `view.provider`, dev reload, fixtures), desktop global hotkeys, and `config.toml` keys (`[cmd_palette]`, `[window_manager.global_shortcuts]`).
3. MUST reconcile the scenario inventory task_10 minted/reset (six program scenarios + NL fallback + four canonical resets incl. `ET-palette-sessions-view-switch` `blocked-verify`) — dedup against the registry, no parallel scenarios for the same behavior.
4. MUST map regression hot spots from `_spec.md` Part II Safety Invariants (1–21) and the ADR set into the cycle's charter selection: targeted tier (approval exactly-once, view-session isolation/generations, keymap conflicts, membership-vs-health, structural revision) + one adjacent canary journey (session landing / attention semantics — BR-20).
5. MUST update `docs/qa/journeys/` flowcharts for the palette-first operator paths and write this cycle's charters in `docs/qa/charters/`.
</requirements>

## Subtasks

- [ ] 11.1 Inventory every public surface from tasks 01–10 (verbs/routes/tools/keys/surfaces) against `_dx.md` and the task Completion Notes
- [ ] 11.2 Reconcile + finalize the scenario set (entry_points per journey row; dedup; content-addressed ids)
- [ ] 11.3 Update journey flowcharts in `docs/qa/journeys/`
- [ ] 11.4 Select charters: invariant-targeted tier + adjacent canary journey; write `docs/qa/charters/`
- [ ] 11.5 Verify every scenario file carries walkable steps + expected evidence for task_12

## Implementation Details

### Skills

`qa-report` (with `qa-docs-path=docs/qa`)

### Relevant Files

- `docs/qa/scenarios/` — existing palette scenarios + the task_10 minted/reset set
- `docs/qa/journeys/`, `docs/qa/charters/`, `docs/qa/bugs/`, `docs/qa/reports/` — the living tree
- `.compozy/tasks/command-palette/_dx.md` — the frozen surface inventory the plan must cover
- `.compozy/tasks/command-palette/_spec.md` § Safety Invariants + `adrs/` — hot-spot source

### Dependent Files

- `docs/qa/` tree — journeys/scenarios/charters updated this cycle

### Related ADRs

- All (hot-spot mapping input) — see `adrs/adr-001..009.md`

### Web/Docs Impact

- `web/` / `packages/site`: none — planning artifacts only, in `docs/qa/`.
- QA impact: this task IS the QA plan; no scenario verdicts change here.

### Extensibility / Agent Manageability / Config Lifecycle

- none — checked surfaces: planning-only task; the plan itself covers those surfaces as scenario entry points.

## Deliverables

- Updated `docs/qa/journeys/` + finalized scenario inventory + this cycle's charters in `docs/qa/charters/`
- Every public surface of tasks 01–10 reachable through at least one scenario entry point
- Charter selection justified against Safety Invariants + ADR hot spots

## Tests

- No `_tests.md` IDs (planning task). Output quality gates: scenario files parse per the `qa-execution` contract; no orphan surface (11.1 inventory cross-check); no duplicate scenarios (11.2 dedup evidence).

## Success Criteria

- task_12 can execute every charter without inventing steps
- Zero public surfaces from tasks 01–10 missing a scenario entry point
- The `ET-palette-sessions-view-switch` re-walk is explicitly scheduled in a charter (standing `blocked-verify` debt)
