---
status: completed
title: Persistence, grammar and gate-plane foundation
type: backend
complexity: high
---

# Task 1: Persistence, grammar and gate-plane foundation

## Overview

Delivers everything the succession rework builds on: the schema migration (two tables, two
`loop_runs` columns, pre-change run-history hard cut), the `metric` DSL block with its lint rules,
the `previous`/`best` namespace grammar, score emission across the three machine criterion types,
metric comparison with best-eligibility, the sanitization boundary, and the typed mutation intents
(including the Go-side `Confidence` → `Score` event-payload swap so the tree compiles closed).

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST activate skills: `eng-schema-migration`, `eng-code-guidelines`, `golang-pro`, `eng-test-conventions`, `testing-boss` (and `eng-consolidate-test-suites` before adding tests).
- MUST follow TechSpec "Data Models": `loop_generations` + `loop_gate_verdicts` with the N-001 CHECK constraints, `loop_runs.best_generation`/`best_score` with paired-nullability CHECK, declarative source + appended gap-free Goose migration + `atlas.sum` + sqlc via `make codegen` (never hand-edited).
- MUST implement the B-005 hard cut inside the migration: delete pre-change loop-run history (runs + children + orphaned blobs) while preserving definitions, catalog entries, and input defaults.
- MUST add the `metric` block (`direction`, optional `min_delta`) to `GateCriterion` with blocking lint rules: single metric per definition, machine-criteria-only placement, direction required, `min_delta` finite and non-negative.
- MUST extend namespace grammar (`ValidatePath`, CEL variable set, linter reference builders) with `previous.*` (nodes, `verdicts.<gate_id>`, `route_causes`, generation) and `best.*` roots — identical accept/reject sets across all three validators; no singular `previous.verdict` path exists.
- MUST implement score emission contracts (command stdout JSON, agent-judge rubric field, extension response field), direction-aware comparison with best-eligibility (Safety Invariants 1-3), and `invalid_output` on non-finite/contract-violating scores.
- MUST implement the sanitization boundary (Safety Invariant 11) reusing the existing diagnostics redaction/bounding primitives + claim-token redactor, applied once before intent construction.
- MUST define `VerdictIntent`/`BestUpdateIntent`/`GenerationIntent` types and delete `CriterionResult.Confidence` together with its Go-side readers (store event builder switches to `score`; TS consumers move in task_04) — no dual field, no compat shim.
- MUST respect the 500-line cap: new behavior lands in new files (`dsl/gate_metric.go`, `linter_metric.go`, `dsl/refs/namespace_history.go`, `gate/metric.go`, `gate/verdict_intent.go`, `gate/sanitize.go`); `linter.go` (457) and `linter_references.go` (493) must not grow.
</requirements>

## Subtasks

- [x] 1.1 Extend `50_loops.sql` declarative fragments with the two tables + two columns + CHECKs; append the Goose migration including the pre-change history hard cut; regen sqlc/Atlas via `make codegen`.
- [x] 1.2 Add sqlc queries: insert/list generations, insert/list verdicts, `ListRouteCausingVerdicts`, best-column updates — package-private behind repository mappings.
- [x] 1.3 Add `MetricSpec` to the DSL + the four blocking lint rules with deterministic diagnostics codes.
- [x] 1.4 Extend `ValidatePath`, CEL env, and linter reference builders with the two roots (new files; shared path-table test).
- [x] 1.5 Implement score emission in the three machine evaluators + typed parse failures → `invalid_output`.
- [x] 1.6 Implement `gate/metric.go` comparison (direction, `min_delta`, eligibility, non-finite rejection).
- [x] 1.7 Implement the sanitizer and route all evaluator diagnostics through it before intent construction.
- [x] 1.8 Define intent types; delete `Confidence` field + Go readers; migrate the store event-builder payload keys (`confidence` → `score`) so the tree compiles with zero references.
- [x] 1.9 Implement all assigned UT/IT cases in their canonical suites; run `make gate`.

## Implementation Details

TechSpec sections: Data Models, Core Interfaces (intents, VerdictReader), Safety Invariants 1-3/11,
Delete Targets (Confidence bullet, history hard cut). Score/verdict semantics: ADR-003.

### Relevant Files

- `internal/store/globaldb/schema/definitions/50_loops.sql` — declarative source (384 lines; headroom)
- `internal/store/globaldb/queries/loop_runtime.sql` — sibling query patterns
- `internal/loop/dsl/gate_start.go` — `GateCriterion`/`CriterionType` (74 lines)
- `internal/loop/dsl/refs/namespace.go:77-109` + `dsl/refs/condition.go:55-60` — path/CEL validation seams
- `internal/loop/gate/types.go` (290) — `CriterionResult`/`Verdict`; `Confidence` at :159
- `internal/loop/gate/evaluator_command.go` (93), `gate/judge.go:120-185` (247), `gate/evaluator_extension.go:90` (155) — score emission points
- `internal/store/globaldb/global_db_loop_event_boundaries.go:18,109-146` — `confidence` payload keys to migrate
- `internal/loop/linter.go:244` (457) — criterion validation home; extend via `linter_metric.go`

### Dependent Files

