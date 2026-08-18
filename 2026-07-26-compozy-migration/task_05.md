---
status: completed
title: Rebrand authored governance content and project memory
type: docs
complexity: medium
---

# Task 05: Rebrand authored governance content and project memory

## Overview

Closes the rebrand with the authored-content layer that code-scoped grep gates never see: `COPY.md`, `PRODUCT.md`, the root `README.md`, the five `CLAUDE.md` files, `docs/_memory/` (glossary, standing directives, playbook, lessons, synthesis), `docs/qa/` command tokens, and the changelog/tweets/prs trees. This is where the OS-first positioning (brief round-6) and the people-first register (round-5) land in the normative doc base. The launch hero and its locks are explicitly owned by task 11, not this governance sweep.

<critical>
- ALWAYS READ `_brief.md`, `_techspec.md`, `_content-plan.md`, `_tests.md`, every ADR, and any per-task memory before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — hard-cut authored identifiers in the declared scopes; do not preserve old names through aliases, historical exemptions, fallback language, or partial grep scopes
</critical>

<requirements>
- MUST rewrite `COPY.md` §2/§3 OS-first per brief round-6: product-name table (`Compozy`/`CompozyOS`/`Compozy Network`/`compozy-network/v0`), integrated-completeness differentiator ladder, Network demoted from lead to the subsystem nobody else has, and the retired open-workplace promise removed as primary narrative.
- MUST preserve `COPY.md` §4 (audience) and §5 (voice) intact except for product names — they are register, not brand (round-5 lock). Reintroducing "operator-first" anywhere is a blocking failure.
- MUST preserve the task-11-owned launch hero, subhead, and their locks as external dependencies. This task may remove retired expansion language from governance prose but MUST NOT install, relock, or otherwise define the launch hero.
- MUST rewrite `PRODUCT.md` purpose/users/register to CompozyOS terms while preserving the people-first §Users, §Brand Personality, and the "Operator cockpits" anti-reference.
- MUST rewrite the root `README.md` prose in the people-first register — outcome-level opening, command density below as proof — with install/module lines following the earlier phases.
- MUST rewrite `docs/_memory/glossary.md` product/protocol vocabulary and add CompozyOS/Compozy Network entries; sweep `standing_directives.md` and `spec-authoring-playbook.md`.
- MUST sweep `docs/_memory/lessons/**` and `_synthesis.md` fully (brief round-4: no historical exemption) preserving L-numbers, dates, and evidence structure.
- MUST sweep command tokens across every authored `docs/qa/` surface: `README.md`, `personas.md`, `journeys/**`, `charters/**`, `automation-backlog/**`, `bugs/**`, `reports/**`, `scenarios/**`, and `_seeds/**`. Preserve the people-first persona roster and story voice already aligned on 2026-07-22; reintroducing "As an operator" stories or renaming the `operator_persona` protocol identifier is forbidden. Never hand-edit generated `state.csv` or binary evidence.
- MUST update the five `CLAUDE.md` files (root, `internal/`, `web/`, `packages/site/`, `packages/ui/`) for skill names, official-skill wording, impact-audit wording, and any line describing a renamed surface. Synchronize `AGENTS.md` only when its actual mirror/symlink contract requires it.
- MUST cover the complete authored-document scope: root governance/security/catalog files; `docs/{articles,design,ecosystem,ideas,opendesign,prompts,prs,rfcs,tweets}/**`; bridge READMEs; SDK example READMEs; slide/theme sources; and internal operational documentation. Fixtures owned by tasks 02–04 are excluded unless this task changes the documented contract they assert.
- MUST run both gates to zero with explicitly versioned scopes: a global authored-document brand gate that excludes generated/binary evidence and the then-locked development-skill path names, plus a separate skill-body gate that proves those locked paths contain no live old product identifier in their bodies (round 9 renamed that tree to `.agents/skills/eng/eng-*`, retiring the exclusion); and the register anti-regression grep (`operator-first|engineer-to-engineer|operator cockpit|operator surface|As an operator`), excluding legitimate technical uses (`home_policy = operator`, `operator HOME`, `operator_persona`, identity-boundary prose).
- MUST NOT add prose/brand-string suite tests — these are task verification commands, and `CLAUDE.md`/`SKILL.md` context-budget rules still apply to every edited instruction file.
</requirements>

