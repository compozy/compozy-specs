---
status: completed
title: Real-User QA Execution
type: qa-execution
complexity: critical
---

# Task 7: Real-User QA Execution

## Overview

Executes the planned cycle in-persona against a bootstrapped lab: walks every in-scope scenario to
a recorded verdict, drives the UI end-to-end, registers defects in the durable registry, applies
governed fixes, and closes with the dated report and mandatory teardown.

<critical>ALWAYS READ the in-scope docs/qa/scenarios/ files, open docs/qa/bugs/, and the cycle's charters in docs/qa/charters/ before executing.</critical>

<requirements>
- MUST activate `qa-execution` with `qa-docs-path=docs/qa`, bootstrapped via `eng-qa-bootstrap` (fresh lab, manifest-driven). Activate `eng-real-scenario-qa` for the runtime-observation harness. Activate `eng-worktree-isolation` (unique `COMPOZY_HOME` + ports + tmux socket) when concurrency is signaled — default home/port forbidden under concurrency.
- E2E directive (UI-bearing): run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright). Drive the highest-risk UI workflow (ratchet run detail: score/best/restore chip/exhausted→best) through `browser-use:browser`; fall back to `agent-browser` only if unavailable. Do not silently substitute shell-only checks.
- CLI/API/agent-manageability directive: exercise `loop status/runs/list/validate` structured output, HTTP/UDS loop routes, SSE replay, and native-tool outputs against the daemon; compare structured CLI output with HTTP/UDS responses for the same persisted state (B-008 mapping is the oracle).
- MUST walk every scenario minted/reset in task_06 to a recorded verdict with evidence; a failing walk means fix the production code and re-walk (never weaken the scenario). Only recorded `blocked-verify`/`blocked-decision` may stay unwalked.
- Register every reproduced defect in `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup first) and link it in the affected scenario files. Fixes follow the fix-loop governor: small/contained only, regression test red-before/green-after, one logical fix per commit; escalate the rest to "Decisions for a Human".
- Update scenario verdicts and write the dated run report at `docs/qa/reports/<YYYY-MM-DD>-loops-paper-adoption.md`. Exit gate: `make gate-full` (workstream close) before Final Status.
- <critical>QA process teardown is mandatory (L-029): every terminal path ends with `eval "$TEARDOWN_COMMAND"` or `make qa-reap`; cite `teardown.json` (`"clean": true`) as evidence; register long-lived lab processes under `<QA_OUTPUT_PATH>/qa/pids/`.</critical>
</requirements>

## Subtasks

- [x] 7.1 Bootstrap the lab (`eng-qa-bootstrap`), record the manifest, provider-home policy per contract.
- [x] 7.2 Walk the three minted scenarios + two resets in-persona with evidence (CLI + HTTP/UDS + web).
- [x] 7.3 Drive the ratchet UI journey via `browser-use:browser`; run both E2E suites.
- [x] 7.4 Register/dedup bugs; apply governed fixes with red-before/green-after regressions.
- [x] 7.5 Update verdicts; write the dated report; run the full verification lane.
- [x] 7.6 Teardown + `teardown.json` clean evidence.

## Implementation Details

### Relevant Files

- `docs/qa/scenarios/` (in-scope set from task_06), `docs/qa/bugs/`, `docs/qa/reports/`
- Bootstrap manifest + `<QA_OUTPUT_PATH>` lab tree (run-scratch evidence only)

### Related ADRs

- [ADR-002](adrs/adr-002.md), [ADR-003](adrs/adr-003.md) — expected behaviors under walk.

## Deliverables

- Every in-scope scenario at a recorded verdict with linked evidence; bugs registered/deduped.
- Dated report at `docs/qa/reports/<YYYY-MM-DD>-loops-paper-adoption.md`.
- `make gate-full` evidence current (cite `make gate-status`); `teardown.json` `"clean": true`.

## Tests

No new `_tests.md` IDs — this task executes the scenario contract and re-runs the shipped suites
(`make test-e2e-runtime`, `make test-e2e-web`) as verification, not authorship.

## Success Criteria

- Zero in-scope scenarios left `untested`/`fail` (only recorded `blocked-*` may remain)
- CLI vs HTTP/UDS state comparison recorded for the same run (structured output parity)
- `make gate-full` green once at close; teardown clean cited
