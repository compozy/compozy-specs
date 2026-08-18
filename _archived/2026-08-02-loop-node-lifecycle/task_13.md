---
status: completed
title: Real-User QA Execution
type: qa-execution
complexity: critical
---

# Task 13: Real-User QA Execution

## Overview

Executes the task_12 plan as a real user against an isolated lab: walks every flagged scenario
persona-in-seat across CLI, HTTP/UDS, native tools, and the web UI, records verdicts with
evidence in the living `docs/qa/` tree, registers content-addressed bugs, runs the fix loop for
failures (production code fixed, never tests weakened), and tears the lab down on every terminal
path.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- R1: MUST bootstrap a fresh isolated lab via `eng-qa-bootstrap` (unique `COMPOZY_HOME`, daemon
  port, tmux-bridge socket, `COMPOZY_WEB_API_PROXY_TARGET` derived from the manifest — never
  hardcoded; L-009) and register long-lived lab processes under `<QA_OUTPUT_PATH>/qa/pids/`.
- R2: MUST walk every scenario task_12 authored/reset per the `qa-execution` contract, recording
  verdicts (`pass`/`fail`/`blocked-*`) with evidence paths; a flagged scenario left
  `untested`/`fail` at completion is a blocking failure.
- R3: UI-bearing walks use Playwright/browser evidence via `browser-use:browser` (fallback
  `agent-browser`); CLI/API walks use `make test-e2e-runtime` plus structured CLI output vs
  HTTP/UDS state comparison (agent-operability proof).
- R4: Failures follow the fix-loop governor: content-addressed `docs/qa/bugs/BUG-*.md`,
  red-before/green-after regression at the owning layer, production fix (never weakened tests),
  re-walk to `pass`.
- R5: MUST run `make gate-full` once at workstream close (this is the train's final task) and
  cite `make gate-status` as evidence (`cy-final-verify`).
- R6: <critical>Teardown is mandatory on every terminal path (L-029): `eval "$TEARDOWN_COMMAND"`
  or `make qa-reap`; cite `teardown.json` (`"clean": true`). Files may stay; processes never
  do.</critical>
- R7: Config writes against the isolated home run sequentially — never parallelize
  `compozy config set` per home.
</requirements>

## Subtasks

- [x] 13.1 Bootstrap lab from manifest; verify isolation envelope
- [x] 13.2 Persona walks: loop author (CLI/API authoring + **editor-authoring-walk** browser evidence)
- [x] 13.3 Persona walks: operator (retry visibility, pause-repair, quarantine-requeue, cancel/kill, inventories, **catalog-runform-walk** — CLI + Web)
- [x] 13.4 Persona walks: approver (link journey, ladder) + managing agent (native tools only)
- [x] 13.5 Chaos walks: daemon restart mid-wait/mid-retry, session kill (death-resume), duplicate deliveries
- [x] 13.6 Bug registry + fix loop + re-walks to green
- [x] 13.7 Dated report + scenario tracker updates + `make gate-full` + teardown evidence

The user explicitly deferred `make gate-full` until after the requested deep-review remediation so
the workstream runs the full gate once against the final tree. Task 13 owns the completed QA report,
scenario verdicts, bug fixes, browser evidence, and clean teardown; the final gate remains a
workstream closeout step.

## Implementation Details

Execute strictly from the task_12 plan; QA state lives in committed `docs/qa/`; lab holds only
run-scratch evidence indexed by path.

### Relevant Files

- `docs/qa/` — scenarios/charters/bugs/reports (durable state)
- `bootstrap-manifest.json` (lab) — env/ports/teardown source of truth

### Dependent Files

- Any production file a bug fix touches (fix loop) — recorded per bug.

### Related ADRs

- All — the walks assert the ADR-decided behaviors end to end.

## Web/Docs Impact

Verdict-driven: UI bugs found here fix production web code; docs corrections route to the
config-toml/loops pages when claims mismatch behavior.

## Extensibility / Agent Manageability / Config Lifecycle

Walks PROVE agent manageability (native-tool-only journey) and config lifecycle (family-default
tuning walk with `loop inspect` sources). No new surfaces.

## QA impact

This task consumes and resolves every flag; completion requires zero `untested`/`fail` flagged
scenarios (only recorded `blocked-*` may remain unwalked).

## Skills

`qa-execution`, `eng-qa-bootstrap`, `eng-real-scenario-qa`, `eng-worktree-isolation`,
`browser-use` (UI evidence), `systematic-debugging` + `no-workarounds` (fix loop),
`cy-final-verify` (close).

## References

- SD-005, SD-006 (forensic fixes), L-009, L-029; task_12 plan report.

## Deliverables

- Every flagged scenario walked to a recorded verdict with evidence; bugs registered/fixed/re-walked
- Dated `docs/qa/reports/<date>-loop-node-lifecycle.md`
- `make gate-full` evidence current; `teardown.json` `"clean": true`
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED — see note)**

## Tests

No new `_tests.md` IDs — this task executes scenario walks (runtime and Playwright journeys
implemented in earlier tasks, plus content-addressed browser walks including
editor-authoring-walk and catalog-runform-walk flagged in tasks 02–11) and must leave every
flagged scenario walked or `blocked-*` in the final gate. Regression tests created by the fix
loop are owned by the layer each bug belongs to (eng-consolidate-test-suites).

## Success Criteria

- Zero flagged scenarios left `untested`/`fail`; only recorded `blocked-verify`/`blocked-decision` unwalked
- `make gate-full` green once at close, evidence cited via `make gate-status`
- Teardown evidence clean on every terminal path
