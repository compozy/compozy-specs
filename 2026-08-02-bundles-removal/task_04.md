---
status: completed
title: "Docs, official skill, instruction checklists, QA scenario flags (Phase F)"
type: docs
complexity: medium
---

# Task 4: Docs, official skill, instruction checklists, QA scenario flags (Phase F)

## Overview

Ships the documentation program of the cut so every public artifact describes the extension-only kit truth: site pages deleted/rewritten (+ new kit-authoring, secrets, and operability guides), the official `skills/compozy` skill rewritten inside its rune budget, product/instruction checklists purged of the living "bundles" surface, ecosystem docs reframed, and the QA tracker flagged (scenario deletions, strips, and the six new `untested` scenarios). This is the last implementation task before the QA pair.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST execute the TechSpec **Delete Targets § Site docs** inventory: delete `core/resources/bundles.mdx`, `api-reference/bundles.mdx`, `cli-reference/bundle/**`; strip every listed section (install/develop/manifest/marketplace/resources/tools/bridges/skills-bundled/config prose); update navs (`runtime-navigation.ts`, the three `meta.json`s) — paths are the **verified** `packages/site/content/runtime/**` layout, not the brief's stale `content/docs/**`.
- MUST write the new docs: static kit authoring (dir-per-agent + sidecars, automation TOML, layout JSON, describe `resources`) in `develop.mdx` + manifest reference; secrets-binding guide; inventory/preview + network-confirm in extension operations; CLI reference regenerated (new verbs/flags, `bundle` group gone).
- MUST update the three site truth suites (`runtime-docs-discovery`, `runtime-tools-canonical-docs`, `runtime-docs-truth`) to the post-cut expectations in the same change.
- MUST rewrite `skills/compozy`: rename `references/capabilities-and-bundles.md` → `capabilities.md` with the Bundles section deleted; update `native-tools.md`, `tools-and-skills.md`, `extension-authoring.md` (+ kit/secrets/confirm/inventory teaching), SKILL.md router row — with deletions landing before additions so the startup prompt stays ≤ 32,000 runes (budget gate green).
- MUST purge the living "bundles" lines from `README.md`, `CLAUDE.md` (impact-audit template ×3), `internal/CLAUDE.md`, `AGENTS.md`/`internal/AGENTS.md`, `docs/_memory/standing_directives.md` (SD-011), `docs/_memory/spec-authoring-playbook.md`, and the `cy-web-docs-impact`/`cy-spec-preflight` skill trigger lists (activate `writing-agents-md` for instruction files, `writing-skills` for skill files).
- MUST reframe ecosystem docs: archive/delete `docs/ecosystem/bundle-opportunities.md`, rewrite the bundle pillar in `docs/ecosystem/README.md`, adjust `extension-opportunities.md`; delete the opendesign `bundle-activation-detail.html` artifact and strip the bundles kind from the shared marketplace prototype; annotate `.compozy/tasks/marketplace/**` legs as pre-hard-cut.
- MUST apply the QA tracker pass (flag, don't retest): delete `ET-024..ET-030`, `NB-023`, `ET-web-bundle-activation-detail`, `ET-web-bundle-preview-activate`; strip `ET-020`, `ET-033`, `ET-cli-marketplace-refresh`, `RT-reserved-builtin-agent-names` (+ verify the grep-positive list from the TechSpec); update journeys/charters/seeds; add the six content-addressed `untested` scenarios (`ET-ext-kit-enable`, `ET-ext-secrets-binding`, `ET-ext-inventory`, `ET-ext-preview`, `ET-ext-network-confirm`, `ET-web-extension-kit-inventory`); keep historical bugs annotated; `ET-dev-cycle-skill-bundle` and `MS-046..048` are homonyms — untouched.
- MUST update `docs/_memory/glossary.md` (bundle entries removed; kit/binding/confirm vocabulary added) and note the brief's site-path drift where relevant.
- Skills to activate: `documentation-writer`, `copywriting` (public product language), `writing-skills`, `writing-agents-md`, `fumadocs`, `qa-report` (scenario file shapes).
</requirements>

## Subtasks

- [x] 4.1 Site deletions + section strips + nav metas (actual `content/docs/**` paths)
- [x] 4.2 New guides: kit authoring, secrets binding, inventory/preview/confirm; manifest v2 reference update; CLI reference regen
- [x] 4.3 Site truth suites updated to post-cut expectations
- [x] 4.4 `skills/compozy` rewrite inside the rune budget (deletions first)
- [x] 4.5 README/CLAUDE/AGENTS/memory/skill-trigger checklist purge
- [x] 4.6 Ecosystem + opendesign cleanup; no `.compozy/tasks/marketplace/**` tree exists in this worktree to annotate
- [x] 4.7 QA tracker pass: deletions, strips, six new untested scenarios, journeys/charters/seeds
- [x] 4.8 Glossary update; docs build + full `make verify` (program completion gate)

## Implementation Details

The site/skill/QA inventories with file:line references are in `_techspec.md` **Delete Targets** (§ Site docs, § Skills / QA / ecosystem / design / memory) and **Web/Docs Impact**. New pages follow the existing extensions docs set structure; scenario files follow the content-addressed shape in `docs/qa/scenarios/` (see `qa-report` skill).

### Relevant Files

- `packages/site/content/runtime/{core/resources/bundles.mdx,api-reference/bundles.mdx,cli-reference/bundle/**}` — deletes
- `packages/site/content/runtime/core/{extensions/*.mdx,marketplace/index.mdx,resources/index.mdx,tools/*.mdx,skills/bundled.mdx,bridges/*.mdx,configuration/config-toml.mdx}` — strips/rewrites
- `packages/site/lib/runtime-navigation.ts:67` + `content/runtime/{api-reference,cli-reference,core/resources}/meta.json` — navs
- `packages/site/lib/__tests__/{runtime-docs-discovery,runtime-tools-canonical-docs,runtime-docs-truth}.test.ts` — truth suites
- `skills/compozy/{SKILL.md,references/capabilities-and-bundles.md,references/native-tools.md,references/tools-and-skills.md,references/extension-authoring.md,references/contributing-to-compozy.md}` — skill rewrite
- `README.md`, `CLAUDE.md`, `internal/CLAUDE.md`, `AGENTS.md`, `internal/AGENTS.md`, `docs/_memory/{standing_directives.md,spec-authoring-playbook.md,glossary.md}` — checklists
- `docs/ecosystem/{bundle-opportunities.md,README.md,extension-opportunities.md}`, `docs/design/opendesign/_done/marketplace/*` — ecosystem/design
- `docs/qa/scenarios/**`, `docs/qa/journeys/**`, `docs/qa/charters/**`, `docs/qa/_seeds/**` — QA tracker pass
- `.claude/skills/{cy-web-docs-impact,cy-spec-preflight}/**` — trigger-list updates

### Dependent Files

- Generated CLI docs under `packages/site/content/runtime/cli-reference/` — regen
- `docs/qa/state.csv` — regenerated view (gitignored; do not hand-edit)

### Related ADRs

- [ADR-001](adrs/adr-001.md) — docs scope of the cut; [ADR-002/003/005/006](adrs/adr-002.md) — the behaviors the new guides teach.

### Web/Docs Impact

- `web/`: none — task_03 owns web source (checked: this task touches only `packages/site`, skills, root docs, `docs/qa`).
- `packages/site`: this task IS the site impact (inventory above).
- QA impact: this task executes the tracker pass itself — deletions, strips, and the six new `untested` scenario files listed in requirements.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: official skill + protocol/authoring docs updated to the extension-only kit narrative; skill trigger lists stop naming bundles as a living surface; no runtime surface changes (checked: docs-only diff).
- Agent manageability: the agent path documentation for the new verbs/tools ships here (`skills/compozy` + site CLI/API references); no contract changes.
- Config lifecycle: none — no `config.toml` key changes; `config-toml.mdx` touched only for stale bundle prose (checked).

## Deliverables

- Site, official skill, README/instruction files, ecosystem docs, and glossary describing only the extension-kit truth
- Three site truth suites green against post-cut expectations; skill rune budget gate green
- QA tracker updated: −10 scenarios, listed strips, +6 untested, journeys/charters/seeds coherent
- Full `make verify` green (program completion gate)
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- No new test IDs are assigned to this task. Its gates are modified existing suites and builds: the three `packages/site/lib/__tests__/` truth suites updated to post-cut expectations, the `skills/compozy` prompt rune-budget gate, the docs build, and the program-final full `make verify` — plus the living-reference grep gate re-run across docs (`compozy bundle `, `/api/bundles`, `compozy__bundles_`) expecting only historical/homonym hits.

## Success Criteria

- Every gate above green (site suites, rune budget, docs build, full `make verify`)
- Grep gate over `packages/site/content`, `skills/`, root docs shows only historical/homonym hits
- QA tracker pass complete and internally consistent (no scenario references a deleted surface as living)