## Subtasks

- [x] 5.1 Rewrite `COPY.md` positioning, message architecture, and vocabulary OS-first without changing the task-11-owned launch hero or its locks
- [x] 5.2 Rewrite `PRODUCT.md` purpose/users/register preserving the people-first authorities
- [x] 5.3 Rewrite the root `README.md` prose in register with install/module lines following earlier phases
- [x] 5.4 Rewrite `docs/_memory/glossary.md` and sweep standing directives + spec-authoring playbook
- [x] 5.5 Sweep `docs/_memory/lessons/**` and `_synthesis.md` preserving structure, dates, and evidence
- [x] 5.6 Sweep every authored `docs/qa/` command-token surface without touching persona roster, story voice, generated state, or binary evidence
- [x] 5.7 Update the five `CLAUDE.md` files and any required `AGENTS.md` mirror for renamed-surface guidance
- [x] 5.8 Sweep `CHANGELOG.md`, release receipts, root governance/security/catalog artifacts, documentation collections, bridge README files, SDK example READMEs, slides/themes, and internal operational documentation
- [x] 5.9 Run the global authored-document brand gate, the locked-skill body gate, and the register anti-regression grep to zero
- [x] 5.10 Run a register review of every rewritten artifact against `COPY.md` §4/§5 and `PRODUCT.md`

## Implementation Details

Classification per `_content-plan.md` §A: MECHANICAL token sweeps for lessons/QA/changelog trees; REWRITE for `COPY.md`, `PRODUCT.md`, `README.md`, and the glossary; REGEN was already handled for `DESIGN.md` in task 04. Locked verbatim launch text and its material allocation remain task 11's external dependency; this task neither moves nor rewrites it.

Register review is a human/agent read against the two authorities with both in context — explicitly not a suite test (repo test-placement policy).

### Relevant Files

- `COPY.md` — §2/§3 rewritten OS-first; §4/§5 preserved; task-11-owned launch hero/subhead locks not changed here
- `PRODUCT.md` — register note, users, product purpose, brand personality, anti-references
- `README.md` (root) — brand prose, positioning, install/module lines
- `docs/_memory/glossary.md`, `standing_directives.md`, `spec-authoring-playbook.md` — canonical vocabulary and posture
- `docs/_memory/lessons/**`, `_synthesis.md` — full sweep preserving structure
- `docs/qa/{README.md,personas.md,journeys,charters,automation-backlog,bugs,reports,scenarios,_seeds}` — command tokens only; exclude generated state and binary evidence
- `CLAUDE.md`, `internal/CLAUDE.md`, `web/CLAUDE.md`, `packages/site/CLAUDE.md`, `packages/ui/CLAUDE.md` (and any required `AGENTS.md` mirror) — governance lines for renamed surfaces, never the task-11 hero lock
- `CHANGELOG.md`, release receipts, root governance/security/catalog artifacts, `docs/{tweets,articles,design,ecosystem,ideas,opendesign,prompts,prs,rfcs}`, bridge READMEs, SDK example READMEs, slides/themes, and internal operational docs — authored-document sweep

### Dependent Files

- `.agents/skills/eng/**` — path names were locked as `agh-*` while this task ran (round-4 lock); their bodies were inspected with the dedicated body gate and only prose naming renamed surfaces changed. Round 9 (2026-07-26) lifted the lock and renamed the tree to `.agents/skills/eng/eng-*`, so the body gate now folds into the global brand gate
- `packages/site/lib/__tests__/protocol-rfc-hard-cut.test.ts` — reads root `docs/rfcs/` and `glossary.md`; already co-shipped in task 03, re-verify after the glossary rewrite
- `packages/site/lib/__tests__/public-install-contract.test.ts` — reads the root `README.md`; must stay green after the prose rewrite

