---
status: completed
title: Catalog loop classification with 4-surface parity
type: backend
complexity: high
---

# Task 2: Catalog loop classification with 4-surface parity

## Overview

Delivers front 1's backend: loop execution records are classified by provenance at the source and excluded from every task listing surface by default, with a typed include filter and structured `LoopProvenance` projected on both catalog items and the single-task read. The same default and filter land on HTTP, UDS, CLI, and the native tool in one change — no divergent defaults, no id-string parsing anywhere.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST extend `internal/task` `CatalogQuery` with NEUTRAL fields only: `ExcludeCreatedBy []ActorRef` + `LoopRunID string` (the one acknowledged correlation name). The literal `"loop-coordinator"` never enters `internal/task` — the `include_loop=false → exclude {daemon, loop-coordinator}` mapping is owned by `internal/api/core` (UT-027).
2. MUST implement exclusion as a server-side SQL predicate on `created_by` columns in the catalog SQL builder; facets, counts, dashboard aggregates, and inbox lanes compute over the same filtered set (Safety Invariant 8). Classification matches provenance columns only — never id or title strings (Safety Invariant 9).
3. MUST implement `loop_run_id` scoping as a correlated `EXISTS` semi-join on `task_runs.loop_run_id`, applied BEFORE ordering/pagination — exactly one row per task across multi-attempt runs, no post-pagination `DISTINCT` (B-006; IT-013 seeds three attempt runs).
4. MUST project ONE shared optional `LoopProvenance` wire type — `{run_id, loop_name?, role: coordinator|cell, generation?, node_id?, item_index?}` — on BOTH the catalog item payload and the single-task payload (`GET /api/tasks/:id` + CLI structured read); `loop_name` omitted (Go omitempty / TS optional) when the owning run was retention-deleted and unrecoverable — the record renders relational facts only (B-003, N-001).
5. MUST add typed query fields `include_loop` (bool, default false) and `loop_run_id` (string, implies include) to `GET /api/tasks` on HTTP and UDS, to `compozy task list` (`--include-loop`, `--loop-run`), and to the native tool `compozy__task_list` (args + descriptor + input-schema digest + tests) — same defaults, same semantics, in the same change (4-surface parity, IT-015).
6. MUST keep `parent_task_id` returning children regardless of `include_loop` (explicit parentage is an explicit request, IT-026); unknown `--loop-run` returns an empty list (filter, not lookup).
7. MUST return deterministic errors per `_dx.md`: non-boolean `include_loop` → `400 {"error":"invalid_query_field","field":"include_loop"}` (status AND body asserted).
8. MUST honor cursor-filter binding: a cursor issued under one filter context replays deterministically (UT-026), and every catalog read stays workspace-scoped (cross-workspace → scoped result, never data, IT-016).
9. MUST co-ship the contract: OpenAPI spec authoring + `make codegen` (generated TS types) + E2E fixture/matcher updates in this task (L-007); `make codegen-check` green.
10. MUST update the official Compozy skill (`skills/compozy/`) task-listing reference: calm default + `--include-loop`/`--loop-run` contract.
11. SHOULD verify performance of the exclusion predicate rides `idx_tasks_created_by` (ADR-004); the generated-column fallback is recorded there if the join proves slow — do not implement it preemptively.
</requirements>

## Subtasks

- [x] 2.1 Extend `CatalogQuery` with the neutral fields (`internal/task`)
- [x] 2.2 Exclusion predicate + `EXISTS` semi-join + `metadata_json` provenance projection in the catalog SQL builder (`internal/store/globaldb`)
- [x] 2.3 `LoopProvenance` shared wire type + projection on catalog item AND single-task payloads, with the truthful retention-degrade shape
- [x] 2.4 Handler mapping in `internal/api/core` (`include_loop`/`loop_run_id` parsing, constant ownership, deterministic 400s)
- [x] 2.5 Facets/counts/dashboard/inbox computed over the filtered set
- [x] 2.6 CLI `compozy task list --include-loop --loop-run` + LOOP column in table output + `-o json` parity
- [x] 2.7 Native tool `compozy__task_list` typed args + descriptor + schema digest + tests
- [x] 2.8 Contract co-ship: OpenAPI authoring, `make codegen`, generated TS, E2E fixtures/matchers
- [x] 2.9 Official skill task-listing reference update (`skills/compozy/`)
- [x] 2.10 Implement assigned unit tests (UT-021..029)
- [x] 2.11 Implement assigned integration tests (IT-010..016, IT-026, IT-029, IT-033)
- [x] 2.12 Flag QA scenarios per the QA impact line

## Implementation Details

Follow `_spec.md` Part II: Core Interfaces (`CatalogQuery`), Data Models (`LoopProvenance` row; side-table-vs-JSON decisions), API Endpoints (`GET /api/tasks`, `GET /api/tasks/:id`), Architectural Boundaries (kernel loop-noun-free). Skills: `eng-code-guidelines` + `golang-master` + `eng-contract-codegen-coship`; tests per `eng-test-conventions` + `testing-boss` + `eng-consolidate-test-suites`; completion per `deslop` + `cy-final-verify`.

Suite placement (from `_tests.md`): catalog SQL cases EXTEND the existing task-catalog SQL suite in `internal/store/globaldb`; handler mapping EXTENDS the task-catalog param-parsing suite in `internal/api/core`; parity cases EXTEND the API parity/testutil harness in `internal/api`.

### Relevant Files

