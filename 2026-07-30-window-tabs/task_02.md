---
status: completed
title: Public surface: contract, codegen, hooks, native tools, CLI, parity
type: backend
complexity: high
---

# Task 2: Public surface: contract, codegen, hooks, native tools, CLI, parity

## Overview

Closes the agent-facing loop for the tab domain in one pass: v3 wire types (History off the snapshot, new fields on window/desktop/client), strict decode for the five new commands and three extended payloads, two new deterministic error codes, the five hook events with ChangeSet outcome mapping and introspection, five new native tools plus schema updates to three existing ones, the new and extended CLI verbs with regenerated docs, and byte-parity across HTTP/UDS/CLI. Zero new routes — everything rides the existing commands/preview/stream/layout surface.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST update `internal/api/contract` wire types per TechSpec §Data Models deltas: `WindowManagerWindow` +`nav_stack`/`pinned`; `WindowManagerDesktop` +`floating_stacks`; `WindowManagerSnapshot` −`history` +`closed_entry_count`; `WindowManagerClientView` +`stack_active`; LayoutDocument v3 fields — no dual shapes, no legacy field retained.
2. MUST register the five new command IDs with strict payload decode (`DisallowUnknownFields`), including mode-dependent route validation (`pop` rejects route) and scope×minimize rejection.
3. MUST add error codes `window_manager_not_stacked` and `window_manager_window_pinned` with HTTP 422 mapping and exact-string coverage.
4. MUST implement the five hook events (`window_manager.window.opened/closed`, `window_manager.stack.grouped/ungrouped/activated`) with ChangeSet `StackGrouped`/`StackUngrouped` outcome fields, dispatch mapping (opened: open+reopen; closed: all close scopes with all ids; activated: set_active + coalescer flush), introspection entries, and NO dispatch for presentation focus flips.
5. MUST add native tools `compozy__window_{group,reorder,activate,pin,reopen}` and extend `compozy__window_{open,navigate,close}` schemas (stack-target/mode/scope) with descriptors, toolmeta entries, digests, and availability gates.
6. MUST add CLI verbs `compozy window group|activate|pin|unpin|reopen` and extend `open --stack-target`, `navigate --mode`, `close --scope`, `group --insert-index`, plus extended `window list` output (stack id, member order, active, pinned, nav depth) — all output modes; regenerate CLI docs via `make codegen`.
7. MUST keep HTTP/UDS byte-parity (existing harness) and run the full codegen loop (`make codegen` + `make codegen-check`) with the OpenAPI/TS types adopted in the same change.
8. MUST use the public noun **activate** in CLI/tool/docs while keeping `window.stack.set_active` as the internal command ID (documented equivalence).
</requirements>

## Subtasks

- [x] 2.1 Wire types + conversions for every v3 delta; delete `history` from the wire snapshot
- [x] 2.2 Strict decode + validation for new/extended command payloads
- [x] 2.3 Error codes with core mapping and exact strings
- [x] 2.4 Hook events, ChangeSet outcomes, dispatch mapping, introspection
- [x] 2.5 Native tools (new + extended schemas) with toolmeta/digests/gates
- [x] 2.6 CLI verbs + extended flags + list output; regenerate docs
- [x] 2.7 Transport parity + codegen loop green
- [x] 2.8 Implement all 25 assigned test cases

## Implementation Details

Follow TechSpec §API Endpoints, §Agent Manageability Plan, §Extensibility Integration Plan, and ADR-009 (activated hook). Exact current shapes in the transcript exploration report: contract types (`windowmanager_{types,commands,command_payloads,conversions,layout,stream}.go`), error codes (`windowmanager_types.go:37-51`), core mapping (`api/core/window_manager_errors.go:43-72`), hooks (`events/payloads/dispatch/introspection_window_manager.go`, `daemon/window_manager_hooks.go:69-82`, `service_execute.go:59-77`), tools (`daemon/native_tool_window_manager_*`, `tools/builtin_ids.go:365-415`, `toolmeta/native_entries.go`), CLI (`cli/window_manager_*.go`, docs via `internal/cli/doc.go` + `magefiles/codegen_cli_docs.go`).

### Relevant Files

