---
status: in_progress
title: Real-User QA Execution
type: qa-execution
complexity: critical
---

# Task 7: Real-User QA Execution

## Overview

Executes the cycle task_06 planned: walks every in-scope scenario to a recorded verdict on macOS + Linux against the shipped shell, drives the browser-e2e coverage, exercises the CLI/HTTP/UDS agent surfaces end to end, runs the real beta N→N+1 auto-update gate, registers defects, and closes the program only when the Part I release gate holds.

<critical>ALWAYS READ the in-scope docs/qa/scenarios/ files, open docs/qa/bugs/, and the cycle's charters in docs/qa/charters/ before executing.</critical>

<requirements>
1. MUST activate `qa-execution` with `qa-docs-path=docs/qa`, plus `eng-real-scenario-qa` (release-grade runtime scope: playbook lab via `eng-qa-bootstrap`, in-persona operator kickoff, runtime observation, strict audit).
2. MUST activate `eng-worktree-isolation` when concurrency is signaled: unique `COMPOZY_HOME`, daemon ports, tmux sockets; isolated Web QA derives `COMPOZY_WEB_API_PROXY_TARGET` from the bootstrap manifest — never hardcoded.
3. UI surfaces: run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright) AND the new `make test-e2e-desktop` lane; drive the highest-risk UI workflow (settings apply → progress → restart truth) through `browser-use:browser`, falling back to `agent-browser` only if unavailable. Do not silently substitute shell-only checks.
4. CLI/API/agent surfaces: exercise `compozy update` (all statuses incl. `blocked`/`staged`/`--cancel`), the preserved `compozy app` verbs, and `GET/POST /api/settings/update{,/apply,/cancel}` over HTTP and UDS with structured output; compare persisted state across surfaces; assert the deterministic error contracts.
5. MUST walk every in-scope scenario (`APP-*`, `REL-*`, and the program's new ids) to a recorded verdict with evidence on **both** macOS and Linux; a failing walk means fix the production code (fix-loop governor) and re-walk — never weaken the scenario.
6. MUST execute the **real release gate**: publish beta N then N+1 through the task_04 authority and observe the installed app auto-update end to end (menubar indicator → apply → restart on N+1), recording the evidence; this is the program's final gate (Part I).
7. Defects: dedup against `docs/qa/bugs/`, register as `BUG-<YYYYMMDD>-<slug>.md`, link from affected scenarios. Fixes: small/contained only, regression test red-before/green-after, one logical fix per commit; escalate the rest to "Decisions for a Human".
8. MUST write the dated run report `docs/qa/reports/<YYYY-MM-DD>-electron-shell.md`, update every scenario verdict, and re-run gates (`make verify` via `make gate-full`) before Final Status.
9. <critical>QA process teardown is mandatory (L-029): every lab ends with `eval "$TEARDOWN_COMMAND"` or `make qa-reap` on every terminal path; cite `teardown.json` (`"clean": true`); register long-lived lab processes at `<QA_OUTPUT_PATH>/qa/pids/<name>.pid`.</critical>
</requirements>

## Subtasks

- [x] 7.1 Bootstrap isolated labs (macOS + Linux) via `eng-qa-bootstrap`; provider-home policy per contract
- [x] 7.2 E2E lanes: `make test-e2e-runtime` + `make test-e2e-web` + `make test-e2e-desktop`
- [x] 7.3 Walk the `APP-*` matrix on both OSes (first run offline, attach/start, quit, links, instance, zoom, updates, recovery, agent verbs)
- [x] 7.4 Walk `REL-*` + channel scenarios (publish, repair, self-update); public-channel legs recorded as `blocked-decision`
- [x] 7.5 Browser-parity + web-update-surface walks (browser-use:browser)
- [ ] 7.6 Real beta N→N+1 gate with recorded evidence — blocked by the explicit decision to delete beta.17–beta.20 and not recreate public release state
- [x] 7.7 Bugs registered/deduped; governor-scoped fixes with red→green regressions
- [ ] 7.8 Verdicts + dated report + `make gate-full` + teardown evidence — report and teardown complete; full gate deferred to PR CI by operator direction

## Implementation Details

The walk order and gate criteria come from task_06's charters. The Part I release gate is the exit condition: all 14 `APP-*` recorded `pass` on both OSes + the real N→N+1 cycle. Any scenario left `untested`/`fail` at completion is a blocking failure; only recorded `blocked-verify`/`blocked-decision` may stay unwalked.

### Relevant Files

- `docs/qa/{scenarios,journeys,charters,bugs,reports}/` — the living contract this task operates on
- `docs/qa/state.csv` — regenerated view (never hand-edited)

### Related ADRs

- [ADR-009](adrs/adr-009.md) — the update-cycle invariants the walks must observe end to end

## Deliverables

- Every in-scope scenario at a recorded verdict with evidence (both OSes)
- Real N→N+1 gate evidence; dated run report; bug registry current
- `make gate-full` green; teardown `"clean": true` cited

## Tests

No new `_tests.md` ids — this task executes the recorded-walk contract over the scenarios and re-runs the program's gates; the automated suites (all 82 UT / 24 IT / 34 E2E) must already be green and are re-verified via `make gate-full`.

## Success Criteria

- Part I release gate holds: 14/14 `APP-*` `pass` on macOS + Linux, real N→N+1 cycle evidenced
- Zero unwalked flagged scenarios (except recorded `blocked-*`)
- Final Status backed by fresh `make gate-full` + teardown evidence
