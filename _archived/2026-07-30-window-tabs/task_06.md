---
status: completed
title: Real-User QA Execution
type: qa-execution
complexity: critical
---

# Task 6: Real-User QA Execution

## Overview

Executes the planned window-tabs QA cycle as a real user: walks every in-scope scenario persona-driven with evidence, registers reproduced defects in the durable bug registry, runs the fix-loop under its governor, records verdicts, and writes the dated run report. The feature is UI-bearing AND CLI/API/agent-manageability-bearing — both e2e directives apply.

<critical>ALWAYS READ the in-scope docs/qa/scenarios/ files, open docs/qa/bugs/, and the cycle's charters in docs/qa/charters/ before executing.</critical>

<requirements>
1. MUST activate `qa-execution` with `qa-docs-path=docs/qa`; for the release-grade runtime scope also activate `eng-real-scenario-qa` (playbook lab + operator kickoff + runtime observation).
2. MUST bootstrap with `eng-qa-bootstrap` (fresh lab) and activate `eng-worktree-isolation`: unique `COMPOZY_HOME`, daemon port, tmux-bridge socket; `COMPOZY_WEB_API_PROXY_TARGET` derived from the bootstrap manifest — never default home/port (L-009).
3. MUST run `make test-e2e-runtime` AND `make test-e2e-web`; drive the highest-risk UI workflow (drag-merge → deck attention routing → close/reopen after reload) through `browser-use:browser`, falling back to `agent-browser` only if unavailable — no shell-only substitution.
4. MUST exercise the CLI/API/agent path end-to-end against the lab daemon: structured `compozy window list|group|activate|pin|reopen` output vs HTTP/UDS state for the same topology, deterministic errors (`window_manager_not_stacked`, `window_manager_window_pinned`, stale revision), config discovery of the two keys, `compozy layout watch` streaming, and hook-event observability.
5. MUST register every reproduced defect in `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup first) and link it from affected scenarios; fixes follow the fix-loop governor (small/contained, regression red-before/green-after, one logical fix per commit); escalate the rest to "Decisions for a Human".
6. MUST update scenario verdicts with evidence paths and write `docs/qa/reports/<YYYY-MM-DD>-window-tabs.md`; a flagged scenario left `untested`/`fail` blocks completion (only recorded `blocked-verify`/`blocked-decision` may stay).
7. MUST re-run gates before Final Status: `make gate-full` (the workstream close), citing `make gate-status` as evidence.
8. MUST end with process teardown on every terminal path: `eval "$TEARDOWN_COMMAND"` (or `make qa-reap`), citing `teardown.json` (`"clean": true`) — files may stay, processes never do (L-029). Register long-lived lab processes at `<QA_OUTPUT_PATH>/qa/pids/<name>.pid` on spawn.
</requirements>

## Subtasks

- [x] 6.1 Bootstrap the isolated lab (`eng-qa-bootstrap`) and record the manifest
- [x] 6.2 Run the automated e2e lanes (`make test-e2e-runtime`, `make test-e2e-web`)
- [x] 6.3 Walk every in-scope scenario per charter (browser + CLI/HTTP cross-surface) with evidence
- [x] 6.4 Register/dedup bugs; run the fix-loop under the governor; re-walk failed scenarios after fixes
- [x] 6.5 Record verdicts + dated report; escalate open decisions
- [x] 6.6 Run `make gate-full` after the final review mutation (teardown evidence is already clean)

## Implementation Details

Scenario set comes from task_05's plan. The lab exercises the real daemon + web + CLI; evidence lives under the lab's `<QA_OUTPUT_PATH>` indexed by path from the scenario files.

### Relevant Files

- `docs/qa/scenarios/*.md`, `docs/qa/charters/*`, `docs/qa/bugs/`, `docs/qa/reports/` — the living QA contract
- `web/e2e/__tests__/os-shell.spec.ts` — automated lane executed here

### Dependent Files

- Production code under `internal/windowmanager`, `internal/api`, `web/src/systems/os`, `packages/ui` — fix-loop targets when walks fail

### Related ADRs

- All ADR-001..013 — the walked behaviors trace to them; verdicts cite the violated ADR/invariant on failure

## Deliverables

- Every in-scope scenario walked to a recorded verdict with evidence; bugs registered and linked
- Dated run report; `make gate-full` evidence current; teardown clean
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED — none newly assigned: this task executes the shipped suites and scenario walks)**

## Tests

No new `_tests.md` IDs — this task runs the full shipped suites (`make gate-full`, both e2e lanes) and the scenario walks as its verification body.

## Success Criteria

- Zero in-scope scenarios left `untested`/`fail` (only recorded `blocked-*` allowed)
- `make gate-full` green once at workstream close (`make gate-status` cited)
- `teardown.json` shows `"clean": true` on the terminal path
- Dated report links every verdict, bug, and evidence path
