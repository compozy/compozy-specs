---
status: completed
title: Desktop-state public surface (contract, HTTP/UDS/WS, CLI, wiring)
type: backend
complexity: critical
---

# Task 2: Desktop-state public surface (contract, HTTP/UDS/WS, CLI, wiring)

## Overview

Expose the engine as the agent-manageable `desktop-state` surface: canonical wire DTO + OpenAPI schemas + generated TS types, HTTP and UDS routes (CRUD + atomic apply + the WebSocket stream upgrade on BOTH listeners), the `agh desktop-state` CLI verb family, daemon composition-root wiring (resolver, boot/shutdown ordering, workspace-deletion purge gate), the `skills/agh` update, and generated site references. SD-011 is the gate: contract, transports, CLI, docs, skills, and tests move together — nothing partial.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST define the canonical DTO once in `internal/api/contract` per §Data Models (key, value object|null, rev, seq, deleted, updated_at RFC3339; numbers guarded ≤ 2^53−1; list sorted by key; tombstones only as `deleted:true` events) and generate every transport type from it — OpenAPI components include the WS frame schemas.
2. MUST implement the §API Endpoints exactly: list/get/put/apply/delete + `GET .../desktop-state/stream` (WS upgrade) with deterministic error codes (`desktop_state_*`, `workspace_not_found`) — identical on HTTP and UDS listeners, stream included (peer-review B-008).
3. MUST implement the §WebSocket connection lifecycle: one reader pump + one single-writer pump per connection, bounded outbound queue (256), 10s write deadline, 30s/60s ping/pong, best-effort eviction error frame (2s deadline) then close; frame-triggered mutations run under a daemon-owned lifecycle context with a 5s per-op deadline (invariant 8, L-001/SD-010 both directions).
4. MUST echo client `req` ids on ack/error; ack carries `results:[{key,rev,seq}]`; snapshot carries `as_of_seq`.
5. MUST ship `agh desktop-state list|get|set|delete|watch` with repo-standard `-o json` / `-o jsonl` output; `watch` consumes the UDS stream upgrade with identical fence/ordering semantics.
6. MUST wire everything in `internal/daemon` only (SD-008): construct service + resolver (backed by the workspace service), register routes on both listeners, hook workspace deletion → deletion gate → purge, enforce shutdown order (stop upgrades → close subs → join pumps → close store), and register the package in the boundaries check.
7. MUST run `make codegen` and pass `make codegen-check` (OpenAPI + generated TS + CLI docs); update `skills/agh/` with the verb family, routes, error codes, and `os_shell` payload key conventions in the same change.
8. MUST assert status code AND body in every handler test (`eng-test-conventions`); `eng-contract-codegen-coship` governs the contract change.
</requirements>

## Subtasks

- [x] 2.1 Contract DTO + error codes + OpenAPI paths/components (incl. WS frames) + `make codegen` (TS types, CLI docs)
- [x] 2.2 HTTP handlers: list/get/put/apply/delete with full failure-shape mapping from sentinel errors
- [x] 2.3 WS endpoint: upgrade, sub/snapshot/apply/ack/event/error/ping frames, reader+writer pumps, deadlines, eviction
- [x] 2.4 UDS parity: identical routes + stream upgrade on the UDS listener
- [x] 2.5 CLI verb family with `-o` outputs; `watch` over UDS with NDJSON lines in seq order
- [x] 2.6 Daemon wiring: resolver impl, composition, boot/shutdown ordering, workspace-delete purge hook, boundaries registration
- [x] 2.7 `skills/agh` + generated site reference updates
- [x] 2.8 Full assigned suite: handler units, real-socket pump tests, and the 9 integration cases

## Implementation Details

Follow the existing handler/contract/spec layering under `internal/api/` (contract → spec → core) and the current CLI verb registration pattern in `internal/cli/`. The WS handler is the product's first browser-facing socket — the pump/deadline/shutdown rules in §WebSocket frames and §Safety Invariant 8 are normative, not advisory. Workspace deletion flow: find the existing deletion path in the workspace service and insert the gate before row delete (per invariant 9 ordering).

### Relevant Files

- `internal/api/contract/` — new `desktopstate.go` DTO + error codes (+ tests)
- `internal/api/spec/` — OpenAPI operation definitions + components for frames
- `internal/api/core/` — handlers (CRUD/apply/stream) + tests
- `internal/cli/` — `desktop_state.go` verb family + tests (follow existing `client_*.go` patterns)
- `internal/daemon/` — service construction, resolver, listeners, shutdown ordering
- `skills/agh/` — official skill surface documentation
- `openapi/agh.json`, `web/src/generated/agh-openapi.d.ts` — regenerated (never hand-edited)

### Dependent Files

- `internal/clientstate/` (task_01) — consumed service; do not modify its contracts
- `web/src/systems/os/lib/os-state-client.ts` (task_04) — will consume the generated frame types
- Vite dev proxy config (task_04 verifies `ws: true` forwarding)

### Related ADRs

- [ADR-004](adrs/adr-004.md) — transport decision + lifecycle amendment
- [ADR-008](adrs/adr-008.md) — public naming (`desktop-state`, no domain axis), quota semantics

## Deliverables

- Complete public surface: contract + OpenAPI/TS/CLI-docs codegen + HTTP/UDS/WS + CLI + daemon wiring + skills/agh + site refs, all in one change set
- Deterministic error contract identical across all four transports
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-022–UT-029 — HTTP handler success + every failure shape (status AND body)
- [x] UT-030–UT-034 — WS sub/snapshot, ack/event with origin, error frames, slow-consumer close, detached-commit-on-disconnect
- [x] UT-035–UT-039 — CLI verb family: `-o json` shapes, if-rev conflict exit, jsonl watch, not-found errors
- [x] UT-078, UT-079 — writer-pump serialization + deadline eviction with real sockets; shutdown joins with queued mutation
- [x] IT-001–IT-009 — daemon-wired cross-transport flows: HTTP↔WS visibility, workspace isolation, purge-on-delete, restart durability, CLI parity, UDS byte-parity, UDS watch parity, commit-order under load

## Success Criteria

- Every assigned test case implemented and passing (`go test -race` scoped + `+integration` lane)
- `make codegen-check` clean; boundaries check passes; `make lint` zero issues
- An agent can list/get/set/delete/watch any workspace desktop via CLI with structured output and deterministic errors, with results identical over HTTP and UDS (IT-006/IT-007/IT-008 as evidence)
- Web/Docs Impact: `web/src/generated/agh-openapi.d.ts` regenerated (consumed by task_04); `packages/site` generated API + CLI references pick up desktop-state; no authored web component changes in this task (checked: web routes/components untouched)
- Extensibility/Agent Manageability/Config Lifecycle: agent-manageability surface IS this task (CLI+HTTP+UDS+errors+skills/agh); no extension/hook/tool/bundle/registry contract changes (checked: Host API method list, hook catalog, native toolsets — unchanged; no public domain namespace per ADR-008); config shipped in task_01, referenced in generated config docs here
- QA impact: new user-visible surface → task_10 mints content-addressed `untested` scenarios for `agh desktop-state` CLI + API behaviors (flag recorded here, scenarios authored by the QA pair)
