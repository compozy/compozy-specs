---
status: completed
title: QA Plan and Session Charters
type: qa-report
complexity: high
---

# Task 6: QA Plan and Session Charters

## Overview

Plan the real-user QA cycle for the mode-anchored cross-workspace access program on the living `docs/qa/` tree: update journeys, mint/update content-addressed scenario files, and write session charters for the cycle. This turns the techspec invariants and the named behavior deltas into an executable QA scope for task_07.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- ALWAYS READ every ADR under `adrs/` (ADR-007 defines the mode mapping) and every per-task memory file before planning
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST activate the `qa-report` skill with `qa-docs-path=docs/qa` (bootstrap the tree if absent).
- MUST express coverage as scenario `entry_points` on journey-derived rows, not standalone test cases.
- MUST cover every public surface touched by tasks 01–05: the mode-mapped enforcement seams (native tool calls, agent-driven CLI, HTTP/UDS identity, task claim, spawn, coordination), the ACP prompt flow (`allow_once | allow_session | reject_once | reject_session` semantics as shipped), audit event visibility (`compozy logs --type <event-type>` / `GET /api/logs` plus native `compozy__logs`/`compozy__observe_search`), the owner projection route, the web deep-link confirm flow, and the updated site/skill docs pages.
- MUST map regression hot spots from `_techspec.md` Safety Invariants (esp. invariant 1 named deltas, invariant 7 deny-all-never-prompts, invariant 9 consent volatility) and ADR-007 risks into the cycle's charter selection (targeted tier + one adjacent canary journey).
- MUST include the scenarios flagged `untested` by task_05 (mode matrix, prompt outcomes, deep-link confirm, rewritten `ET-web-session-deep-link-isolation`) as this cycle's scope.
- MUST dedup against existing `docs/qa/scenarios/` and `docs/qa/bugs/` entries before minting new files (content-addressed ids).
</requirements>

## Subtasks

- [x] 6.1 Read the shipped state: `_techspec.md`, ADR-001/004/006/007, task_01–05 completion notes, and the current `docs/qa/` tree.
- [x] 6.2 Update `docs/qa/journeys/` flowcharts with the cross-workspace access journeys (agent hits boundary per mode; operator deep-link switch).
- [x] 6.3 Mint/update content-addressed scenario files in `docs/qa/scenarios/` covering the mode matrix across seams, prompt outcome matrix incl. session-consent reuse and expiry-on-stop, audit visibility, and the rewritten deep-link contract.
- [x] 6.4 Write session charters in `docs/qa/charters/` for this cycle: persona-driven charters for operator (mode configuration, prompt answering, deep-link) and agent (boundary denials, hint copy, no-self-widening) perspectives.
- [x] 6.5 Select the charter tier: targeted charters for the new behavior + one adjacent canary journey (existing workspace isolation regression).
- [x] 6.6 Verify every public surface touched by tasks 01–05 appears as an `entry_point` in at least one scenario row.

## Implementation Details

Planning-only task operating on the committed `docs/qa/` tree. No runtime code changes. Scenario files use content-addressed ids; `state.csv` is generated output and never hand-edited.

### Relevant Files

- `docs/qa/scenarios/` — scenario rows for this cycle (including the ones task_05 flagged `untested`; neighbors to dedup against: `ET-native-workspace-scope-isolation`, `MS-workspace-resolution-chain`).
- `docs/qa/journeys/J-operate-workspace-context.md` — its Mermaid DENY/END nodes encode the old refusal semantics and must reflect the confirm-first + mode-gated contract; other journeys as coverage demands.
- `docs/qa/charters/` — cycle charters to write.
- `docs/qa/bugs/` — registry to dedup against.
- `.compozy/tasks/cross-workspace-access/_techspec.md` — invariants and named deltas driving hot-spot selection.

### Dependent Files

- `docs/qa/reports/` — task_07 writes the dated report against this plan.

### Related ADRs

- [ADR-007: PermissionMode anchoring](adrs/adr-007.md) — the mode mapping and accepted trade-offs QA must exercise.
- [ADR-004: Deep links prompt before switching](adrs/adr-004.md) — the rewritten web contract.
- [ADR-006: Beta posture](adrs/adr-006.md) — accepted gaps that QA documents but does not fail on.

## Deliverables

