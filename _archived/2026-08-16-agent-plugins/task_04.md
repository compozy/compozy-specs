---
status: completed
title: Operator/agent surfaces: CLI output, contracts, error codes, parity
type: backend
complexity: high
---

# Task 4: Operator/agent surfaces: CLI output, contracts, error codes, parity

## Overview

Makes the feature visible and operable: install/validate/update/remove/status/inventory/list render the frozen `_dx.md` outputs (Ingested/Skipped blocks, Format lines, skip-aware Summary, quarantine copy, dual-manifest note), the extension and inventory payloads carry `format` + ordered `diagnostics` uniformly across CLI/HTTP/UDS/native tools, and the five deterministic `extension_agent_plugin_*` error codes land in the error-kind enum. This is where ADR-003 (first-class diagnostics) becomes user-visible and agent-branchable, beating both reference clients' log-only behavior.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST render the `_dx.md` transcripts byte-for-shape: install `Format:`/`Ingested`/`Skipped` blocks between `✓` and `next:` (portable only — native install output byte-identical to today), validate `Status/Format/Package/Would ingest` + severity rows with warn→exit 0 / fatal→exit 1, remove `removed:`/quarantine lines, the dual-manifest `note:` line, and `Summary: running (N components skipped)`.
2. MUST implement all four output modes (`human`, `json`, `jsonl`, `toon`) for every touched verb — a missing formatter is a runtime error in this CLI.
3. MUST widen `ExtensionPayload` (always-present `format`, ordered `diagnostics`) and `ExtensionInventoryPayload` (`format`, `diagnostics`) with ONE payload builder — list rows, status, info, and native tools all carry the full payload (uniform list contract, round-2 B-009); ingest set merges before live runtime entries per the total order.
4. MUST map the new failure kinds onto the existing error-kind enum with the exact codes and envelopes from `_dx.md` Errors: `extension_agent_plugin_client_layout`, `extension_agent_plugin_not_manifest`, `extension_agent_plugin_schema_unsupported` (422 generic), `extension_agent_plugin_manifest_invalid` (422 validation envelope with `issues[]`), `extension_agent_plugin_component_skipped` (diagnostic, not an error).
5. MUST extend `compozy extension validate` with the portable branch (stateless, daemon-less, same ladder) and `compozy__extensions_validate` with the structured result (`status`/`format`/`would_ingest`/`issues`); native tool descriptors/output schemas/digests refresh for every touched tool.
6. MUST hold CLI/HTTP/UDS parity (shared `BaseHandlers`) with the transport-parity suite extended for payload, list rows, and inventory; unknown-name and consent errors keep identical shapes.
7. MUST run the contract co-ship set: `make codegen`, `make codegen-check`, `make bun-typecheck` (generated TS consumers compile).
</requirements>

## Subtasks

- [x] 4.1 Payload/inventory contract widening + one payload builder + diagnostic merge/order
- [x] 4.2 Error-kind enum additions + envelope mapping + HTTP status wiring
- [x] 4.3 Install/update human blocks + note line + quarantine remove copy + Summary skip-count
- [x] 4.4 Validate portable branch (CLI + native tool), exit-code contract, four output modes
- [x] 4.5 Status/inventory/list renders with Format lines + Diagnostics/Skipped sections
- [x] 4.6 Native tool descriptor/output-schema/digest refresh (`_install/_update/_validate/_info/_list/_inventory`)
- [x] 4.7 Parity suite extension (payload + list rows + inventory + error shapes)
- [x] 4.8 CLI journey E2Es against the `_dx.md` transcripts
- [x] 4.9 Contract co-ship gates (`make codegen`, `codegen-check`, `bun-typecheck`)

## Implementation Details

Follow `_spec.md` Part II → API Endpoints + Agent Manageability Plan; the frozen `_dx.md` is the pixel-level contract. Existing anchors: `internal/api/contract/extensions.go:144-240` (payloads), `internal/api/core/extensions_errors.go:19-136` (error-kind enum + envelope selection), `internal/extension/describe.go:60-100` (payload builder + `extensionType`/`extensionState`/summary computation), `internal/cli/extension_output.go` (success bundles, sections, list columns), `internal/cli/extension_authoring.go:92-246` (validate verb + render), `internal/cli/format.go:121-189` (four output modes — jsonl/toon REQUIRED), `internal/tools/builtin/extensions.go` + `internal/daemon/native_extension_tools.go` (tool descriptors), `internal/api/httpapi/transport_parity_integration_test.go` (owning parity suite), `internal/api/spec/registry_extensions.go` (OpenAPI registry).

