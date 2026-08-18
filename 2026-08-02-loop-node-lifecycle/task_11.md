---
status: completed
title: Site docs, config reference and official skill
type: docs
complexity: low
---

# Task 11: Site docs, config reference and official skill

## Overview

Ships the documentation half of the contract: a new "failure handling" chapter in the loops docs,
grammar/reference updates for every new DSL surface, the operator verbs and inventories in the
running guide, the config-toml lifecycle keys (manual file — nothing enforces it), the extension
`event_key` breaking-change note, and the official Compozy skill update so agents learn the new
tool set.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- R1: MUST add `packages/site/content/docs/loops/failure-handling.mdx` (precedence chain,
  classes, effects, verbs, inventories, quarantine/requeue, liveness posture) and register it in
  `meta.json` (unregistered pages don't render).
- R2: MUST update `dsl-reference.mdx` (retry/deadline/result_contract/on_*/wait/on_parent_close),
  `reference-grammar.mdx` (`effect.*` root), `running.mdx` (cancel/kill, node verbs,
  inventories, canceled terminal), `configure.mdx` + `configuration/config-toml.mdx`
  (per-kind lifecycle keys + `[loops.breaker]` + autopause examples), `extensions.mdx`
  (`event_key` requirement — breaking, no compat), `guardrails.mdx` (parked states vs bounds).
- R3: MUST update `skills/compozy/references/loops.md`: tool/CLI table (+8/−1 incl. Destructive
  flags), failure contract section, event kinds, inventories, verbs, `canceled` terminal.
- R4: Copy follows `COPY.md` + `docs/_memory/glossary.md` (canonical vocabulary; `capability`
  never misused); runtime truth wins over aspiration; CLI reference pages are GENERATED — never
  hand-edit (they regenerated in task_07).
- R5: Docs claims MUST match shipped behavior — cross-check every claim against the task_02–07
  implementations, not the spec alone.
</requirements>

## Subtasks

- [x] 11.1 `failure-handling.mdx` + `meta.json` registration
- [x] 11.2 DSL/grammar/running/guardrails updates
- [x] 11.3 config-toml + configure pages (keys, defaults, examples, breaker note)
- [x] 11.4 extensions page `event_key` breaking-change section
- [x] 11.5 Official skill loops reference update
- [x] 11.6 Site build + link check + `make codegen-check` (generated pages untouched)

## Implementation Details

Authored MDX only — the CLI pages under `content/docs/cli/loop/` are generated (task_07's
codegen). Sidebar order lives in `content/docs/loops/meta.json`.

### Relevant Files

- `packages/site/content/docs/loops/` — 16 authored pages + `meta.json`
- `packages/site/content/docs/configuration/config-toml.mdx:84-86,609-661` — loops key rows/examples
- `skills/compozy/references/loops.md` + `skills/compozy/SKILL.md` — official skill
- `COPY.md`, `docs/_memory/glossary.md` — language authority

### Dependent Files

- `packages/site/content/docs/cli/loop/` — generated; verify drift-free only

### Related ADRs

- All PRD+techspec ADRs inform prose; cite ADR-002/003/004/005 posture in failure-handling.mdx.

## Web/Docs Impact

This IS the docs impact. QA impact: none — docs-only (no runtime behavior).

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: documents the `event_key` provide-surface contract.
- Agent manageability: the official skill is how agents discover the new verbs — R3 is the
  agent-path documentation requirement (SD-011).
- Config lifecycle: config-toml rows + examples for every task_01 key (manual, unenforced —
  explicit deliverable).

## Skills

`documentation-writer`, `fumadocs`, `copywriting` (public copy), `compozy` (skill reference
shape).

## References

- Surfaces exploration §12 (docs file inventory, generated-vs-authored split)
- `.compozy/tasks/loops-paper-adoption/_techspec.md` Web/Docs Impact — the co-ship precedent

## Deliverables

- All pages above updated/added; site builds; sidebar renders the new chapter
- Official skill teaches the full new surface
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED — see note)**

## Tests

No `_tests.md` IDs are assigned to this docs task. Verification is gate-owned: site build via
repo-root Turbo lane (`make gate`), link integrity, and `make codegen-check` proving generated
pages untouched. Prose accuracy is checked against shipped behavior in R5 (no static prose tests
— stronger gates own these artifacts).

## Success Criteria

- Site build + link check green through `make gate`; `make codegen-check` clean
- Every new DSL field, verb, state, config key, and event kind appears in exactly one canonical
  docs home
- Official skill table matches the shipped native-tool inventory (8 added, 1 removed)
