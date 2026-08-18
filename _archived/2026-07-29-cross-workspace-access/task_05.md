---
status: completed
title: Docs, official Compozy skill, glossary, QA scenario flags
type: docs
complexity: low
---

# Task 5: Docs, official Compozy skill, glossary, QA scenario flags

## Overview

Make the shipped behavior true in every knowledge surface: the site documents the mode → cross-workspace mapping and the beta posture in plain language, the bundled `skills/compozy/` skill teaches agents the boundary contract, the glossary gains the canonical entry, and the QA tracker is flagged (reset + new `untested` scenarios) for the task_06/07 cycle. Docs-only in artifact type, but it closes the program's truth loop — runtime behavior always wins over aspirational wording.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST document on the site: the mode mapping (`approve-all` crosses / `deny-all` never / `approve-reads` asks at the tool seam), the session-scoped volatile consent semantics (`allow_session` dies with the session, no management surface), the per-axis `deny-all` asymmetry (ask-everything for tools vs never-cross for workspaces), the coupled-axes trade-off, and the ADR-006/007 beta posture + accepted gaps in plain language (COPY.md claim standards apply).
- MUST update `skills/compozy/` with the boundary contract agents experience: mode mapping, prompt option semantics, the deny-hint copy verbatim, audit event types, and the guidance to surface denials to the operator instead of working around them.
- MUST add the glossary entry "Cross-workspace access" (mode-anchored definition; explicitly NOT grants/toggle — those were never shipped).
- MUST flag the QA tracker per the flag-don't-retest rule: reset `docs/qa/scenarios/ET-web-session-deep-link-isolation.md` (`expected` rewritten to the confirm-first contract, `qa_status: untested`) and add content-addressed `untested` scenarios for: the mode matrix across seams, the prompt outcome matrix (incl. session-consent reuse + expiry-on-stop), and the deep-link confirm flow. Dedup against existing scenario ids first.
- MUST follow the content-addressed scenario frontmatter shape already in the tree (id/area/title/persona/journey/expected/entry_points/qa_status/... fields).
- MUST NOT document surfaces that don't exist (no grants CLI, no Settings section, no `[workspace_access]` key — ADR-007 deleted them from the design).
- Skills: `documentation-writer`, `copywriting`, `fumadocs`; glossary edit follows `docs/_memory/` authoring conventions.
</requirements>

## Subtasks

- [x] 5.1 Update `packages/site/content/runtime/core/sessions/permissions.mdx`: extend the three-mode table (~L19-27), the workspace-boundary section (~L54-68), the decision order (~L88-102), and the surface-parity table (~L186-214) with the cross-workspace mapping, prompt options, consent lifetime, and asymmetry note.
- [x] 5.2 Update the workspace isolation narrative in the existing pages — `packages/site/content/runtime/core/workspaces/index.mdx` (workspace-as-identity framing) and `resolver.mdx`: isolation default, the mode-anchored door, operator deep-link confirm, beta posture + accepted gaps in plain language. No new page (no dedicated security page exists; keep the distributed structure).
- [x] 5.3 Cross-check `packages/site/content/runtime/core/configuration/config-toml.mdx` `[permissions]` rows (~L48, ~L282, ~L1024-1028) still read true (no new keys; wording gains the cross-workspace consequence of `mode`).
- [x] 5.4 Update `skills/compozy/` at the exact touchpoints: `references/native-tools.md` ~L29 (the hard-refusal sentence "Bound sessions cannot override…" — now mode-governed), `references/agent-definitions.md` ~L44 mode enumeration + ~L30 example, and the `SKILL.md` router row + Reference Index line only if a new reference file is warranted.
- [x] 5.5 Add the `docs/_memory/glossary.md` entry.
- [x] 5.6 QA tracker: rewrite `ET-web-session-deep-link-isolation` (`expected` directly contradicts confirm-and-switch — rewrite frontmatter `expected` + body repro, keep `qa_status: untested`, append the dated QA-impact note, extend `overlaps`); mint the new content-addressed `untested` scenario files (e.g. `ET-web-session-cross-workspace-confirm` per the area-code README, dedup first). Never touch `state.csv` (generated, gitignored).
- [x] 5.7 Verify docs build + links (site build lane) and that every claim matches runtime truth from tasks 01–04 completion notes.

## Implementation Details

Read `COPY.md` and `docs/_memory/glossary.md` before writing any public copy; the canonical artifact vocabulary applies (never `recipe`/`workflow`-as-artifact). Cite exact denial/hint copy from the shipped code, not from memory. The QA scenario files follow the frontmatter shape of `docs/qa/scenarios/ET-web-session-deep-link-isolation.md`.

### Relevant Files

- `packages/site/content/runtime/core/sessions/permissions.mdx` — mode documentation home (9.2K today).
- `packages/site/content/runtime/core/workspaces/index.mdx` + `resolver.mdx` — isolation narrative.
- `packages/site/content/runtime/core/configuration/config-toml.mdx` — `[permissions]` reference cross-check.
- `skills/compozy/SKILL.md` + `skills/compozy/references/` — bundled official skill.
- `docs/_memory/glossary.md` — vocabulary entry.
- `docs/qa/scenarios/ET-web-session-deep-link-isolation.md` — reset target + frontmatter exemplar.

### Dependent Files

- `docs/qa/` cycle scope — task_06 consumes the flagged scenarios as its planning input.