Anti-patterns to beat (cite in review): Codex renders no manifest/mcp diagnostics anywhere (`.resources/codex/codex-rs/cli/src/plugin_cmd.rs:241-330` — no format column, silent disappearance per its tracing-only table); Hermes CLI shows no portable indicator while its GUI does (`.resources/hermes/hermes_cli/plugins_cmd.py:1811-1850` vs `.resources/hermes/tui_gateway/methods_tools.py:2404-2406`).

Skills to activate: `eng-code-guidelines`, `golang-master`, `eng-contract-codegen-coship`, `eng-test-conventions`, `testing-boss`, `eng-consolidate-test-suites`.

### Relevant Files

- `internal/cli/extension.go` — verb tree (no new verbs; flags unchanged)
- `internal/cli/extension_inventory_output.go` — inventory render gains Format/Skipped
- `internal/api/core/extensions.go` + `extensions_operation_errors.go` — handler + envelope construction
- `internal/toolmeta/native_entries.go` — native tool metadata refresh

### Dependent Files

- `web/src/generated/compozy-openapi.d.ts` — regenerates; task_05 consumes the new fields
- `packages/site/content/docs/cli/extension/*.mdx` — generated CLI docs refresh (task_06 runs the regeneration)
- `docs/qa/scenarios/` — behavior this task ships gets flagged in task_06

### Related ADRs

- [ADR-003: First-class diagnostics](adrs/adr-003.md) — the visibility contract this task implements
- [ADR-004: Verbatim names](adrs/adr-004.md) — identity rendering across surfaces

### Web/Docs Impact

- `web/`: `web/src/generated/compozy-openapi.d.ts` (regenerated fields consumed by task_05's components/hooks in `web/src/systems/extensions/`); no component edits in this task.
- `packages/site`: generated CLI reference pages under `content/docs/cli/extension/` refresh when task_06 runs `make cli-docs`-equivalent regeneration; prose pages land in task_06.
- QA impact: new behavior (install report blocks, validate verdicts, error codes, quarantine copy) — add content-addressed `untested` scenario files (task_06 sweep; walked by task_08).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: native tool descriptors + output schemas + digests refresh for the six touched `compozy__extensions_*` tools; no new tool IDs (checked: `builtin_extension_ids.go` unchanged); no hook/provide/registry change.
- Agent manageability: THE agent-surface task — structured outputs on all four modes, branchable deterministic codes, HTTP/UDS parity, status/inventory discovery; consent-in-non-interactive fails deterministically (existing rule regression-covered).
- Config lifecycle: zero config keys (checked: `ExtensionsConfig`); OpenAPI/TS co-ship in this task's gates.

## Deliverables

- Every `_dx.md` CLI transcript reproducible against a live daemon; native-extension outputs byte-identical to today
- Widened contracts regenerated (OpenAPI + TS) with parity proven
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**
- `make gate` green; co-ship gates clean

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-042, UT-043, UT-044, UT-045 — install blocks + native byte-parity, validate render/exit, remove/quarantine copy, dual-manifest note
- [x] UT-047 — four output modes for install/validate payloads
- [x] UT-054 — Summary skip-count computation
- [x] IT-003 — HTTP install failures: client-layout + not-manifest codes, 422 envelopes, registry untouched
- [x] IT-012 — transport parity: payload + list rows + inventory + not-found shapes (owning suite: `internal/api/httpapi`)
- [x] IT-013 — native tools: install/validate structured results + branchable error codes + non-interactive consent failure
- [x] E2E-002 — golden-path CLI journey (transcript-matched)
- [x] E2E-003 — failure-path CLI journey (client layout, fatal name, dual note, validate layout error; exit codes)

## Success Criteria

- Every assigned test case implemented and passing
- An agent can distinguish every failure class from `code` alone — no prose parsing — across CLI/HTTP/UDS/native
- A degraded install is diagnosable from `status` and `inventory` on every surface without reading logs
- `make codegen-check` and `make bun-typecheck` clean
