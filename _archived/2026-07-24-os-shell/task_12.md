---
status: completed
title: Real-User QA Execution
type: qa-execution
complexity: critical
---

# Task 12: Real-User QA Execution

## Overview

Execute the cycle planned by task_11 as a real user: persona-driven sessions through the public surfaces (browser desktop, CLI, API), evidence-first verdicts on the scenario files, bugs into the durable registry, and a dated run report. The OS shell is release-grade runtime scope — the playbook lab applies.

<critical>ALWAYS READ the in-scope docs/qa/scenarios/ files, open docs/qa/bugs/, and the cycle's charters in docs/qa/charters/ before executing.</critical>

<requirements>
1. MUST activate `qa-execution` with `qa-docs-path=docs/qa`, and `eng-real-scenario-qa` for the release-grade runtime scope (playbook lab via `eng-qa-bootstrap`, one in-persona operator kickoff, runtime observation, strict audit).
2. MUST activate `eng-worktree-isolation`: unique `AGH_HOME`, unique daemon port, unique tmux-bridge socket (L-009) — default home/port forbidden.
3. UI e2e directive: run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright). Drive the highest-risk UI workflow — multi-session observation with live convergence across two browser contexts — through `browser-use:browser`; fall back to `agent-browser` only if `browser-use:browser` is unavailable. Do not silently substitute shell-only checks.
4. CLI/API directive: exercise `agh desktop-state list|get|set|delete|watch` end-to-end against the daemon-served runtime (unique `AGH_HOME`), compare structured CLI output with HTTP/UDS responses for the same persisted state, probe deterministic errors (`workspace_not_found`, rev conflicts, quota), and validate the `[desktop_state]` config lifecycle (defaults, override, invalid values).
5. MUST register every reproduced defect in `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup against the registry first) and link it in the affected scenario files.
6. Fixes follow the fix-loop governor: small/contained only, regression test red-before/green-after, one logical fix per commit; escalate the rest to "Decisions for a Human".
7. MUST update scenario-file verdicts and write the dated run report at `docs/qa/reports/<YYYY-MM-DD>-os-shell.md`; exit gate: re-run `make verify` before Final Status.
8. <critical>QA teardown is mandatory (L-029): every lab ends with `eval "$TEARDOWN_COMMAND"` or `make qa-reap` on every terminal path; cite `teardown.json` (`"clean": true`) as evidence; register long-lived processes at `<QA_OUTPUT_PATH>/qa/pids/`.</critical>
</requirements>

## Subtasks

- [ ] 12.1 Bootstrap isolated lab (`eng-qa-bootstrap`; manifest persisted)
- [ ] 12.2 Execute charters: persona journeys through the desktop (windows, snap, spaces, palette, attention, compact)
- [ ] 12.3 Multi-context convergence + degraded-recovery probes (two browsers + CLI writer)
- [ ] 12.4 CLI/HTTP/UDS cross-surface comparison for desktop-state
- [ ] 12.5 Bugs registered + linked; contained fixes via the governor
- [ ] 12.6 Verdicts + dated report + `make verify` re-run
- [ ] 12.7 Teardown with clean evidence

## Implementation Details

### Relevant Files

- `docs/qa/` living tree (scenarios/charters/bugs/reports)
- Bootstrap manifest + `bootstrap.env` (lab-scoped; `AGH_WEB_API_PROXY_TARGET` derived, never hardcoded)

### Related ADRs

- [ADR-004](adrs/adr-004.md)/[ADR-008](adrs/adr-008.md) — the sync/persistence contracts the convergence probes attack

## Deliverables

- Executed charters with evidence-linked verdicts on scenario files; bug registry entries; dated run report
- Cross-surface parity evidence (CLI vs HTTP/UDS) and convergence/degraded evidence
- Teardown evidence (`teardown.json` clean)

## Tests

No new implementation test IDs — this task executes the planned cycle against the shipped program; regression tests it adds ride the fix-loop governor (red-before/green-after) into the owning suites.

## Success Criteria

- Every in-scope scenario carries a fresh verdict with evidence paths; zero unregistered reproduced defects
- `make verify` green at exit; lab torn down clean (L-029 evidence cited)
