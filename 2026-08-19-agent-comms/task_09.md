---
status: pending
title: "Real-User QA Execution"
type: qa-execution
complexity: critical
---

# Task 9: Real-User QA Execution

## Overview

Execute task_08's charters as a real user against a bootstrapped lab: walk every in-scope scenario to a recorded verdict, register reproduced defects in the content-addressed bug registry, fix what the governor allows, and close the workstream with the dated report and full gates.

<critical>ALWAYS READ the in-scope docs/qa/scenarios/ files, open docs/qa/bugs/, and the cycle's charters in docs/qa/charters/ before executing.</critical>

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Activate `qa-execution` with `qa-docs-path=docs/qa`; for release-grade scope on the runtime also activate `eng-real-scenario-qa` (playbook lab + operator kickoff + runtime observation) and start from `eng-qa-bootstrap` (fresh lab per pass).
2. Activate `eng-worktree-isolation` (unique `COMPOZY_HOME` + daemon ports + tmux socket) when concurrency is signaled; isolated Web QA exports `COMPOZY_WEB_API_PROXY_TARGET` derived from the bootstrap manifest — never hardcoded `:2123`; config writes against one isolated home run sequentially.
3. E2E directive (`requires_e2e=true` — `_uiux.md` present): run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright). Drive the highest-risk UI workflow through `browser-use:browser`; fall back to `agent-browser` only if `browser-use:browser` is unavailable. Do not silently substitute shell-only checks.
4. CLI/API/agent-manageability: exercise structured CLI output, HTTP/UDS routes, status/config discovery, deterministic errors, and compare persisted state across surfaces (CLI vs HTTP vs app for the same call/message records).
5. Register every reproduced defect in `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup against the registry first) and link it in the affected scenario files. Fixes follow the fix-loop governor: small/contained only, regression test red-before/green-after, one logical fix per commit; escalate the rest to "Decisions for a Human".
6. Walk every scenario flagged by tasks 02/05/06 plus the charter selection to a recorded verdict; a failing walk means fix the production code and re-walk — never weaken a scenario. Only recorded `blocked-verify`/`blocked-decision` may stay unwalked.
7. Update scenario verdicts and write the dated run report at `docs/qa/reports/<YYYY-MM-DD>-<slug>.md`. Exit gate: re-run gates (`make verify` via `make gate-full`) before Final Status.
8. <critical>QA process teardown is mandatory (L-029): every lab ends with `eval "$TEARDOWN_COMMAND"` (from the bootstrap manifest) or `make qa-reap` on every terminal path; cite `teardown.json` (`"clean": true`) as evidence; register long-lived lab processes at `<QA_OUTPUT_PATH>/qa/pids/<name>.pid` on spawn.</critical>
</requirements>

## Subtasks

- [ ] 9.1 Bootstrap the lab (`eng-qa-bootstrap`, fresh manifest) with worktree isolation
- [ ] 9.2 Walk the charter scenarios: CLI/HTTP/tool journeys (golden path, follow-up/revive, batch, cancel, repair, extraction, TTL, drain, publish, task `--expect`, config effects, spawn-removal negative)
- [ ] 9.3 Walk the web journeys via Playwright/browser (Activity, call detail, inbox, roster compose, Calls tab, attention)
- [ ] 9.4 Register + link defects; run the fix-loop governor; escalate what exceeds it
- [ ] 9.5 Record verdicts, write the dated report, re-run `make gate-full`, tear down the lab

## Implementation Details

Charters and scenario files from task_08 drive everything; evidence lives in the lab indexed by path, verdicts and bugs in the committed `docs/qa/` tree.

### Relevant Files

- `docs/qa/charters/` + `docs/qa/scenarios/` + `docs/qa/bugs/` — the executing contract
- `.compozy/tasks/agent-comms/_dx.md` — exact invocations to reproduce

### Dependent Files

- `docs/qa/scenarios/*.md` — verdicts recorded
- `docs/qa/reports/<YYYY-MM-DD>-agent-comms.md` — the dated run report

### Related ADRs

- [ADR-011: Accounting-only call activations in v1](adrs/adr-011.md) — completions never denied: a QA observation of a dropped wake is a P0 bug, not tuning

## Deliverables

- Every in-scope scenario walked to a recorded verdict with evidence; defects registered and linked
- Dated run report; `make gate-full` green after the last mutation; `teardown.json` clean

## Tests

No new `_tests.md` IDs are assigned (all 264 are owned by tasks 01-06); this task executes the scenario walks and re-runs the full suites as its gate: `make test-e2e-runtime`, `make test-e2e-web`, `make gate-full`.

## Success Criteria

- Zero flagged scenarios left `untested`/`fail` (only recorded `blocked-verify`/`blocked-decision` may remain)
- Cross-surface state comparison clean (CLI vs HTTP vs app); deterministic errors verified per the `_dx.md` table
- `make gate-full` green; teardown evidence cited
