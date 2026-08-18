---
status: completed
title: Real-User QA Execution
type: qa-execution
complexity: critical
---

# Task 7: Real-User QA Execution

## Overview

Execute the QA cycle planned by task_06 against a daemon-served runtime the way a real operator and real agents experience it: persona-driven sessions across CLI, HTTP/UDS, native tools, and the web SPA, with evidence, bug registration, and a dated report on the living `docs/qa/` tree.

<critical>
- ALWAYS READ the in-scope docs/qa/scenarios/ files, open docs/qa/bugs/, and the cycle's charters in docs/qa/charters/ before executing
- ALWAYS READ `_techspec.md` Safety Invariants and ADR-007 before judging expected behavior — daemon truth wins
- QA process teardown is mandatory (L-029): end every terminal path with `eval "$TEARDOWN_COMMAND"` or `make qa-reap`; cite `teardown.json` (`"clean": true`)
- TESTS REQUIRED — implement every verification listed in ## Tests
</critical>

<requirements>
- MUST activate `qa-execution` with `qa-docs-path=docs/qa`; for release-grade runtime scope also activate `eng-real-scenario-qa` (playbook lab + operator kickoff + runtime observation).
- MUST bootstrap a fresh lab via `eng-qa-bootstrap` and activate `eng-worktree-isolation`: unique `COMPOZY_HOME`, daemon ports, and `tmux-bridge` socket (L-009); default home/port is forbidden.
- UI features: MUST run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright); drive the highest-risk UI workflow (foreign-session deep-link confirm) through `browser-use:browser`, falling back to `agent-browser` only if unavailable. Do not silently substitute shell-only checks.
- CLI/API/agent-manageability: MUST exercise the deterministic denial contracts end-to-end — agent-driven `compozy` CLI inside a session (exit 77 + mode hint, daemon-origin), HTTP/UDS identity denials, `ReasonWorkspaceAccessDenied` on native tools — and compare structured CLI output with HTTP/UDS responses for the same persisted state.
- MUST verify the mode matrix live: `deny-all` never prompts (zero `session/request_permission` traffic), `approve-reads` prompts at the tool seam only, `approve-all` crosses promptless, session consent dies with the session (restart/new-session re-prompts).
- MUST verify audit visibility: every crossing/denial appears via `compozy logs --type <event-type>` / `GET /api/logs` and the native `compozy__logs`/`compozy__observe_search` tools with actor-home `WorkspaceID`, seam, source, and mode.
- MUST register every reproduced defect in `docs/qa/bugs/BUG-<YYYYMMDD>-<slug>.md` (dedup against the registry first) and link it in the affected scenario files.
- Fixes follow the fix-loop governor: small/contained only, regression test red-before/green-after, one logical fix per commit; escalate the rest to "Decisions for a Human".
- MUST update scenario-file verdicts and write the dated run report at `docs/qa/reports/<YYYY-MM-DD>-cross-workspace-access.md`.
- Exit gate: report the machine-readable QA bootstrap block (manifest path, lab root, runtime home, base URL, verification evidence). The single full `make verify` is reserved for the program's final Phase E gate after review remediation.
</requirements>

## Subtasks

- [x] 7.1 Bootstrap an isolated lab (`eng-qa-bootstrap`; unique home/ports/sockets) and record the bootstrap manifest.
- [x] 7.2 Execute the operator charters: configure agent `permissions.mode` values, answer prompts (`allow_once`/`allow_session`/`reject_*`), flip a session's agent to `approve-all`, observe audit events.
- [x] 7.3 Execute the agent charters: drive boundary hits per mode across native tools, agent CLI, task claim, and spawn; verify hint copy and deterministic errors.
- [x] 7.4 Execute the web charter through `browser-use:browser`: foreign-session deep link → confirm dialog → switch; cancel path; nonexistent session path.
- [x] 7.5 Run the E2E gates: `make test-e2e-runtime` and `make test-e2e-web`.
- [x] 7.6 Register bugs (content-addressed, dedup first), apply governor-scoped fixes with red-before/green-after regression tests, escalate the rest.
- [x] 7.7 Update scenario verdicts, write the dated report, tear down the lab (`teardown.json` clean), and emit the QA bootstrap block; defer the single full `make verify` to final Phase E after review remediation.

## Implementation Details

Execution-only task over the plan from task_06. Provider-home policy follows the provider contract from the bootstrap manifest (`native_cli` + `home_policy = operator` preserves operator HOME unless a charter tests isolation). Isolated Web QA exports `COMPOZY_WEB_API_PROXY_TARGET` derived from the manifest — never hardcode `:2123`. Never parallelize `compozy config set` writes against one isolated home.

### Relevant Files

