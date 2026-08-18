---
status: completed
title: "Public surfaces co-ship: contract, routes, CLI, native tools"
type: backend
complexity: high
---

# Task 7: Public surfaces co-ship — contract, routes, CLI, native tools

## Overview

Closes the no-partial-surface loop for everything tasks 02–06 built: contract payload extensions
(`canceled` status, lifecycle fields, node controls/waits projections), the node-verb and
inventory routes on HTTP + UDS, run cancel/kill routes, the `compozy loop node …` + `compozy loop
nodes` + `loop cancel|kill` CLI verbs, eight new native tools (two Destructive) with schema-digest
refresh, hook payload field additions, the `stop` surface deletion everywhere, and one
`make codegen` pass propagating OpenAPI + TypeScript.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- R1: MUST extend contract types exactly per TechSpec API Endpoints: `LoopGenerationOutput`
  (+attempt/next_attempt_at/failure_class/disposition), `LoopRunResponse` (+node_controls,
  +waits), new `LoopNodeInventoryResponse`, `LoopRunStatus` + terminal-state + cause enums
  (+`canceled`/`operator_cancel`/`operator_kill`), `LoopRunEventKind` (+15) — then
  `make codegen` + `make codegen-check` green (eng-contract-codegen-coship).
- R2: MUST add routes on BOTH httpapi and udsapi via shared `BaseHandlers`: run
  `cancel`/`kill`, node `pause|resume|cancel|kill|requeue`, and the workspace
  `GET /loop-nodes` inventory (paginated, stable order); DELETE the `stop` route.
  Deterministic `ReasonError` answers name actual state + allowed transitions; concurrent verbs
  answer with winner provenance (US-026).
- R3: MUST add CLI verbs `compozy loop cancel|kill`, `compozy loop node
  pause|resume|cancel|kill|requeue`, `compozy loop nodes --state …` with table + `-o json|jsonl`
  golden output; DELETE `compozy loop stop` (command, client, docs regen).
- R4: MUST register native tools `compozy__loop_cancel`, `compozy__loop_kill` (Destructive),
  `compozy__loop_node_pause|resume|cancel|kill|requeue` (kill Destructive, rest Mutating),
  `compozy__loop_nodes` (Read); DELETE `compozy__loop_stop`; refresh descriptors + schema
  digests; update `compozy__loop_status` output schema for the extended generation object; NO
  new capability gates (grill decision).
- R5: MUST extend `LoopNodeTerminalPayload` hook fields ({failure_class, disposition, attempt,
  target}) and expose effective family defaults with sources in `compozy loop inspect`
  (US-027 AC-3).
- R6: Cross-surface parity is test-enforced: the same verb via HTTP, UDS, CLI, and native tool
  returns the identical structured shape; workspace isolation asserted on every new
  read path (Compozy Impact Audit).
</requirements>

## Subtasks

- [x] 7.1 Contract types + enums + `make codegen` (OpenAPI + TS) + codegen-check
- [x] 7.2 BaseHandlers verbs + inventory handler + stop deletion; httpapi + udsapi registration
- [x] 7.3 CLI verb group + renderers + golden JSON; client methods; stop deletion
- [x] 7.4 Native tool descriptors/bindings/digests + loop_status schema update + stop deletion
- [x] 7.5 Hook payload fields + `loop inspect` effective-defaults sources
- [x] 7.6 Parity + isolation + inventory-scale integration suites; native-tool E2E journey

## Implementation Details

Follow TechSpec "API Endpoints", "Agent Manageability Plan", "Impact Analysis". New files: CLI
`loop_node.go`, `loop_nodes.go`; contract `loop_nodes.go`; handlers in `internal/api/core`
(new `loops_nodes.go` if `loops.go` nears cap).

### Relevant Files

- `internal/api/contract/loop_runs.go` + `loop_enums.go` — payloads/enums to extend
- `internal/api/core/loops.go:332-405` + `loop_interfaces.go` — BaseHandlers + LoopService seam
- `internal/api/httpapi/loops_routes.go:35-63` + `internal/api/udsapi/loops_routes.go:38-70` — route registration (stop deletion here)
- `internal/cli/loop.go:41-65` + `loop_runs.go:147` — verb tree + run-action factory (stop deletion)
- `internal/tools/builtin/loops.go:14-237` + `loops_output_schemas.go` + `internal/tools/builtin_ids.go:273-304` — descriptors/digests/IDs
- `internal/daemon/native_loop_tools.go:23-92` — tool bindings
- `internal/hooks/payloads_task_loop.go:152-159` — node-terminal payload
- `magefiles/codegen.go:13-77` — codegen pipeline

### Dependent Files

- `web/src/generated/compozy-openapi.d.ts` — regenerated (task_08 consumes)
- `packages/site/content/docs/cli/loop/` — GENERATED CLI docs (regen here; prose in task_11)
- `skills/compozy/references/loops.md` — verb table (task_11)

### Related ADRs

- [ADR-016](adrs/adr-016.md) — verb semantics · grill decisions (gating, stop hard cut) in TechSpec Key Decisions

## Web/Docs Impact

- `web/`: regenerated TS types; UI consumption is task_08.
- `packages/site`: generated CLI docs regenerate in this task's codegen run; authored docs are
  task_11. QA impact below.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: hook payload additions (additive); native tool digest refresh; no new
  capability gates; extension protocol docs for `event_key` finalized in task_11.
- Agent manageability: THE deliverable — full CLI/HTTP/UDS/native parity with deterministic
  errors and paginated inventories.
- Config lifecycle: `loop inspect` source-labeled effective defaults; no new keys.

## QA impact

Flag new `untested` scenarios: agent-operates-lifecycle-via-native-tools (US-026); reset any
scenario invoking `loop stop` (CLI/API) to `untested` — replaced by cancel/kill walks.

## Skills

`eng-code-guidelines`, `golang-pro`, `eng-contract-codegen-coship`, `eng-test-conventions`,
`testing-boss`, `eng-consolidate-test-suites`, `eng-data-boundaries` (read-path parity).

## References

- `.compozy/tasks/loops-paper-adoption/_techspec.md` §API Endpoints — the B-008-style DTO-parity precedent this task mirrors
- Surfaces exploration report §6–§8 (route/CLI/native inventories with file:line)

## Deliverables

- Every new state/verb operable with parity across four surfaces; `stop` gone everywhere
- `make codegen-check` green; digests updated
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] UT-153..UT-155 — deterministic verb answers, winner provenance, pagination
- [x] UT-183..UT-186 — payload projections, CLI golden JSON, descriptors/digests, status schema
- [x] UT-188 — deleted `stop` surface answers as absent at runtime
- [x] IT-026 — inventories per run/loop/workspace + aggregation + empty states
- [x] IT-029 — four-surface parity + status/body assertions
- [x] IT-030 — workspace isolation on node/wait/claim/inventory paths
- [x] E2E-014 — managing-agent journey via native tools only

## Success Criteria

- Every assigned test case implemented and passing
- One truth across surfaces: identical structured shapes CLI/HTTP/UDS/native for every verb and list
- `stop` returns unknown-route/unknown-command/not-found on all four surfaces
