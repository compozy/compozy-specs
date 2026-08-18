---
status: completed
title: Workspace-access policy core, mode source, session consent cache, audit substrate
type: backend
complexity: high
---

# Task 1: Workspace-access policy core, mode source, session consent cache, audit substrate

## Overview

Create `internal/workspaceaccess` — the single decision funnel for cross-workspace access (ADR-001) anchored on the session `PermissionMode` (ADR-007) — plus the daemon-owned substrate it depends on: a `ModeSource` over live session state, an in-memory `SessionConsentCache` with session-stop cleanup, and the audit emitter + event registry entries. After this task the policy is fully testable and wired into the daemon composition root, but no seam consults it yet (task_02).

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST implement the TechSpec Core Interfaces verbatim (`ActorRef`, `Seam`, `Request`, `Decision` with `PromptEligible`, `Mode`, `ModeSource`, `Consent`, `SessionConsentCache`, `AuditEmitter`, `AccessRecord`, `Deps`, `New`) and the fixed decision order (0–5). The decision chain is owned exclusively by this package's unit suite.
- MUST fail closed everywhere: step-zero validation (unknown kind, empty/non-canonical target, empty session id on agent-session actors), nil constructor deps, mode-lookup errors, unrecognized mode values, unknown sessions.
- MUST keep `internal/workspaceaccess` importing stdlib + `internal/logger` only; add the package rule to `magefiles/boundaries.go` in the same change.
- MUST implement the daemon `ModeSource` against the session manager's live session state (the `EffectivePermissions` the manager loads at start — `internal/session/manager_start.go:68`); unknown/stopped sessions return an error. Gap to close first: `session.Info` (`internal/session/session.go:54-86`) does NOT expose `EffectivePermissions` today (unexported accessors only) — extend `Info` and populate it in both builders (`manager_start_run.go` `sessionInfoForRead`, `query_metadata.go` `sessionInfoFromMeta`).
- MUST implement the consent cache as daemon-owned in-memory state keyed by session ID, cleared on session stop by registering on the daemon session-lifecycle fanout (`internal/daemon/hooks_bridge_observers.go` `sessionLifecycleFanout.OnSessionStopped`, wired in `boot_hook_observers.go`; release pattern mirrors `hostedMCP.ReleaseSession` at `internal/session/manager_stop_finalization.go:68`); no persistence, no list/revoke surface (ADR-007 invariant 9).
- MUST register `workspace.access_denied` (notify-eligible warning) and `workspace.access_granted` (success, non-notify) in the events registry and emit one best-effort `AccessRecord` per `Authorize` call through the observe `EventSummary` store, written with `context.WithoutCancel`; append failures log `Warn` and never change the decision.
- MUST populate `EventSummary` scoping per TechSpec: `WorkspaceID` = actor home (global when workspace-less); target/seam/source/mode/error in payload; `SessionID`/`AgentName`/`ActorKind` in summary columns.
- MUST NOT add config keys, tables, migrations, HTTP routes, CLI verbs, or native tools — the ADR-007 cut is absolute.
- Skills: `eng-code-guidelines`, `golang-pro`; tests under `eng-test-conventions` + `testing-boss`; concurrency of the consent cache under `golang-pro` discipline (race-safe map, `-race` gate).
</requirements>

## Subtasks

- [x] 1.1 Create `internal/workspaceaccess` with the contract file split decided up front (types/interfaces, default policy, audit record) — no god files.
- [x] 1.2 Implement `DefaultPolicy.Authorize` with the fixed chain: validation → operator → kind gate → same-workspace → mode (`approve-all` allow / `deny-all` deny / `approve-reads` → consent cache → `PromptEligible` miss).
- [x] 1.3 Implement the audit emission path: one record per call, best-effort, error-populated on failure denials.
- [x] 1.4 Add the `workspaceaccess` import rule to `magefiles/boundaries.go`.
- [x] 1.5 Expose `EffectivePermissions` on `session.Info` (both info builders) and implement the daemon `ModeSource` over `Sessions.Status` (session id → effective permission mode; error on unknown/empty).
- [x] 1.6 Implement the daemon in-memory `SessionConsentCache` (race-safe) and register its release on the session-lifecycle fanout (`OnSessionStopped`).
- [x] 1.7 Register the two event types in `internal/events` (own registry fragment file, mirroring `registry_network_coordination.go`) and implement the `AuditEmitter` over the observe event store.
- [x] 1.8 Wire policy construction in the daemon composition root (deps: mode source, consent cache, audit emitter, logger) — exported for task_02 seam injection.
- [x] 1.9 Implement all assigned unit + integration cases.

## Implementation Details

Follow TechSpec §Implementation Design (interface code blocks are normative) and §Monitoring and Observability. The consent cache and mode source live in `internal/daemon` (composition root, SD-008) as small dedicated files; `workspaceaccess` never imports session/config/events packages — everything arrives via the injected interfaces.

### Relevant Files

