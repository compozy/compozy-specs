---
status: pending
title: QA execution — run the scenario matrix and close the report with a release verdict
type: test
complexity: high
---

# Task 13: QA execution — run the scenario matrix and close the report with a release verdict

## Overview

Executes the scenario matrix authored by task_12 (rows + charters in `docs/qa/`) against a fresh
deterministic QA lab, records timestamped evidence per scenario, registers defects in the global
bug registry, and closes the dated run report with the release-readiness verdict. A red scenario
is a production bug, never a test to weaken.

This is a backend-heavy program with CLI/HTTP/UDS/agent surfaces (`requires_cli_e2e=true`) and
web surfaces for suggestion/checkpoint/approval UI from earlier tasks (`requires_e2e=true` —
tasks 01/07/08 touch web).

**Authoritative sources:** this suite has no `_prd.md` / `_user_stories.md` / `_tests.md`.
`_techspec.md`, `analysis/summary.md`, and task_12's plan (`docs/qa` rows + charters) are
authoritative.

Merges former task 35. Depends on task_12 (edge `12→13`).

Activate `eng-qa-bootstrap` + `eng-real-scenario-qa` + `qa-execution` before any lab work. Honor
L-016 provider-home policy and L-029 teardown (mandatory on every terminal path).

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST bootstrap a fresh QA lab via `eng-qa-bootstrap` (unique `AGH_HOME`, ports, tmux-bridge
   sockets — worktree-isolation rules if parallel) and persist the bootstrap manifest.
2. MUST respect provider home-policy contracts in the lab (L-016): `native_cli` +
   `home_policy=operator` keeps operator HOME unless the scenario tests isolation. Bound-secret/
   brokered creds use `PROVIDER_HOME`/`PROVIDER_CODEX_HOME` from the bootstrap manifest.
3. MUST execute all 17 scenarios from task_12's matrix, filling each `state.csv` verdict
   (`pass`/`fail`/`blocked-verify`) with timestamped evidence per the evidence contract (SD-006).
4. MUST register every red observation in `docs/qa/bugs/BUG-NNNN.md` (dedup against the registry
   first) with a forensic reproduction (timestamp, command, observed output) — zero unfiled reds.
5. MUST write the dated run report `docs/qa/reports/<YYYY-MM-DD>-hermes-comparison.md` (created at
   matrix time, updated incrementally): verdict summary per scenario, bug list, bootstrap block
   (manifest path, lab root, runtime home, base URL, evidence paths), and the release-readiness
   Final Status per task_12's criteria, distinguishing "verified in lab" from "covered by
   unit/integration only".
6. Isolated Web QA MUST export `AGH_WEB_API_PROXY_TARGET` derived from the manifest — never
   hardcoded `:2123`; config writes against the lab home run sequentially.
7. MUST tear down the lab on every terminal path (pass/fail/blocked/abort) via
   `eval "$TEARDOWN_COMMAND"` or `make qa-reap` (L-029); cite `teardown.json` (`"clean": true`) as
   evidence. Register long-lived lab processes at `<QA_OUTPUT_PATH>/qa/pids/<name>.pid` on spawn.
8. E2E directive (`requires_e2e=true` + `requires_cli_e2e=true`): run `make test-e2e-runtime` AND
   `make test-e2e-web`; drive highest-risk UI workflows through `browser-use:browser` (fallback
   `agent-browser`); exercise CLI/HTTP/UDS/agent-manageability paths against the lab daemon.
</requirements>

## Subtasks

- [ ] 13.1 Lab bootstrap + manifest persisted (machine-readable block into the report); export
      `AGH_WEB_API_PROXY_TARGET` from manifest; register pids.
- [ ] 13.2 Execute scenarios 1–17; fill `state.csv` verdicts + evidence; exercise CLI/HTTP/UDS +
      web E2E surfaces as required.
- [ ] 13.3 Register bugs (forensic form, dedup); close the dated report with the release verdict;
      tear down lab (L-029) and cite `teardown.json`.

## Implementation Details

Skills own the mechanics: `eng-qa-bootstrap` (lab), `qa-execution` (sessions + tracker write-back,
`qa-docs-path=docs/qa`), `eng-real-scenario-qa` (runtime-observation orchestration). Lean evidence
goes to `docs/qa/evidence/<date>-hermes-comparison/`; bulky lab artifacts stay under the
bootstrap-managed root, indexed by path from the report. Provider-home policy matches L-016.
Never parallelize config writes against one isolated QA home.

### Relevant Files

- `docs/qa/` — state.csv verdicts, bugs/, reports/, evidence/
- QA lab artifacts under the bootstrap-managed root (not the repo)
- task_12 plan — scenario rows, charters, evidence contract, verdict criteria

### Dependent Files

- none (terminal task)

### Related ADRs

- All ADR-001..010 as mapped by task_12's coverage matrix — execution verifies each scenario's
  ADR mapping with lab evidence
- TechSpec §3.10 — O1–O5 scenarios 13–17

### Competitor References

- Not applicable — validation task

## Deliverables

- Executed 17-scenario matrix with per-scenario timestamped evidence (verdicts in `state.csv`)
- Closed dated run report: verdict summary, registry bugs, bootstrap block, release-readiness
  Final Status
- Clean lab teardown (`teardown.json` `"clean": true`)

## Tests

No `_tests.md` for this suite — concrete inline cases below. The 17 scenarios ARE the E2E
surface.

- Unit: N/A — this task executes scenarios; code-level tests live in tasks 01–11
- Integration: N/A — same rationale
- E2E (lane: scenario execution + `make test-e2e-runtime` + `make test-e2e-web`):
  - [ ] All 17 scenarios from task_12 executed with timestamped evidence
  - [ ] Every `fail` row carries a registered `BUG-NNNN` (no silent skips)
  - [ ] CLI/HTTP/UDS parity exercised for agent-manageable surfaces in applicable scenarios
  - [ ] Web UI flows exercised for approval grants, suggestions, and checkpoint timeline
        (tasks 01/07/08)
  - [ ] `AGH_WEB_API_PROXY_TARGET` derived from manifest (never hardcoded)
  - [ ] Provider-home policy honored (L-016)
  - [ ] Lab teardown complete; `teardown.json` reports `"clean": true` (L-029)
- All defects found MUST reproduce before being filed (SD-006)

## Success Criteria

- 17/17 scenarios executed with timestamped evidence (every row terminal in `state.csv`; `fail`
  rows carry a registered `BUG-NNNN` — no silent skips)
- Report closed: zero evidence-free claims, Final Status explicit, blocking defects enumerated
  with reproductions
- Continuation can resume the lab from the bootstrap block alone (within the same active QA loop)
- Lab processes reaped; `teardown.json` clean
