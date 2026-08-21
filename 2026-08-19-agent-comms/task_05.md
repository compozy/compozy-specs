---
status: pending
title: "Surfaces: native tools, CLI, HTTP/UDS, roster, hooks, publish"
type: backend
complexity: high
---

# Task 5: Surfaces: native tools, CLI, HTTP/UDS, roster, hooks, publish

## Overview

Complete the full house chain in one pass — contract types → core handlers → HTTP + UDS routes → spec registry → codegen → CLI family → native tools — for calls, messages, publish, and subtree stop; plus the subagents surface (AGENT.md `description`, roster injection, depth-wall toolset strip), the `call` hooks family + Host API reads, the one-way Network publish bridge, task result-contract authoring (`expect` on every task surface + run snapshots + completion admission), and the hard deletion of the spawn surface. Closes with the acpmock co-ship and all 21 runtime/HTTP E2E journeys from `_dx.md`.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Every route, verb, tool, payload, status code, wake text, and error row MUST match `_dx.md` exactly — it is the frozen public surface; deviations are spec violations, not judgment calls. Deterministic errors: same condition, same code, same shape, every surface.
2. Native tools: add the calls family (`compozy__agent_call`, `call_return`, `call_await`, `call_cancel`, `call_result`, `call_publish`, `agent_message`) as `ToolsetIDCalls` with descriptors, computed schema digests, risk classes per the Impact Audit, bindings, availability closures; extend `compozy__session_stop` with `subtree`; extend `agent_create`/`agent_list` with `description`. DELETE `compozy__session_spawn` completely (descriptor, schemas, toolset entry, binding, availability, handler, input DTO) in the same pass — UT-160 asserts presence AND absence together.
3. CLI: `compozy call <agent|session> …` (with `--expect`, `--strict`, `--result-budget/--result-overflow`, `--idle-ttl`, `--deadline`, `--idempotency-key`, `--runtime`, four narrowing flags), `call list/show/await/cancel/result/publish`, `compozy message send/list`, `compozy session stop --subtree`, `compozy agent list` — all four output formats, `_dx.md` exit codes (0/2/3), structured stderr errors. DELETE `internal/cli/spawn.go` + its generated doc page.
4. HTTP/UDS: all `/calls` routes (create+batch 200 per-item, get, result, cancel, await, publish), `/messages` (POST 202/GET), session-stop `subtree` — dual registration + UDS parity entries, contract types with previews only (refs never raw), spec-registry operations, `make codegen` regenerating OpenAPI + TS types.
5. Roster injection renders at the hosted-MCP projection seam at serve time (name + description, workspace shadows global, 32-entry/120-char caps + overflow line); the call tool is ABSENT from a child's toolset at the depth wall and the child's context states its literal remaining depth; `description` parses/renders/digests through AGENT.md machinery with the 500-char authoring bound.
6. Task result contracts: `expect` (+ optional budget fields) accepted on `compozy task create/update`, HTTP/UDS task bodies, and native task tools; digest + effective budget echoed on every read projection; completion admission validates against the run's start-time snapshot with one worker resubmission round through the existing completion authority (sanitized errors).
7. Hooks family `call` — exactly the seven events sharing the canonical naming table — dispatch wired at service transitions with sanitized payloads, fail-open; Host API gains `calls/list|get|result`, `messages/list` with `calls:read` consent contracts; the observability coverage-matrix test extends to every call/message lifecycle event.
8. Publish: `Service.Publish` (completed-with-result only) through the daemon-implemented `PublishBridge` (`internal/calls` never imports `internal/network`); idempotent per conversation via `call_publications` (replay returns recorded id, `published:false`); bounded evidence under the Network ceiling.
9. `deadline_seconds` is the wire field on every create surface (tool/HTTP/UDS/batch items); CLI `--deadline` converts; validation → `call_deadline_invalid`; budget and deadline participate in idempotency identity.
10. acpmock learns `call_return`/`agent_message` behaviors in this change (runtime-contract co-ship, L-007); E2E packages stay within the registered lanes in `internal/e2elane/lanes.go` (daemon/httpapi/udsapi/testutil-e2e/cli) — register any net-new package there.
11. Remaining delete targets land here: `SpawnOpts.IdempotencyKey` (inert input), HTTP/UDS spawn route + `AgentSpawnRequest`/`AgentSpawnPayload` + spec op + parity entries + generated TS, and the stale `compozy exec` line in `internal/CLAUDE.md`. No fallback, no compat shim.
</requirements>

