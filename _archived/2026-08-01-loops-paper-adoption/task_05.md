---
status: completed
title: Site docs, generated references and official skill
type: docs
complexity: low
---

# Task 5: Site docs, generated references and official skill

## Overview

Documents the new grammar and semantics where users and agents read them: loop docs on the site
(including the evidence-gating pattern the paper re-read flagged as its sharpest transferable
idea), regenerated CLI/API references, and the official Compozy skill so agents operating loops
learn the new roots, metric criteria, origins, and re-attempt semantics.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST activate skills: `documentation-writer`, `fumadocs`, `copywriting` (public product language per `COPY.md`; vocabulary per `docs/_memory/glossary.md` — the artifact name is `capability`, loops are Loops).
- MUST update `packages/site/content/docs/loops/`: `dsl-reference.mdx` (+`metric` block), `reference-grammar.mdx` (+`previous.*`/`best.*` roots incl. `verdicts.<gate_id>` + `route_causes`), `guardrails.mdx` (+re-attempt semantics table + evidence-gating pattern section: "weak evidence is a route, not a hidden prompt instruction"), `running.mdx` (origin/best in run inspection), and add `ratchet.mdx` (concept, eligibility rule, seed-from-best, exhaustion→best) + `meta.json` entry.
- MUST regenerate CLI/API reference content affected by task_03 (`api/loops.mdx`, `cli/loop/*` via the docs codegen path) rather than hand-editing generated output.
- MUST update `skills/compozy/` loops guidance: new namespace roots, metric criterion contract (incl. extension scorer response field), origins vocabulary, both `next_generation` semantics, best/score in `loop status` output.
- MUST document runtime truth only — no aspirational claims (`COPY.md` claim standards); examples must lint against the real grammar.
- Docs-only exception applies: close on site build/link checks, not Go gates.
</requirements>

## Subtasks

- [x] 5.1 `ratchet.mdx` new page + meta.json ordering.
- [x] 5.2 Grammar/DSL reference updates (`dsl-reference`, `reference-grammar`) with lintable examples.
- [x] 5.3 `guardrails.mdx`: re-attempt semantics + evidence-gating pattern section.
- [x] 5.4 `running.mdx`: origin chips, best fields, `loop status` vs `loop runs` output shapes.
- [x] 5.5 Regenerate CLI/API reference pages; verify no drift.
- [x] 5.6 `skills/compozy/` loops section update.

## Implementation Details

TechSpec section: Web/Docs Impact. Terminology: ADR-002 route semantics, ADR-003 ratchet
vocabulary, ADR-004 origins.

### Relevant Files

- `packages/site/content/docs/loops/{dsl-reference,reference-grammar,guardrails,running,index}.mdx` + `meta.json`
- `packages/site/content/docs/api/loops.mdx`, `packages/site/content/docs/cli/loop/*` — generated refs
- `packages/site/content/docs/examples/{software-delivery-loop,review-and-fix-loop}.mdx` — update if grammar examples appear
- `skills/compozy/` — bundled official skill (loops guidance)

### Dependent Files

- `packages/site/CLAUDE.md` — site conventions (read before editing)
- Generated CLI JSON export — source for `cli/loop/*` regen

### Related ADRs

- [ADR-001](adrs/adr-001.md), [ADR-002](adrs/adr-002.md), [ADR-003](adrs/adr-003.md), [ADR-004](adrs/adr-004.md) — the documented contract.

### Web/Docs Impact

- `web/`: none — checked: no web code consumes site content.
- `packages/site`: this task IS the docs impact (paths above).
- QA impact: none — no runtime behavior change; doc accuracy is walked as part of task_07 scenario evidence (docs cited in walk notes).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: documents the extension scorer response field (`score`) in the extension criterion docs — the only extensibility surface touched, as documentation.
- Agent manageability: documents the agent path (`loop status`/`runs` output fields, validate diagnostics) — no surface change.
- Config lifecycle: none — no keys changed (spec Config Lifecycle: zero-key evidence).

## Deliverables

- Five updated/new MDX pages + regenerated references + updated official skill.
- Examples lint-valid against the shipped grammar.
- Site build + link checks green (`make gate` classifies docs lanes).

## Tests

No `_tests.md` IDs assigned — documentation task. Gates: site build, link check, and
`make codegen-check` (generated refs drift) own the invariants; per repo policy no prose/static
tests are added.

During runtime-truth review, the documented in-body `next_generation` route exposed a production
normalization defect. Its invariant belongs to `internal/loop/gate`; the canonical evaluator suite
now proves that `on_result.fail: next_generation` yields `RouteNextGeneration` instead of silently
falling back to `revise`.

## Success Criteria

- Site builds clean; internal links resolve; generated pages carry zero hand edits
- Official skill reflects the shipped contract (spot-checked against `loop status -o json` output)
- Copy follows `COPY.md`/glossary (no forbidden synonyms)

## Completion Evidence

- Site source generation, typecheck, 56 test files / 312 tests, and production build: PASS.
- Generated OpenAPI and CLI reference checks: PASS with zero generated-page drift.
- Internal links: 28 links across five changed Loop docs resolved, including anchors.
- Official skill metadata validation: PASS; `loops.md` remains 410 lines with routed contents.
- Gate: full escalation PASS, fingerprint `5c1a75196407170e6f46b5b7d9ff24d84fb97a6b`,
  log `.cache/gate/logs/full-1785590520.log` (18,685 Go tests; build and boundaries green).

## Compozy Impact Audit

- Native tools: no descriptor/schema change in this task; docs were checked against the shipped
  `compozy__loop_status` detail and `compozy__loop_runs` summary contracts from Task 03.
- Extensibility and hooks: documented the shipped extension scorer `score` field and existing
  origin vocabulary; no extension, hook, bundle, registry, bridge, MCP, or config lifecycle change.
- Workspace data isolation: no datum or access-path change; docs describe existing run-scoped,
  workspace-gated history. The routing fix changes only route normalization.
- Official Compozy skill: updated `skills/compozy/references/loops.md` for history roots, metric
  criteria, best state, origins, and succession semantics.
- QA tracker: the Task 06 succession scenario now explicitly covers in-body `next_generation`; Task
  07 must walk it with the other minted/reset scenarios before workstream completion.
