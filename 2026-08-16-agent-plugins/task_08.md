---
status: completed
title: Real-User QA Execution
type: qa-execution
complexity: critical
---

# Task 8: Real-User QA Execution

## Overview

Executes the chartered cycle in-persona against a real daemon: every flagged scenario walks to a recorded verdict, the three-provider delivery matrix and 8-item conformance walk produce the evidence task_09 cites, browser e2e covers the UI surfaces, and every reproduced defect lands in the content-addressed bug registry with the fix-loop governor applied.

<critical>ALWAYS READ the in-scope docs/qa/scenarios/ files, open docs/qa/bugs/, and the cycle's charters in docs/qa/charters/ before executing.</critical>

<requirements>
1. MUST activate `qa-execution` with `qa-docs-path=docs/qa`, plus `eng-real-scenario-qa` + `eng-qa-bootstrap` for the runtime lab (fresh lab, bootstrap manifest, in-persona operator kickoff), and `eng-worktree-isolation` (unique `COMPOZY_HOME`, daemon port, tmux socket — default home/port forbidden).
2. MUST run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright), and drive the highest-risk UI workflow (marketplace card → install → trust → inventory Skipped rows) through `browser-use:browser`, falling back to `agent-browser` only if unavailable — never shell-only substitutes (UI-bearing feature: `_uiux.md` exists).
3. MUST exercise the CLI/API/agent-manageability surfaces end-to-end against the daemon-served runtime: structured CLI output vs HTTP/UDS responses for the same persisted state, status/config discovery, deterministic error codes, consent flows.
4. MUST walk every scenario flagged by tasks 02–06 to a recorded verdict; a failing walk means fix the production code (fix-loop governor: small/contained only, regression test red-before/green-after, one logical fix per commit) — escalate the rest to "Decisions for a Human". Only recorded `blocked-verify`/`blocked-decision` may stay unwalked.
5. MUST execute the provider-matrix charter (Claude Code, OpenClaw, Hermes real sessions consuming the fixture package) and the 8-item conformance walk, recording per-item evidence paths in the run report — task_09's gate.
6. MUST register every reproduced defect as `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup first), update scenario verdicts, and write the dated report at `docs/qa/reports/<YYYY-MM-DD>-agent-plugins.md`.
7. MUST end every lab envelope with `eval "$TEARDOWN_COMMAND"` or `make qa-reap` on every terminal path, citing `teardown.json` (`"clean": true`) — live processes at completion are a blocking failure (L-029). Exit gate: `make gate-full` current before Final Status.
</requirements>

## Subtasks

- [x] 8.1 Bootstrap isolated lab (`eng-qa-bootstrap`); record manifest + pids
- [x] 8.2 Walk the install/lifecycle + authoring/validate scenario set (CLI + native tools)
- [x] 8.3 Walk the secrets/remote + degraded/diagnostics scenario set
- [x] 8.4 Browser e2e pass (marketplace + settings surfaces) via `browser-use:browser`
- [x] 8.5 Provider-matrix charter execution + conformance walk with evidence paths
- [x] 8.6 Bug registry + scenario verdicts + dated report
- [x] 8.7 Fix loop (governed) for reproduced defects; re-walk to green
- [x] 8.8 Teardown + `make gate-full` + Final Status

## Implementation Details

The charters from task_07 are the execution contract; `_dx.md` transcripts are the expected outputs; `docs/qa/` is the only durable home. Provider auth follows the provider contract (bound-secret/brokered → isolated `PROVIDER_HOME`; `native_cli` + operator home policy preserved unless the scenario tests isolation).

### Relevant Files

- `docs/qa/charters/` (cycle), `docs/qa/scenarios/` (flags), `docs/qa/bugs/` (registry)
- `.compozy/tasks/agent-plugins/_dx.md` — expected transcripts
- Bootstrap manifest + `<QA_OUTPUT_PATH>/qa/pids/` — lab lifecycle

### Dependent Files

- `docs/qa/reports/<date>-agent-plugins.md` — the evidence artifact task_09 cites

### Related ADRs

- [ADR-003](adrs/adr-003.md) — diagnostics visibility is a primary verification target

### Web/Docs Impact

- `web/` / `packages/site`: none authored — verification only; defects route through the fix loop or escalation.
- QA impact: this task records the verdicts for every flagged scenario (the verify half of flag-then-verify).

### Extensibility / Agent Manageability / Config Lifecycle

- none authored — checked surfaces: verification-only task; findings escalate rather than mutate design.

## Deliverables

- Every flagged scenario walked to a recorded verdict; bug registry + dated report written
- Provider-matrix + conformance-walk evidence recorded with paths
- Teardown proof (`teardown.json` clean) + current `make gate-full` record
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED — none newly assigned; this task executes the shipped suites + scenario walks)**

## Tests

Cases assigned from `_tests.md`: none new — runs `make test-e2e-runtime`, `make test-e2e-web`, and the full gate as verification instruments; automated case ownership stays with tasks 01–05.

## Success Criteria

- Zero scenarios left `untested`/`fail` (only recorded `blocked-*` exceptions)
- Conformance + provider evidence complete enough for task_09 to cite by path
- `make gate-full` current at close; lab processes provably dead
