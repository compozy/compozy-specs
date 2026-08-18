---
status: pending
title: "P1 Daemon — Unified Registry, Approval Substrate, Agent Surface"
type: backend
complexity: critical
---

# Task 1: P1 Daemon — Unified Registry, Approval Substrate, Agent Surface

## Overview

Builds the daemon-canonical command registry (`internal/cmdpalette` + `corecmds`) that absorbs every core command id, extends the windowmanager client registry into the palette's attachment/targeting authority, and lands the async `ApprovalCoordinator` on the tools layer (durable `tool_approval_pending`, migration `00069`) that the frozen 202 invoke contract requires. Ships the complete P1 agent surface in the same slice: `/api/cmd-palette/{commands,clients,invoke,stream}` + `/api/tools/approvals/*` on HTTP **and** UDS, OpenAPI + generated TS, CLI `cmd-palette list|inspect|invoke|clients` + `approvals show|cancel`, native tools, SSE invalidation, events, skills/glossary/docs.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting; read every ADR in `adrs/` and any `memory/task_NN.md` present
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — fix root causes; frozen `_dx.md`/Part II contracts are binding; surface gaps as clarifications, never invent
</critical>

<requirements>
1. MUST implement `internal/cmdpalette` exactly per Part II Core Interfaces: serializable `Descriptor` (closed `Action` union; execution site DERIVED from `Action.Kind`; declared `ExecutionPolicy`; `Destructive ⇒ Confirmation`), `Registry.Catalog/Invoke`, `Provider`, `Catalog.Revision` computed over STRUCTURAL state only with a separate `context_revision` (SI: never client-resolved availability).
2. MUST absorb the full core inventory into `corecmds` (every `internal/windowmanager` action id, every OS app launch, every settings destination, view entries) — split per section under the 500-line cap; duplicate core id = boot failure; `ext.*` collision rejects the later registration (SI-11).
3. MUST extend the windowmanager client registry as the single identity authority (no palette-local registry): `ClientView` gains `kind` (shell|browser) + volatile palette-context snapshot fields; correlated `client_command` envelopes ride the existing fenced WS with ack/result frames, bounded delivery timeout, and `client_disconnected` structured failure. Adding the frame kind MUST co-ship the Go contract const/struct AND the web Zod strict-union branch (the union rejects unknown kinds).
4. MUST enforce the two authorization paths: self-originated calls prove attachment with the registration-minted token (`X-Compozy-Client-Token`, validated token↔ClientID); control-plane callers (CLI/UDS, privileged HTTP, native tools) target a validated `ClientID` under their own authorization with no token. Listing never auto-selects a client; invoke auto-selects only with exactly one attachment; multiple → 409 `multiple_clients` listing ids; none for client-context → 412 `no_attached_shell`.
5. MUST land the async approval substrate in `internal/tools` (never palette-local): `ApprovalCoordinator` Begin/Resolve/Status/Cancel; fragment `43_tool_approval_pending.sql` → migration `00069` (append-only, BEFORE the palette migration `00070`); separate `approval_status`/`execution_status` machines; `invocation_id` as provider idempotency key; `resume_fence` claimed atomically on `approved→dispatching`; crash recovery → `uncertain`, never silently retried; `cannot_defer_secrets` (password-typed args reject at invoke; secrets never persisted); single-flight lease released only on the execution terminal (SI-1).
6. MUST ship every public surface of this slice per frozen `_dx.md`: routes + status codes byte-exact, `GET /api/tools/approvals/{id}` + `POST .../cancel`, CLI transcripts + exit codes (0/1/2), native tools `compozy__cmd_palette_list|invoke` with the `client` param (frozen — deliberate divergence from the WM `client_id` precedent), `-o json|jsonl|toon`, deterministic errors with reasons byte-identical across UI and structured errors (BR-8).
7. MUST resolve public `workspace` inputs (ID/name/path) through the canonical resolver; native tools bind the trusted session workspace before validation; every storage row/SSE payload/interface carries the typed canonical workspace ID only (L-033).
8. MUST register the `cmd_palette` event family (`internal/events/names.go` + registry + a component decision) for this slice's events (`catalog.changed`, `command.invoked` with `invocation_id`/`approval_id`) and emit at owning domain boundaries; SSE stream via `PrepareSSE` with reconcile-on-open (no per-event replay cursor).
9. MUST co-ship contract artifacts in this slice: OpenAPI ops + `make codegen` (generated TS), `native-tool-catalog.json` regen, transport-parity test rows, `skills/compozy/` update, glossary `cmd_palette` entry (ADR-005), owning docs page updates.
10. MUST NOT create: a second client registry, a palette-local pending-approval table, hooks tailing events, compat shims, or any `execute_hook` usage (that seam is untouched here; its precise deletion is task_08).
11. SHOULD generalize from the clarify runtime's pending/resolve/timeout/cancel shape (`internal/daemon/clarify_bridge*.go`) and reuse existing approval reason codes (`internal/tools/reason.go`) — never invent new reason vocabulary.
</requirements>

