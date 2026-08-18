---
status: completed
title: Real-User QA Execution
type: qa-execution
complexity: critical
---

# Task 10: Real-User QA Execution

## Overview

Executes the cycle's charters as a real user against a daemon-served runtime: walks every in-scope worktree scenario to a recorded verdict with evidence, drives the UI-bearing journeys through the browser, exercises the CLI/API/agent-manageability surfaces end-to-end, registers reproduced defects in the bug registry, and closes the program with the dated run report and the full gate.

<critical>ALWAYS READ the in-scope docs/qa/scenarios/ files, open docs/qa/bugs/, and the cycle's charters in docs/qa/charters/ before executing.</critical>

<requirements>
1. MUST activate `qa-execution` with `qa-docs-path=docs/qa`; for release-grade runtime scope also activate `eng-real-scenario-qa` (playbook lab via `eng-qa-bootstrap`, one in-persona operator kickoff, read-only runtime observation, strict audit).
2. MUST activate `eng-worktree-isolation` when concurrency is signaled: unique `COMPOZY_HOME`, daemon ports, and `tmux-bridge` sockets; never the default home/port; isolated Web QA exports `COMPOZY_WEB_API_PROXY_TARGET` derived from the bootstrap manifest.
3. UI features: run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright). Drive the highest-risk UI workflow — per-run fan-out with parallel worktrees through the exit ladder to removal — through `browser-use:browser`; fall back to `agent-browser` only if `browser-use:browser` is unavailable. Do not silently substitute shell-only checks.
4. CLI/API/agent-manageability: exercise structured CLI output (`-o json|jsonl`), HTTP/UDS routes, native tools under permission modes, status/config discovery, deterministic errors — comparing persisted state across surfaces for the same fixture (worktree lifecycle, policy patch, exit actions, forge zero-credential tier and, when a binding or `gh` login is present in the lab, the credentialed tier).
5. Config/extensibility: validate the `[worktrees]` config lifecycle (defaults, workspace overlay, invalid-value refusal naming the key) and the `worktree` hook events + GitHub extension enable/disable truthful degradation.
6. MUST register every reproduced defect in `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup against the registry first) and link it in the affected scenario files.
7. Fixes follow the fix-loop governor: small/contained only, regression test red-before/green-after, one logical fix per commit; escalate the rest to "Decisions for a Human". When a walk fails, fix the production code and re-walk — never weaken a scenario.
8. MUST update scenario-file verdicts with evidence paths and write the dated run report at `docs/qa/reports/<YYYY-MM-DD>-worktree-support.md`. Exit gate: re-run gates (`make gate-full`) before Final Status.
9. <critical>QA process teardown is mandatory (L-029): every lab ends with `eval "$TEARDOWN_COMMAND"` (bootstrap manifest) or `make qa-reap` on every terminal path; register long-lived lab processes at `<QA_OUTPUT_PATH>/qa/pids/<name>.pid`; cite `teardown.json` (`"clean": true`) as evidence.</critical>
</requirements>

## Subtasks

- [x] 10.1 Bootstrap isolated lab (`eng-qa-bootstrap`; fresh manifest) + provider-home policy per contract
- [x] 10.2 Walk CLI/API/tool scenarios to verdicts with cross-surface state comparison
- [x] 10.3 Walk web journeys (browser-driven) incl. the per-run fan-out → exit → removal charter
- [x] 10.4 Config lifecycle + hook/extension degradation walks
- [x] 10.5 Register bugs (dedup) + fix-loop for small/contained defects + escalation list
- [x] 10.6 Verdicts + evidence in scenario files; dated report in `docs/qa/reports/`
- [x] 10.7 `make gate-full` + teardown with `teardown.json` evidence

## Implementation Details

The lab exercises the real single-binary daemon (CLI + Web + API), not mocks. Scenario walk order follows the charters; the canary journey (workspace lifecycle) runs even if untouched by diffs.

### Relevant Files

- `docs/qa/{scenarios,charters,journeys,bugs,reports}/` — the living contract
- Bootstrap manifest + `<QA_OUTPUT_PATH>` lab tree (run-scratch evidence, pids, teardown.json)

### Dependent Files

- `docs/qa/state.csv` — regenerated view

### Related ADRs

- [ADR-006](adrs/adr-006.md) — the data-loss surfaces the walks stress hardest

### Web/Docs Impact

- `web/` / `packages/site`: none authored here; doc-accuracy findings become bugs/escalations (checked: QA artifacts only).
- QA impact: this task IS the verification — no flagged scenario may remain `untested`/`fail` at completion; only recorded `blocked-verify`/`blocked-decision` may stay unwalked.

### Extensibility / Agent Manageability / Config Lifecycle

- Verified, not changed: agent-operable surfaces, hook events, extension degradation, and config lifecycle are exercised as shipped; findings route to bugs — no surface changes originate here.

## Deliverables

- Every in-scope scenario walked to a recorded verdict with evidence
- Bug registry entries + fix-loop commits (small/contained) + escalation list
- Dated run report + regenerated `state.csv` + `teardown.json` (`"clean": true`)
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED — none assigned; verification task)**

## Tests

No `_tests.md` IDs assigned — the automated contract (152 UT / 41 IT / 17 E2E) ships inside tasks 01-07; this task runs the persona-driven walks plus `make test-e2e-runtime`, `make test-e2e-web`, and the closing `make gate-full`.

## Success Criteria

- Zero flagged scenarios left `untested`/`fail`; blocked states recorded with reasons
- `make gate-full` green at close (the program's single full gate)
- Teardown evidence clean; no lab processes survive
- Run report leads with the highest-value findings and names every escalation