- Updated `docs/qa/journeys/` flowcharts for cross-workspace access.
- Scenario files minted/updated in `docs/qa/scenarios/` (content-addressed, journey-derived, with `entry_points`).
- Cycle charters in `docs/qa/charters/` (targeted tier + canary journey).
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

No `_tests.md` IDs are assigned to this planning task — the automated contract is fully owned by tasks 01–04. This task's verification is structural:

- [x] Every public surface touched by tasks 01–05 appears as a scenario `entry_point` (audited list in the charter).
- [x] Every scenario file passes the `docs/qa/` schema conventions (content-addressed id, `qa_status`, journey linkage).
- [x] Charter selection names the regression hot spots from invariants 1, 7, and 9 explicitly.

## Success Criteria

- Every assigned test case implemented and passing
- `docs/qa/` tree contains journeys, scenarios, and charters sufficient for task_07 to execute without further planning.
- No duplicate/conflicting scenario ids against the existing registry.

## Completion Notes

- Added `J-cross-workspace-access` for the six enforcement entry paths, three permission-mode
  outcomes, four approval answers, cross-seam session-consent reuse, stop/restart expiry, and
  best-effort audit observability. Updated `J-operate-workspace-context` so foreign input
  canonicalizes and reaches the shared policy before any handler instead of being unconditionally
  rejected.
- Added `J-open-foreign-session` for canonical and short permalinks, the three-field owner projection,
  routed confirmation, confirm/cancel/unknown outcomes, and pre-confirmation data isolation. Updated
  the existing open/return journeys with narrow ownership links rather than duplicating the flow.
- Re-homed the five affected `untested` scenarios and widened their `entry_points` to 32 audited
  public-surface groups, including explicit HTTP/UDS parity, exact approval CLI syntax, all four
  audit readers, both permalink forms, every changed site page, and both changed official-skill
  references. No new scenario was minted because the existing five rows already own the behavior.
- Added four immutable targeted charters: mode/seam Feature Tour, consent/audit Interrupt Tour,
  foreign-link Back-Button Tour, and same-workspace Feature Tour canary. Their coverage names safety
  invariants 1, 7, and 9 plus the adjacent risks they exercise.
- Claude Opus produced the bounded QA document slice through Herdr direct mode; the controller
  corrected best-effort audit wording, executable approval syntax, and the complete documentation
  surface inventory, then independently revalidated the result. The worker was retired.

Compozy Impact Audit:

- Native tools: QA-plan impact only — checked `compozy__workspace_info`, memory, automation, hooks,
  task-claim, denial reason/hint, prompt options, audit readers, and capability behavior; no tool ID,
  descriptor, schema digest, risk flag, diagnostic, or capability gate changed.
- Extensibility and hooks: no runtime impact — checked extensions, hooks, skills/capabilities,
  tools/resources, bundles, registries, bridge SDKs, MCP sidecars, and config lifecycle. The plan
  explicitly verifies the changed site and official-skill guidance against runtime behavior.
- Workspace data isolation: QA-plan impact only — scenarios distinguish actor-home audit scope,
  canonical target scope, volatile session consent, global operator-only owner projection, and
  workspace-scoped detail/transcript/cache data; Task 07 must collect live cross-surface evidence.
- Official Compozy skill: no file change — checked and added both changed references as QA entry
  points so Task 07 verifies their guidance against observed behavior.

VERIFICATION REPORT
-------------------
Claim: Task 06 leaves an executable, deduplicated living QA plan for Task 07.
Commands: canonical `materialize_state.py docs/qa`; YAML parsing for all changed blocks; scripted
journey, charter, invariant, and public-surface audit; `git diff --check`.
Executed: 2026-07-29 after controller corrections to the Claude Opus artifact pass.
Exit code: 0 for every accepted check.
Output summary: 660 scenarios materialized with 660 unique ids; five affected scenarios remain
`untested`; 32 required public-surface groups present; two new journeys have true ends and
abandonment paths; four targeted charters each name one tour; invariants 1, 7, and 9 are covered.
Warnings: the corpus retains pre-existing loose frontmatter that the canonical materializer accepts;
all changed YAML blocks also pass a strict parser.
Errors: none.
Contract parity: PASS — traced to the TechSpec, ADR-001/004/006/007/008, task 01–05 memories, and the
actual CLI/HTTP/UDS/site/skill surfaces.
Verdict: PASS.