## Subtasks

- [ ] 1.1 `internal/cmdpalette` domain types + validation (descriptor/action/policy/predicate; file split decided up front: types, validation, registry, catalog, invoke)
- [ ] 1.2 `internal/cmdpalette/corecmds` — full core absorption (WM actions, apps, settings destinations, view entries) with boot-time uniqueness
- [ ] 1.3 Context-key contract v1 (closed set per Assumptions) + daemon-side predicate evaluator resolving against a named client's snapshot
- [ ] 1.4 Windowmanager client registry extension: `kind`, volatile context snapshot + debounced refresh, attachment token mint/validate, `client_command` envelopes (Go const/struct + readPump + ack/result + web Zod branch co-ship)
- [ ] 1.5 Approval substrate: `ApprovalCoordinator` + `tool_approval_pending` (fragment 43 → migration `00069`, sqlc queries, store wrapper, delete-trigger), fence, boot recovery, `cannot_defer_secrets`
- [ ] 1.6 Approvals public surface: status/cancel routes (both transports) + CLI `compozy approvals show|cancel` + native parity
- [ ] 1.7 Invoke orchestration: schema validation (422 fields) → targeting → availability re-eval (412 with catalog-identical reason) → single-flight (409) → route by action kind (tool via `internal/tools` policy; client ops over the client channel)
- [ ] 1.8 API family: `internal/api/core/cmd_palette*.go` handlers + port interface on `BaseHandlers`, `cmd_palette_routes.go` on httpapi + udsapi, OpenAPI registry, SSE stream
- [ ] 1.9 CLI `compozy cmd-palette list|inspect|invoke|clients` (family files + output bundle + structured errors per transcripts)
- [ ] 1.10 Native tools: `builtin_ids_cmd_palette.go` + `builtin/cmd_palette.go` descriptors/schemas + toolset + catalog regen
- [ ] 1.11 Events + observability: family registration, `command.invoked` correlation fields, slog fields, failure counters
- [ ] 1.12 Co-ship: `make codegen` artifacts, transport-parity rows, `skills/compozy/`, glossary entry, docs pages; wire everything in `internal/daemon/boot_resource_graph.go`

## Implementation Details

Reference `_spec.md` Part II: Core Interfaces (descriptor/registry/approval blocks), Data Models (`tool_approval_pending` column rationale), API Endpoints, Key Decisions (attached-client model, workspace resolution, approval ownership), Safety Invariants 1, 5, 8, 9, 11, 17.

### Skills

`eng-code-guidelines` · `golang-master` · `eng-schema-migration` · `eng-contract-codegen-coship` · `eng-test-conventions` · `eng-consolidate-test-suites` · `testing-boss` · `eng-cleanup-failure-paths`

### Relevant Files