- `internal/task/catalog.go:57-73` — `CatalogQuery` to extend (neutral fields only).
- `internal/store/globaldb/global_db_task_catalog_sql.go:308-405` — `taskCatalogBaseFilter`/columns/order/limit: where predicate, join, and projection land.
- `internal/api/core/task_catalog.go:16-46` — `ParseTaskListQuery` → domain query; `include_loop` mapping + the `loop-coordinator` constant ownership.
- `internal/api/core/task_crud_handlers.go:20,89` — shared `ListTasks`/`GetTask` handlers (HTTP + UDS parity by construction).
- `internal/api/core/task_payload_mapping.go` + `task_catalog_conversion.go` — domain→wire mapping (where `loop` projects).
- `internal/api/contract/task_catalog.go` + `tasks.go` + `task_catalog_validation.go` — wire contract for list + single-task payloads.
- `internal/api/httpapi/routes.go:162-174` — tasks route registration.
- `internal/cli/task_list.go:13-89` — CLI flags + output columns.
- `internal/cli/task_output_core.go` + `json_flags.go` + `list_page_output.go` — table/JSON rendering helpers.
- `internal/daemon/native_task_list.go:13-76` — native tool args/descriptor/digest.
- `internal/daemon/scheduler_loop_coordinator.go:13` — the existing `loop-coordinator` ref constant (seed side).
- `internal/api/spec/` — OpenAPI authoring for tasks routes (mirror the loops pattern in `spec/loops.go`).
- `skills/compozy/` — official skill task-listing reference.
- `internal/loop/coordinator_metadata.go:20-50` + `internal/loop/constants.go:4-7` — cell metadata keys the projection reads.

### Dependent Files

- `web/src/generated/compozy-openapi.d.ts` — regenerates with `loop` object + query fields (consumed by task_04).
- `web/src/systems/tasks/adapters/tasks-catalog-api.ts` + `mocks/{handlers,fixtures}.ts` — task_04 binds these to the new contract (ownership split: this task ships backend + generated types only).
- `web/e2e/fixtures/scenario-contracts.ts` + `runtime-seed.ts` — E2E contract fixtures updated here (co-ship).
- `internal/api/core/tasks_test.go` + `tasks_surface_test.go` + `internal/api/httpapi/transport_parity_integration_test.go` — suites extended.
- `packages/site/content/docs/cli/**` — CLI docs regenerate via `make codegen`.

### Competitor References

- `.resources/temporal/common/persistence/visibility/store/query/converter.go:187-210,486-488` — the mention-based opt-out trap our typed `include_loop` field avoids; mirror the typed-field discipline, reject query-text inference.

### Related ADRs

- [ADR-001: Loop execution records leave the Tasks surfaces by default](adrs/adr-001.md) — the default this task implements.
- [ADR-004: Classification rides existing provenance columns](adrs/adr-004.md) — predicate/join/projection design + recorded fallback.

## Deliverables

- Server-owned calm default + typed include on all four listing surfaces with identical semantics
- `LoopProvenance` on catalog + single-task payloads (retention-degrade shape included)
- Regenerated OpenAPI/TS artifacts + E2E fixtures (co-ship), `make codegen-check` green
- Official skill task-listing reference updated
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-021, UT-022, UT-023, UT-024, UT-025, UT-026 — SQL predicate exclusion, true-empty facets, provenance-not-name, include-path projection, `LoopRunID` join, cursor-filter binding
- [x] UT-027, UT-028, UT-029 — handler mapping (constant ownership, `loop_run_id` implies include, deterministic 400 with body)
- [x] IT-010 — first-boot backfill (task_01) then catalog include-path projects `role=coordinator`
- [x] IT-011, IT-012, IT-013, IT-014 — clean default under active-run churn; include path provenance; run scoping with 3 attempt runs (one row per task, coherent facets, gap-free cursors); dashboard/inbox aggregates exclude loop records
- [x] IT-015, IT-016 — 4-surface semantic parity; cross-workspace denial with asserted bodies
- [x] IT-026 — `parent_task_id` drill-down without the global flag
- [x] IT-029 — provenance backfill on an ACTIVE run's coordinator (metadata-only) + truthful degraded `loop` shape for deleted runs
- [x] IT-033 — single-task read carries the same `loop` object as the catalog item; retention-deleted run omits `loop_name`

## Success Criteria

- Every assigned test case implemented and passing
- `compozy task list` (default) shows zero loop rows during an active run on every page; `--include-loop --loop-run` matches the `_dx.md` transcripts verbatim
- HTTP, UDS, CLI `-o json`, and `compozy__task_list` return identical item sets and provenance fields for the same query (semantic parity, N-003)
- `make codegen-check` green; `make gate` green on the task's diff

### Web/Docs Impact

- `web/`: `web/src/generated/compozy-openapi.d.ts` regenerates (new query fields + `loop` object); component/hook/fixture changes belong to task_04 (`web/src/systems/tasks/**`) — no web component edits in this task.
- `packages/site`: generated CLI reference for `task list` regenerates via `make codegen` (`content/docs/cli/**`); generated API reference reflects the new query fields.
- QA impact: user-visible CLI/API behavior change (default excludes loop records everywhere) → add a content-addressed `untested` scenario for "task listings calm default + typed include parity (CLI/HTTP/UDS/native)"; flag only — the walk runs in the loop's QA phase.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: native tool `compozy__task_list` descriptor + input-schema digest updated with tests (the only `compozy__*` tool reading task listings — checked `internal/daemon/native_*`); bridge task subscriptions unaffected (terminal notifications by task id, not listings); Extension Host API method set unaffected; no new extension points.
- Agent manageability: typed flags with documented defaults; deterministic `400 invalid_query_field`; `-o json` provenance fields replace id parsing; documented in the official skill.
- Config lifecycle: none — no `config.toml` keys added/changed/removed (checked `internal/config`: `include_loop`/`loop_run_id` are per-request query fields, not settings).
