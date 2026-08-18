---
status: completed
title: QA Plan and Session Charters
type: qa-report
complexity: high
---

# Task 12: QA Plan and Session Charters

## Overview

Plans the pre-publish QA cycle that gates readiness for a beta cut. Every public surface the active migration touched — renamed binary, home, environment, tool IDs, wire protocol, per-task runtime selection, the review-and-fix loop, bundled skills, input defaults, deep link, and both guide surfaces — becomes journey-derived scenario coverage in the living `docs/qa/` tree. The cycle also plans fresh IT-017 evidence for the pinned `github.com/compozy/releasepr@v0.0.24` release contract. The live config migrator and first-boot legacy-state journeys are deferred with task 14 and are not charter gates for this cycle. The separate beta installer/registry/cosign live-check checklist is authored by Task 10 and executes only after publication; it is not a Task 13 prerequisite.

<critical>
- ALWAYS READ `_techspec.md`, applicable ADRs under `adrs/`, `_brief.md` (rounds 1-12), `_content-plan.md`, `_tests.md`, and task bodies 01-11 before planning
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — do not turn post-publish checks into synthetic pre-publish evidence, skip journey derivation, or invent a beta install result without a published artifact
</critical>

<requirements>
- MUST activate the `qa-report` skill with the repo QA docs path, bootstrapping the tree only if absent.
- MUST update journey flowcharts in `docs/qa/journeys/`, mint or update scenario files in `docs/qa/scenarios/`, and write this cycle's charters in `docs/qa/charters/`.
- MUST cover every public surface touched by tasks 01-11 — CLI verbs, HTTP endpoints, UDS routes, web routes, doc pages, extension points, agent-operation paths, and `config.toml` keys — expressed as scenario `entry_points` on journey-derived rows, never as standalone test cases.
- MUST use content-addressed scenario ids for new files, deduplicating same-behavior entries rather than coordinating a shared counter.
- MUST map regression hot spots from the TechSpec's fifteen safety invariants and the eight ADRs into charter selection (targeted tier plus one adjacent canary journey).
- MUST write the exact executable scenario-ID list into each cycle charter. The Task 13 set is the union of those named targeted-tier IDs plus the one named adjacent-canary ID; it is not an implicit query for every preexisting `qa_status: untested` scenario.
- MUST include the non-technical persona lens on the hero journeys, consistent with the people-first roster already aligned in the QA tree.
- MUST plan the seeded delivery/deep-link upgrade guidance journey for the active MVP; record the live in-place config-migrator / first-boot legacy-state journey as deferred to task 14, not as a Task 13 charter gate. Record the live beta install/registry boundary as a post-publish, externally executed Task 10 single-cut checklist item rather than a Task 13 session.
- MUST plan fresh IT-017 release-PR evidence that identifies both `releasepr v0.0.24` pins, proves the explicit candidate ref equals checked-out `HEAD`, exercises leading-`v` and local/`origin` tag-collision rejection, records every authoritative planner output, and proves the workflow consumes them without re-derivation while retaining annotated tag creation. This remains read-only pre-publish evidence, not a simulated release.
- MUST plan verification that both migration-guide surfaces carry identical normalized content and that the disposition ledger accounts for every audited legacy CLI/web/extension/SDK surface.
- MUST NOT execute sessions, drive browsers, or file bugs — planning only; execution belongs to task 13.
- MUST NOT reset or edit `qa_status` verdicts as part of planning beyond the flags implementation tasks already set.
- MUST ensure every public surface changed by tasks 01-11 appears in at least one scenario's `entry_points`, while distinguishing coverage inventory from execution selection. Scenarios outside the charter ID list remain in the living backlog for later cycles.
</requirements>

## Subtasks

- [x] 12.1 Read the full spec corpus and inventory every user-visible surface changed by tasks 01-11
- [x] 12.2 Update or add journey flowcharts covering rebrand, parity, upgrade, and release-channel journeys
- [x] 12.3 Mint content-addressed scenario files for new behavior and reconcile scenarios flagged `untested` by implementation tasks
- [x] 12.4 Derive charters from the safety invariants and ADR risk surfaces, selecting a targeted tier plus one canary journey
- [x] 12.5 Plan the persona coverage including the non-technical lens on hero journeys
- [x] 12.6 Plan the active MVP delivery/deep-link journey and pinned `releasepr v0.0.24` IT-017 evidence, mark the live migrator/first-boot journey deferred to task 14, and hand the post-publish live-check list to Task 10's single-cut runbook
- [x] 12.7 Plan cross-surface parity checks (CLI vs HTTP vs UDS) for the runtime-selection and input-default paths
- [x] 12.8 Record the cycle plan and hand Task 13 an explicit deduplicated scenario-ID set for the targeted tier plus one adjacent canary

