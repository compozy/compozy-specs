---
status: completed
title: Bundled skills inside the dev-cycle extension
type: backend
complexity: medium
---

# Task 08: Bundled skills inside the dev-cycle extension

## Overview

Ships the nine `cy-*`/`git-rebase` skills as bundled resources of the dev-cycle extension, closing a silent correctness gap: the dev-cycle agents already reference `cy-execute-task`, `cy-fix-reviews`, and `cy-final-verify` by name while nothing installs them. Skills publish at global scope and project into every Compozy-managed session; the legacy `compozy` skill is retired because the renamed official runtime skill now owns that name and its product-overview role.

<critical>
- ALWAYS READ the migration brief, TechSpec, ADRs, and test contract (`_brief.md`, `_techspec.md`, `adrs/`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS: bundle only the nine named skills from the repository source; do not add aliases, a compatibility `compozy` skill, external-CLI installation, or an unreviewed decision-capture skill.
</critical>

<requirements>
- MUST ship exactly nine skills inside the extension package: `cy-create-prd`, `cy-create-techspec`, `cy-create-tasks`, `cy-execute-task`, `cy-workflow-memory`, `cy-review-round`, `cy-fix-reviews`, `cy-final-verify`, `git-rebase`.
- MUST source those directories from `.agents/skills/{cy-create-prd,cy-create-techspec,cy-create-tasks,cy-execute-task,cy-workflow-memory,cy-review-round,cy-fix-reviews,cy-final-verify,git-rebase}`; `.claude/skills` is not their repository source.
- MUST retire the legacy `compozy` skill entirely — it is NOT bundled, and the bundle MUST NOT contain a skill named `compozy` (the renamed official runtime skill owns that name in the global scope).
- MUST declare the skills through the manifest resources key and add the directory to the embed set.
- MUST verify each ported `SKILL.md` parses through the extension skill loader with its source classification — this was flagged as an open verification item in ADR-004 and MUST be encoded as a test, not assumed.
- MUST sweep skill content for retired CLI surfaces before bundling: no `tasks run`, `tasks validate`, `reviews fix`, or `setup` invocations; `tasks validate` is removed rather than falsely mapped to `loop validate`, and all remaining guidance uses loop-based surfaces and CompozyOS branding.
- MUST NOT recreate the legacy external-CLI install behavior — no `--into <cli>` targeting, no writes into external agent-CLI homes.
- MUST bump the extension version so the manifest checksum change re-enrolls the bundled install at boot.
- MUST keep bundled skills exempt from content scanning via their bundled source classification, with the manifest checksum covering skill content (Safety Invariant 12).
- MUST prove that exemption is classification-bound rather than a general scan bypass: the same content from a non-bundled source still passes through `VerifyContent`, while changing bundled bytes changes the checksum/version enrollment identity and re-enrolls the updated resource.
- MUST add the reviewer role's required bundled skill references, explicitly including `cy-review-round`; the role must not rely on a silent prompt dependency.
- MUST prove global publication does not bypass workspace-local overrides or leak a workspace-local skill into another workspace: the bundled source is global, while lookup/cache/prompt projection preserves the existing workspace boundary.
</requirements>

## Subtasks

- [x] 8.1 Port exactly the nine source directories from `.agents/skills` into the extension package; exclude `cy-capture-decisions` and every other skill
- [x] 8.2 Sweep every ported skill body for retired CLI surfaces and legacy branding
- [x] 8.3 Declare the skills resource in the manifest and extend the embed set
- [x] 8.4 Bump the extension version and confirm the checksum change re-enrolls the install
- [x] 8.5 Verify loader compatibility for every ported skill frontmatter
- [x] 8.6 Add the missing skill references to the reviewer agent definition
- [x] 8.7 Confirm global-scope publication and session prompt projection while proving workspace-local override/isolation behavior in two workspaces, bundled-only scan exemption, and checksum/version-triggered re-enrollment
- [x] 8.8 Implement UT-045, UT-046, UT-047 and IT-013

## Implementation Details

Follow ADR-004 and TechSpec §Development Sequencing step 3. The wiring is three small changes (skills directory, manifest resources key, embed directive) on a loader path that is already complete end-to-end; the real work is the content sweep and the loader-compatibility verification.

Documentation for coding agents outside Compozy-managed sessions (manual download/copy) is guidance in the migration guide (task 09), not a product surface here.

### Relevant Files

- `extensions/dev-cycle/extension.json:1-10` — name, version, capabilities, and the `resources` block that gains the skills key
- `extensions/dev-cycle/embed.go:8-14` — the embed directive set to extend
- `extensions/dev-cycle/embed_test.go` — bundled-resource assertions and prompt contracts
- `extensions/dev-cycle/agents/{code_implementer,review_fixer,reviewer}/AGENT.md` — skill references; reviewer currently has none
- `internal/extension/**` — manifest loading, resource publication scope, checksum verification
- `internal/skills/**` and skill-cache/query paths — skill parsing, source classification, bundled-source scan exemption, global publication, and workspace-local lookup isolation
- `.agents/skills/{cy-create-prd,cy-create-techspec,cy-create-tasks,cy-execute-task,cy-workflow-memory,cy-review-round,cy-fix-reviews,cy-final-verify,git-rebase}` — the exact repo skill sources to port and sweep

### Dependent Files

- `internal/session/**` prompt augmentation — projects published skills into managed sessions
- `skills/compozy/` (renamed in task 04) — owns the `compozy` skill name and the product-overview role
- Migration guide (task 09) — documents where skills went and the retired install verb

### Related ADRs

- [ADR-004: Dev-Cycle Bundles Nine Skills From `.agents/skills`; Authoring Runs in Managed Sessions](adrs/adr-004.md) — the nine-skill decision, the retirement of the legacy `compozy` skill, and the rejection of an external-CLI install surface

## Deliverables

- Nine skills bundled inside the dev-cycle extension package, embedded and manifest-declared
- Swept skill content free of retired CLI surfaces, with no skill named `compozy` in the bundle
- Verified loader compatibility for every ported skill
- Reviewer agent definition carrying its required skill references
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from [`_tests.md`](_tests.md), the test contract — read each ID's full definition there before writing tests.

- [x] UT-045 — embedded FS contains all nine skill files; the manifest declares the skills resource and normalizes on load. The assertion remains in `embed_test.go` because embedded bytes are the shipped product contract
- [x] UT-046 — every bundled skill parses through the extension skill source (the ADR-004 verification item, encoded as a test)
- [x] UT-047 — content gate: no `tasks run`, `tasks validate`, `reviews fix`, or `setup` invocation in bundled skill bodies and no skill named `compozy` in the bundle (the single permitted prose assertion — the embedded content IS the shipped product contract and `embed_test.go` already owns this class)
- [x] IT-013 — boot enrollment publishes the skills at global scope, session prompt augmentation lists all nine, viewing returns the bundled body, and two-workspace coverage proves local overrides do not leak; bundled classification bypasses `VerifyContent`, non-bundled content is still scanned, and a manifest checksum/version change re-enrolls updated embedded bytes

### Web/Docs Impact

- `web/`: none — checked surfaces: `web/src/systems/**` skill views, extension views; reason: skills surface through existing extension/skill listing paths with no new payload fields or controls. The nine skills appear in existing skill lists as data.
- `packages/site`: the task 09 migration-guide section "Where your skills went" includes manual download/copy guidance for non-Compozy-managed agent CLIs; no additional product surface or standalone installer page is created.
- QA impact: added `ET-dev-cycle-skill-bundle` and `ET-dev-cycle-legacy-skill-retired` as content-addressed `untested` scenarios. No existing scenario entry point cites the retired `compozy setup` verb or an extension-owned duplicate `compozy` skill, so no prior verdict required a reset.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: this task IS an extensibility change — the extension manifest gains a skills resource, the embed set grows, the version/checksum bump re-enrolls the bundled install, and resource publication happens at global scope (recorded as an intentional first-party projection in the workspace-isolation audit). Bundles (technical concept) remain unable to carry skills — verified, not attempted.
- Agent manageability: `compozy skill list` and `compozy skill view <name>` resolve the bundled skills with structured output; `compozy extension list` reports the bumped version. Authoring flows run through existing session verbs (`session new/prompt/resume`) — no new verb is added.
- Config lifecycle: no `config.toml` keys added, changed, or removed — checked surfaces: `[extensions]` enrollment keys, skill configuration; reason: bundled resources are declared in the extension manifest, not in runtime config.

### Compozy Impact Audit

- Native tools: no tool IDs, toolsets, descriptors, schemas, digests, risk flags, availability diagnostics, capability gates, or CLI/API fallbacks change. Checked the existing `compozy__skill_list`/`compozy__skill_view` registry and transport surfaces; IT-013 proves the newly published declarations and bodies resolve through their shared skill registry.
- Extensibility and hooks: the dev-cycle extension adds exactly the nine embedded skill resources, `resources.skills`, version/checksum re-enrollment, and the existing global registry projection. Checked hooks, technical bundles, bridge SDKs, and MCP sidecars; none change because skills remain extension resources.
- Workspace data isolation: the nine extension declarations are intentionally global; workspace-local overrides and additional skills remain workspace-scoped through lookup, cache, content view, and managed-session prompt projection. IT-013 proves workspace A's override/local-only skill never appears in workspace B.
- Official Compozy skill: updated `skills/compozy/references/tools-and-skills.md` with the exact nine dev-cycle skills, their global availability in managed sessions, workspace-local shadowing, the retired duplicate `compozy` skill, and the absence of external-CLI installation. `skills/compozy/` remains the sole owner of the `compozy` skill name.

## Success Criteria

- Every assigned test case implemented and passing
- Nine skills published at global scope and visible in a managed session's prompt augmentation
- Zero retired CLI invocations in bundled skill bodies; no bundled skill named `compozy`
- The dev-cycle agents' referenced skills all resolve, closing the silent dependency gap
- `make verify` green
