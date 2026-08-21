---
status: pending
title: Docs, official skill, and instructions updates
type: docs
complexity: medium
---

# Task 7: Docs, official skill, and instructions updates

## Overview

Ships the documentation closure for skill sources: site configuration and skills pages (including a user-first sources/exposures page with a step-by-step tutorial), regenerated CLI docs for the new verbs, the official Compozy skill's references, and the `internal/CLAUDE.md` discovery-layer amendment. Public behavior changed across config, CLI, API, and sessions — the official skill and site must state it in the same program.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — no compat shims, no fallbacks, no placeholders (greenfield hard cuts)
</critical>

<requirements>
1. MUST update `packages/site/content/docs/configuration/config-toml.mdx` (section index rows, example `[skills]` block, key table for `sources`/`custom_sources` with defaults + live semantics), `file-locations.mdx` (location tables + tree gain the preset/custom roots), and `lifecycle-matrix.mdx` (both keys as Live rows).
2. MUST update `packages/site/content/docs/skills/index.mdx` — discovery order + precedence with root lists, origin attribution, suppression behavior (automatic, injection-only, explicit invocation wins), qualified invocation forms — and add a dedicated sources & exposures docs page with its `meta.json` entry, written user-first: a step-by-step end-to-end tutorial (enable a source → see skills in the picker → expose a skill → verify from another tool) with expected outputs per step.
3. MUST refresh generated CLI docs via `make codegen` (new verbs `skill sources|expose|unexpose`, changed `create`/`info`/`where`/`list`) and order the new entries in `packages/site/content/docs/cli/skill/meta.json` (ordering is manual); update `content/docs/api/skills.mdx` for the expose routes + payload fields.
4. MUST keep the AGENT.md docs truthful: `agents/definitions.mdx:324` and `configuration/agent-md.mdx:526` keep `skills.extra_sources` rejected, adding the pointer to runtime `skills.sources`.
5. MUST update the official Compozy skill: `skills/compozy/references/configuration.md` (the two keys, scopes, tri-state override) and `references/tools-and-skills.md` (origin fields on native tools, expose operations, sources inspection paths).
6. MUST amend `internal/CLAUDE.md`'s five-layer discovery sentence to "each layer may own multiple roots" (activate `writing-agents-md` before editing — keep the delta lean, no restated prose).
7. MUST follow `COPY.md` claim standards and `docs/_memory/glossary.md` vocabulary (skills stay "skills"; never "recipe"/"playbook"; sandbox ≠ environment); runtime truth wins over aspirational wording — every example output must match the `_dx.md` transcripts byte-for-byte.
8. MUST verify docs build + links through the standard gates (no standalone prose tests — `make gate` docs lanes + `codegen-check` own drift).
</requirements>

## Subtasks

- [ ] 7.1 `config-toml.mdx` index + example + key table; `file-locations.mdx` trees/tables; `lifecycle-matrix.mdx` rows
- [ ] 7.2 `skills/index.mdx` discovery-order/precedence/suppression rewrite for root lists
- [ ] 7.3 New sources & exposures page (user-first tutorial with expected outputs) + `meta.json`
- [ ] 7.4 CLI docs regen + `cli/skill/meta.json` ordering + `api/skills.mdx` expose routes
- [ ] 7.5 AGENT.md docs pointers (`definitions.mdx`, `agent-md.mdx`)
- [ ] 7.6 Official skill references (`configuration.md`, `tools-and-skills.md`)
- [ ] 7.7 `internal/CLAUDE.md` five-layer amendment (lean delta)
- [ ] 7.8 Cross-check every example against `_dx.md` transcripts; gates green

## Implementation Details

Sources of truth: `_dx.md` (transcripts/shapes — copy verbatim), `_spec.md` Config Lifecycle + Agent Manageability Plan (surface inventory), shipped behavior from tasks 01-05. Site conventions per `packages/site/CLAUDE.md` (read before editing). Feature docs follow the user-first tutorial pattern (own sidebar entry, step-by-step with expected outputs).

### Relevant Files

- `packages/site/content/docs/configuration/config-toml.mdx:79-80,580-590,1932-1945` — section index, example block, `[skills]` key table (hand-maintained)
- `packages/site/content/docs/configuration/{file-locations,lifecycle-matrix,skill-md}.mdx` — location trees, Live rows, definition-field notes (allowlist inertness per ADR-008)
- `packages/site/content/docs/skills/{index.mdx,meta.json}` — discovery narrative + new page slot
- `packages/site/content/docs/cli/skill/` + `meta.json` — generated verb pages (regen) + manual ordering
- `packages/site/content/docs/api/skills.mdx` — API reference
- `packages/site/content/docs/agents/definitions.mdx:324` + `configuration/agent-md.mdx:526` — `skills.extra_sources` rejection + pointer
- `skills/compozy/references/{configuration.md,tools-and-skills.md}` — official skill references
- `internal/CLAUDE.md` — five-layer discovery sentence
- `magefiles/codegen_cli_docs.go:11-40` — CLI docs generation contract (`make codegen`, checked by `codegen-check`)
- `COPY.md` + `docs/_memory/glossary.md` — language authority

### Dependent Files

- None downstream — this task closes the program's documentation surface (QA pair follows).

### Related ADRs

- [ADR-002](adrs/adr-002.md) — preset table documented as curated · [ADR-008](adrs/adr-008.md) — recognized-but-inert fields documented · [ADR-013](adrs/adr-013.md) — origin visibility narrative · [ADR-014](adrs/adr-014.md) — non-goals stated in docs (no sync, symlinks never copies)

### Web/Docs Impact

- `web/`: none — checked: no component/hook/type surface; docs-only change.
- `packages/site`: the full list above (configuration ×4, skills ×2 + new page, cli/skill regen + meta, api/skills).
- QA impact: none — no runtime behavior change; docs describe behavior shipped and flagged by tasks 01-06 (their scenarios cover the walks).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: documents the closed preset table and extension non-participation (ADR-014); no extension surface changes (checked: manifests, hooks, registries untouched).
- Agent manageability: documents the agent paths (sources inspection, config-set semantics, expose operations, native-tool fields) — the docs ARE the agent-path documentation required by SD-011.
- Config lifecycle: completes the lifecycle for both keys — examples, generated CLI/site docs, lifecycle-matrix rows (structs/defaults/validation/tests shipped in task_01).

## Deliverables

- Site docs updated end-to-end (configuration, skills, CLI, API) including the new user-first sources & exposures tutorial page
- Official skill references updated; `internal/CLAUDE.md` amended
- Regenerated CLI docs with ordered navigation
- Every example byte-consistent with `_dx.md` transcripts

## Tests

No `_tests.md` IDs assigned — the behavior IDs live with their implementing tasks. Verification is gate-owned (no standalone prose/snapshot tests per test-placement policy):

- [ ] `make codegen` + `make codegen-check` — CLI docs drift-free after regen
- [ ] `make gate` — site lint/typecheck/build lanes green
- [ ] Manual cross-check recorded in completion notes: every transcript in the new/changed pages matches `_dx.md` exactly

## Success Criteria

- All listed pages updated; new page present in navigation; zero broken links (site build green)
- Official skill and CLAUDE.md deltas are lean (no restated rules) and pass their authoring-skill review
- No doc claims a behavior the runtime does not ship (COPY.md claim standards applied)
