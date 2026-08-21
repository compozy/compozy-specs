---
status: pending
title: Real-User QA Execution
type: qa-execution
complexity: critical
---

# Task 13: Real-User QA Execution

## Overview

Executes the task_12 charters against a real daemon-served runtime: walks every in-scope scenario to a recorded verdict, registers reproduced defects in the bug registry, runs the fix loop for small contained fixes, and closes the program with the dated run report and full gates.

<critical>ALWAYS READ the in-scope docs/qa/scenarios/ files, open docs/qa/bugs/, and the cycle's charters in docs/qa/charters/ before executing.</critical>

**Skills**: activate `qa-execution` with `qa-docs-path=docs/qa`; activate `eng-real-scenario-qa` (release-grade runtime scope: playbook lab via `eng-qa-bootstrap`, operator kickoff, runtime observation); activate `eng-worktree-isolation` (unique `COMPOZY_HOME` + ports + tmux socket — concurrency is signaled by parallel lanes).

<requirements>
- MUST run both e2e lanes: `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright). Drive the highest-risk UI workflow (profile switch → aggregate → create-under-All → deep link) through `browser-use:browser`; fall back to `agent-browser` only if unavailable. Do not silently substitute shell-only checks.
- MUST exercise the CLI/API/agent-manageability surface end to end against a daemon-served runtime (unique `COMPOZY_HOME`): structured CLI output, HTTP/UDS routes, status/config discovery, deterministic errors — and compare structured CLI output with HTTP/UDS responses for the same persisted state (profiles list/current/selection, scoped vs aggregate listings, enablement state, needs-setup).
- MUST validate the extension/config lifecycle paths live: mock-kit install with declared profile, per-profile enablement, layered config write/feedback/denylist, per-profile secret journey.
- MUST walk every in-scope scenario per the `qa-execution` contract and record the verdict with evidence; a failing walk means fix the production code and re-walk until it passes — never weaken the scenario. Only recorded `blocked-verify`/`blocked-decision` may stay unwalked.
- MUST register every reproduced defect in `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup against the registry first) and link it from the affected scenario files.
- Fixes follow the fix-loop governor: small/contained only, regression test red-before/green-after, one logical fix per commit; escalate the rest to "Decisions for a Human".
- MUST write the dated run report at `docs/qa/reports/<YYYY-MM-DD>-profiles.md` and update scenario verdicts.
- Exit gate: re-run `make verify` (via `make gate-full`) before Final Status.
- <critical>QA process teardown is mandatory (L-029): every lab ends with `eval "$TEARDOWN_COMMAND"` (bootstrap manifest) or `make qa-reap` on every terminal path; cite `teardown.json` (`"clean": true`) as evidence; register long-lived lab processes at `<QA_OUTPUT_PATH>/qa/pids/<name>.pid` on spawn.</critical>
- Provider-home policy follows the provider contract (bound-secret/brokered → isolated `PROVIDER_HOME`; `native_cli` + `home_policy=operator` keeps operator HOME); isolated web QA exports `COMPOZY_WEB_API_PROXY_TARGET` from the manifest — never hardcoded; config writes against one isolated home run sequentially.
</requirements>

## Subtasks

- [ ] 13.1 Bootstrap the lab (`eng-qa-bootstrap`, fresh manifest) with isolated home/ports/sockets.
- [ ] 13.2 Walk the charters: lifecycle+selection, scoped/aggregate (leak probes), layers/config/credentials, extensions/presets, per-profile state, phase-0 semantics.
- [ ] 13.3 Cross-surface state comparison (CLI × HTTP × UDS × web) for the persisted profile state.
- [ ] 13.4 Bug registry entries + fix loop + re-walks.
- [ ] 13.5 Verdicts recorded, dated report written, `make gate-full` green, lab teardown proven clean.

## Implementation Details

The task_12 charters are the execution plan; this task owns evidence, verdicts, fixes, and the report. Browser lane covers S1–S13 journeys; runtime lane covers `_dx.md` transcripts.

### Relevant Files

- `docs/qa/charters/` (this cycle), `docs/qa/scenarios/` (in-scope rows), `docs/qa/bugs/`, `docs/qa/reports/`.
- `.compozy/tasks/profiles/_dx.md` — CLI/API truth for cross-surface comparison.
- Bootstrap manifest + `teardown.json` — lab lifecycle evidence.

### Related ADRs

- [ADR-004](adrs/adr-004.md) / [ADR-015](adrs/adr-015.md) — the fail-closed scoping the leak probes attack.

## Deliverables

- Every in-scope scenario walked to a recorded verdict with evidence; bugs registered and linked; fixes committed per the governor.
- Dated run report at `docs/qa/reports/<date>-profiles.md`; scenario verdicts updated.
- `make gate-full` green after the last mutation; teardown evidence (`"clean": true`).

## Tests

No new `_tests.md` IDs — this task executes the shipped suites and the scenario walks; regression tests it adds belong to the fix loop (red-before/green-after in the owning canonical suite).

## Success Criteria

- Zero scenarios left `untested`/`fail` in scope (only recorded `blocked-*` may remain).
- Cross-surface comparisons agree byte-for-byte where the contract promises parity.
- Final Status backed by a current full-gate record and clean teardown.
