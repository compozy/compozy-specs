---
status: pending
title: Real-User QA Execution
type: qa-execution
complexity: critical
---

# Task 7: Real-User QA Execution

## Overview

Executes this cycle's charters against a real daemon-served runtime: walks every in-scope scenario to a recorded verdict, registers reproduced defects in the content-addressed bug registry, applies governor-scoped fixes, and closes the program with the full gate.

<critical>ALWAYS READ the in-scope `docs/qa/scenarios/` files, open `docs/qa/bugs/`, and the cycle's charters in `docs/qa/charters/` before executing.</critical>

<requirements>
1. MUST activate `qa-execution` with `qa-docs-path=docs/qa` AND `eng-real-scenario-qa` (release-grade runtime scope: playbook lab + operator kickoff + runtime observation), bootstrapping via `eng-qa-bootstrap` (fresh lab, unique `COMPOZY_HOME`).
2. MUST activate `eng-worktree-isolation` (unique `COMPOZY_HOME` + ports + tmux socket) when concurrency is signaled; isolated Web QA exports `COMPOZY_WEB_API_PROXY_TARGET` derived from the bootstrap manifest (never hardcoded `:2123`).
3. UI-bearing scope: run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright). Drive the highest-risk UI workflow — the redesigned run page two-register read with a live run — through `browser-use:browser`; fall back to `agent-browser` only if `browser-use:browser` is unavailable. Do not silently substitute shell-only checks.
4. CLI/API scope: exercise the affected CLI verbs (`task list --include-loop/--loop-run`, `loop why/nodes/events/runs`, `task timeline` audit reasons), HTTP/UDS routes, native tool, and the `loops.reconcile_interval` config lifecycle end-to-end against the daemon-served runtime; compare structured CLI output with HTTP/UDS responses for the same persisted state (semantic parity, N-003). Include the settlement journey: crash/kill a run, verify zero live records + audit reasons; boot with seeded orphans, verify repair.
5. MUST register every reproduced defect in `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup against the registry first) and link it in the affected scenario files; verify and record closure of `BUG-20260719-autonomous-progress-unobservable`.
6. Fixes follow the fix-loop governor: small/contained only, regression test red-before/green-after, one logical fix per commit; escalate the rest to "Decisions for a Human".
7. MUST update scenario-file verdicts (no flagged scenario left `untested`/`fail`; only recorded `blocked-verify`/`blocked-decision` may stay unwalked) and write the dated run report at `docs/qa/reports/<YYYY-MM-DD>-loop-task-legibility.md`.
8. Exit gate: re-run gates (`make verify` via `make gate-full`) before Final Status.
9. <critical>QA process teardown is mandatory (L-029): every lab/runtime envelope ends with `eval "$TEARDOWN_COMMAND"` (from the bootstrap manifest) or `make qa-reap` on every terminal path; cite `teardown.json` (`"clean": true`) as evidence; register long-lived lab processes at `<QA_OUTPUT_PATH>/qa/pids/<name>.pid` on spawn.</critical>
</requirements>

## Subtasks

- [ ] 7.1 Bootstrap the isolated lab (`eng-qa-bootstrap`; manifest, ports, homes)
- [ ] 7.2 Run the automated lanes: `make test-e2e-runtime` + `make test-e2e-web`
- [ ] 7.3 Walk the charters: browser journeys (two-register run page, Tasks default/reveal, runs roster) via `browser-use:browser`
- [ ] 7.4 Walk the CLI/API journeys incl. settlement/orphan-repair and cross-surface parity comparison
- [ ] 7.5 Register/dedup bugs; apply governor-scoped fixes with red→green regression tests
- [ ] 7.6 Record scenario verdicts + closure verification of the P1 bug
- [ ] 7.7 Write the dated report; run `make gate-full`; teardown with evidence
- [ ] 7.8 Escalate non-governor findings to "Decisions for a Human"

## Implementation Details

QA state lives in the committed `docs/qa/` tree; the lab holds only run-scratch evidence indexed by path. Provider-home policy per the bootstrap manifest. Never parallelize config writes against one isolated QA home.

### Relevant Files

- `docs/qa/charters/` (this cycle, from task_06) · `docs/qa/scenarios/` (in-scope ids) · `docs/qa/bugs/` (registry + P1 closure) · `docs/qa/reports/`.
- `.compozy/tasks/loop-task-legibility/_dx.md` — CLI/API journey transcripts and error table (walk verbatim).
- `.compozy/tasks/loop-task-legibility/_uiux.md` — surface states the browser walks assert.
- `docs/design/opendesign/loop-legibility/` — visual contracts for the two-register run page, Tasks reveal, and runs roster walks (`DESIGN-NOTES.md` + the six boards).

### Related ADRs

- [ADR-006](adrs/adr-006.md) — settlement journeys; [ADR-001](adrs/adr-001.md)/[ADR-002](adrs/adr-002.md) — default-read walks (SD-012 briefing test is scenario-owned, L-036).

## Deliverables

- Recorded verdicts on every in-scope scenario; bugs registered/linked/deduped; P1 closure recorded
- Dated run report at `docs/qa/reports/<YYYY-MM-DD>-loop-task-legibility.md`
- `make gate-full` evidence (fingerprint-cached record) + `teardown.json` (`"clean": true`)

## Tests

No `_tests.md` IDs are assigned to this task (all 99 belong to tasks 01-05 and run in their suites + the automated lanes above). This task's verification objects are the scenario walks, the cross-surface parity comparisons, and the exit gates.

## Success Criteria

- Zero scenarios left `untested`/`fail` in scope (only recorded `blocked-verify`/`blocked-decision` may remain)
- `make gate-full` green after the last mutation; teardown evidence clean
- Final Status states pass/fail per charter with evidence paths
