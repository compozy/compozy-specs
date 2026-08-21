---
status: pending
title: Editorial Docs Synthesis
type: docs
complexity: low
---

# Task 11: Editorial Docs Synthesis

## Overview

Phase 7 — editorial only; no public behavior waits for it. Stitches the per-slice docs (shipped inside tasks 01–09) into the user-first Profiles narrative on the site, adds the glossary entry, and runs the final generated-reference sweep. Content decisions follow `COPY.md` claim standards and the user-first tutorial convention (own sidebar category, step-by-step end-to-end tutorial with expected outputs).

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

**Skills**: `documentation-writer`, `copywriting`. Read `packages/site/CLAUDE.md` and `COPY.md` before writing.

<requirements>
- MUST create the Profiles sidebar category stitching the per-slice pages into one narrative: concept → tutorial (create → work → switch → aggregate → layers → credentials → extensions) with expected outputs from `_dx.md` transcripts.
- MUST add the glossary entry: bare "Profile" names only this concept (ADR-008); existing compounds stay; update `docs/_memory/glossary.md`.
- MUST verify the final generated-reference state: `make cli-docs-check` green; OpenAPI-derived pages current; no hand-authored content inside generated trees.
- MUST apply `COPY.md` claim standards — non-technical-audience claims stay future-framed (D1); "separation, not security" stated where operators might assume otherwise.
- MUST cross-link configuration pages (precedence, file locations) and extension-authoring pages updated by tasks 04/08/09 — no restated content, links only.
- MUST NOT change runtime behavior, tests, or config — editorial only (docs-only gate exemption applies).
</requirements>

## Subtasks

- [ ] 11.1 Profiles docs category: index/concept page + user-first tutorial with expected outputs.
- [ ] 11.2 Glossary entry ("Profile") in site docs + `docs/_memory/glossary.md`.
- [ ] 11.3 Cross-link + de-duplicate the per-slice pages (lifecycle, selection, scoped reads, layers, credentials, extensions, presets).
- [ ] 11.4 Generated-reference sweep: `make cli-docs-check`, OpenAPI pages, meta.json ordering.
- [ ] 11.5 Copy pass per `COPY.md` (claims, maturity labels, future-framing).

## Implementation Details

Docs live under `packages/site/content/docs/`; the sidebar map is code (`lib/docs-navigation.ts` owns category ordering). Blog/announcement content is out of scope.

### Relevant Files

- `packages/site/content/docs/` — profiles category home; `configuration/`, `extensions/`, `notifications/` cross-link targets.
- `packages/site/lib/docs-navigation.ts` — sidebar category registration.
- `packages/site/content/docs/cli/profile/` — generated (verify only, never edit).
- `docs/_memory/glossary.md` — the arbitration line for "Profile".
- `COPY.md` — claim standards.

### Dependent Files

- `packages/site/content/docs/index.mdx` / `how-to-use-these-docs.mdx` — category mention if the IA requires it.

### Related ADRs

- [ADR-008](adrs/adr-008.md) — naming + glossary line.
- [ADR-010](adrs/adr-010.md) — "separation, not security" copy posture.

## Deliverables

- Profiles docs category with the end-to-end tutorial; glossary entry; clean generated-reference state; copy-reviewed pages.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)** — none assigned; the gates below own verification.

## Tests

No `_tests.md` IDs assigned — docs-only editorial task. Verification is owned by existing gates: site build + link integrity via `make gate` (bun lanes) and `make cli-docs-check` for generated drift; no bespoke prose tests (forbidden by test-placement rules).

### Web/Docs Impact

- `web/`: none — site-only.
- `packages/site`: this task is the impact (category, tutorial, glossary, cross-links).
- QA impact: none — no runtime behavior change; docs review rides task_12's journey coverage.
