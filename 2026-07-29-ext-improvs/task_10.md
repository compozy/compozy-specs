---
status: completed
title: QA Plan and Session Charters
type: qa-report
complexity: high
---

# Task 10: QA Plan and Session Charters

## Overview

Plan the real-user QA cycle for the extension DX program over the living `docs/qa/` tree: update journeys, mint/update scenario files for every public surface tasks 01–09 touched, and select this cycle's session charters from the TechSpec's regression hot spots and the brief's scorecard targets.

<critical>ALWAYS READ _techspec.md, every ADR, and every per-task memory file before planning.</critical>

## Requirements

- Activate the `qa-report` skill with `qa-docs-path=docs/qa` (bootstrap the tree if absent).
- Output: journey flowcharts updated in `docs/qa/journeys/`, scenario files minted/updated in `docs/qa/scenarios/` (content-addressed ids; consolidate the flags accumulated by tasks 02–09), session charters in `docs/qa/charters/` for this cycle.
- Coverage: every public surface touched by tasks 01–09 — the 9 new/changed CLI verbs (`init|build|validate|dev|reload|logs|publish|commands|exec`), the 5 new HTTP/UDS routes, the source-union install request, the nine native tools, web marketplace/extensions surfaces, doc pages, extension points (manifest v2, hook source, command metadata), and every `[extensions.*]` config key — expressed as scenario `entry_points` on journey-derived rows, not standalone test cases.
- Map regression hot spots from `_techspec.md` Safety Invariants 1–17 and ADRs 001–008 into the charter selection (targeted tier + one adjacent canary journey — marketplace/skills flows share surfaces with extensions).
- Bind the brief's scorecard targets as cycle acceptance measures: concepts ≤10, first-success actions ≤4, own-install ≤1 command + 1 consent, passive update discovery, SDK re-grades ≥ A−/B/B.
- Include the newcomer-outside-the-repo journey (release-stamped binary, public docs verbatim) and the agent-driven authoring journey (native tools + `skills/compozy/` guidance) as first-class journeys.

## Deliverables

- Updated `docs/qa/journeys/` + scenario files + this cycle's charters committed
- Charter selection rationale referencing invariants/ADRs
- No per-round `qa/` trees; `state.csv` remains generated output only

## Success Criteria

- Every public surface from tasks 01–09 reachable through at least one scenario `entry_point`
- Charters cover the targeted tier + one canary journey
- Scenario set includes the authoring path (scaffold→build→validate→dev→reload→publish→install→invoke→update→remove)

## Completion Evidence

- Added first-class newcomer, agent-authoring, dev-lifecycle, and distribution journeys; hard-cut the policy journey to current configuration/declaration truth; extended the command journey with its manifest metadata boundary. Every new or revised flow has a true end and at least one abandonment/resume path.
- Consolidated the Task 02–09 flags into the ET-015–ET-023 family and content-addressed extension rows, then added `ET-extension-manifest-v2-surfaces` and `ET-extension-dx-scorecard`. All cycle rows remain `qa_status: untested` for Task 11.
- Selected six targeted charters and one adjacent Marketplace/Skills canary. Their selection rationales map Safety Invariants 1–17 and ADRs 001–008 to an owning mission.
- A generated entry-point audit found all 44 required contract markers: nine CLI verbs, five HTTP/UDS routes, the source-union request, nine new native tools, exact extension config keys, Web surfaces, public docs, official-skill guidance, and manifest/hook/command metadata.
- The binding acceptance measures are explicit scenario/charter observables: concepts ≤10, first success ≤4 actions, own install ≤1 command + 1 consent, passive update discovery, and SDK re-grade ≥ A−/B/B.

VERIFICATION REPORT
-------------------
Claim: Task 10 QA-plan artifacts and contract coverage pass; the monorepo `make verify` remains reserved for final Task 11 completion by operator instruction.
Command: `python3 .agents/skills/qa-report/scripts/materialize_state.py docs/qa`; Ruby charter/journey reference and YAML validators; 44-marker scenario entry-point audit; `git diff --check`.
Executed: 2026-07-29, after all Task 10 deliverable changes.
Exit code: 0 for every command.
Output summary: tracker materialized 676 scenarios; seven cycle charters resolved every scenario and journey reference; five extension journey YAML blocks have a true end and abandonment path; all 44 required public entry-point markers were present; diff check was clean.
Warnings: none.
Errors: none.
Contract parity: PASS against `_brief.md` scorecard/verification requirements, `_techspec.md` Safety Invariants 1–17 and ADRs 001–008, all task memories 01–09, and Task 10's complete public-surface list.
Visual contract: n/a — no named visual reference found; the operator waived visual evidence for this planning task.
Verdict: PASS for Task 10 focused scope.

## Compozy Impact Audit

- Native tools: no runtime descriptor or schema changed. Scenario entry points and the agent-authoring charter explicitly cover all nine new IDs (`extensions_init|build|validate|dev|reload|logs|search|provenance|publish`), their risk gates, and canonical lifecycle-tool handoffs.
- Extensibility and hooks: manifest v2 provides/permissions, extension hook source, command groups/tool metadata, public SDK harnesses, source-union distribution, official-skill guidance, and config lifecycle all have journey-derived scenario and charter owners. No runtime hook, registry, bundle, bridge, MCP, or SDK code changed.
- Workspace data isolation: no runtime datum changed. The dev-recovery and agent-authoring charters bind dev links, reload, logs/SSE, native reads, handler identity, and global-vs-workspace instances to trusted server-side workspace scope.
- Official Compozy skill: no skill content changed. `skills/compozy/SKILL.md` and `references/extension-authoring.md` are first-class entry points in the agent journey and charter.
