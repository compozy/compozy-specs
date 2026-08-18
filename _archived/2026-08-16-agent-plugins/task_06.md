---
status: completed
title: Docs, official skill, and QA scenario flags
type: docs
complexity: low
---

# Task 6: Docs, official skill, and QA scenario flags

## Overview

Closes the surface loop in documentation and QA tracking: the site's extension docs teach the third install layout, the manifest widening, and the remote-header binding; a short interop page anchors the standard relationship; the official Compozy skill reference learns the new operating facts; generated CLI docs refresh; and every user-visible behavior shipped by tasks 02–05 gets its content-addressed `untested` scenario file in `docs/qa/scenarios/` for the QA pair to walk.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST update `packages/site/content/docs/extensions/install.mdx` (third layout in the source-union and layout tables, deterministic non-conformant errors, ingestion report), `manifest.mdx` (MCPServerConfig transport/url/headers), `secrets.mdx` (`--remote-header`), and add one short interop page in the extensions section (registered in `meta.json`) — extending existing tables per the site's own instruction file, `packages/site/CLAUDE.md`, and the user-first tutorial posture (step-by-step with expected outputs).
2. MUST update `skills/compozy/references/extensions.md` with the load-bearing operating facts: agent-plugin install path + detection matrix, `format` field + badge semantics, diagnostics (ingest vs live codes, ordering), data-directory lifecycle (first-launch creation, quarantine), `--remote-header` binding, validate portable branch (Compozy Impact Audit obligation).
3. MUST regenerate the CLI reference pages so `install`, `validate`, `remove`, `status`, `inventory`, `list`, and `secrets` docs match the shipped output shapes.
4. MUST add the content-addressed `untested` scenario files in `docs/qa/scenarios/` covering the flagged behaviors from tasks 02–05 (portable install from dir/git, dual-manifest note, degraded install visibility, remove-with-data/quarantine, dev-link reload, authenticated remote binding, marketplace catalog entry + badges, validate journeys), deduplicating against existing scenario ids — flag only; task_08 walks them.
5. MUST follow `COPY.md` register and the glossary — "extension" vocabulary, "agent plugin" strictly as a format name, `capability` never misused; claim standards applied before "supported"/"conformant" wording (the conformance claim language stays conditional until task_09's evidence citation).
6. MUST keep runtime truth authoritative: every documented transcript is reproduced from the shipped binary, not hand-authored.
</requirements>

## Subtasks

- [x] 6.1 `install.mdx` — layout table, detection matrix, ingestion report, error table rows
- [x] 6.2 `manifest.mdx` — MCP declaration widening (+ explicit sidecar-not-widened note)
- [x] 6.3 `secrets.mdx` — remote-header binding walkthrough with presence-only reads
- [x] 6.4 Interop page (standard relationship, what maps where, non-goals) + `meta.json` registration
- [x] 6.5 Generated CLI docs refresh for the seven touched verbs
- [x] 6.6 `skills/compozy/references/extensions.md` update
- [x] 6.7 `docs/qa/scenarios/` sweep: add the content-addressed untested scenario files, dedup against the registry
- [x] 6.8 Link-check + docs build green

## Implementation Details

Anchors: `packages/site/content/docs/extensions/{index,install,manifest,secrets}.mdx` + `meta.json` (pages array), `packages/site/content/docs/cli/extension/*.mdx` (generated — regenerate, never hand-edit), `skills/compozy/references/extensions.md` (existing load-bearing statements listed in the extensions map §9 — extend, don't restate), `docs/qa/scenarios/` (content-addressed ids per the QA tracker contract), `_dx.md` (transcripts to reproduce). Precedent to avoid: Hermes' stale adapter-scope copy (`.resources/hermes/website/docs/user-guide/cli.md:71` still says "stdio MCP entries" after streamable-http shipped) — scope statements live in ONE place per page.

Skills to activate: `documentation-writer`, `copywriting`, `writing-skills` (for the official-skill reference edit), plus `packages/site/CLAUDE.md` dispatch.

### Relevant Files

- `packages/site/CLAUDE.md` — site-specific rules (read before editing)
- `COPY.md` + `docs/_memory/glossary.md` — register + vocabulary authority
- `docs/qa/scenarios/` + `docs/qa/bugs/` — living QA tree (dedup source)

### Dependent Files

- `packages/site/content/docs/meta.json` — sidebar order (extensions section)
- `skills/compozy/SKILL.md` — router; only touched if the reference index changes

### Related ADRs

- [ADR-001](adrs/adr-001.md), [ADR-002](adrs/adr-002.md) — the positioning the interop page documents

### Web/Docs Impact

- `web/`: none — checked: no component/route/type change in this task.
- `packages/site`: the four prose pages + interop page + generated CLI pages named above.
- QA impact: this task **is** the flag sweep — it adds the untested scenario files for tasks 02–05's behaviors; verdicts recorded by task_08 (intermediate tasks flag only, per house rule).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: official Compozy skill updated (audit obligation); protocol/site docs updated; no runtime surface change (checked: docs-only diff).
- Agent manageability: documents the agent paths (structured verbs, codes, discovery) shipped in task_04 — no new surface.
- Config lifecycle: documents "zero new keys" truthfully; no config artifact edits (checked: no `config.toml`/sidecar docs claims beyond the not-widened note).

## Deliverables

- All named pages updated/created; generated CLI docs current; official skill reference current
- Scenario files added, content-addressed, deduplicated, all `untested`
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED — none assigned; the docs build, link-check, and `make gate` docs lane are this task's verification)**

## Tests

Cases assigned from `_tests.md`: none — documentation task. Verification is the site build + link check + `make gate` (docs lane) + reproduced transcripts matching shipped output. Static prose tests are forbidden by house rule; the QA pair owns behavioral verification.

## Success Criteria

- Docs build + link-check green; every transcript reproduced from the shipped binary
- Scenario files exist for every flagged behavior with valid content-addressed ids and `qa_status: untested`
- Official skill reference reflects every public fact this feature shipped
