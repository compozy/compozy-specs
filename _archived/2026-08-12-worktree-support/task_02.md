---
status: completed
title: "Worktree public surface: API, CLI, native tools, streams"
type: backend
complexity: high
---

# Task 2: Worktree public surface: API, CLI, native tools, streams

## Overview

Exposes the task-01 domain through every agent-operable surface in one pass: contract types, shared `BaseHandlers`, byte-parity HTTP + UDS routes (CRUD, adopt, create-cancel, status, remove, dismiss), the per-worktree and catalog SSE streams, OpenAPI/TS codegen, the `compozy worktree` CLI verb group, and the `compozy__worktree` native toolset. This is the "no partial-surface completion" slice — contract → handlers → transports → CLI → tools → codegen land together.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST add `internal/api/contract/worktrees.go` payloads and register the route table from `_techspec.md` §API Endpoints (list w/ `refresh`, create 202+pending, adopt, inspect, status w/ `refresh`/`forge`, create-cancel, remove w/ `force` + 409 refusal payload, dismiss, per-worktree stream, catalog stream) identically in `internal/api/httpapi` and `internal/api/udsapi` via shared `BaseHandlers` (exit routes are task 05).
2. MUST map every sentinel error to its deterministic wire code (TechSpec table) — one code per cause, byte-identical across HTTP/UDS/CLI/tools; refusal bodies carry the machine-readable risk inventory.
3. MUST implement the per-worktree SSE stream (durable events, `after_sequence` replay) and the `worktree_catalog_changed` catalog stream mirroring the session catalog stream's shape and trust tier (operator surface; frames workspace-attributed).
4. MUST add the `compozy worktree` verb group — `list`, `create`, `cancel`, `adopt`, `inspect`, `status`, `remove`, `dismiss` in this task (exit verbs are task 05) — with `-o json|jsonl` via the `outputBundle` convention, deterministic exit codes, and the workspace resolution ladder.
5. MUST add native toolset `compozy__worktree` (`_list`/`_inspect` RiskRead, `_create` RiskMutating, `_remove` RiskDestructive + `Destructive: true`) with the workspace input binder/authorizer, catalog snapshot + digests, and git-capability availability diagnostics.
6. MUST regenerate OpenAPI + TS (`make codegen`) in the same change; `make codegen-check` is an acceptance gate.
7. MUST keep list payloads carrying per-workspace `repo` metadata (`git_backed`, `git_available`, diagnostic) and the full state vocabulary (`pending|ready|failed|removing|missing|removed`, discovered merge, stale flags) exactly as the UI renders it.
8. MUST place the new route group in the correct surface tier (`RegisterSurfaceRoutes`) and update `magefiles/boundaries.go` if any new `internal/api/*` subpackage appears.
</requirements>

## Subtasks

- [x] 2.1 Contract payloads + error-code mapping table (sentinel → wire code, exhaustive test)
- [x] 2.2 `BaseHandlers` worktree methods in `internal/api/core` (workspace scope via the existing `workspaceScope` object)
- [x] 2.3 HTTP + UDS route registration (surface tier) + refusal payload shapes
- [x] 2.4 Per-worktree stream + catalog stream (durable append → broadcast, `after_sequence` replay)
- [x] 2.5 CLI verb group with structured output + exit codes
- [x] 2.6 Native toolset + descriptors + toolset catalog + digests + availability diagnostics
- [x] 2.7 `make codegen` artifacts (OpenAPI, TS types) co-shipped
- [x] 2.8 Transport/CLI parity + isolation integration tests

## Implementation Details

Follow `_techspec.md` §API Endpoints, §Agent Manageability Plan. Handlers import `internal/worktree` directly (established pattern). Route param stays `:workspace_id` with the dual-id scope object.

### Relevant Files

- `internal/api/core/workspaces.go:19-257` + `workspace_scope.go:18-157` — handler + scope patterns to mirror
- `internal/api/httpapi/routes.go:68-74` + `internal/api/udsapi/routes.go:77-84` — byte-identical registration precedent; `surface_routes.go:9-24` tiers
- `internal/api/httpapi/session_routes.go:8` (`/sessions/catalog-stream`) — catalog stream shape to mirror
- `internal/cli/workspace.go:32-302`, `format.go:63-190`, `workspace_resolution.go:84-146` — CLI verb group + output bundle + resolution ladder
- `internal/tools/builtin/workspace.go:13-104`, `descriptors.go:28-89`, `toolsets.go:6-10`, `builtin_ids.go:105-112,399-455` — native tool patterns; `internal/daemon/native_workspace_input_binder.go`/`_authorizer.go` — workspace input enforcement
- `internal/codegen/openapits` + `magefiles/codegen.go:35,81` — generation entry points

### Dependent Files

- `openapi/compozy.json`, `web/src/generated/compozy-openapi.d.ts` — regenerated
- `internal/tools/builtin/testdata/native-tool-catalog.json` — new toolset snapshot + digests
- `internal/daemon/boot_tool_registry.go` — toolset registration
- `internal/api/httpapi/handlers_test.go` + `internal/api/testutil` — parity suites

### Related ADRs

- [ADR-007](adrs/adr-007.md) — read-model shapes, tombstone visibility
- [ADR-002](adrs/adr-002.md) — adopt-on-select surface semantics

### Competitor References

- `.resources/herdr/src/app/api/worktrees.rs:48-172` + `api/schema/worktrees.rs:3-50` (deferred create/remove API returning real outcomes; JSON error shape with distinct codes)
- `.resources/herdr/src/cli/worktree.rs` (worktree CLI verb group shape)

### Web/Docs Impact

- `web/`: `web/src/generated/compozy-openapi.d.ts` regenerates (consumed by task 06's data layer); no component work here.
- `packages/site`: generated CLI/API references regenerate; prose pages land in task 08.
- QA impact: new scenarios — add content-addressed `untested` files under `docs/qa/scenarios/` for CLI worktree lifecycle (create/list/inspect/status/remove two-step/dismiss via `-o json`) and API parity; flag only — the walk runs in the program QA phase (task 10).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: native tool catalog + toolset registry gain the `compozy__worktree` entries with digests; no manifest/hook/MCP/bridge changes (checked: `internal/extension*`, `internal/bridgesdk` untouched).
- Agent manageability: this task IS the plan — CLI verbs with structured output, HTTP/UDS parity, native tools with risk classes, deterministic error table, state/config discovery via list/inspect payloads.
- Config lifecycle: none — no new keys (checked: `internal/config` untouched; `[worktrees]` shipped in task 01).

## Deliverables

- Full route table live on both transports with parity-tested payloads and error codes
- `compozy worktree` verbs + native toolset shipping with catalog/digests
- Streams live with replay; codegen artifacts committed
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-119, UT-120 — sentinel→wire mapping completeness, refusal payload shape
- [x] UT-121, UT-122 — CLI JSON shapes, exit codes, empty-list semantics
- [x] IT-034 — stream lifecycle events + `after_sequence` replay + ordering (bound-session turn-end leg continues in task 03)
- [x] IT-038 — cross-workspace isolation across transports and tools
- [x] E2E-003 — CLI journey against the daemon
- [x] E2E-004 — native-tools journey under permission modes

## Success Criteria

- Every assigned test case implemented and passing
- `make codegen-check` clean; `make gate` green
- Same fixture state yields byte-equal JSON on HTTP, UDS, and CLI `-o json` for every route shipped here
- Removal two-step refusal identical across all four surfaces (API/UDS/CLI/tool)
