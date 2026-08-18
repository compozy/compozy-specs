---
status: completed
title: Docs, config examples, official Compozy skill
type: docs
complexity: medium
---

# Task 8: Docs, config examples, official Compozy skill

## Overview

Documents the shipped feature across every public knowledge surface: the new Worktrees section on the site with user-first step-by-step tutorials, configuration and lifecycle-matrix pages, the `forge.provider` extension protocol docs with GitHub setup, regenerated CLI/API references, example configs, and the official `skills/compozy/` runtime-truth update so agents learn the new verbs, tools, policies, and error codes.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST create the Worktrees docs section in `packages/site` with its own sidebar category and step-by-step end-to-end tutorials with expected outputs (user-first docs convention): concepts/index, creating & adopting, navigation & selection, task isolation (per-run + fan-out), loop environments, assisted exit (incl. GitHub extension setup: secret binding + `gh` fallback + zero-credential tier), removal & recovery, configuration, agent surface (CLI/API/tools).
2. MUST update `config-toml.mdx` + `lifecycle-matrix.mdx` for `[worktrees]` and `task.orchestration.profile.default_worktree_mode`, and the example `config.toml` files.
3. MUST document the `forge.provider` extension surface (provide, service methods, capabilities contract incl. vocabulary/draft/compare-template, conformance expectations) in the extension docs.
4. MUST regenerate CLI and API references (`make codegen` artifacts + generated CLI docs) and verify with `make codegen-check` + the site build.
5. MUST update `skills/compozy/` (activate `writing-skills`) with worktree verbs, native tools, policy modes, exit flow, and the deterministic error-code vocabulary — runtime truth only.
6. MUST follow `COPY.md` + `docs/_memory/glossary.md` (canonical term Worktree; never "recipe"/"workflow" for capabilities; claim standards for "supported"/"shipping"), activating `documentation-writer` + `copywriting` for public prose.
7. MUST cross-link the workspaces docs (resolver/multi-root pages) to the worktree model without rewriting their contracts.
</requirements>

## Subtasks

- [x] 8.1 Worktrees section: concepts + creating/adopting + navigation tutorials (with expected outputs)
- [x] 8.2 Task isolation + loop environments tutorials
- [x] 8.3 Assisted exit + GitHub extension setup + zero-credential tier
- [x] 8.4 Removal & recovery + configuration pages (+ config-toml/lifecycle-matrix updates + examples)
- [x] 8.5 Agent-surface page (CLI/API/tools + error codes) + `forge.provider` protocol docs
- [x] 8.6 Regenerated CLI/API references verified; sidebar/IA wiring
- [x] 8.7 `skills/compozy/` update

Execution note: the generated OpenAPI Worktrees page, lifecycle matrix, and root config example were
already current from the owning implementation tasks. Cobra reference regeneration changed only the
fan-out page after its public required-flag contract was corrected. Runtime QA remains in Task 10.

## Implementation Details

Site conventions: `packages/site/CLAUDE.md` (read before working there). Docs navigation: `packages/site/lib/docs-navigation.ts`. Existing anchors: `packages/site/content/docs/workspaces/{index,resolver,multi-root,config-overlays}.mdx`, `configuration/config-toml.mdx`, `configuration/lifecycle-matrix.mdx`, extension docs section.

### Relevant Files

- `packages/site/content/docs/` — section home; `lib/docs-navigation.ts` — sidebar category
- `_techspec.md` §Agent Manageability, §Config Lifecycle, §API Endpoints — the truth being documented
- `skills/compozy/` — official bundled skill
- `COPY.md`, `docs/_memory/glossary.md` — language authority

### Dependent Files

- Generated CLI reference pages (cobra export), generated API reference, `openapi/compozy.json`

### Related ADRs

- [ADR-005](adrs/adr-005.md) — placement docs; [ADR-010](adrs/adr-010.md) — forge/credential docs; [ADR-011](adrs/adr-011.md) — namespace/reclamation docs

### Competitor References

- none — documentation task; behavior authority is the shipped runtime + TechSpec.

### Web/Docs Impact

- `web/`: none (checked: no SPA changes).
- `packages/site`: everything above — this task IS the docs impact.
- QA impact: the existing fan-out isolation scenario now also checks that the CLI rejects a missing
  idempotency key before enqueue; all doc-accuracy walks ride the Task 10 scenario walks.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: `forge.provider` protocol documented; extension docs updated (no extension code surfaces change).
- Agent manageability: the agent path is documented (CLI/API/tools + errors), and CLI fan-out now
  enforces the backend-required idempotency identity before enqueue.
- Config lifecycle: docs + examples leg of the `[worktrees]` and profile-default keys (structs/validation shipped in tasks 01/04).

## Deliverables

- Worktrees docs section live with tutorials and expected outputs
- Config/extension/agent-surface docs + regenerated references
- `skills/compozy/` updated to runtime truth
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED — none assigned; gates below stand in)**

## Tests

No `_tests.md` IDs assigned — documentation carries no unit/integration/e2e cases (static/prose tests are forbidden by default). Owning gates instead:

- [x] Codegen check clean through the repo-root Turbo task graph
- [x] `packages/site` lint, typecheck, and build green through repo-root Turbo
- [x] Link integrity through the site build's existing checks

## Success Criteria

- Site builds green from repo root; `make codegen-check` clean
- Every public surface from tasks 01-07 (verbs, routes, tools, config keys, error codes) appears in docs with at least one tutorial path exercising it
- `skills/compozy/` names only shipped behavior (runtime-truth audit)
