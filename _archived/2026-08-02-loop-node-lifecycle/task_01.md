---
status: completed
title: Schema, classification, DSL grammar and config contracts
type: backend
complexity: high
---

# Task 1: Schema, classification, DSL grammar and config contracts

## Overview

Delivers every contract the rest of the train builds on: the single Goose migration (five new
tables, three row-preserving rebuilds, the `loop_run_events.delivery_key` column), the closed
`FailureClass` classifier with payload-declared detection and hint propagation, the full DSL
failure vocabulary (`on_*`, `effects`, promoted `retry:`, new `deadline:`, `result_contract`,
`wait` kind, `on_parent_close`) with its lint rules, and the `[loops.defaults.*]` +
`[loops.breaker]` config lifecycle. No behavior changes ship here beyond classification and
validation — later tasks consume these shapes.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- R1: MUST ship ONE next-version Goose migration covering: CREATE `loop_node_controls`,
  `loop_node_attempts`, `loop_node_waits`, `loop_admission_claims`, `loop_effect_outbox`;
  row-preserving rebuilds of `loop_generation_outputs` (+`attempt`, `next_attempt_at`,
  `first_scheduled_at`, `epoch`, status CHECK), `loop_generations` (origin CHECK + `requeue`),
  and `dead_entities` (kind CHECK + `loop_target`); `loop_run_events` ADD COLUMN `delivery_key`
  + partial UNIQUE index; `loop_runs` ADD COLUMN `cancel_requested` + `cancel_kind` (run-level
  cancel authority, round-3 B-001) — declarative sources `50_loops.sql` + `41_reliability.sql`,
  then `make codegen` (Atlas + sqlc) and `make codegen-check` (TechSpec Data Models; ADR-011).
- R2: MUST implement `FailureClass` + `ClassifyNodeFailure` as a pure function with the fixed
  evaluation order, the v0 payload truth table, per-node `result_contract`, hint extraction from
  `ActionFailure.Recovery`, and single-pass sanitization (ADR-013; TechSpec Core Interfaces).
- R3: MUST extend the DSL envelope (`Deadline`, `ResultContract`, `OnError{Route XOR AllowFail,
  Effects}`, six trigger effect lists, `on_parent_close`), `RetrySpec` (+`Backoff`,
  `NonRetryable`), the `wait` control kind params, and contract terminal triggers including
  `on_canceled` — with normalize→lint→normalize round-trip stability (ADR-010).
- R4: MUST add every lint rule split by severity exactly as the TechSpec DSL Grammar section
  lists (blocking vs warnings), in NEW linter files (`linter.go`/`linter_references.go` are
  frozen), plus the CEL 80% cost-warn via `OptTrackCost`.
- R5: MUST land the config lifecycle: per-kind `[loops.defaults.delivery|watch]`
  retry/liveness/resume/predicates/waits/admission keys + `[[…autopause]]` rules + global
  `[loops.breaker]` — structs, defaults, validation (incl. monotonic pairs), pointer overlays,
  20 agent-mutable tool-surface paths, and `TestToolConfigPathPolicy` coverage (US-027).
- R6: MUST NOT add any behavior consumers here (no planner, no verbs) — contracts only; goal
  nodes MUST reject generic retry fields and `deadline` (`CodeRetryOnGoalNode`).
</requirements>

## Subtasks

- [x] 1.1 Declarative schema fragments + the single Goose migration + Atlas/sqlc regen
- [x] 1.2 sqlc queries + store row types for the five new tables and extended outputs
- [x] 1.3 `FailureClass` enum, evidence struct, pure classifier with fixed order
- [x] 1.4 Payload-declared detection (built-in rules + `result_contract`) and hint propagation
- [x] 1.5 DSL envelope + `RetrySpec`/`BackoffSpec` + effect/wait grammar + `on_canceled`
- [x] 1.6 Lint rules (blocking + warning split) in new linter files + CEL cost-warn
- [x] 1.7 Config structs/defaults/validation/overlay + `[loops.breaker]` + tool-surface paths
- [x] 1.8 Canonical migration suites extended (fresh/reopen/ahead/integrity/equivalence + CASCADE)

## Implementation Details

Follow TechSpec "Data Models", "DSL Grammar", "Core Interfaces" (classifier), and "Config
Lifecycle". New files per the Architectural Boundaries 500-cap plan: `failure_class.go`,
`failure_classify.go`, `failure_result_contract.go`, `linter_lifecycle.go`, `linter_wait.go`,
`dsl/lifecycle.go`, `dsl/effects.go`, `dsl/wait.go`, store `global_db_loop_node_*.go`,
`global_db_loop_admission_claims.go`.

