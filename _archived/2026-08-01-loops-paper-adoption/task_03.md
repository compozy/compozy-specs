---
status: completed
title: Public surfaces, codegen co-ship and runtime E2E
type: backend
complexity: high
---

# Task 3: Public surfaces, codegen co-ship and runtime E2E

## Overview

Closes the loop end-to-end across every public surface in one pass (no-partial-surface rule):
contract DTO mapping (summary vs detail), the `loop_generations`-backed payload query, the
`gate_verdict` payload migration on SSE, UDS parity, CLI renderers, native-tool schemas, hook
payload fields, generated artifacts, and the full runtime E2E suite with co-shipped acpmock
matchers (L-007). Also proves the secret-redaction boundary across every surface.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST activate skills: `eng-contract-codegen-coship`, `eng-code-guidelines`, `golang-pro`, `eng-test-conventions`, `eng-data-boundaries` (read-model changes).
- MUST implement the B-008 DTO mapping exactly: `best_generation`/`best_score` on `LoopRunPayload` (list/`loop runs`/`loop list`/`compozy__loop_runs`); `parent_generation`, `origin`, `verdicts[]{gate_id,outcome,score,route_cause_rank?}` on `LoopRunResponse.Generations` (detail/`loop status`/`compozy__loop_status`). Summaries never embed per-generation history.
- MUST replace the counted `1..run.Generation` payload loop with the `loop_generations` query (delete target).
- MUST ship the migrated `gate_verdict` SSE payload exactly as specified (before/after schema in TechSpec API Endpoints) and the enriched `generation_started` payload; broadcast only post-commit; contract enum untouched (kind exists).
- MUST keep HTTP/UDS parity through shared `BaseHandlers`; CLI renders via contract types only; native tool descriptor/schema digests refreshed; hooks payload fields added.
- MUST run `make codegen` + `make codegen-check` in-task (contract co-ship); OpenAPI + generated TS types move with this change.
- MUST co-ship acpmock matchers/fixtures for every runtime E2E case in the same change as the contract (L-007), including the prompt-content matcher for `previous.verdicts.*` interpolation (E2E-003).
- MUST prove workspace isolation (404 + zero rows cross-workspace) and cross-surface secret redaction (IT-024).
</requirements>

## Subtasks

- [x] 3.1 Contract types: `LoopRunPayload` best fields; `LoopGenerationPayload` provenance + verdicts; SSE payload structs; `make codegen`.
- [x] 3.2 Payload assembly from `loop_generations`/`loop_gate_verdicts` queries; delete the counted loop.
- [x] 3.3 SSE: migrated `gate_verdict` + enriched `generation_started`, post-commit ordering + replay.
- [x] 3.4 CLI renderers (`loop status` detail; `loop runs`/`loop list` summary) in table + `-o json|jsonl`.
- [x] 3.5 Native tool output schemas + descriptor digests (`compozy__loop_status`, `compozy__loop_runs`); toolmeta entries.
- [x] 3.6 Hook payload fields (`loop.gate.post`, `loop.generation.pre`) + dispatch assertions.
- [x] 3.7 Runtime E2E suite (E2E-001..005, 007) with co-shipped acpmock matchers; deterministic oracles.
- [x] 3.8 Isolation + redaction integration proofs (IT-021, IT-024); run `make gate`.

## Implementation Details

TechSpec sections: API Endpoints, Agent Manageability Plan, Monitoring and Observability, Delete
Targets (counted loop, event `confidence` keys Go-side already gone in task_01).

### Relevant Files

- `internal/api/contract/loops.go:296-308` + `loop_catalog.go` + `loop_enums.go` — payload/enum surface
- `internal/daemon/loop_api_runs.go:303-321` — counted loop (delete target); payload assembly
- `internal/daemon/loop_api_payloads.go:77-82` — summary payload builder
- `internal/api/httpapi/loops_routes.go:35-62` + `internal/api/udsapi/loops_routes.go:39+` — route parity
- `internal/cli/loop_runs.go:14-184`, `internal/cli/loop_list.go:16-80` — renderers + output bundles
- `internal/tools/builtin/loops.go:36-192,225` + `internal/toolmeta/native_entries.go:90-103` — schemas/digests/gates
- `internal/hooks/events.go:141-147` + payload structs — hook fields
- `internal/api/spec/loops.go` + `loop_schemas.go` — OpenAPI spec source

### Dependent Files

- `web/src/generated/*` — TS types regenerate (consumed in task_04)
- `openapi/compozy.json` — regenerated artifact (codegen-check)
- E2E runtime harness + acpmock fixtures — matchers co-ship here

### Related ADRs

- [ADR-003](adrs/adr-003.md) — verdict/score exposure; [ADR-004](adrs/adr-004.md) — provenance payloads; [ADR-002](adrs/adr-002.md) — deterministic terminal oracles for E2E.

### Web/Docs Impact

