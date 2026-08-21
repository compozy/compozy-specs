---
status: pending
title: Real-User QA Execution
type: qa-execution
complexity: critical
---

# Task 9: Real-User QA Execution

## Overview

Executes the task_08 charters as a real user against a daemon-served runtime: walks every in-scope scenario to a recorded verdict, registers reproduced defects in the bug registry, fixes what the fix-loop governor allows, and closes the program with the dated run report and full gates.

<critical>ALWAYS READ the in-scope `docs/qa/scenarios/` files, open `docs/qa/bugs/`, and the cycle's charters in `docs/qa/charters/` before executing.</critical>

<requirements>
1. Activate `qa-execution` with `qa-docs-path=docs/qa`; for release-grade scope on the Compozy runtime also activate `eng-real-scenario-qa` (playbook lab via `eng-qa-bootstrap`, one in-persona operator kickoff, runtime observation, strict audit).
2. Activate `eng-worktree-isolation` when concurrency is signaled — unique `COMPOZY_HOME`, daemon ports, and `tmux-bridge` sockets; default home/port forbidden under concurrency.
3. UI surfaces: run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright). Drive the highest-risk UI workflow (Settings sources toggle → picker → expose panel) through `browser-use:browser`; fall back to `agent-browser` only if unavailable. Do not silently substitute shell-only checks.
4. CLI/API/agent-manageability surfaces: exercise structured CLI output (`skill sources -o json`, expose envelopes, config set/get/unset), HTTP/UDS routes, status/config discovery, deterministic errors — and compare structured CLI output with HTTP/UDS responses for the same persisted state (parity by observation).
5. Provider-home policy per contract: bound-secret/brokered lanes use `PROVIDER_HOME` from the bootstrap manifest; `native_cli` + `home_policy = operator` preserves the operator HOME unless a scenario tests isolated provider-home (the suppression matrix scenarios DO test both — honor each scenario's stated lane). Isolated Web QA exports `COMPOZY_WEB_API_PROXY_TARGET` derived from the manifest — never hardcoded.
6. Register every reproduced defect in `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup against the registry first) and link it in the affected scenario files. Fixes follow the fix-loop governor: small/contained only, regression test red-before/green-after, one logical fix per commit; escalate the rest to "Decisions for a Human".
7. Walk every scenario flagged by tasks 01-06 to a recorded verdict — a failing walk means fix the production code and re-walk; only recorded `blocked-verify`/`blocked-decision` may stay unwalked. Update scenario verdicts and write the dated run report at `docs/qa/reports/<YYYY-MM-DD>-skill-sources.md`.
8. Exit gate: re-run gates (`make gate-full` — the workstream close) before Final Status. <critical>QA process teardown is mandatory (L-029): every lab ends with `eval "$TEARDOWN_COMMAND"` or `make qa-reap` on every terminal path; cite `teardown.json` (`"clean": true`) as evidence; register long-lived lab processes at `<QA_OUTPUT_PATH>/qa/pids/<name>.pid` on spawn.</critical>
</requirements>

## Subtasks

- [ ] 9.1 Bootstrap the isolated lab (`eng-qa-bootstrap`); record manifest + pids
- [ ] 9.2 Walk charter scenarios: source configuration + live apply (CLI/API/web)
- [ ] 9.3 Walk absorption + session usage: picker origins, qualified invocation, suppression matrix per provider lane
- [ ] 9.4 Walk expose lifecycle: create/expose/unexpose/remove, four states, external-tool readability of links
- [ ] 9.5 Walk diagnostics: truncation, unreadable root, collisions, skipped links; cross-surface parity comparison
- [ ] 9.6 Browser pass via `browser-use:browser` on the highest-risk UI flow; `make test-e2e-runtime` + `make test-e2e-web`
- [ ] 9.7 Register bugs / apply governed fixes / escalate decisions; update scenario verdicts
- [ ] 9.8 Dated run report + `make gate-full` + teardown evidence

## Implementation Details

Execution operates on the committed `docs/qa/` tree; the lab holds only run-scratch evidence indexed by path. Canary journey per task_08's charter selection.

### Relevant Files

- `docs/qa/{scenarios,journeys,charters,bugs,reports}/` — living QA tree (verdicts, registry, report)
- `.compozy/tasks/skill-sources/_dx.md` — transcripts to reproduce as a real user
- `bootstrap-manifest.json` (lab) — env/ports/teardown authority for this pass

### Dependent Files

- None — program close.

### Related ADRs

- Safety-critical walks map to [ADR-009](adrs/adr-009.md)/[ADR-010](adrs/adr-010.md) (suppression), [ADR-011](adrs/adr-011.md)/[ADR-015](adrs/adr-015.md) (expose lifecycle), [ADR-005](adrs/adr-005.md) (live apply), [ADR-012](adrs/adr-012.md) (symlink containment).

## Deliverables

- Every in-scope scenario at a recorded verdict; bugs registered/deduped; governed fixes committed
- Dated report `docs/qa/reports/<YYYY-MM-DD>-skill-sources.md`
- `make gate-full` green at close; teardown evidence (`teardown.json` clean)

## Tests

No new `_tests.md` IDs — this task executes the shipped suites and the scenario walks: `make test-e2e-runtime`, `make test-e2e-web`, cross-surface CLI↔HTTP/UDS comparison, and `make gate-full` as the exit gate.

## Success Criteria

- Zero scenarios left `untested`/`fail` (only recorded `blocked-verify`/`blocked-decision` may remain)
- Every reproduced defect registered (or deduped) with evidence; fix-loop discipline observable in commit history
- Final Status backed by a current `make gate-full` record and clean teardown