### Related ADRs

- No ADR governs authored content. The governing locks are `_brief.md` round-5 (register), round-6 (OS-first positioning with verbatim hero), and round-4 (full historical sweep).

## Deliverables

- `COPY.md` carrying OS-first positioning with §4/§5 register intact, while the task-11-owned launch hero remains untouched
- `PRODUCT.md`, root `README.md`, and glossary rewritten in the people-first register
- Fully swept memory, QA, changelog, and communications trees with structure preserved
- Five updated `CLAUDE.md` files and any required `AGENTS.md` mirror, with no task-11 hero change
- Global authored-document and locked-skill body gates at zero, with a register review recorded per rewritten artifact
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from [`_tests.md`](_tests.md), the test contract — read each ID's full definition there before writing tests.

- [x] No new `_tests.md` IDs are assigned to this task. Verification is the markdown-scope brand grep gate, the register anti-regression grep, the register review against `COPY.md` §4/§5 + `PRODUCT.md`, and the two existing site specs that read root files (`protocol-rfc-hard-cut`, `public-install-contract`) staying green. Prose tests are forbidden by repo policy when a stronger gate owns the invariant.

### Web/Docs Impact

- `web/`: none — checked surfaces: `web/src/**`, `web/CLAUDE.md` (governance lines only, no code); reason: this task edits authored documents, not runtime code.
- `packages/site`: `packages/site/CLAUDE.md` governance wording only; task 11 owns the launch hero lock. Authored MDX prose was swept in task 04 — re-verify no register regression was introduced there.
- QA impact: none — no user-visible behavior change. `docs/qa/` edits in this task are command-token sweeps and do not alter product behavior; scenario `qa_status` values MUST NOT be reset by a docs sweep.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: extension manifests, hooks, skills/capabilities, tools/resources, bundles, registries, bridge SDKs; reason: `.agents/skills/**` dispatch names were brief-locked unchanged during this task, which edits only prose describing already-renamed surfaces.
- Agent manageability: none — checked surfaces: CLI verbs, HTTP endpoints, UDS routes, structured output; reason: no runtime surface is added, removed, or reshaped. The instruction files that agents read are updated so agent guidance stays truthful.
- Config lifecycle: none — checked surfaces: `config.toml` keys, defaults, validation, examples; reason: documentation-only change. Config reference regeneration belongs to task 09 with the migrator.

### AGH Impact Audit

- Native tools: no impact — checked native tool IDs/toolsets, descriptors, schema digests, capability gates, CLI/API fallbacks, and generated references; this task changes authored governance text only.
- Extensibility and hooks: no runtime impact — checked extension manifests, hooks, skills/capabilities, tools/resources, bundles, registries, bridge SDKs, and config lifecycle. The development-skill path names were locked during this task and their bodies received a separate truthfulness gate; the later round-9 rename to `.agents/skills/eng/eng-*` is internal tooling and touches no runtime surface.
- Workspace data isolation: no impact — checked global/workspace/session/agent scope, workspace propagation through CLI/HTTP/UDS/core/store/web/SSE/cache/events; no runtime datum or isolation path changes.
- Official AGH skill: no impact — checked the renamed `skills/compozy/**`; this authored-governance sweep changes no public tool, CLI path, hook event, capability, bundle/resource, or memory/network/task semantic. The separate `.agents/skills/eng/eng-*` development-skill paths are not the official product skill.

## Success Criteria

- Every assigned test case implemented and passing
- Global authored-document brand gate and locked-skill body gate at zero, with generated/binary evidence and locked skill path names explicitly excluded from the former
- Register anti-regression grep at zero outside legitimate technical uses
- `COPY.md` §4/§5 diff shows product-name changes only; the task-11-owned launch hero is not added, moved, relocked, or changed by this task
- `make verify` green and both root-file-reading site specs passing
