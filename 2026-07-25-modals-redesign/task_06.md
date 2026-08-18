---
status: pending
title: Real-User QA Execution
type: qa-execution
complexity: critical
---

# Task 06: Real-User QA Execution

## Overview

Executes the modal-redesign QA cycle against a real daemon-served web UI: runs in-scope `docs/qa/scenarios/`, drives Playwright for the highest-risk dialogs, verifies Visual Contract Mode evidence still passes, registers bugs, and writes the dated report. Closes the wave only when `make verify` is green and teardown is clean.

<critical>
ALWAYS READ the in-scope `docs/qa/scenarios/` files, open `docs/qa/bugs/`, and the cycle's charters in `docs/qa/charters/` before executing.
</critical>

<requirements>
- MUST activate `qa-execution` with `qa-docs-path=docs/qa`. For release-grade runtime scope also activate `eng-real-scenario-qa` (playbook lab + operator kickoff + runtime observation).
- MUST activate `eng-worktree-isolation` (unique `AGH_HOME` + ports + tmux socket) when concurrency is signaled; never use the default home/port under concurrency.
- MUST run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright). Drive the highest-risk UI workflows through `browser-use:browser`; fall back to `agent-browser` only if `browser-use:browser` is unavailable. Do not silently substitute shell-only checks.
- MUST exercise at minimum: agent create Simple/Advanced (no MCP field), bridge create secrets + edit delivery (D2), MCP secret preserve/rotate, provider auth_mode gate (T3), vault write-only secret, workspace DirectoryBrowser split, session advanced overrides.
- MUST verify Visual Contract evidence from tasks 01–04 still resolves with zero unresolved blocking divergences for charter-selected rows (re-capture only when implementation drifted).
- MUST register every reproduced defect in `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup first) and link it from affected scenario files.
- MUST follow the fix-loop governor: small/contained fixes only, red-before/green-after regression test, one logical fix per commit; escalate the rest to "Decisions for a Human".
- MUST update scenario-file verdicts and write `docs/qa/reports/<YYYY-MM-DD>-modals-redesign.md`.
- MUST tear down every lab/runtime envelope (`eval "$TEARDOWN_COMMAND"` or `make qa-reap`) and cite `teardown.json` with `"clean": true` — processes never left alive.
- MUST exit with `make verify` green.
</requirements>

## Subtasks

- [ ] 6.1 Bootstrap isolated QA lab from charters; register long-lived PIDs
- [ ] 6.2 Execute targeted charter scenarios (dense dialogs + provider auth gate)
- [ ] 6.3 Execute canary charter (SecretField shared path / marketplace or MCP)
- [ ] 6.4 Run `make test-e2e-runtime` and `make test-e2e-web`; drive highest-risk UI via browser-use
- [ ] 6.5 Confirm Visual Contract bundles for charter rows; re-capture only on drift
- [ ] 6.6 File bugs, apply contained fixes with regression tests, escalate the rest
- [ ] 6.7 Update scenario verdicts + write dated report
- [ ] 6.8 Teardown to clean; run `make verify`

## Implementation Details

Activate `qa-execution`, `eng-real-scenario-qa`, `eng-qa-bootstrap`, and `eng-worktree-isolation` as required. UI feature ⇒ Playwright mandatory. Highest-risk workflows: agent wizard collapse, bridge secrets/delivery, provider T3 gate. Evidence paths under `.compozy/tasks/modals-redesign/evidence/visual/` and lab `qa/` scratch.

### Relevant Files

- `docs/qa/scenarios/`, `docs/qa/charters/`, `docs/qa/bugs/`, `docs/qa/reports/`
- `docs/qa/journeys/`
- Playwright specs under `web/e2e/`
- Visual evidence from tasks 01–04

### Dependent Files

- Production dialogs from tasks 01–04 (fix targets only when defects are small/contained)

## Deliverables

- Scenario verdicts updated for the in-scope set
- Dated report at `docs/qa/reports/<YYYY-MM-DD>-modals-redesign.md`
- Bugs filed/linked; contained fixes landed with tests
- Teardown clean + `make verify` green

## Tests

- [ ] `make test-e2e-runtime` passes in the isolated lab
- [ ] `make test-e2e-web` passes; highest-risk dialogs exercised via browser-use (or documented agent-browser fallback)
- [ ] Charter scenarios executed with recorded verdicts
- [ ] Visual Contract charter rows remain PASS (or re-captured PASS)
- [ ] `make verify` green
- [ ] Teardown evidence `"clean": true`

### Web/Docs Impact

- `web/`: only contained bugfix patches discovered in execution.
- `packages/site`: none unless a bug proves a public docs lie — then fix in the same contained change.
- QA impact: scenario verdicts + report are the deliverable; do not leave scenarios stuck in untested after a pass.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none expected — UI QA.
- Agent manageability: where charters include CLI canaries for the same entities, compare structured CLI/HTTP results with UI-persisted state.
- Config lifecycle: none — checked no config writes beyond lab bootstrap.

### AGH Impact Audit

- Native tools: no impact unless a bug fix touches them — checked default is UI-only.
- Extensibility and hooks: no impact unless a bug fix requires it.
- Workspace data isolation: scoped dialog canaries must not leak across workspaces in lab evidence.
- Official AGH skill: no impact unless public guidance was wrong — then update `skills/agh/` in the fix commit.

## Success Criteria

- In-scope scenarios have execution verdicts; dated report filed
- E2E runtime + web green; Visual Contract charter rows PASS
- Teardown clean; `make verify` green
- Residual defects either fixed with tests or escalated as human decisions