- `web/`: generated types change (`web/src/generated/compozy-openapi.d.ts`, `web/src/generated/loop-enums.ts`); component/hook/reducer updates happen in task_04 — this task must leave `make bun-typecheck` actionable for it (breaking TS changes expected and owned by task_04's edge).
- `packages/site`: generated API/CLI reference pages change content (`api/loops.mdx`, `cli/loop/*`) — regenerated in task_05.
- QA impact: user-visible CLI/API changes land here; scenarios minted/reset in task_06 and walked in task_07 (`LP-loop-run-deep-link` reset; new `LP-ratchet-*`/`LP-dod-reject-retry`/`LP-revise-repair-context` cover CLI+API evidence).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: hook payload additions surfaced to extensions (additive; matcher taxonomy unchanged); extension scorer contract documented in task_05; no manifest/bundle/registry/bridge/MCP changes — checked `internal/extension/contract/host_api.go` (no loop methods affected).
- Agent manageability: `loop status`/`loop runs`/`loop list` structured outputs updated with deterministic shapes; `compozy__loop_status`/`compozy__loop_runs` schemas updated; errors unchanged; UDS = HTTP parity asserted (IT-018).
- Config lifecycle: none — no keys; checked `internal/config` loop surfaces (unchanged defaults govern).

## Deliverables

- All public surfaces closed in one pass: contract → HTTP → UDS → CLI → native tools → hooks → codegen artifacts.
- Runtime E2E suite green with deterministic oracles and co-shipped matchers.
- `make codegen-check` green; redaction + isolation proofs recorded.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-036 — migrated `gate_verdict` payload shape (no `confidence` keys)
- [x] UT-037, UT-038 — CLI detail vs summary rendering (golden JSON)
- [x] UT-039 — hook payload fields
- [x] IT-008 — payload assembly via `loop_generations` query
- [x] IT-016, IT-017, IT-018, IT-019 — HTTP field-by-field, SSE durable ordering + replay, UDS parity, native tool schema digest
- [x] IT-020 — hook dispatch observation + patch rejection
- [x] IT-021 — workspace isolation (zero rows + 404)
- [x] IT-024 — secret absence across SQL/HTTP/UDS/CLI/native/SSE/namespace/logs
- [x] E2E-001, E2E-002, E2E-003, E2E-004, E2E-005, E2E-007 — ratchet climb/restore, DoD retry prompt-content proof, livelock bound (both variants), exhaustion→best

## Success Criteria

- Every assigned test case implemented and passing
- `make gate` green; `make codegen` + `make codegen-check` green in-task
- `rg 'for generation := 1'` returns nothing under `internal/daemon` (counted loop deleted)
- UDS/HTTP/CLI/native outputs byte-consistent for the same persisted state (IT-018 evidence)

## Completion Evidence

- `make codegen && make codegen-check` passed; OpenAPI, Go SDK, TypeScript SDK, web types, enums,
  native descriptors, and schema digests are synchronized.
- Runtime feedback E2E passed all six assigned journeys: ratchet climb/restore, DoD feedback,
  producer revision, both livelock bounds, and exhaustion-to-best.
- `TestLoopRuntimeSelectionIntegration`, loop event replay, hook dispatch, native schemas, workspace
  isolation, and the expanded redaction E2E all passed. The redaction proof covers SQL, HTTP, UDS,
  CLI JSON, native tools, SSE replay, rendered previous-verdict context, logs, and the isolated home.
- `rg 'for generation := 1' internal/daemon` returned no matches; detail assembly reads durable
  `loop_generations` and `loop_gate_verdicts` instead of synthesizing counted generations.
- `make gate` passed with fingerprint `d78bf49775c7ffbcf42cb96f11a733bb52fdd9f3` and log
  `.cache/gate/logs/full-1785583357.log`.

## Compozy Impact Audit

- Native tools: updated `compozy__loop_status` detail and `compozy__loop_runs` summary schemas,
  descriptors, digests, catalog fixture, and contract tests; no capability-gate or CLI fallback change.
- Extensibility and hooks: added generation `origin`/`parent_generation` and gate
  `outcome`/`score`/`best_generation` hook fields with strict patch decoding; checked extension Host
  API, bundles, registries, bridge SDKs, and MCP sidecars — no other surface changes. No new
  `config.toml` keys or defaults; the existing gate revision limit remains authoritative.
- Workspace data isolation: generation lineage, verdicts, and best state are workspace-scoped through
  store, daemon, HTTP/UDS, CLI, native tools, SSE, and caches/events. Integration coverage proves
  foreign-workspace 404s, zero query rows, no SSE leakage, and native denial.
- Official Compozy skill: updated `skills/compozy/references/loops.md` with summary/detail boundaries,
  provenance, verdict, best-state, hook, SSE, and command-criterion fields. Broad site and generated
  reference prose remains owned by Task 05.
