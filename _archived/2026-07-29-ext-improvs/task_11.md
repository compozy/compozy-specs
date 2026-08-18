---
status: in_progress
title: Real-User QA Execution
type: qa-execution
complexity: critical
---

# Task 11: Real-User QA Execution

## Overview

Execute the planned QA cycle as a real user against a release-stamped runtime: persona-driven sessions over the charters from task_10, evidence-backed verdicts on the in-scope scenario files, defects into the durable bug registry, and a dated run report — closing the extension DX program.

<critical>ALWAYS READ the in-scope docs/qa/scenarios/ files, open docs/qa/bugs/, and the cycle's charters in docs/qa/charters/ before executing.</critical>

## Requirements

- Activate `qa-execution` with `qa-docs-path=docs/qa`. For release-grade scope on the Compozy runtime, also activate `eng-real-scenario-qa` (playbook lab via `eng-qa-bootstrap` + operator kickoff + runtime observation).
- Activate `eng-worktree-isolation` (unique `COMPOZY_HOME` + daemon ports + tmux socket) when concurrency is signaled; never default home/port.
- Run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright). Drive the highest-risk UI workflow (marketplace install forms + extension logs panel) through `browser-use:browser`; fall back to `agent-browser` only if `browser-use:browser` is unavailable. Do not silently substitute shell-only checks.
- Exercise the affected CLI verbs, HTTP/UDS routes, native tools, status/config discovery, and deterministic errors end-to-end against a daemon-served runtime (unique `COMPOZY_HOME`); compare structured CLI output with HTTP/UDS responses for the same persisted state.
- Validate the extensibility/config lifecycle: manifest v2 load matrix on a stamped binary, `[extensions.*]` keys via `compozy config set` (sequential per home — never parallel config writes), legacy-key rejection UX.
- Execute the newcomer journey outside the repo on a release-stamped binary (the `dev`-build version gate short-circuit must NOT mask the compat floor) and the agent-driven loop via native tools + official skill.
- Register every reproduced defect in `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup against the registry first) and link it in the affected scenario files.
- Fixes follow the fix-loop governor: small/contained only, regression test red-before/green-after, one logical fix per commit; escalate the rest to "Decisions for a Human".
- Update scenario-file verdicts and write the dated run report at `docs/qa/reports/<YYYY-MM-DD>-<slug>.md`. Exit gate: re-run gates (`make verify`) before Final Status.
- <critical>QA process teardown is mandatory (L-029): every lab ends with `eval "$TEARDOWN_COMMAND"` or `make qa-reap` on every terminal path; cite `teardown.json` (`"clean": true`) as evidence; register long-lived PIDs at `<QA_OUTPUT_PATH>/qa/pids/<name>.pid` on spawn.</critical>

## Deliverables

- Scenario verdicts updated across the in-scope set; bugs registered content-addressed; dated report in `docs/qa/reports/`
- Scorecard measurements recorded against the brief's targets (concepts, actions, install chain, update discovery, SDK grades)
- Teardown evidence (`teardown.json` clean) + `make verify` green at Final Status

## Success Criteria

- Every in-scope scenario carries an evidence-backed verdict
- The newcomer and agent journeys complete on a release-stamped binary following public docs verbatim
- Zero lab processes alive at exit; machine-readable QA bootstrap block reported for continuations