- `internal/windowmanager/clients.go`, `client_projection.go`, `client_revision.go`, `subscription.go` — the client registry this task extends (identity authority)
- `internal/api/core/window_manager_ws.go` + `internal/api/contract/windowmanager_stream.go` — WS transport + frame envelopes; today server-push-only (`readPump` rejects client frames) — `client_command` changes both
- `web/src/systems/os/lib/window-manager-stream-schema.ts` — strict Zod union that MUST gain the `client_command`/ack branch in the same change
- `internal/command/catalog.go` + `types.go` — daemon slash-command catalog: the sha256 `Revision` + `Descriptor` precedent (sits beside, never merged)
- `internal/extension/command.go` — `Manager.Commands(workspaceID)` storage-free health-gated projection (the pattern `Manager.CmdPalette` follows in task_07)
- `internal/tools/tool.go:259` (`ApprovalBridge`), `dispatch_hooks.go:47` (`requestApproval`), `internal/daemon/tool_approval_bridge.go` — today's synchronous, id-less approval seam this task extends
- `internal/daemon/clarify_bridge.go` + `clarify_bridge_lifecycle.go` + `internal/tools/clarify.go` — the pending/resolve/timeout/cancel runtime model (without durability)
- `internal/tools/reason.go` — existing `ReasonApproval*` codes to reuse
- `internal/store/globaldb/schema/definitions/33_extensions.sql` (trigger pattern) + `42_tool_approval_grants.sql` (FK pattern) + `schema/migrations/` (head `00068`) + `sqlc.yaml` + `global_db_tool_approval_grants.go` (store-wrapper model)
- `internal/api/core/{base_handlers.go,window_manager_port.go,window_manager_handlers.go,window_manager_errors.go,session_catalog_stream.go,sse.go,parsers.go}` — handler/port/error/SSE exemplars
- `internal/api/httpapi/window_manager_routes.go` + `internal/api/udsapi/window_manager_routes.go` + `internal/api/spec/registry_extensions.go` — route + OpenAPI registration patterns
- `internal/cli/window_manager_common.go` + `window_manager_desktop.go` + `format.go` + `workspace_resolution.go` — CLI family + output + resolver patterns
- `internal/tools/builtin_ids.go` (add sibling file, 30.5K — do not grow), `builtin/descriptors.go`, `builtin/window_manager.go` + schemas, `builtin/testdata/native-tool-catalog.json` — native-tool declaration + catalog contract
- `internal/events/names.go` + `registry_base.go` + `components.go` — event registration (first UI-shell family; component decision required)
- `internal/daemon/boot_resource_graph.go` — composition-root wiring site

### Dependent Files

- `internal/api/core/base_handlers.go` — gains `CmdPalette` port field
- `internal/api/httpapi/routes.go` + `internal/api/udsapi/routes.go` — call the new route registrars
- `internal/api/core/transport_parity_integration_test.go` — parity rows for every new route
- `internal/cli/root.go` — registers `newCmdPaletteCommand` (+ approvals verbs)
- `openapi/compozy.json`, `web/src/generated/compozy-openapi.d.ts`, `sdk` contract tables — regenerate via `make codegen`
- `internal/store/globaldb/schema/migrations/atlas.sum` — new migration appended via codegen
- `packages/site/content/docs/cli/` — CLI docs regenerate (`make cli-docs` inside codegen)
- `skills/compozy/`, `docs/_memory/glossary.md` — co-ship updates
- `internal/store/migrate_streams_test.go` — this task extends fresh/reopen/ahead/integrity for `00069` (the formal ID IT-020 completes in task_03 with `00070`)

### Competitor References

- Legacy commander (cited by path in `analysis/01_legacy_commander.md`; repo `~/dev/compozy-code` — not vendored under `.resources/`): descriptor contract `packages/tauri/src/app-core/shared/commander.ts` — the serializable handler-free descriptor keystone.

### Related ADRs

