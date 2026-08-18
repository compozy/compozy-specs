---
status: completed
title: Docs, official skill, examples, QA scenario flags (Phase G)
type: docs
complexity: medium
---

# Task 9: Docs, official skill, examples, QA scenario flags (Phase G)

## Overview

Make the program real for outsiders and agents (R1/R10): the extensions docs set is rewritten around the code-first journey, the official Compozy skill gains authoring knowledge (a Compozy agent can build a Compozy extension), the examples corpus becomes importable, and every user-visible change is flagged in the QA tracker. The docs-verbatim E2E guard kills the "cannot be managed-installed as written" class forever.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST rewrite `develop.mdx` around init → dev → reload → publish; add the quickstart (getting-started/guides), permissions + consent-areas reference, publish guide, manifest v2 single-source reference, and the contributed-commands guide (declaring commands/groups, `/`-paths + depth, the closed flag subset, `--input` escape, `exec` vs `tool invoke`, a sample-extension worked example — the dev-cycle `archive` walkthrough lands with `cmd-archive`).
- MUST update `config-toml.mdx` with the full `[extensions.*]` table (incl. `resources.*`), regen the CLI reference (9 new verbs + changed flags), correct the hook event catalog (extension source documented), specify tool-ID (`ext__<extension>__<tool>`) and command-path grammars, adopt ADR-006 positioning in bridge-wall paragraphs, and note the hard cuts on the migration page.
- MUST add the authoring reference to `skills/compozy/` (manifest v2, permissions, dev loop, publish, native tools) wired into the router table.
- MUST clean `sdk/examples/`: the two `internal/*`-importing fixtures relocate to `internal/extension/testdata`-backed E2E fixtures; remaining/new examples import only the public SDKs; `prompt-enhancer` phantom capability corrected (delete-target sweep).
- MUST fix `README.md:160` (`extension inspect` → real verbs) + quickstart pointer, and add glossary entries (Permissions, Provides, capability disambiguation note).
- MUST flag QA scenarios per the tracker rule across the whole program (content-addressed adds for new behavior; `qa_status` resets for changed behavior) including the authoring-path scenario set (scaffold→build→validate→dev→reload→publish→install→invoke→update→remove).
- MUST apply `COPY.md` claim standards + `docs/_memory/glossary.md` vocabulary to every public page; runtime truth wins over aspiration.
- MUST implement the docs-verbatim guard as a runtime-lane journey replaying the published quickstart exactly as written.
</requirements>

## Subtasks

- [x] 9.1 `develop.mdx` rewrite + quickstart + references (permissions/consent, manifest v2, publish, contributed commands)
- [x] 9.2 `config-toml.mdx` `[extensions.*]` + CLI reference regen + hook catalog + grammars + bridge positioning + migration notes
- [x] 9.3 `skills/compozy/` authoring reference + router wiring
- [x] 9.4 `sdk/examples` cleanup/relocation; README fix; glossary entries
- [x] 9.5 QA tracker flags: new authoring/dev/distribution/commands scenarios + resets across ET-015..ET-023 family
- [x] 9.6 E2E-007 docs-verbatim guard + docs build green

## Implementation Details

TechSpec: Web/Docs Impact (site bullets), Extensibility (protocol docs), Delete Targets (examples, README), Assumptions (scorecard binds Phase G acceptance). Skills: `documentation-writer`, `copywriting`, `fumadocs`; QA flags per `qa-report` contract. Follow `packages/site/CLAUDE.md`.

### Relevant Files

- `packages/site/content/runtime/core/extensions/{index,develop,install}.mdx` + `meta.json` (new pages register in `pages`) — the docs set (develop.mdx:399-401,488 walls die)
- `packages/site/content/runtime/core/configuration/config-toml.mdx` — `[extensions.*]` table (sibling `lifecycle-matrix.mdx` is codegen-owned — never hand-edit)
- `packages/site/content/runtime/cli-reference/extension/*.mdx` — regenerated via `Makefile:106` (`compozy doc --output-dir …` + oxfmt); new verbs regenerate, never hand-written
- `skills/compozy/SKILL.md` (router table) + `skills/compozy/references/capabilities-and-bundles.md` — authoring content lands under its Extensibility Surfaces section
- `sdk/examples/{prompt-enhancer,secret-guard,telegram-reference,clarify-tool}` — cleanup/relocation set
- `README.md:160`, `docs/_memory/glossary.md`
- `docs/qa/scenarios/` — content-addressed scenario files (ET-015..ET-023 family + new)
- `internal/daemon/daemon_extension_authoring_e2e_integration_test.go` — E2E-007 lives beside E2E-001 (Suite Placement)

### Dependent Files

- Everything shipped by tasks 01–08 — docs must describe final surfaces (runtime truth); any doc/runtime conflict resolves toward the runtime

### Related ADRs

- [ADR-006: Closed-surface positioning](adrs/adr-006.md) — bridge-wall paragraphs + AGH-105 pointers
- [ADR-005: GitHub/git-first distribution](adrs/adr-005.md) — publish guide + source docs
- [ADR-008: Extension-contributed commands](adrs/adr-008.md) — commands guide scope (no product command)
- [ADR-004: Single permissions list](adrs/adr-004.md) — permissions/consent reference

