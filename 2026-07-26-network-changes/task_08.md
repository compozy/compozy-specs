---
status: pending
title: "Real-User QA Execution"
type: docs
complexity: critical
---

# Task 8: Real-User QA Execution

## Overview

Execute the planned QA cycle against a release-grade isolated lab: exercise local-default, coordinated-run flagship, invitation, discoverability, administration, live bounds, and agent-manageability journeys; drive highest-risk UI through Playwright/`browser-use`; register bugs; update `state.csv` verdicts; write the dated report; tear down all lab processes (L-029).

<critical>
- ALWAYS READ docs/qa/state.csv and the cycle's charters in docs/qa/charters/ before executing
- ALWAYS tear down with `eval "$TEARDOWN_COMMAND"` or `make qa-reap` on every terminal path — cite teardown.json `"clean": true`
- Activate eng-worktree-isolation — unique AGH_HOME + ports + tmux socket; never use default home/port under concurrency
- Fixes follow fix-loop governor: small/contained only; escalate the rest to "Decisions for a Human"
</critical>

<requirements>
1. Activate `qa-execution` with `qa-docs-path=docs/qa` and `eng-real-scenario-qa` for release-grade runtime observation; start with `eng-qa-bootstrap`.
2. Run `make test-e2e-runtime` AND `make test-e2e-web`. Drive highest-risk UI (invitation → enable → conversation; network empties) through `browser-use:browser` with `agent-browser` fallback only if unavailable — do not silently substitute shell-only checks.
3. Exercise structured CLI output vs HTTP/UDS for participation, coordination, usage, and status; assert deterministic error codes (`network_participation_unavailable`, `not_participating`, `loop_requires_live`, etc.).
4. Register every reproduced defect in `docs/qa/bugs/BUG-NNNN.md` (dedup first; never reset ids) and link in affected `state.csv` rows.
5. Provider-home policy MUST match provider contract; isolated Web QA MUST export `AGH_WEB_API_PROXY_TARGET` from bootstrap (never hardcode `:2123`).
6. Never parallelize config writes against one isolated QA home.
7. Update `state.csv` verdicts and write `docs/qa/reports/<YYYY-MM-DD>-network-changes.md`.
8. Exit gate: re-run `make verify` before Final Status; teardown evidence required.
9. Do not expand scope into mailbox or spend caps.
10. One worktree owns `docs/qa/` tracker edits and BUG minting this cycle.
</requirements>

## Subtasks

- [ ] 8.1 Bootstrap isolated lab from charters; register PIDs under QA output path.
- [ ] 8.2 Execute targeted + canary journeys (local, coordinated+invitation, live bounds, admin, agent).
- [ ] 8.3 Run runtime + web E2E gates; browser-drive flagship UI.
- [ ] 8.4 File bugs, update state.csv, write dated report.
- [ ] 8.5 Teardown; cite clean teardown.json; final verify if fixes landed.

## Implementation Details

Activate `eng-qa-bootstrap`, `eng-real-scenario-qa`, `qa-execution`, `qa-report` (for report shape), `eng-worktree-isolation`. UI features require Playwright/`browser-use` per cy-tasks-tail-qa-pair E2E directive.

Skills to activate: `eng-qa-bootstrap`, `eng-real-scenario-qa`, `qa-execution`, `eng-worktree-isolation`, `agent-browser`.

### Relevant Files

- `docs/qa/state.csv`, `docs/qa/charters/`, `docs/qa/journeys/`, `docs/qa/bugs/`, `docs/qa/reports/`
- Bootstrap manifest + teardown.json from lab
- `.compozy/tasks/network-changes/_tests.md` E2E catalog for traceability

### Dependent Files

- Implementation tasks 01–06 must be complete
- task_07 charters/state.csv must exist

### Related ADRs

- [ADR-002](adrs/adr-002.md) — flagship path under test
- [ADR-004](adrs/adr-004.md) — reject partial-mode release findings as blockers

## Deliverables

- Executed charters with verdicts in state.csv
- Dated QA report + any BUG-NNNN filings
- teardown.json with `"clean": true`
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)** — none; execution gate

## Tests

No new UT/IT/E2E authoring — execute the living QA plan and automated E2E gates.

- [ ] `make test-e2e-runtime` and `make test-e2e-web` exercised for this cycle
- [ ] Flagship invitation → coordination → conversation browser path observed
- [ ] Teardown clean on pass/fail/blocked/abort

## Success Criteria

- Charters executed; state.csv updated; report written
- Lab processes reaped; teardown evidence cited
- Blockers filed as bugs or escalated; no silent scope expansion

### Web/Docs Impact

- not applicable — QA execution only (runtime behavior already shipped)

### Extensibility / Agent Manageability / Config Lifecycle

- Execution MUST cover agent CLI/HTTP/UDS parity and extension confirmation paths listed in task_07 scenarios.