- [ADR-001](adrs/adr-001.md) — one unified registry, full OS coverage (this task's absorption checklist)
- [ADR-003](adrs/adr-003.md) — daemon owns signals + weights authority (scoring is task_03's TS scorer; no Go scorer here)
- [ADR-005](adrs/adr-005.md) — `cmd_palette`/`cmd-palette` identifier on every surface; glossary co-ship
- [ADR-006](adrs/adr-006.md) — daemon-canonical registry with web projection; execution-site derivation

### Web/Docs Impact

- `web/`: `web/src/generated/compozy-openapi.d.ts` (regen only) and the co-shipped `client_command` branch in `web/src/systems/os/lib/window-manager-stream-schema.ts` + `lib/window-manager-types.ts` + `hooks/use-window-manager-stream.ts` (envelope handling only — the consuming palette UI is task_02).
- `packages/site`: generated CLI reference under `content/docs/cli/` (new `cmd-palette`, `approvals` verb dirs); generated API reference from `openapi/compozy.json`; no hand-written page this slice (owning pages update in tasks 05/07/09).
- QA impact: flag only (loop rule — walks run in task_12): add content-addressed `untested` scenario for the **agent invoke path** (list → inspect → invoke → approval via CLI/native); reset `ET-session-command-catalog-parity.md` is NOT affected (different catalog — checked).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: no manifest family this slice (task_07); no `view.provider` (task_08); no hook-catalog change (checked `internal/hooks/events_catalog.go` — untouched per spec); Surface table untouched (not a ResourceKind).
- Agent manageability: CLI `cmd-palette list|inspect|invoke|clients` + `approvals show|cancel` with `-o json|jsonl|toon`; HTTP/UDS parity for all listed routes; native tools `compozy__cmd_palette_list|invoke`; deterministic error table per `_dx.md` § Errors.
- Config lifecycle: none — checked surfaces: no new `config.toml` keys this slice (`[cmd_palette]` lands in task_05; `[window_manager.global_shortcuts]` in task_09); no existing key touched.

## Deliverables

- `internal/cmdpalette` + `corecmds` compiled, wired at the composition root, with the full core inventory absorbed and boot-validated
- `ApprovalCoordinator` + `tool_approval_pending` migration `00069` + approvals public surface (routes/CLI/native) live on both transports
- Windowmanager client registry extension + `client_command` channel + attachment-token authorization (both paths)
- `/api/cmd-palette/{commands,clients,invoke,stream}` + CLI verbs + native tools + SSE + events, matching `_dx.md` byte-for-byte
- Contract co-ship: OpenAPI/TS regen, native-tool catalog regen, transport-parity rows, skills/glossary/docs updates
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [ ] UT-001, UT-002, UT-003, UT-004, UT-005 — catalog assembly, absorption count, structural revision vs `context_revision`, duplicate-id rejection, degraded sources
- [ ] UT-006, UT-007, UT-008, UT-009, UT-010 — invoke availability/args/unknown-id/single-flight-per-policy-class/approval-pending
- [ ] UT-011 — native-tool list ↔ catalog parity
- [ ] UT-012 — settings-destination descriptors cover the route inventory
- [ ] IT-001 — commands GET with seeded providers + jsonl completeness at scale
- [ ] IT-007 — invoke with inline args → tool result envelope
- [ ] IT-009 — invoke policy matrix (404/422/412/202 + denial without side effect)
- [ ] IT-010 — single-flight + failure + approval-lifetime races (409 while pending; deny/timeout release; late approval resolves exactly once)
- [ ] IT-021 — native-tool catalog contract + approval gate honored
- [ ] IT-022 — transport parity for every new route
- [ ] IT-031 — two clients: divergent availability, targeting rules, forged-token rejection, control-plane targeting without token, identical structural revision
- [ ] IT-032 — workspace resolution matrix (ID/name/path/cwd) + two-workspace isolation across CLI/HTTP/UDS/native
- [ ] E2E-022, E2E-023, E2E-024, E2E-026 — agent journeys via Go harness/UDS using `_dx.md` transcripts verbatim (list, invoke + errors, approval approve/deny, native-tool parity)

## Success Criteria

- Every assigned test case implemented and passing
- `make gate` green including migrate-suite extension for `00069`, transport parity, and `make codegen-check` (no drift in OpenAPI/TS/native catalog/CLI docs)
- Absorption assertion holds: every `internal/windowmanager` action id appears in the catalog exactly once (UT-002's count pins it)
- A restart during an approved-but-undispatched approval never double-executes (fence claimed once; recovery to `uncertain` surfaced, never retried)
- Zero new god files; every new file within the 500-line cap
