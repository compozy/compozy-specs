---
status: pending
title: Serve AGH as an MCP server by re-projecting the extension host API
type: backend
complexity: high
---

# Task 10: Serve AGH as an MCP server by re-projecting the extension host API

## Overview

External MCP-speaking agents cannot drive AGH: it consumes MCP servers but never serves one, even
though the extension host API already exposes the needed sessions/tasks/network/memory methods.
`agh mcp serve` makes AGH natively drivable by any MCP client — the highest-strategic-value item
of the extensibility slice and a direct SD-011 fulfillment.

**Authoritative sources:** this suite has no `_prd.md` / `_user_stories.md` / `_tests.md`.
`_techspec.md` §3.9 and `adrs/adr-008.md` are authoritative. Concrete test cases are inline below
(exact input/condition/expected).

Merges former task 26. Independent of task_09's registry/skills machinery (W5 parallelizable;
both feed `→12`).

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST publish sessions/tasks/network/memory/resources as MCP tools by re-projecting existing
   host-API methods — no new business logic in the façade.
2. MUST NOT mint new `agh__*` native tool IDs (ADR-008 §2): the MCP projection is a separate
   façade; native descriptors/digests untouched.
3. stdio transport MAY run unauthenticated for local spawn; any non-stdio transport REQUIRES
   token auth.
4. MUST enforce the same workspace isolation the host API enforces — no cross-workspace data
   through the façade.
5. MUST document the command + tool surface in `skills/agh/` and site docs.
</requirements>

## Subtasks

- [ ] 10.1 MCP serve adapter (tool schema projection from host-API methods).
- [ ] 10.2 `agh mcp serve` CLI verb + transport/auth options.
- [ ] 10.3 Isolation + auth tests; docs (`skills/agh/` + site guide).

## Implementation Details

See `_techspec.md` §3.9 / ADR-008. New serve adapter lives beside `internal/mcp/` consumer code
(own files, boundary rules; update `mage Boundaries` if a new subpackage appears). Capability
projection is a mechanical re-map of host-API methods with their isolation semantics intact.

### Relevant Files

- `internal/mcp/` — serve adapter home
- `internal/extension/` host API — projected methods (consumed)
- `internal/cli/` — `mcp serve` verb

### Dependent Files

- `skills/agh/` + `packages/site` — MCP serve documentation

### Related ADRs

- [ADR-008: `agh mcp serve` re-projects the extension host API as an MCP server](adrs/adr-008.md)
  — no new `agh__*` IDs; stdio may be unauth; non-stdio requires token; workspace isolation

### Competitor References

- `.resources/hermes/mcp_serve.py` — projection + transport model

## Deliverables

- Working MCP server exposing AGH management surfaces
- Zero new native tool IDs (digest diff clean)
- Docs for `agh mcp serve` + projected tool surface
- Every test case in `## Tests` implemented and passing **(REQUIRED)**

## Tests

No `_tests.md` for this suite — concrete inline cases below. Coverage target ≥80% on touched
packages; all Go tests under `-race`.

- Unit (serve-adapter tests (new, beside `internal/mcp` suites)):
  - [ ] Projected tool list mirrors the host-API method set (drift test: new host method without
        projection decision → failing test)
  - [ ] No projected tool ID collides with the `agh__*` namespace
  - [ ] Non-stdio transport without token → connection rejected deterministically
- Integration (`make test-integration`):
  - [ ] Real MCP client (test harness) over stdio lists sessions and creates a task through the
        façade; effects visible via native HTTP API
  - [ ] Workspace isolation: client scoped to workspace A cannot read workspace B sessions
- E2E (`make test-e2e-runtime`):
  - [ ] Harness scenario: external MCP client drives a small session lifecycle end-to-end

## Success Criteria

- Every assigned test case implemented and passing
- Coverage ≥80% on touched packages
- An unmodified third-party MCP client can list/inspect/operate AGH surfaces
- Zero new native tool IDs (digest diff clean)
- Non-stdio without token is rejected; workspace isolation holds through the façade