### Related ADRs

- [ADR-007](adrs/adr-007.md) — the mapping, trade-offs, and asymmetry to document.
- [ADR-006](adrs/adr-006.md) — beta posture + accepted gaps for the plain-language section.
- [ADR-004](adrs/adr-004.md) — deep-link confirm contract for the rewritten scenario.

### Web/Docs Impact

- `web/`: none — checked surfaces: no code change; UI copy shipped in task_04.
- `packages/site`: the pages enumerated above (this task IS the site impact).
- QA impact: this task performs the flags — `ET-web-session-deep-link-isolation` reset + new content-addressed `untested` scenarios (mode matrix, prompt outcomes, deep-link confirm). Flag, don't retest — execution belongs to task_07.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: `skills/compozy/` update is the extensibility deliverable (bundled skill = agent-facing contract surface); no other extension surfaces — checked: hooks/tools/bundles/registries/bridges/MCP unchanged by the program.
- Agent manageability: documents the agent path (deterministic denials + hint + audit reads + prompt flow); no new surfaces to document beyond what tasks 01–04 shipped.
- Config lifecycle: no key changes; the `[permissions]` reference gains consequence wording only — checked: structs/defaults/overlay/validation untouched.

## Deliverables

- Site pages updated (permissions, workspaces isolation, config cross-check) with COPY.md-compliant plain-language beta posture.
- `skills/compozy/` boundary contract update; glossary entry.
- QA tracker flagged: 1 reset + new `untested` content-addressed scenarios, deduped.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

No `_tests.md` IDs assigned — the automated contract is owned by tasks 01–04 (docs carry no runtime invariant; static-prose tests are forbidden by default). Verification gates for this task:

- [x] Site build lane green (`bunx turbo run lint typecheck test --filter=./packages/site` as applicable + site build); link integrity clean.
- [x] Every documented claim traced to shipped behavior (completion notes map claim → task/test that proves it).
- [x] Scenario files pass the docs/qa frontmatter conventions and dedup audit.

## Success Criteria

- Every assigned test case implemented and passing
- No documentation references a surface ADR-007 deleted (grep proof: no `workspace access` CLI verbs, no `[workspace_access]`, no grants/Settings mentions).
- Glossary + skill + site agree on the same mode mapping and copy.

## Completion Notes

- Documented the mode matrix, native-tool prompt options, volatile session consent, `deny-all`
  asymmetry, best-effort audit semantics, coupled-axis trade-off, and beta limitations in the existing
  permissions, workspace, resolver, and config reference pages. No new configuration or management
  surface is claimed.
- Updated the official Compozy skill's existing `native-tools` and `agent-definitions` references.
  `skills/compozy/SKILL.md` already routes native-tool and agent-definition questions to those files,
  so no redundant router growth was necessary.
- Added the canonical glossary definition and corrected the native binding scenario from unconditional
  refusal to canonicalize-then-policy. Reset the affected deep-link scenario to `untested` and added
  three deduplicated content-addressed `untested` scenarios for the mode, prompt, and operator-confirm
  journeys. No QA execution was performed in this flagging task.
- Claim trace: task 01's policy tests own mode/session-consent/audit truth; task 02's seam and runtime
  E2E own enforcement, prompt, hint, and agent-manageability truth; task 03's parity integration owns
  the minimal owner projection; task 04's router/E2E tests own confirm-before-switch and pre-confirm
  isolation.

Compozy Impact Audit:

- Native tools: documentation impact only — checked native tool IDs, descriptors, binding schemas,
  reason code, denial hint, prompt options, audit reads, and capability gates against task 02's shipped
  contract; no runtime descriptor or schema changed in this task.
- Extensibility and hooks: updated the canonical bundled `skills/compozy/` references agents use for
  native tool and agent-definition guidance; checked extensions, hooks, bundles, registries, bridge
  SDKs, MCP sidecars, and config lifecycle with no additional impact.
- Workspace data isolation: documentation and QA-tracker impact only — classified policy data as
  session-scoped volatile consent, audit data as actor-workspace-scoped, owner projection as global
  operator-scoped, and session detail/catalog data as workspace-scoped; tasks 01–04 own propagation and
  leak-prevention tests.
- Official Compozy skill: updated `references/native-tools.md` and `references/agent-definitions.md` with
  the shipped mode mapping, exact denial hint, prompt/session semantics, audit discovery, and
  no-workaround guidance; the existing router already reaches both references.

VERIFICATION REPORT
-------------------
Claim: Task 05 knowledge surfaces truthfully describe the shipped cross-workspace behavior and flag
the next QA cycle.
Commands: `bunx turbo run lint typecheck test --filter=./packages/site`; site build lane; rendered-anchor
audit; docs/qa frontmatter parser and duplicate-id audit; stale-surface and formatting sweeps.
Executed: 2026-07-29 after controller corrections to the Claude Opus direct-mode artifact pass.
Exit code: 0 for every lane.
Output summary: Site Turbo 7/7 with 51 files and 248 tests; build 4/4 with 1,845 generated pages; all
touched rendered anchors resolved; 660 QA scenarios parsed with zero duplicate IDs; all five affected
scenarios are `untested`; stale-surface, formatting, and diff checks are clean.
Warnings: none.
Errors: none.
Contract parity: PASS — documentation claims were traced to tasks 01–04 completion evidence and current
runtime/CLI surfaces.
Verdict: PASS.