## Subtasks

- [ ] 5.1 Contract types (`internal/api/contract/calls.go`) + core handlers (`calls.go`/`calls_interfaces.go` on `BaseHandlers`) + HTTP/UDS route registration + spec-registry operations + UDS parity entries + `make codegen`
- [ ] 5.2 CLI family: client extension, call/message/session-stop/agent-list verbs, four-format bundles, exit-code mapping; delete `spawn.go`
- [ ] 5.3 Native tools family: ids, descriptors, toolsets, bindings, availability, handlers with structured (redacting) results; session-stop `subtree`; delete `compozy__session_spawn` end to end
- [ ] 5.4 AGENT.md `description` (parse/render/digest/validate 500c) + payloads + roster renderer + serve-time injection + depth-wall toolset strip + literal-remaining-depth context
- [ ] 5.5 Task `expect` authoring on every surface + read projections + run-snapshot completion admission with one resubmission round
- [ ] 5.6 Hooks family `call` (events/payloads/catalog/dispatch) + Host API read methods with consent contracts + coverage-matrix extension
- [ ] 5.7 `Service.Publish` + `PublishBridge` daemon implementation + `call_publications` idempotency + bounded evidence
- [ ] 5.8 `deadline_seconds` wire parity + idempotency-identity inclusion across every create surface
- [ ] 5.9 acpmock `call_return`/`agent_message` behaviors + the 14 runtime CLI journeys (E2E-001..014) + 7 HTTP journeys (E2E-023..029) with `_dx.md` transcripts verbatim
- [ ] 5.10 Sweep the remaining delete targets; implement every assigned case; close on `make gate`

## Implementation Details

Follow the house chain exactly as the loop surfaces did (the Relevant Files anchor every seam). Handlers share query structs across HTTP/UDS; projections expose previews with fetch routes, never raw refs. Batch create returns status 200 with a per-item array. Native message-bearing tool results use the `structuredNetworkResult`-style redaction variant. E2E suites live inside already-registered lane packages (`internal/daemon`, `internal/api/httpapi`, `internal/api/udsapi`, `internal/testutil/e2e`, `internal/cli`) — extend those, and register in `internal/e2elane/lanes.go` only if a genuinely new package is created.

### Relevant Files

- `internal/tools/builtin_ids.go:101` + `builtin_loop_ids.go:1` — tool-ID declaration pattern; new family gets `builtin_call_ids.go`
- `internal/tools/builtin/descriptors.go:29,76,81` + `builtin/sessions_orchestration.go:9,44,106-134` + `builtin/toolsets.go:6,46,69` — descriptor registration, the family-file shape to copy (and the spawn schemas to delete), toolset membership
- `internal/tools/descriptor.go:35,113` + `internal/tools/schema_digest.go:44` — descriptor contract + computed digests
- `internal/daemon/native_tools.go:71,93` + `native_tool_bindings.go:5` + `native_tool_session_orchestration_bindings.go:5` + `native_tool_availability.go:23,62,101` + `native_tool_results.go:13,28` — provider join, bijective boot validation, bindings, availability, structured/redacting results
- `internal/daemon/native_tool_session_orchestration.go:31,74` + `native_session_prompt_stream.go:11` — handler shape; the response-discard lesson
- `internal/cli/root.go:63,166,172,216` + `network.go:93` + `network_usage_cmd.go:8` + `format.go:63,71,121,154` + `client.go:26` + `client_network_coordination.go:34,137` + `client_api_errors.go:178` + `command_exit_error.go:22` — CLI DI, family/verb/format/exit-code patterns
- `internal/api/contract/loop_requests.go:9-65` + `core/loop_interfaces.go:186,223` + `core/base_handlers.go:111,135` + `httpapi/loops_routes.go:35` + `udsapi/loops_routes.go:38` + `udsapi/loop_options.go:13` + `udsapi/handlers_test.go:296` + `spec/spec.go:237` + `spec/operation_registry.go:5` — the full chain shape + parity test
- `magefiles/codegen.go:13,48` — codegen order and drift gate
- `internal/config/agent.go:17,36,213` + `agent_create.go:24,71,145` + `agent_discovery.go:14` + `internal/api/contract/agent_observe_payloads.go:28` — AGENT.md parsing/rendering/digest/shadowing + payloads gaining `description`
- `internal/tools/builtin/workspace.go:15,71` + `internal/daemon/native_entity_catalog_tools.go:1` — `agent_list`/`agent_create` surfaces
- `internal/hooks/events.go:6,25,32` + `events_network.go:3` + `events_catalog.go:7` + `payloads_spawn.go:6` — hook family template
- `internal/codegen/hostapi/catalog.json:1` + `internal/extension/contract/permissions.go:9,30` — Host API methods + consent
- `internal/testutil/acpmock/` (+ `driver/`, `fixture.go`) + `internal/testutil/e2e/runtime_harness.go` + `internal/e2elane/lanes.go` — acpmock co-ship + E2E lane registration