- `docs/qa/charters/` — the cycle's charters (from task_06).
- `docs/qa/scenarios/` — in-scope scenario files, incl. the rewritten `ET-web-session-deep-link-isolation`.
- `docs/qa/bugs/` — registry for dedup + new entries.
- `docs/qa/reports/` — dated run report target.

### Dependent Files

- `docs/qa/scenarios/*.md` — verdict updates (`qa_status` transitions).

### Related ADRs

- [ADR-007: PermissionMode anchoring](adrs/adr-007.md) — expected mode behavior per seam.
- [ADR-006: Beta posture](adrs/adr-006.md) — accepted gaps: document, do not fail the run on them.
- [ADR-004: Deep links prompt before switching](adrs/adr-004.md) — web contract under test.

## Deliverables

- Executed charters with evidence indexed by path in the lab (run-scratch only; durable state in `docs/qa/`).
- Bug registry entries for every reproduced defect; governor-scoped fixes committed with regression tests.
- Updated scenario verdicts + dated report `docs/qa/reports/<YYYY-MM-DD>-cross-workspace-access.md`.
- Machine-readable QA bootstrap block + `teardown.json` (`"clean": true`) evidence.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

No new `_tests.md` IDs — automated cases ship with tasks 01–04. This task's gates:

- [x] `make test-e2e-runtime` and `make test-e2e-web` green on the completed program.
- [x] Mode matrix verified live per charter evidence (deny-all zero-prompt, approve-reads prompt-at-tool-seam-only, approve-all promptless, consent expiry on stop).
- [x] CLI vs HTTP/UDS cross-surface comparison recorded for denial contracts and audit reads.
- [x] Focused runtime/Web gates green; the program-level `make verify` remains deferred to final Phase E.

## Success Criteria

- Every assigned test case implemented and passing
- Every in-scope scenario carries an updated verdict; every defect is registered or fixed under the governor.
- Dated report exists and names Final Status; teardown evidence is clean.

## Completion Notes

- Bootstrapped fresh isolated lab `northstar-pay-20260729-124649-419333` at the manifest-assigned
  runtime home, UDS, port `55292`, Web proxy, provider homes, and tmux bridge. The exact teardown
  completed with `teardown.json` reporting `clean: true` and `survivors: []`.
- The six selected feature scenarios pass or pass-after-fix across native tools, agent CLI,
  HTTP/UDS, audit readers, same-workspace resolution, and both Web permalink forms. Four browser
  captures and the complete live matrix are indexed from the dated report.
- Fixed `BUG-20260729-coordination-cli-drops-agent-identity` in `4ef8e8c` and
  `BUG-20260729-nearest-workspace-case-alias` in `4e81f17`. The first invariant belongs to the CLI
  agent-transport suites; the second belongs to the workspace resolver suite. Both regressions were
  red before their owning production corrections and green afterward.
- Focused evidence passed: 1,329 CLI race tests, 107 workspace race tests, zero-issue `make lint`,
  `make test-e2e-runtime`, and `make test-e2e-web`.
- The autonomous playbook completed all 12 Tasks and produced 19 successful Network sends, but only
  one of three reviews reached a verdict, none of the three declared disruption seeds was delivered,
  no resolved disagreement is evidenced, and runtime-owned journey progress remains absent. The
  strict audit truthfully retains C11/C17 plus deferred C14; `BUG-0028` and
  `BUG-20260719-autonomous-progress-unobservable` own those out-of-scope scenario gaps.
- The program-level `make verify` remains intentionally deferred to final Phase E after the requested
  deep-review remediation rounds.

### Compozy Impact Audit

- **Native tools:** no tool ID, toolset, descriptor, I/O schema, digest, risk flag, or capability
  gate changed in Task 07. Live `workspace_info`, `logs`, and `observe_search` behavior was checked;
  only CLI identity forwarding and workspace discovery production code changed.
- **Extensibility and hooks:** no extension, hook, capability, skill/bundle registry, bridge SDK,
  MCP sidecar, or config lifecycle changed. The existing workspace-access policy and audit hooks were
  exercised without adding a surface.
- **Workspace data isolation:** actor, target, nested, and broad-control workspace IDs were traced
  through CLI, HTTP/UDS, native tools, audit events, owner projection, Web routing, caches, and SSE
  behavior. The CLI fix preserves the validated actor session; the resolver fix selects filesystem-
  identical registered roots without exposing foreign data. Pre-confirm Web traffic fetched only the
  global owner projection and actor-scoped 404 detail.
- **Official Compozy skill:** no public command, tool ID, hook event, capability, bundle/resource,
  memory, Network, or Task semantic changed beyond the already documented contract; checked
  `skills/compozy/` and no update is required.