## Implementation Details

The QA tree is the living repo contract: scenario files, journeys, charters, the content-addressed bug registry, and dated reports. Plans become durable documents there, never per-round throwaway trees.

Highest-risk surfaces for charter selection, drawn from the program's invariants: runtime precedence and provenance truthfulness (invariants 1-5, 13), review artifact atomicity and path containment (6-9), rename co-ship completeness (10), bundled skill immutability (12), input-default validation (14), and single-source release planning plus workflow-owned publication (15). Migrator idempotency and non-destruction (11) remain deferred with task 14.

### Relevant Files

- `docs/qa/journeys/` — journey flowcharts to extend
- `docs/qa/scenarios/` — discover the current scenario inventory; existing ids may be numeric or content-addressed, while every new file uses a content-addressed id
- `docs/qa/charters/` — this cycle's session charters
- `docs/qa/personas.md` — the people-first roster including the non-technical persona
- `docs/qa/README.md`, `docs/qa/templates/` — tree conventions
- `_techspec.md` §Safety Invariants and `adrs/adr-001..008.md` — regression hot spots
- `_brief.md` rounds 6-12 — positioning claims, the two guide surfaces, single-cut beta delivery, the pinned release-planning contract, and the agent-authored review reset
- `.github/workflows/release.yml` and `.agents/skills/releasepr/**` — IT-017 entry points for the pinned planner, authoritative output consumption, and workflow-owned tag boundary

### Dependent Files

- `docs/qa/state.csv` — generated view, gitignored; never hand-edited
- Task 13 — consumes the charters and scenario scope produced here

### Related ADRs

- All eight ADRs inform charter selection; ADR-005 (distribution identity) drives both the pre-publish `releasepr v0.0.24` evidence and the deferred post-publish channel checklist. ADR-006 (migrator policy) informs deferred task 14 planning notes, not this cycle's executable charters. ADR-008 binds the provider-free review journey and deletes the historical CodeRabbit/watch path from executable scope.

## Cycle Scope Handoff

The targeted tier is defined by eight immutable charters; one additional charter supplies the one
adjacent canary. Task 13 executes the deduplicated union below, not every `untested` tracker row:

- `CH-compozy-platform-hard-cut`: `RT-compozy-cli-binary`, `RT-compozy-global-database`,
  `RT-compozy-home-layout`, `RT-compozy-home-isolation`,
  `RT-compozy-environment-namespace`, `ET-compozy-native-tool-invocation`,
  `ET-compozy-extension-contract-identity`, `ET-compozy-official-skill-discovery`.
- `CH-compozy-wire-public-hard-cut`: `NB-compozy-wire-identity`,
  `RT-compozy-claim-token-redaction`, `ET-compozy-public-brand-navigation`.
- `CH-compozy-mixed-runtime-delivery`: `LP-runtime-selection-overrides`,
  `LP-runtime-provenance-observation`, `LP-loop-run-deep-link`.
- `CH-compozy-run-plain-language`: `LP-runtime-provenance-observation`,
  `LP-loop-run-deep-link` (non-technical persona re-walk; duplicate IDs are deduplicated in the
  executable union).
- `CH-compozy-runtime-input-preflight`: `LP-loop-input-defaults`,
  `LP-runtime-validation-preflight`.
- `CH-compozy-agent-authored-review`: `LP-agent-authored-review-run`,
  `LP-review-artifact-inspection`, `LP-review-round-finalization`.
- `CH-compozy-dev-cycle-skills`: `ET-dev-cycle-skill-bundle`,
  `ET-dev-cycle-legacy-skill-retired`.
- `CH-compozy-beta-candidate`: `REL-release-candidate-plan`,
  `REL-migration-guide-parity`, `REL-beta-channel-contract`.
- `CH-compozy-landing-canary` (**the only adjacent canary**): `REL-os-landing-proof`.