### Relevant Files

- `internal/store/globaldb/schema/definitions/50_loops.sql` — loop DDL declarative source
- `internal/store/globaldb/schema/definitions/41_reliability.sql` — `dead_entities` kind CHECK
- `internal/store/globaldb/schema/migrations/` — Goose stream (verify the CURRENT head at execution time; it advances with parallel work — never assume a pinned number)
- `internal/loop/action_failure.go` — `ActionFailure{Code,Cause,Recovery}` hint slot
- `internal/loop/dsl/graph.go:28-56` — flat Node envelope (fields attach here)
- `internal/loop/dsl/node_params.go:68` — existing `RetrySpec`
- `internal/loop/dsl/types_nodes.go` — control-kind enum (gains `wait`)
- `internal/loop/types.go:47-124` — lint code inventory; `internal/loop/linter.go:42` phase pipeline (frozen — extend via new files)
- `internal/loop/dsl/refs/condition.go` — `ConditionCompiler`, cost limit
- `internal/config/loops.go` + `merge_loops.go` + `tool_surface_loops.go` — config pattern
- `internal/config/autonomy.go:60-84` — monotonic cross-field validation precedent
- `internal/tools/errors.go` — ToolError codes feeding classification

### Dependent Files

- `internal/loop/generation_snapshot.go` — outputs writer learns new columns (task_02 consumes)
- `internal/store/globaldb/global_db_task_claim_complete.go:349` — second outputs writer
- `internal/store/globaldb/sqlcgen/` — regenerated queries
- `packages/site/content/docs/configuration/config-toml.mdx` — key docs (task_11 finalizes)

### Related ADRs

- [ADR-010](adrs/adr-010.md) — DSL surface · [ADR-011](adrs/adr-011.md) — persistence shape
- [ADR-013](adrs/adr-013.md) — classification · [ADR-014](adrs/adr-014.md) — dead_entities rebuild + global breaker policy

## Web/Docs Impact

None in this task — schema/grammar/config contracts only; `web/` types regenerate in task_07,
`packages/site` docs land in task_11 (config-toml rows drafted there).

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: DSL grammar additions are authoring surface; no manifest/hook changes here.
- Agent manageability: 20 new agent-mutable config paths (`compozy config set`) with
  `TestToolConfigPathPolicy` proof; `compozy loop validate` surfaces every new lint code.
- Config lifecycle: full R5 scope — structs, defaults, merge/overlay, validation, tool surface,
  tests; docs finalized in task_11.

## QA impact

None user-walkable yet (contracts only); behavior scenarios are flagged by tasks 02–11.

## Skills

`eng-code-guidelines`, `golang-pro`, `eng-schema-migration`, `eng-test-conventions`,
`testing-boss`, `eng-consolidate-test-suites`.

## References

- `.compozy/tasks/loop-ideas/analysis/behavior-defaults.md` (§compozy-v0 payload rules — the truth table for R2)
- `../compozy-v0/engine/task/tasks/shared/response_handler.go:144-231` — payload detection reference
- `.resources/sim/apps/sim/executor/utils/block-reference.ts:34-48` — available-fields diagnostics
- `.resources/temporal/common/retrypolicy/retry_policy.go:25-97` — retry policy shape

## Deliverables

- The single migration applied + `make codegen-check` green
- Classifier + payload detection + DSL grammar + lint + config packages compiling with full test coverage
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-001..UT-009 — payload-declared detection + result_contract + truncation
- [x] UT-010..UT-016 — authoring vs runtime diagnostics
- [x] UT-017..UT-021 — hint propagation + sanitization
- [x] UT-022..UT-028 — predicate policy + cost bounds (compile/validation level)
- [x] UT-156..UT-163 — config defaults/validation/overlay/tool-surface/resolution
- [x] UT-164..UT-169 — classifier truth tables, ordering, purity
- [x] UT-170..UT-176 — lint codes (blocking + warning), goal exclusions, round-trip
- [x] IT-032 — migration suites (fresh/reopen/ahead/integrity/equivalence)
- [x] IT-034 — CASCADE + claims horizon survival

## Success Criteria

- Every assigned test case implemented and passing
- `make gate` green on the affected lanes; migration suites cover all five creates + three rebuilds + events column
- Zero-annotation definitions lint clean; every new lint code fires on its minimal bad definition
