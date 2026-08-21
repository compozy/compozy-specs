---
status: completed
title: Real-User QA Execution
type: qa-execution
complexity: critical
---

# Task 11: Real-User QA Execution

## Overview

Executes the task_10 charters against a real daemon-served runtime: walks every in-scope scenario to a recorded verdict, drives the highest-risk UI workflows through a real browser, exercises the agent-operation paths with structured output and cross-surface state comparison, registers reproduced defects in the content-addressed bug registry, and closes with the dated run report and green gates.

<critical>ALWAYS READ the in-scope docs/qa/scenarios/ files, open docs/qa/bugs/, and the cycle's charters in docs/qa/charters/ before executing.</critical>

<requirements>
1. Activate `qa-execution` with `qa-docs-path=docs/qa`; for release-grade scope on the multi-agent runtime also activate `eng-real-scenario-qa` (playbook lab via `eng-qa-bootstrap` + operator kickoff + runtime observation).
2. Activate `eng-worktree-isolation` when concurrency is signaled — unique `COMPOZY_HOME`, daemon ports, and tmux sockets; isolated Web QA exports `COMPOZY_WEB_API_PROXY_TARGET` from the bootstrap manifest.
3. UI directive: run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright). Drive the highest-risk UI workflow (request answer → resume; fork dialog → new run) through `browser-use:browser`; fall back to `agent-browser` only if unavailable. Do not silently substitute shell-only checks.
4. CLI/API directive: exercise the seven operations end to end against the lab daemon — structured CLI output vs HTTP/UDS responses for the same persisted state (requests inventory/detail, respond provenance, diff payloads, rerun/fork lineage), capability grants/denials (`loops.respond`, `loops.timetravel`, self-denial), deterministic errors per `_dx.md`, and the `requests.expire_after` config lifecycle.
5. Register every reproduced defect in `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup against the registry first) and link it in the affected scenario files. Fixes follow the fix-loop governor: small/contained only, regression test red-before/green-after, one logical fix per commit; escalate the rest to "Decisions for a Human".
6. Update scenario-file verdicts and write the dated run report at `docs/qa/reports/<YYYY-MM-DD>-graph-eng.md`. No flagged scenario may remain `untested`/`fail` at completion (only recorded `blocked-verify`/`blocked-decision` may stay unwalked).
7. **QA teardown is mandatory (L-029)**: every lab ends with `eval "$TEARDOWN_COMMAND"` or `make qa-reap` on every terminal path; cite `teardown.json` (`"clean": true`) as evidence; register long-lived lab processes at `<QA_OUTPUT_PATH>/qa/pids/`.
8. Exit gate: re-run `make verify` (via `make gate-full` — workstream close) before Final Status.
</requirements>

## Subtasks

- [x] 11.1 Bootstrap the isolated lab (`eng-qa-bootstrap`) and record the manifest
- [x] 11.2 Walk every in-scope scenario to a recorded verdict (CLI/API tier first)
- [x] 11.3 Browser tier: request forms, diff view, fork dialog, bell journey via `browser-use:browser`
- [x] 11.4 Cross-surface comparisons + capability matrix spot checks against persisted state
- [x] 11.5 Bug registry entries + governed fixes (red-before/green-after) or escalations
- [x] 11.6 Verdicts + dated report + teardown evidence + `make gate-full`

## Deliverables

- Every in-scope scenario carrying a recorded verdict; bug registry entries with links; `docs/qa/reports/<date>-graph-eng.md`; `teardown.json` clean; `make gate-full` green

## Tests

- None assigned from `_tests.md` — this task verifies the shipped suites and walks scenarios; automated cases live in tasks 01–09.

## Success Criteria

- Zero `untested`/`fail` flagged scenarios at completion (blocked states recorded explicitly)
- `make gate-full` passes after the last mutation; teardown evidence cited