- `internal/loop/gate/evaluator.go:159-231` — aggregation consumes `Score`/eligibility
- `internal/store/globaldb` migration test suites — fresh/reopen/ahead/integrity/equivalence extensions
- `internal/api/contract/loop_enums.go` — outcome vocabulary mirrored in the verdict CHECK (no Go change here)

### Related ADRs

- [ADR-003: Metric-gated ratchet](adrs/adr-003.md) — metric block, eligibility, verdict persistence shape
- [ADR-004: Generation provenance](adrs/adr-004.md) — `loop_generations` schema + hard-cut rationale
- [ADR-001: Generation-boundary repair context](adrs/adr-001.md) — namespace grammar shape

### Web/Docs Impact

- `web/`: none in this task — TS consumers of the migrated event payload move in task_04 (`web/src/systems/loops/lib/loop-events.ts`, generated types); checked surfaces: no contract/OpenAPI change lands here.
- `packages/site`: none in this task — grammar/metric docs land in task_05; no generated CLI/API refs change here.
- QA impact: the foundation itself does not change a public route. The gate exposed and this task
  repaired an existing user-visible CLI manual-auth deadline defect; canonical scenario
  `ET-cli-mcp-auth-manual-exchange` was reset, replayed, and returned to `pass`. Loop succession
  behavior scenarios remain owned by task_06 after tasks 02-03 land.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: `extension` criterion response contract gains optional `score` (documented in task_05); criterion-kind enum stays closed; no manifest/bundle/registry/bridge/MCP change — checked `internal/extension/contract`, `internal/bundles`, `internal/registry`.
- Agent manageability: `compozy loop validate` surfaces the four new deterministic lint codes; no verb/route changes in this task.
- Config lifecycle: none — no `config.toml` keys added/changed; checked `internal/config/loops.go`, `defaults.go`, `merge_loops.go`, `tool_surface_loops.go`; existing bounds govern the new paths.

## Deliverables

- Migration applied: fresh + reopen(hard-cut) + equivalence + `make codegen-check` green.
- Metric grammar + namespace roots linting deterministically; shared path-table proving validator parity.
- Score emission + comparison + sanitizer + intents compiled and covered; zero `Confidence` references in Go.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-001, UT-002, UT-003, UT-004, UT-005, UT-006, UT-041 — metric grammar + lint rules
- [x] UT-007, UT-008, UT-009, UT-010 — namespace path validation + CEL parity
- [x] UT-015, UT-016, UT-017, UT-018, UT-019, UT-020 — metric comparison + best eligibility
- [x] UT-021, UT-022, UT-023, UT-024, UT-025, UT-026 — score emission contracts (3 evaluators)
- [x] UT-027, UT-040 — verdict intent projection + sanitizer redaction/bounding
- [x] IT-004, IT-005 — verdict queries ordering + human-only `loop_gate_decisions`
- [x] IT-013, IT-014, IT-015 — migration fresh / hard-cut reopen / equivalence + atlas integrity

## Success Criteria

- Every assigned test case implemented and passing
- `make gate` green; `make codegen-check` green; zero lint warnings
- No production file exceeds 500 lines; frozen files did not grow
- Grep proof: zero Loop gate `CriterionResult.Confidence` identifiers and zero `confidence` keys in
  Loop gate-verdict event payloads; unrelated task-review and memory confidence contracts remain.

## Completion Evidence

- Fresh/reopen/equivalence/integrity migration coverage, GlobalDB integration suites, and focused
  Loop/refs/gate suites pass under `-race`; `make codegen-check` and Go lint report zero drift/issues.
- The first full gate found `BUG-20260801-mcp-manual-input-timeout`; its root correction passes the
  complete `internal/cli` race suite and Windows cross-compile. The rebuilt public replay returned
  the canonical timeout in 0.966 seconds while CLI/HTTP status remained unauthenticated.
- QA evidence:
  `~/dev/qa-labs/compozy-loops-paper-task01-mcp-manual-timeout-20260801-075922-632496-lab/qa-artifacts/qa/notes/manual-input-timeout-public.json`.
  Teardown evidence is the same lab's `qa/teardown.json` with `"clean": true` and no survivors.
- Final task gate evidence is the current `make gate-status` record captured by the checkpoint.

## Compozy Impact Audit

- **Native tools:** no direct change; checked Loop and MCP native IDs, toolsets, descriptors, schema
  digests, risk flags, capability gates, and CLI/API fallbacks. Task 01 changes internal foundation
  types and the existing CLI input boundary only.
- **Extensibility and hooks:** the extension criterion may emit the optional internal `score` used by
  the gate evaluator. Checked extension contracts, hooks, skills/capabilities, tools/resources,
  bundles, registries, bridge SDKs, MCP sidecars, and config lifecycle; no new surface or key lands.
- **Workspace data isolation:** generation, verdict, and best data are run-scoped and reached only
  through workspace-fenced run ownership. Foreign-workspace GlobalDB tests cover history/best reads;
  no CLI/API/UDS/SSE/cache/event path gains a workspace-free lookup.
- **Official Compozy skill:** no Task 01 content change; no public Loop verb, tool ID, or final grammar
  documentation ships until task_05. The existing MCP timeout statement already matches the repair.
