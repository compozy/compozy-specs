---
status: completed
title: "Real-User QA Execution"
type: qa-execution
complexity: critical
---

# Task 6: Real-User QA Execution

## Overview

Executes the cycle planned in task_05 against a real daemon-served runtime in persona: the extension-kit lifecycle end to end, the secrets/consent gates, the hard-cut negatives, the web surfaces, and the homonym keep-set — registering defects in the living bug registry, applying only governor-scoped fixes, and closing with the dated run report.

<critical>ALWAYS READ the in-scope docs/qa/scenarios/ files, open docs/qa/bugs/, and the cycle's charters in docs/qa/charters/ before executing.</critical>

<requirements>
- MUST activate `qa-execution` with `qa-docs-path=docs/qa`, plus `eng-real-scenario-qa` (release-grade runtime scope: playbook lab via `eng-qa-bootstrap`, in-persona operator kickoff, runtime observation, strict audit).
- MUST activate `eng-worktree-isolation`: unique `COMPOZY_HOME`, daemon ports, and tmux sockets from the bootstrap manifest; default home/port forbidden; isolated Web QA exports `COMPOZY_WEB_API_PROXY_TARGET` derived from the manifest.
- MUST run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright). Drive the highest-risk UI workflow (extension detail: inventory panel + confirm affordance + enable result) through `browser-use:browser`; fall back to `agent-browser` only if `browser-use:browser` is unavailable. Do not silently substitute shell-only checks.
- MUST exercise the CLI/API/agent-manageability surfaces end to end: structured CLI output (`-o json|jsonl|toon`) for `extension inventory|preview|secrets *|enable|update`, HTTP/UDS parity, native tools (`compozy__extensions_inventory|preview`, enable confirm field), deterministic errors (409 confirm w/ digest, 409 agent conflict, 400 binding classes), status/config discovery (`doctor` probe), and compare persisted state across surfaces.
- MUST verify the removal negatives (no `compozy bundle`, `/api/bundles/*` 404/absent, no bundle marketplace kind, no `compozy__bundles_*`) and the homonym keep-set (`compozy support bundle` round trip, `--source bundled` skills) in persona.
- MUST register every reproduced defect in `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup against the registry first) and link it in the affected scenario files.
- Fixes follow the fix-loop governor: small/contained only, regression test red-before/green-after, one logical fix per commit; escalate the rest to "Decisions for a Human".
- MUST update scenario-file verdicts and write the dated run report at `docs/qa/reports/<YYYY-MM-DD>-bundles-removal.md`. Exit gate: re-run `make verify` before Final Status.
- <critical>QA process teardown is mandatory (L-029): end every lab/runtime envelope with `eval "$TEARDOWN_COMMAND"` or `make qa-reap` on every terminal path; cite `teardown.json` (`"clean": true`) as evidence; register long-lived lab processes at `<QA_OUTPUT_PATH>/qa/pids/<name>.pid`.</critical>
</requirements>

## Subtasks

- [x] 6.1 Bootstrap isolated lab (`eng-qa-bootstrap`; fresh manifest)
- [x] 6.2 Execute charters in persona (kit lifecycle, secrets/consent, hard-cut negatives, homonyms, web)
- [x] 6.3 E2E gates (`make test-e2e-runtime`, `make test-e2e-web`) + `browser-use:browser` UI drive
- [x] 6.4 Bug registry entries + governor-scoped fixes + escalations
- [x] 6.5 Scenario verdicts + dated report + final `make verify`
- [x] 6.6 Teardown with `teardown.json` evidence

## Implementation Details

Charters and isolation parameters come from task_05; the scenario set is the one task_04 shipped. Provider-home policy follows the provider contract from the bootstrap manifest.

### Relevant Files

- `docs/qa/{charters,scenarios,bugs,reports}/**` — execution contract and outputs
- Bootstrap manifest + `<QA_OUTPUT_PATH>` lab tree — run-scratch evidence indexed by path

### Related ADRs

- [ADR-001..008](adrs/adr-001.md) — the behaviors under test.

## Deliverables

- Executed charters with evidence; bug registry updated; scenario verdicts current
- Dated report `docs/qa/reports/<YYYY-MM-DD>-bundles-removal.md`
- `teardown.json` with `"clean": true`

## Tests

- No `_tests.md` IDs (execution task). Gates: `make test-e2e-runtime`, `make test-e2e-web`, final full `make verify`, and the in-persona charter evidence.

## Success Criteria

- Every charter executed or explicitly blocked with evidence
- Zero lab processes alive at exit (`teardown.json` cited)
- Final Status recorded in the dated report after a green `make verify`
