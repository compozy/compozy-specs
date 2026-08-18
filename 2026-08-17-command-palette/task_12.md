---
status: pending
title: "Real-User QA Execution"
type: qa-execution
complexity: critical
---

# Task 12: Real-User QA Execution

## Overview

Executes the task_11 charters as a real user against a fresh isolated lab: walks every in-scope scenario to a recorded verdict, registers reproduced defects in the bug registry, runs the fix-loop governor on contained fixes, and closes the program with the dated report and the full gate.

<critical>ALWAYS READ the in-scope `docs/qa/scenarios/` files, open `docs/qa/bugs/`, and the cycle's charters in `docs/qa/charters/` before executing.</critical>

<requirements>
1. MUST activate `qa-execution` with `qa-docs-path=docs/qa`; for the release-grade runtime scope also activate `eng-real-scenario-qa` (playbook lab + operator kickoff + runtime observation). Bootstrap the lab with `eng-qa-bootstrap` (fresh lab per pass).
2. MUST activate `eng-worktree-isolation` when concurrency is signaled: unique `COMPOZY_HOME`, daemon ports, tmux sockets; never the default home/port. Isolated Web QA exports `COMPOZY_WEB_API_PROXY_TARGET` from the bootstrap manifest.
3. MUST run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright). Drive the highest-risk UI workflows through `browser-use:browser`, falling back to `agent-browser` only if unavailable — never silently substitute shell-only checks. Desktop global-hotkey scenarios additionally cite `make test-e2e-desktop` (not in the default lanes — run it explicitly).
4. MUST exercise the CLI/API/agent-manageability surfaces end-to-end against the daemon-served runtime: structured CLI output (`-o json|jsonl`), HTTP/UDS routes, status/config discovery, deterministic errors — comparing structured CLI output with HTTP/UDS state for the same persisted data (bindings, aliases, pins, personalization, approvals, view sessions). Extension/config lifecycle validation: fixture enable/disable/dev-reload, `[cmd_palette]` + `[window_manager.global_shortcuts]` set/get/PATCH round-trips.
5. MUST walk every in-scope scenario to a recorded verdict — including the six program scenarios, **NL fallback**, the four canonical resets, and the **`ET-palette-sessions-view-switch` re-walk** (standing `blocked-verify` — resolve, don't supersede). A failing walk means fix the production code and re-walk; only recorded `blocked-verify`/`blocked-decision` may stay unwalked.
6. MUST register every reproduced defect in `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup against the registry first) and link it in the affected scenario files. Fixes follow the fix-loop governor: small/contained only, regression test red-before/green-after, one logical fix per commit; escalate the rest to "Decisions for a Human".
7. MUST write the dated run report at `docs/qa/reports/<YYYY-MM-DD>-command-palette.md` and update scenario verdicts. Exit gate: re-run gates (`make verify` via `make gate-full`) before Final Status.
8. MUST tear down every lab process on every terminal path: `eval "$TEARDOWN_COMMAND"` (from the bootstrap manifest) or `make qa-reap`; cite `teardown.json` (`"clean": true`) as evidence; register long-lived processes at `<QA_OUTPUT_PATH>/qa/pids/` on spawn (L-029).
</requirements>

## Subtasks

- [ ] 12.1 Bootstrap the isolated lab (`eng-qa-bootstrap`; fresh pass; manifest recorded)
- [ ] 12.2 Run the harness lanes: `make test-e2e-runtime` + `make test-e2e-web` + explicit `make test-e2e-desktop`
- [ ] 12.3 Walk the operator charters (browser via `browser-use:browser`): root/search/personalization, views (all kinds), action panel/args/confirmation, extension contributions + programmable views, global summon
- [ ] 12.4 Walk the agent charters: CLI/HTTP/UDS/native cross-surface comparisons, approval lifecycle, bindings/aliases/pins mutation parity
- [ ] 12.5 Re-walk `ET-palette-sessions-view-switch` + the canonical resets to recorded verdicts
- [ ] 12.6 Bug registry + fix-loop governor on contained fixes (red-before/green-after; one fix per commit)
- [ ] 12.7 Dated report + scenario verdict updates + `make gate-full` + teardown evidence

## Implementation Details

### Skills

`qa-execution` (with `qa-docs-path=docs/qa`) · `eng-real-scenario-qa` · `eng-qa-bootstrap` · `eng-worktree-isolation` · `qa-report` (verdict recording contract) · `systematic-debugging` + `no-workarounds` (fix loop)

### Relevant Files

- `docs/qa/charters/` (task_11 output) + `docs/qa/scenarios/` in-scope set + `docs/qa/bugs/` registry
- `web/e2e/` + `desktop/e2e/_electron/` + `internal/e2elane/` — the harness lanes
- `.compozy/tasks/command-palette/_dx.md` — transcripts the CLI walks follow verbatim
- Bootstrap manifest + `<QA_OUTPUT_PATH>` lab tree — run-scratch evidence indexed by path

### Dependent Files

- `docs/qa/scenarios/*` (verdicts), `docs/qa/bugs/*`, `docs/qa/reports/<date>-command-palette.md`
- Production code + regression suites touched by governor-approved fixes

### Related ADRs

- All — verdict evidence cites the invariant/ADR a failure violates.

### Web/Docs Impact

- QA impact: this task records the verdicts for every flagged scenario; a scenario left `untested`/`fail` at completion is a blocking failure (only recorded `blocked-verify`/`blocked-decision` may remain).

### Extensibility / Agent Manageability / Config Lifecycle

- Exercised, not changed — checked: this task validates those surfaces end-to-end; any gap found becomes a registered bug or a Decision for a Human, never an inline redesign.

## Deliverables

- Every in-scope scenario at a recorded verdict with evidence; `ET-palette-sessions-view-switch` resolved
- Bug registry entries + governor-compliant fixes with regression tests
- Dated run report at `docs/qa/reports/` + updated verdicts
- `make gate-full` green at close (the program's single full gate) + `teardown.json` `"clean": true`

## Tests

- No `_tests.md` IDs (the automated contract lives in tasks 01–10). This task's contract: scenario walks to recorded verdicts per the `qa-execution` skill, harness lanes green (`test-e2e-runtime`, `test-e2e-web`, explicit `test-e2e-desktop`), CLI↔HTTP/UDS cross-surface comparisons for bindings/aliases/pins/personalization/approvals, and red-before/green-after regression evidence for every governor fix.

## Success Criteria

- Zero in-scope scenarios left `untested`/`fail` (only recorded `blocked-*` verdicts may remain)
- `make gate-full` green after the last mutation; cited via `make gate-status`
- Teardown evidence clean on every terminal path (L-029)
- Dated report published; program ready for commit/PR close