### Competitor References

- `.resources/pi/packages/coding-agent/docs/extensions.md` — opens with "pi can create extensions. Ask it to build one" (the agent-as-author bar for `skills/compozy/`)
- `.resources/eve/docs/extensions.md` — examples-are-the-documentation corpus shape
- `.resources/flue/blueprints/README.md:1-78` — open contribution catalog docs precedent

## Web/Docs Impact

This IS the docs task: `packages/site` set above + `skills/compozy/` + README + glossary. No `web/` code (task_08). Docs build + verbatim-follow QA gate it.

**QA impact**: this task EXECUTES the program-wide flagging (subtask 9.5) — every new/reset scenario lands here with content-addressed ids; pure-docs pages carry no scenario themselves.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: protocol docs (describe contract, dev-lane lifecycle, permissions model, source union, command grammar) become public authoring surface; official skill teaches the extension surfaces.
- Agent Manageability: `skills/compozy/` authoring reference makes the agent path first-class (R10); CLI reference documents every verb's structured output.
- Config Lifecycle: documents the task_02 keys (reference table + examples); no key changes — checked: `internal/config` untouched.

## Deliverables

- Rewritten docs set + quickstart + references; official-skill authoring content; importable examples; corrected README/glossary
- Program-wide QA scenario flags committed
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] E2E-007 — scripted verbatim replay of the published quickstart completes with an installed, invocable extension

## Success Criteria

- Every assigned test case implemented and passing; docs build green in `make verify`
- Brief scorecard: concepts ≤10, first-success actions ≤4 counted by walking the published path
- `skills/compozy/`-guided authoring produces a working extension (exercised again in qa-execution)
- Zero phantom capabilities/verbs anywhere in docs, templates, examples, or README (repo-wide grep clean)

## Completion Evidence

- Published path scorecard: three extension commands after daemon startup, four actions including the startup prerequisite, and nine named concepts.
- Public docs now cover code-first authoring, manifest v2, permissions and derived consent, contributed commands, publishing, the full extension config lifecycle, hook source ordering, tool-ID and command-path grammars, and the closed bridge boundary. Per the operator's greenfield hard-cut direction, no new removed-surface history was added; stale SDK claims in existing migration material were corrected to current truth.
- `skills/compozy/references/extension-authoring.md` is routed from the official skill; the bundled-resource suite proves the reference is embedded and readable.
- `secret-guard` and `telegram-reference` are internal conformance fixtures under `internal/extension/testdata`; public Go examples build against the public SDK, and `prompt-enhancer` no longer advertises a phantom provide surface.
- QA tracker materialization reports 674 scenarios. ET-015–ET-023 remain `untested`; authoring, dev, distribution, command, Web, official-skill, and quickstart behaviors have content-addressed `untested` coverage.

VERIFICATION REPORT
-------------------
Claim: Task 09 focused implementation and contract evidence pass; the monorepo-wide `make verify` remains reserved for the final program gate by operator instruction.
Command: `bunx turbo run lint typecheck test build --filter=./packages/site`; `bunx turbo run lint typecheck test --filter=./web`; scoped Go race/integration suites; `make lint`; QA materializer; phantom/path greps; `git diff --check`.
Executed: 2026-07-29, after all Task 09 changes.
Exit code: 0 for every required command; clean greps intentionally return 1 with no matches.
Output summary: site 51 files/248 tests plus production build; Web 516 files/4,071 tests; extension unit tree 919 tests; reference/bridge/quickstart integrations 16 cases; skills 29 tests; mage boundaries 119 tests; global lint 0 issues; codegen-check all modules verified; QA tracker 674 scenarios.
Warnings: none.
Errors: none.
Contract parity: PASS against `_brief.md`, `_techspec.md`, `_tests.md` E2E-007, ADR-004, ADR-005, ADR-006, and ADR-008; `_user_stories.md` is absent by program design.
Visual contract: n/a — no named visual reference found.
Verdict: PASS for Task 09 focused scope.

## Compozy Impact Audit

- Native tools: no descriptor, schema digest, capability gate, or tool-ID change. Public docs and the official skill now describe the existing extension authoring/distribution native tools and their risk boundaries; E2E-007 exercises the CLI/tool invocation path.
- Extensibility and hooks: extension authoring, manifest v2, permissions, provide surfaces, hook source ordering, commands, publishing, and bridge positioning are now documented. The initialize wire contract now emits empty `provides` and grant collections as JSON arrays rather than `null`, protected by the manager suite. Public examples use public SDKs; bridge conformance fixtures are internal and registered as bundled.
- Workspace data isolation: no datum or storage ownership changed. The quickstart guard proves a dev link remains workspace-scoped and is read/invoked with the owning `workspace_id`; global installed state and foreign workspaces are unchanged.
- Official Compozy skill: added and routed `references/extension-authoring.md`; bundled-resource coverage includes the new reference.