### Dependent Files

- `internal/calls/**` — gains `Publish` + hook-dispatch calls at transitions (service extension, still behind the settlement fences)
- `internal/task/**` — completion admission against run snapshots; task create/update surfaces
- `internal/session/**` — toolset computation honoring the depth wall
- `internal/CLAUDE.md` — stale `compozy exec` line (delete target)
- `web/src/generated/compozy-openapi.d.ts` — regenerated by `make codegen` (consumed by task_06)

### Competitor References

- `.resources/codex/codex-rs/core/src/agent/role.rs:270-348` + `tools/spec_plan.rs:673-683` — roster rendered into the param description (ADR-007's mechanism)
- `.resources/pi/packages/coding-agent/examples/extensions/subagent/index.ts:287-294,473-518` — error-driven roster (anti-pattern) and exactly-one-mode enforcement
- `.resources/hermes/tools/delegate_tool.py:4597-4730` — batch schema + control-plane multiplexing; the 4,793-line file is the layout anti-pattern

### Related ADRs

- [ADR-002: One agent-facing call verb; the spawn surface is deleted](adrs/adr-002.md) — the hard cut executed here
- [ADR-005: Type-level disjunction from Compozy Network with a one-directional bridge](adrs/adr-005.md) — publish bridge rules
- [ADR-007: Explicit registry-name invocation, injected roster, `description` field, batch fan-out](adrs/adr-007.md) — the subagents surface
- [ADR-008: Recursive delegation, default max depth 3, budget-based containment](adrs/adr-008.md) — strip-at-wall + literal depth
- [ADR-009: Async-by-default calls; result-carrying wake; explicit bounded await](adrs/adr-009.md) — await surfaces and clamp

### Web/Docs Impact

- `web/`: `web/src/generated/compozy-openapi.d.ts` regenerates with the call/message/publish/task-expect operations (task_06 consumes it); no hand-written web files change in this task.
- `packages/site`: generated CLI reference pages change with the new/deleted verbs (`make cli-docs` re-run belongs to task_07 alongside the docs area); reason: docs co-ship at the docs task against final surfaces.
- QA impact: new user-visible behavior across CLI/HTTP/tools/config — add content-addressed `untested` scenarios in `docs/qa/scenarios/` for: the call golden path (create→await→result), follow-up/revive, batch fan-out, cancel, publish, mailbox send/list, session stop `--subtree`, task `--expect`, `[calls]` config get/set effects, and spawn-verb removal (negative). Flag only; task_09 walks them.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: `call` hook family (7 events, one naming table, sanitized payloads, fail-open) + Host API `calls/list|get|result`, `messages/list` under `calls:read` consent; AGENT.md `description` flows to extension-provided agent resources unchanged; no bridge SDK / MCP sidecar contract changes (checked: `internal/bridgesdk`, sidecar lifecycle — roster renders at the hosted-MCP projection seam without contract change).
- Agent manageability: the complete `_dx.md` surface — `compozy call/message/agent list/session stop --subtree` with 4 formats + deterministic exit codes, HTTP/UDS parity routes, the 7 native tools + `session_stop.subtree`, `compozy config get/set calls.*`, status discovery via `call list/show`; every failure a typed code from the `_dx.md` table on every surface.
- Config lifecycle: consumes `[calls]` end to end (IT-062 proved enforcement in task_04); no new keys here; generated CLI/config docs refresh in task_07. Checked: no key additions/removals beyond task_01's section.

## Deliverables

- Full `_dx.md` surface live on tools, CLI, HTTP, and UDS with parity + deterministic errors
- Spawn surface completely deleted (tool, CLI, routes, DTOs, spec op, TS types, inert `SpawnOpts.IdempotencyKey`)
- Roster/description/depth-wall subagents surface; task `expect` authoring + snapshot admission; hooks + Host API; publish bridge
- acpmock co-ship + all 21 runtime/HTTP E2E journeys green
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-078 — native handler unknown-agent structured error decode (code + roster)
- [ ] UT-088 — `agent_list` payload carries description + scope, matching roster injection
- [ ] UT-089, UT-090, UT-091, UT-092, UT-093, UT-094, UT-095 — CLI call/list/await/result/message shapes, exit codes, malformed `--expect`
- [ ] UT-096, UT-097, UT-098, UT-099, UT-100, UT-101, UT-102 — HTTP create/errors/batch-200/pagination/await/result (status AND body)
- [ ] UT-112, UT-113, UT-114, UT-115, UT-116 — description parse/render/digest/optional/bound/inert-markup
- [ ] UT-117, UT-118, UT-119 — roster renderer shadow-aware/bounded/zero-state
- [ ] UT-120, UT-121 — depth-wall toolset strip + literal remaining-depth context
- [ ] UT-123 — shadowed rows distinguishable in list
- [ ] UT-124 — task completion admission against the run's start-time snapshot with one resubmission
- [ ] UT-141, UT-142, UT-143, UT-144 — publish evidence/participation/eligibility-table/bounded-envelope
- [ ] UT-145, UT-146 — hook family catalog + dispatch; Host API consent registry
- [ ] UT-147 — session-stop handler `subtree` → drain report
- [ ] UT-150, UT-151, UT-152 — publish HTTP/CLI/tool surfaces
- [ ] UT-154, UT-155, UT-156, UT-157, UT-158, UT-159 — messages HTTP errors, `deadline_seconds` parity, GET call, idempotent cancel route, POST/GET messages
- [ ] UT-160 — native family registration + spawn deletion verified in one test
- [ ] UT-162 — CLI `session stop --subtree` formats; `--strict`/`--result-budget`/`--result-overflow` parsing
- [ ] UT-163 — publish replay idempotency per conversation
- [ ] IT-045, IT-046, IT-047, IT-049 — description round-trip + shadowing, serve-time roster re-render, depth chain with strip, concurrent update+list snapshot
- [ ] IT-052, IT-053 — task `--expect` journey with mid-run snapshot rules; task budget behavior
- [ ] IT-054 — four-way validation parity (call/loop/task/ask)
- [ ] IT-056 — spawn hooks on the call path + `call.created` after commit
- [ ] IT-057 — cross-workspace denial + list/read/SSE isolation across surfaces
- [ ] IT-060 — publish to Live channel timeline; no reverse path
- [ ] IT-063, IT-064 — UDS route parity; OpenAPI operations + generated TS compile
- [ ] IT-066 — Host API consent gate through a test extension
- [ ] IT-067 — observability coverage matrix over every call/message lifecycle event
- [ ] E2E-001..E2E-014 — the 14 runtime/CLI journeys (golden path, follow-up, batch, cancel, await-resume, idempotent retry, repair, silence, extraction, mailbox, TTL, capability subtraction, drain, recursion)
- [ ] E2E-023, E2E-024, E2E-025, E2E-026, E2E-027, E2E-028, E2E-029 — HTTP journeys + agent-definitions API + task-contract journey + publish (CLI/HTTP) + deadline

## Success Criteria

- Every assigned test case implemented and passing; `make test-e2e-runtime` green with the acpmock co-ship
- `make codegen-check` green; UDS parity test lists every new route; spawn surface grep-clean across tools/CLI/API/TS
- Every `_dx.md` transcript reproduces verbatim (states, texts, exit codes, error rows)
- `make gate` green at close
