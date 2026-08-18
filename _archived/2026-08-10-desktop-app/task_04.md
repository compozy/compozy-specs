---
status: completed
title: "Product language + docs: brand hard cut, site pages, official skill"
type: docs
complexity: medium
---

# Task 4: Product language + docs: brand hard cut, site pages, official skill

## Overview

Executes the ADR-012 product-language hard cut (Compozy → CompozyOS across every product surface, with `compozy` codified as the command identifier) and ships the desktop documentation: install page, getting-started, support runbook, threat-model note, and the official skill update. One sweep, enumerated delete targets, an occurrence gate that keeps it cut.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST sweep every product-language "Compozy" occurrence to "CompozyOS" in: `packages/site` (marketing + docs prose), `COPY.md`, `docs/_memory/glossary.md`, `PRODUCT.md`, `web/` display strings, `skills/compozy/` prose, README product prose, release display names — per the TechSpec Delete Targets list. No aliases, no transition prose.
2. MUST codify the boundary in the glossary and COPY.md: *CompozyOS is the product; `compozy` is its command* — command identifiers (binary, `COMPOZY_*` env, module path, `@compozy/cli`, brew formula, socket names, `compozy__*` tool IDs, config paths) stay untouched by rule (ADR-012.2).
3. MUST implement the occurrence gate (UT-113): script scanning swept surfaces for product-language "Compozy" outside the command-identifier allowlist; wired as a regression gate.
4. Brand marks in `web/`/`@compozy/ui` route through the designer lane (SD-007/L-032) — string changes only in this task; never invent or alter marks.
5. MUST ship the desktop docs set in `packages/site`: install page (desktop as primary interactive install; `libfuse2` note + `--appimage-extract-and-run`), desktop getting-started (first run, attach behavior, update experience, ownership rules, recovery state), support runbook (log paths per platform, WebKitGTK/NVIDIA remediations, manual-download fallback, update-recovery + roll-forward procedures), threat-model note (localhost posture inherited; remote = gateway).
6. MUST update `skills/compozy/` with the desktop surface: `compozy app` verbs, update/recovery semantics, ownership rules, routing entries (N-004).
7. Web UI strings: display-text edits only — zero behavior/route/API changes (the shell's zero-web-changes property is unaffected).
</requirements>

## Subtasks

- [x] 4.1 Glossary + COPY.md boundary codification (delete the old product-name definition; write the new one)
- [x] 4.2 Site-wide sweep (`packages/site` marketing + docs prose) + PRODUCT.md + README
- [x] 4.3 `web/` display-string sweep (designer-lane flag for any mark encounter)
- [x] 4.4 `skills/compozy/` content + routing update (desktop surface + rename)
- [x] 4.5 Occurrence-gate script + allowlist + CI wiring (UT-113)
- [x] 4.6 Install page + desktop getting-started + support runbook + threat-model note
- [x] 4.7 Frontend gates via Turbo from repo root (`make bun-lint`, `bun-typecheck`, site build) — never package-local

## Implementation Details

Delete targets are enumerated in TechSpec §Delete Targets — treat that list as the sweep checklist. Docs structure follows `packages/site/CLAUDE.md` conventions and the `documentation-writer`/`copywriting` skills; runbook content sources: `analysis/06` §9–10 (WebKitGTK, log paths), `analysis/07` §3 (libfuse2), §6b (roll-forward), TechSpec recovery-state semantics (task_03 behaviors).

### Relevant Files

- `docs/_memory/glossary.md`, `COPY.md`, `PRODUCT.md`, `README.md` — boundary + sweep
- `packages/site/content/**` — prose sweep + new pages (install, getting-started, runbook)
- `web/src/**` — display strings only
- `skills/compozy/` — official skill
- New gate script under the repo's script conventions (explicit repo-root path)

### Dependent Files

- `packages/site/content/docs/cli/**` — generated in task_02; not hand-edited here
- `.github/workflows` / `Makefile` — gate wiring

### Related ADRs

- [ADR-012](adrs/adr-012.md) — the cut + boundary (this task's contract)
- [ADR-006](adrs/adr-006.md) — durable identity context

## Deliverables

- One product name across every product-language surface; boundary codified; occurrence gate green and wired
- Desktop docs set live (install/getting-started/runbook/threat-model) + official skill updated
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] UT-113 — brand-sweep occurrence gate (fails on product-language "Compozy" outside the allowlist)

## Web/Docs Impact

This task IS the web/docs impact: `web/` strings, full `packages/site` sweep + new pages, COPY/glossary/PRODUCT, official skill. **QA impact:** flag only — copy changes are user-visible; the site build/link gates own correctness; visual walks stay in task_07.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: `skills/compozy/` content/routing update (the official skill is the touched surface); no manifest/hook/registry contract change.
- Agent manageability: docs for the agent path (`compozy app`, recovery semantics) ship here.
- Config lifecycle: none — checked: no key added/changed; config reference edits for `[app]` happened in task_02.

## References

- TechSpec §Delete Targets, §Web/Docs Impact; `analysis/06` §§9–10; `analysis/07` §§3, 6b
- Skills to activate: `copywriting`, `documentation-writer`, `writing-skills` (official skill), `eng-design` only if marks surface

## Success Criteria

- Every assigned test case implemented and passing; occurrence gate green
- `make bun-lint` / site build green from repo root; no behavior diff in `web/`
- Reading any product page yields exactly one product name; command examples still say `compozy`
