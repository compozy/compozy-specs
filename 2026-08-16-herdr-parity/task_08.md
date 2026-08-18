---
status: completed
title: "Real-User QA Execution"
type: qa-execution
complexity: critical
---

# Task 8: Real-User QA Execution

## Overview

Executes the cycle task_07 planned: persona-driven walks of every in-scope scenario across CLI, HTTP/UDS, and the web shell, with evidence, verdicts, registry bugs, and the dated run report. The program is UI-bearing (`_uiux.md`), so browser E2E is mandatory alongside the daemon harness.

<critical>ALWAYS READ the in-scope docs/qa/scenarios/ files, open docs/qa/bugs/, and the cycle's charters in docs/qa/charters/ before executing.</critical>

<requirements>
1. Activate `qa-execution` with `qa-docs-path=docs/qa`; for release-grade runtime scope also activate `eng-real-scenario-qa` (playbook lab via `eng-qa-bootstrap`, one in-persona operator kickoff, runtime observation, strict audit).
2. Activate `eng-worktree-isolation` (unique `COMPOZY_HOME` + ports + tmux socket) when concurrency is signaled; isolated web QA exports `COMPOZY_WEB_API_PROXY_TARGET` from the bootstrap manifest; config writes against one isolated home run sequentially.
3. Run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright). Drive the highest-risk UI workflow (cross-workspace bell jump → answer → done/seen clearing) through `browser-use:browser`; fall back to `agent-browser` only if `browser-use:browser` is unavailable. Do not silently substitute shell-only checks. Browser walks judge the in-scope piece against the named board in `docs/design/opendesign/herdr-parity/`; a missing or failing Visual Contract bundle from tasks 03/05/06 is a blocking defect.
4. Exercise the agent-manageability surfaces end-to-end: structured CLI output vs HTTP/UDS state for the same persisted data (wait outcomes, interactions, summary counts, effective keymap), deterministic errors (exit 65/66/69/75/78, `agent_scope_denied`, `queue-full`), and config lifecycle (`[attention]` + shortcuts value shape) against a daemon-served runtime.
5. Register every reproduced defect in `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup against the registry first) and link it in the affected scenario files.
6. Fixes follow the fix-loop governor: small/contained only, regression test red-before/green-after, one logical fix per commit; escalate the rest to "Decisions for a Human".
7. Update scenario-file verdicts and write the dated run report at `docs/qa/reports/<YYYY-MM-DD>-herdr-parity.md`. Exit gate: re-run gates (`make verify` via `make gate-full`) before Final Status.
8. <critical>QA process teardown is mandatory (L-029): every lab ends with `eval "$TEARDOWN_COMMAND"` or `make qa-reap` on every terminal path; cite `teardown.json` (`"clean": true`) as evidence; register long-lived lab processes at `<QA_OUTPUT_PATH>/qa/pids/<name>.pid` on spawn.</critical>
</requirements>

## Subtasks

- [x] 8.1 Bootstrap the isolated lab (`eng-qa-bootstrap`; fresh manifest).
- [x] 8.2 Walk every in-scope scenario per charter order (CLI/API first, then browser journeys).
- [x] 8.3 Drive the highest-risk UI workflow via `browser-use:browser`; capture evidence indexed by path.
- [x] 8.4 Register/dedup bugs; apply governor-eligible fixes with red→green regression tests.
- [x] 8.5 Record verdicts, write the dated report, re-run `make gate-full`.
- [x] 8.6 Teardown (`teardown.json` clean) — `clean: true`, zero survivors.

## Implementation Details

Consumes task_07's charters/scenarios. Evidence lives in the lab, indexed by path in the report; QA state lives only in `docs/qa/`.

### Relevant Files

- `docs/qa/{scenarios,journeys,charters,bugs,reports}/` — the contract tree.
- Bootstrap manifest + `<QA_OUTPUT_PATH>/qa/pids/` — lab lifecycle.

### Related ADRs

- All six — verdict rationale references the invariants they encode.

## Deliverables

- Every in-scope scenario walked to a recorded verdict (only `blocked-verify`/`blocked-decision` may remain unwalked).
- Bug registry updated; dated report written; `make gate-full` green; teardown clean.

## Tests

QA execution task — walks the scenario tree; the `_tests.md` suite already ran inside tasks 01–06. Gate evidence: `make gate-status` current full-gate record.

## Success Criteria

- Zero scenarios left `untested`/`fail` in scope; every fix has a red-before/green-after regression test.
- `teardown.json` reports `"clean": true`; final report links every verdict to evidence.
