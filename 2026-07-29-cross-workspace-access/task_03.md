---
status: completed
title: Session owner projection endpoint (phase 0 backend)
type: backend
complexity: low
---

# Task 3: Session owner projection endpoint (phase 0 backend)

## Overview

Ship `GET /api/sessions/:session_id/owner` — the minimal owner projection (`{session_id, workspace_id, workspace_name}`, 404 unknown) that lets the web deep-link confirm flow (task_04) resolve a foreign session's owning workspace without fetching any foreign payload pre-confirmation (ADR-004). Independent of the policy track; parallelizable from day one.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST return exactly three fields — `session_id`, `workspace_id`, `workspace_name` — and 404 for unknown sessions; no agent/provider/activity/state fields ever (minimality is the contract, UT-061/IT-020).
- MUST land the handler once in `internal/api/core` with HTTP and UDS registering the same handler under the existing operator-surface read posture (same as `GET /api/sessions`); no new guards, no new auth semantics.
- MUST add the contract type in `internal/api/contract/` and co-ship codegen: `make codegen` regenerates OpenAPI + TS; `make codegen-check` clean (skill `eng-contract-codegen-coship`).
- MUST resolve the owner from the global session catalog the daemon already maintains — no new store queries beyond what the existing session lookup surface provides; workspace name resolved via the workspace registry.
- Skills: `eng-code-guidelines`, `golang-pro`, `eng-contract-codegen-coship`, `eng-data-boundaries` (public read boundary: scope, completeness, parity), `eng-test-conventions`.
</requirements>

## Subtasks

- [x] 3.1 Add the `SessionOwner` contract type (`internal/api/contract/`, small dedicated file per the existing naming pattern).
- [x] 3.2 Implement the core handler (`internal/api/core/session_owner.go`): global session lookup → owner projection or 404.
- [x] 3.3 Register the route in both `session_routes.go` tables (HTTP + UDS) with byte-identical payloads; update the golden route table in `httpapi/handlers_test.go`.
- [x] 3.4 Run `make codegen`; commit regenerated OpenAPI + TS artifacts.
- [x] 3.5 Implement the assigned unit + integration cases (minimality, 404, HTTP/UDS parity, codegen drift).

## Implementation Details

Follow TechSpec §API Endpoints (the single-row table is the whole surface) and ADR-004 Implementation Notes (B-006 amendment: the projection exists precisely because the global catalog returns full `SessionPayload` objects and the client must not receive one pre-confirm).

### Relevant Files

- `internal/api/core/session_detail.go` — the 39-line read-only handler exemplar (`GetSessionByID`: param guard → `h.Sessions.Status` → `respondError` with `StatusForSessionError` → `c.JSON`); mirror its shape.
- `internal/api/contract/drain.go` / `session_catalog.go` — exemplars of tiny dedicated contract files (closed payload, doc comments, json tags).
- `internal/api/httpapi/session_routes.go` + `internal/api/udsapi/session_routes.go` — the byte-identical route tables; a new endpoint MUST land in both in the same change.
- `internal/api/httpapi/handlers_test.go` ~:196-205 — the golden route table that must gain the new route.
- `internal/api/httpapi/transport_parity_integration_test.go` + `internal/api/udsapi/transport_parity_integration_test.go` — parity harness for IT-020.
- `internal/workspace` — workspace name resolution from the registry id.

### Dependent Files

- `openapi/compozy.json` + `web/src/generated/compozy-openapi.d.ts` — regenerate via `make codegen` (new operation; the existing `getSessionByID` returns the full envelope and is not a substitute).
- `web/src/` task_04 modules consume the generated type (`OperationResponse` pattern in `web/src/systems/session/types.ts`).

### Related ADRs

- [ADR-004](adrs/adr-004.md) — minimal owner projection contract and its rationale.

### Web/Docs Impact

- `web/`: `web/src/generated/compozy-openapi.d.ts` gains the projection type (generated); consuming query-key module/loaders are task_04's deliverable.
- `packages/site`: API reference regenerates from OpenAPI (`content/runtime/api-reference/`); no hand-written page in this task (task_05 covers narrative docs).
- QA impact: none — no user-visible behavior change until task_04 consumes the projection (loopback-open operator read, same posture as `GET /api/sessions`).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: no hooks/tools/resources/bundles/bridges/MCP; protocol docs = OpenAPI regen only.
- Agent manageability: the route is an operator-surface read consumed by the SPA; agents don't need it (policy governs agents, ADR-004); HTTP/UDS parity asserted in IT-020.
- Config lifecycle: none — checked surfaces: no keys, no defaults, no overlay behavior.

## Deliverables

- Contract type + core handler + HTTP/UDS registration for the owner projection.
- Regenerated OpenAPI + TS with `make codegen-check` clean.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-061 — handler returns exactly the three fields; unknown session → 404.
- [x] IT-020 — HTTP and UDS byte-identical minimal payloads; no extra fields present.
- [x] IT-013 — `make codegen-check` clean after the contract addition.

## Success Criteria

- Every assigned test case implemented and passing
- `make lint` + scoped `go test -race ./internal/api/...` green; `make codegen-check` clean.
- Payload field-set proof (test-asserted) that no foreign session data beyond the three fields crosses the wire.