- `internal/tools/policy.go` — existing `PermissionMode` semantics the mapping mirrors (`applyPermissionCeiling`).
- `internal/config/config_agent_session.go:87-94` — canonical mode values (`deny-all`/`approve-reads`/`approve-all`).
- `internal/session/session.go:54-86` (`Info` — lacks `EffectivePermissions`; extend), `internal/session/manager_start_run.go` (`sessionInfoForRead`) + `internal/session/query_metadata.go` (`sessionInfoFromMeta`) — the two info builders to populate; `manager_helpers.go:56` (`startPermissions`) + `manager_start.go:68` — where the mode is resolved/loaded.
- `internal/store/session_meta_types.go:38` (`SessionMeta.EffectivePermissions`) — persisted mode field.
- `internal/admission/gate.go` — package model for `workspaceaccess` (tiny closed-set state, sentinel errors, nil-safe methods, compile-time interface assertion).
- `internal/events/registry.go` (`ToolCallDenied` const ~:178 and the `notify(global(warning(...)))` row ~:345 are the analogues) + `registry_metadata.go` (`notify`/`global` builders) — registry pattern for the two new types.
- `internal/store/types_event_summary.go:12-28` (`EventSummary` shape + `Validate` global-scope gating) and `internal/daemon/extensions_events.go:154-180` — the canonical `context.WithoutCancel` append to copy for the audit emitter.
- `internal/daemon/hooks_bridge_observers.go:38-70` (`sessionLifecycleFanout`) + `boot_hook_observers.go` — where the consent cache registers `OnSessionStopped`; release naming mirrors `hostedMCP.ReleaseSession` (`manager_stop_finalization.go:68`).
- `internal/daemon/tool_approval_boot.go` — boot-file pattern (build deps, stash on state, ~50 lines) for the policy/cache boot.
- `magefiles/boundaries.go:181-185` — the 4-rule `internal/store/workspacedb` block is the exact model for the `workspaceaccess` rules (forbid daemon/httpapi/udsapi/cli imports).

### Dependent Files

- `internal/daemon` boot files — gain policy construction + consent cleanup wiring.
- `internal/events/registry_test.go` — coverage matrix must include the two new event names (UT-046).
- `internal/session` info-builder tests — `EffectivePermissions` exposure on `Info`.
- Task_02 seams consume the exported policy handle.

### Related ADRs

- [ADR-007](adrs/adr-007.md) — mode anchoring, consent semantics, decision chain inputs.
- [ADR-001](adrs/adr-001.md) — central policy at seams, fail-closed posture, canonical-ID rule.
- [ADR-006](adrs/adr-006.md) — audit-as-detection beta posture.

### Web/Docs Impact

- `web/`: none — checked surfaces: no contract/OpenAPI change, no web consumption of the policy package; web work is task_03/04's owner projection + confirm flow.
- `packages/site`: none in this task — mode-mapping docs land in task_05 (`content/runtime/core/sessions/permissions.mdx`, workspaces pages).
- QA impact: none — no user-visible behavior change yet (policy is built but unconsulted; seams still hard-deny).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: no new extension points, hooks, tools, resources, bundles, registries, bridge SDKs, or MCP surfaces; the actor-kind gate (`ActorAgentSession` only) is the invariant that keeps extension/automation/network-peer actors out — asserted in UT-051.
- Agent manageability: no new surfaces by design (ADR-007 — no new manageable state exists); audit events become filterable via existing `GET /api/logs`, `compozy logs --type <event-type>`, `compozy__logs`, and `compozy__observe_search` surfaces once emitted.
- Config lifecycle: none — checked surfaces: no new keys; `[permissions] mode` untouched; `permissions.*` trust-root protection pre-existing (`internal/config/tool_surface.go:283`).

## Deliverables

- `internal/workspaceaccess` package with the normative interfaces, fixed decision chain, and audit contract.
- Daemon `ModeSource`, race-safe `SessionConsentCache` with session-stop cleanup, and `AuditEmitter` over the observe store.
- Event registry entries for `workspace.access_granted`/`workspace.access_denied` with the TechSpec notify classes.
- `magefiles/boundaries.go` rule for the new package.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-001, UT-002, UT-007, UT-011, UT-012, UT-013, UT-051, UT-053, UT-055, UT-063, UT-076, UT-077, UT-078, UT-079, UT-080, UT-081, UT-082 — `workspaceaccess` decision chain: operator/same-workspace fast paths, step-zero validation, constructor fail-closed, audit best-effort, kind gate, full mode mapping incl. consent hits/misses, error denials, workspace-less sessions.
- [x] UT-084 — daemon consent cache cleared on session stop.
- [x] UT-046 — events registry declares the two types with correct notify classes.
- [x] IT-003 — representative allow/deny/error decisions each append exactly one queryable `EventSummary`; forced append failure leaves the decision unchanged.
- [x] IT-024 — real `ModeSource` resolves a live managed session's mode; unknown/stopped session denies with wrapped error.
- [x] IT-025 — consent lifecycle through the daemon: `allow_session` visible to subsequent `Authorize`; session stop clears it.

## Success Criteria

- Every assigned test case implemented and passing
- `go test -race ./internal/workspaceaccess/... ./internal/daemon/... ./internal/events/...` green; `make lint` zero issues; boundaries check passes with the new rule.
- The decision chain matrix lives only in the `workspaceaccess` unit suite (suite-ownership audit: no integration case re-asserts precedence).
- No new persisted state anywhere (schema, config, routes verified unchanged).