The exact deduplicated set contains 25 IDs. It covers Tasks 01–11 as follows: Tasks 01–05 use the
two hard-cut charters; Task 06 uses the runtime/preflight charters; Task 07 uses only the current
agent-authored review rows; Task 08 uses the exact-nine-skills charter; Task 09 uses input defaults,
deep-link, and guide-parity rows; Task 10 uses the local beta-candidate rows; and Task 11 uses local
public-brand coverage plus the landing canary.

Explicitly excluded from Task 13 are `REL-beta-install-paths`,
`REL-beta-installer-provenance`, and `REL-beta-self-update` (post-publish Task-10 backlog), the
Task-14 live migrator/first-boot legacy-state journey, and the historical provider-backed review
rows/contracts such as `LP-029`.

## Deliverables

- Updated journey flowcharts covering every migration-touched surface
- Content-addressed scenario files minted or reconciled for the cycle
- Session charters targeting the invariant-derived hot spots plus one canary journey, each naming the exact scenario IDs Task 13 executes
- A finite IT-017 evidence procedure for the pinned planner and workflow-owned tag/publication boundary
- A recorded cycle plan handing scope to task 13
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from [`_tests.md`](_tests.md), the test contract — read each ID's full definition there before writing tests.

- [x] No `_tests.md` IDs are assigned to this task — it produces QA planning artifacts, not automated tests. The automated contract (66 UT + 18 IT + 4 E2E) is fully assigned across tasks 02-10.

### Web/Docs Impact

- `web/`: none — checked surfaces: `web/src/**`, `web/e2e/**`; reason: planning task producing QA documents. Web e2e execution belongs to task 13.
- `packages/site`: none — checked surfaces: `content/**`; reason: planning only. Doc pages are in scope as scenario `entry_points`, not as edits.
- Root docs: `MIGRATION_GUIDE.md` uses the same supported `text` fence label as the site guide after the canonical parity gate exposed Task 11's one-sided lexer repair; normative content is unchanged.
- QA impact: this task IS the QA tracker update — journeys, scenarios, and charters for the cycle. No runtime, UI, CLI, API, or config behavior changes; every new row remains `untested`.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: extension manifests, hooks, skills, tools/resources, bundles, registries; reason: planning artifacts only. Extension points appear as scenario `entry_points` to be exercised in task 13.
- Agent manageability: none — checked surfaces: CLI verbs, HTTP/UDS routes, structured output; reason: no runtime surface changes. Agent-operation paths are planned as coverage, not modified.
- Config lifecycle: none — checked surfaces: `config.toml` keys; reason: planning only. The new `[loops.inputs.*]` and `runtime_*` keys are planned as scenario coverage.

### Compozy Impact Audit

- Native tools: no impact — checked tool IDs, toolsets, descriptors, schema digests, capability gates, and CLI/API fallbacks; this task plans coverage without changing contracts.
- Extensibility and hooks: no impact — checked extensions, hooks, skills/capabilities, tools/resources, bundles, registries, bridge SDKs, MCP sidecars, and config lifecycle; they are scenario scope only.
- Workspace data isolation: no impact — checked workspace/session/agent scope, CLI/HTTP/UDS/core/store/web/SSE/cache/event propagation; no runtime datum or route changes.
- Official Compozy skill: no impact — checked `skills/compozy/`; planning artifacts change no public tool, CLI path, hook event, capability, bundle/resource, or memory/network/task semantic.

## Success Criteria

- Every assigned test case implemented and passing
- Every public surface touched by tasks 01-11 appears as a scenario `entry_point` on a journey-derived row
- Charters cover the invariant-derived hot spots plus one adjacent canary journey
- The charter ID union is explicit and finite; every tasks 01-11 surface is represented in scenario `entry_points` without requiring Task 13 to execute every preexisting `untested` scenario
- The active MVP delivery/deep-link journey is planned with explicit expected outcomes, the live migrator journey is recorded as deferred to task 14, and the live beta install/registry boundary is handed to Task 10 as a clearly deferred post-publish checklist
- IT-017 planning names both `releasepr v0.0.24` pins, the candidate-ref/HEAD and local/remote tag guards, the complete authoritative output set, and the no-re-derivation/workflow-owned-tag assertions without treating them as live release evidence
- New scenario ids are content-addressed with no duplicate same-behavior entries