- `internal/api/contract/windowmanager_types.go`, `windowmanager_commands.go`, `windowmanager_command_payloads.go`, `windowmanager_conversions.go`, `windowmanager_layout.go`, `windowmanager_stream.go` — wire deltas
- `internal/api/core/window_manager_handlers.go`, `window_manager_errors.go`, `window_manager_ws.go` — decode/mapping (no route changes)
- `internal/hooks/events_window_manager.go`, `payloads_window_manager.go`, `dispatch_window_manager.go`, `introspection_window_manager.go` — 5 events
- `internal/windowmanager/commands.go` (`ChangeSet`), `service_execute.go` (`isObservableCommand`) — outcome plumbing
- `internal/daemon/window_manager_hooks.go`, `native_tool_window_manager_bindings.go`, `native_tool_window_manager_commands.go`, `native_tool_window_manager_input.go` — dispatch + tools
- `internal/tools/builtin_ids.go`, `internal/toolmeta/native_entries.go` — IDs + display metadata
- `internal/cli/window_manager_window.go`, `window_manager_navigate.go`, `window_manager_output.go`, `root.go` — verbs/flags/output
- `openapi/compozy.json`, `web/src/generated/compozy-openapi.d.ts` — regenerated

### Dependent Files

- `internal/api/contract/contract_test.go`, `internal/api/core/window_manager{,_ws}_test.go`, `internal/api/udsapi/transport_parity_integration_test.go`, `internal/hooks/window_manager_dispatch_test.go`, `internal/daemon/native_tools_test.go`, `internal/cli/window_manager_test.go` — canonical suites extended
- `packages/site/content/docs/cli/window/*.mdx` — regenerated pages (new verbs + flags)
- `web/src/systems/os/lib/window-manager-types.ts` — command-ID union mirror (adopted fully in task_03)

### Related ADRs

- [ADR-009](adrs/adr-009.md) — activated hook + naming
- [ADR-010](adrs/adr-010.md) — optional server-generated IDs on the wire
- [ADR-012](adrs/adr-012.md) — v2 layout documents rejected (no converter)
- [ADR-013](adrs/adr-013.md) — reopen command surface

### Web/Docs Impact

- `web/`: `web/src/generated/compozy-openapi.d.ts` regenerates (breaking shape: snapshot loses `history`); `web/src/systems/os/lib/window-manager-types.ts` command union gains 5 IDs. Full adoption + UI is task_03 (typecheck may stay red between 02 and 03 only inside task_03's branch scope — land 02+03 in the same workstream before gate-full).
- `packages/site`: `content/docs/cli/window/{group,activate,pin,unpin,reopen}.mdx` generated new; `{open,navigate,close,list}.mdx` regenerated with flags — via `make codegen`, verified by `make codegen-check`.
- QA impact: CLI/API behavior changes. Scenario resets owned by task_05: `ET-window-manager-public-parity`, `ET-window-manager-hooks-resources`, `MS-layout-profile-cli-roundtrip` — walked in task_06.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: +5 typed hook events with introspection; native toolset `compozy__window_manager` grows 26→31 with 3 schema updates; `window_layout` resource docs unchanged here (v3 validation landed in task_01). Checked/no impact: extension manifests, provide/permission surfaces, bundles, bridge SDKs, MCP sidecars, Compozy Network protocol docs.
- Agent manageability: every tab read/mutation reachable via CLI (`-o human|json|jsonl|toon`), HTTP, UDS, and native tools with the documented deterministic errors; `compozy layout watch` streams tab changes; parity asserted by the existing harness.
- Config lifecycle: no new keys here (task_01 owns them); settings GET/PATCH already exposes the section — extended-key exposure verified by existing settings tests.

## Deliverables

- v3 wire contract live across HTTP/UDS with regenerated OpenAPI/TS/CLI docs and zero codegen drift
- 5 hook events observable end-to-end with introspection listings
- 31-tool window-manager toolset with accurate schemas/digests
- CLI verbs + flags shipping with generated docs and parity evidence
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-110..UT-115 — wire shapes, stack_active, history absence, strict decode, error codes, revision cap
- [x] UT-120..UT-122 — hook dispatch mapping incl. activated, introspection
- [x] UT-150..UT-154 — CLI verbs/flags/list/errors
- [x] UT-160 — native tools presence/schemas/gates (no global count)
- [x] IT-003, IT-004 — HTTP end-to-end + UDS parity; concurrent CAS conflict over the API
- [x] IT-005, IT-006 — close/reopen over HTTP with live sessions; WS frames (event vs client-only activation)
- [x] IT-008..IT-012 — native-tool loop, codegen drift, CLI↔HTTP behavior parity, hook integration, layout-profile round-trip + v2 rejection
- [x] E2E-017 — CLI journey: list → group → activate → pin → reopen → export/apply with deterministic stale-revision error

## Success Criteria

- Every assigned test case implemented and passing
- `make codegen-check` green; transport-parity suite green; `make gate` green on affected lanes
- An agent driving only the CLI can inspect and rearrange tabs to the same end state as the HTTP path, byte-identical snapshots
- Generated CLI docs contain the new verbs/flags with the activate naming and error contracts
