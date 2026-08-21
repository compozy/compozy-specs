---
status: pending
title: "Docs and official skill: agent-comms area"
type: docs
complexity: low
---

# Task 7: Docs and official skill: agent-comms area

## Overview

Ship the documentation and the official Compozy skill updates for agent-comms: a new `content/docs/agent-comms/` area (modeled on the `network/` folder pattern) with user-first tutorials, the `[calls]` config reference, regenerated CLI/API references, the spawn-removal rewrite of the orchestration page, and the `skills/compozy/` router + reference updates. Runtime truth wins over aspirational wording — every transcript and key comes from `_dx.md` and the shipped surfaces of task_05.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Activate `documentation-writer` + `copywriting`; follow `COPY.md` claim standards and `docs/_memory/glossary.md` vocabulary (a **call** with a **result contract**; a **parked** child; never "handoff"; the artifact name is `capability`). Read `packages/site/CLAUDE.md` before touching the site.
2. New area `packages/site/content/docs/agent-comms/` with `meta.json` (`{title, icon, pages}` shape, ordered slugs) and five pages: `index.mdx`, `calls.mdx`, `mailbox.mdx`, `subagents.mdx`, `budgets-and-safety.mdx` — user-first, step-by-step tutorials with expected outputs, transcripts verbatim from `_dx.md`.
3. Wire the area: parent docs `meta.json`, `lib/docs-navigation.ts`, `lib/docs-icons.ts`.
4. `configuration/config-toml.mdx` gains the `[calls]` section block + index row with the exact keys/defaults and the observable-effects wording from `_dx.md`.
5. `sessions/orchestration.mdx` is REWRITTEN: spawn sections deleted (no aliases, no "formerly known as"), delegation now documented as calls; the Loops docs note the `output_schema` ≡ `expect` equivalence.
6. Regenerate CLI reference pages (`make cli-docs` flow) — new `call`/`message` families, `session stop --subtree`, `agent list` fields; the spawn CLI doc page disappears with its verb. API reference regenerates from task_05's OpenAPI output.
7. `skills/compozy/` updates: `SKILL.md` router row + new `references/agent-comms.md` (modeled on `references/network.md`), plus `references/native-tools.md` (new family section + spawn removal), `references/agent-definitions.md` (`description`), `references/configuration.md` (`[calls]`), `references/runtime-operations.md` (call/message operations, spawn removal). Keep them lean — these load into prompts.
8. Docs claim only shipped behavior: nine states, no default deadline, idle-TTL suspension, no read/seen state, publish one-way — exactly as the runtime enforces them.
</requirements>

## Subtasks

- [ ] 7.1 Author the five-page `agent-comms` area with `meta.json`, tutorials, and verbatim `_dx.md` transcripts
- [ ] 7.2 Wire navigation, icons, and parent `meta.json`
- [ ] 7.3 Add the `[calls]` block + index row to `configuration/config-toml.mdx`
- [ ] 7.4 Rewrite `sessions/orchestration.mdx` (spawn out, calls in); add the Loops equivalence note
- [ ] 7.5 Regenerate CLI + API reference pages; verify the spawn page is gone
- [ ] 7.6 Update `skills/compozy/` (router + five reference files, `writing-skills` discipline)
- [ ] 7.7 Run site build + link checks; close the docs lanes green

## Implementation Details

Copy the `network/` area shape exactly (explorer-verified: `meta.json` + `index.mdx` + topic pages; nested folders appear as plain page entries). Per the user-first docs convention, each feature page carries an end-to-end tutorial with expected outputs before reference material.

### Relevant Files

- `packages/site/content/docs/network/meta.json` + the `network/` folder — the area pattern to copy
- `packages/site/content/docs/meta.json` + `packages/site/lib/docs-navigation.ts:6` + `packages/site/lib/docs-icons.ts` — area wiring
- `packages/site/content/docs/configuration/config-toml.mdx:99` — config reference gaining `[calls]`
- `packages/site/content/docs/sessions/orchestration.mdx:55` — the spawn rewrite target
- `skills/compozy/SKILL.md:20` + `skills/compozy/references/native-tools.md:41-364` + `references/agent-definitions.md` + `references/network.md:29` (model) + `references/configuration.md` + `references/runtime-operations.md` — the official skill set
- `.compozy/tasks/agent-comms/_dx.md` — the transcript source of truth

### Dependent Files

- Generated CLI reference pages under `packages/site/content/runtime/cli/` (or the current generated-docs root) — refreshed by the cli-docs flow
- Generated API reference — refreshed from task_05's OpenAPI document

### Related ADRs

- [ADR-002: One agent-facing call verb; the spawn surface is deleted](adrs/adr-002.md) — the docs hard cut
- [ADR-005: Type-level disjunction from Compozy Network with a one-directional bridge](adrs/adr-005.md) — the boundary the docs must state plainly
- [ADR-007: Explicit registry-name invocation, injected roster, `description` field, batch fan-out](adrs/adr-007.md) — the subagents page content

### Web/Docs Impact

- `web/`: none — checked surfaces: no web code; reason: docs-only task.
- `packages/site`: the full impact IS this task — new `content/docs/agent-comms/` (5 pages + meta), navigation/icons, config reference section, orchestration rewrite, regenerated CLI/API references.
- QA impact: none — no runtime behavior change; docs correctness is owned by the site build/link gates and the QA doc-scenario sweep in task_09's charters if selected.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: documents the `call` hook family, Host API read methods, and the `description` flow for extension-provided agents — no surface changes; checked: hooks catalog, Host API (docs describe task_05's shipped set).
- Agent manageability: documents the full `_dx.md` surface; no new surface.
- Config lifecycle: documents `[calls]` keys/defaults/observable effects exactly as shipped; no key changes.

## Deliverables

- The five-page `agent-comms` docs area, wired and linked
- Config reference, orchestration rewrite, regenerated CLI/API references (spawn page gone)
- `skills/compozy/` router + reference updates, lean
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED — none are assigned; the gates below stand in)**

## Tests

No `_tests.md` IDs are assigned to this task (docs carry no runtime behavior). Owning gates instead:

- [ ] Site build + link check green through repo-root Turbo (`make bun-lint`, `bun-typecheck`, web build lane)
- [ ] `make codegen-check` unaffected (generated references match shipped surfaces)
- [ ] Every `_dx.md` transcript quoted in docs matches the shipped output byte-for-byte (spot-verified against a running daemon)

## Success Criteria

- Docs area renders with correct navigation/icons; zero broken links
- No spawn vocabulary survives anywhere in docs or the official skill (grep-clean)
- Copy passes the `COPY.md` claim standards; glossary vocabulary holds
