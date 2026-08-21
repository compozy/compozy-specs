---
status: completed
title: P4+P5 strategies, partial, progress namespace, iteration names
type: backend
complexity: critical
---

# Task 5: P4+P5 strategies, partial, progress namespace, iteration names

## Overview

Delivers the fan-out completion contract: the `strategy:` grammar (string shorthand + object form with the strict `StrategyThreshold` codec and the mandatory `missing: acceptable` coverage declaration), the strategy-aware join settlement (`wait_all`/`fail_fast`/`best_effort`/`race`) with monotonic admission and real per-lane cancellation (task_04's primitive), first-class partiality from the collect cell up to the run boundary (`completion_state`), the `progress.*` namespace, and `bind_as`/`index_as` iteration naming. P4+P5 merged: threshold conditions and progress are one slice.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. `strategy:` MUST land on the fan-out envelope with the dual-form codec (`fail_fast`/`race` shorthand; `{kind, threshold, missing}` object; `StrategyThreshold` round-trips exactly `"66%"` and `{count: N}` and rejects mixed/malformed forms); lint MUST enforce `strategy_coverage_undeclared`, `strategy_threshold_invalid`, and the 100%-≡-wait_all hint.
2. Join settlement MUST be a pure function over lane states honoring: definitive-failure-only triggers, deterministic tiebreaks (lowest item index), monotonic admission (late terminals → `late_arrival` only), completed lanes never rewritten, cancel intents delivered post-commit via the per-lane primitive, canceled-by-strategy causes distinct from failure.
3. Partiality MUST be first-class end to end: collect status `partial` + structured output `{total, succeeded, failed, canceled, coverage_rate, partial}`, and `loop_runs.completion_state (complete|partial)` set at terminal commit and propagated through run payloads, CLI, native tools, SSE `status_changed` — P4's owned migration (completion_state + `partial` status CHECK + `branch_pruned` event).
4. `nodes.<fanout>.progress.*` MUST be readable wherever the node id is referenceable, with the bare `progress.*` alias body-only (lint); counts settled-based with totals including unsettled; zero-collection rates = 0.
5. `bind_as`/`index_as` MUST un-shadow nested fan-outs with collision/reserved-root/scope lint (`iteration_name_conflict`).
6. Absence causes MUST be recorded: `branch_pruned` events with deciding cause, bounded aggregate payloads at width; single deterministic cause when skip and strategy-cancel overlap.
7. Docs co-ship: strategy/partial/progress/naming sections in `dsl-reference.mdx` + `failure-handling.mdx`, skill grammar rows, editor codec round-trip.
</requirements>

## Subtasks

- [x] 5.1 Grammar + codecs (`StrategySpec`, `StrategyThreshold`, `bind_as`/`index_as`) + lint table
- [x] 5.2 Pure join-settlement function (`control_join.go`) covering the four strategies with the invariants above
- [x] 5.3 Settlement → plan integration: cancel intents, collect output/status, `late_arrival`, `branch_pruned`
- [x] 5.4 P4 migration (completion_state, `partial` status, `branch_pruned`) + terminal-commit projection through every read surface
- [x] 5.5 `progress.*` namespace projection + alias scoping in `dsl/refs` + history builder
- [x] 5.6 Iteration-name scoping through fan-out materialization and namespace resolution
- [x] 5.7 Docs co-ship + editor codec/inspector data updates
- [x] 5.8 Implement all assigned tests; run focused task verification (the batch gate is deferred by the execution directive)

## Implementation Details

Reference `_spec.md` Part II — Strategy engine + Progress projection components, Data Models (completion_state, collect output), Safety Invariants 4/5/6(partial)/10, Key Decisions (progress scope rule), ADR-004.

### Relevant Files

- `internal/loop/dsl/graph.go`, `dsl/node_params.go` — envelope fields + specs
- `internal/loop/control_readiness.go:46-174` — the all-success barrier to replace with settlement (`control_join.go`)
- `internal/loop/fanout_materialization.go` — lane model + `bind_as` binding
- `internal/loop/control_namespace.go` (+`control_namespace_progress.go` new) — progress + alias scoping
- `internal/loop/coordinator_terminal_helpers.go` — completion_state at terminal commit
- `internal/loop/coordinator_cancellation.go` + task_04's per-lane delivery — strategy cancels
- `internal/store/globaldb/schema/definitions/50_loops.sql`, `global_db_loop_events.go`
- `internal/api/contract/loops.go`/`loop_runs.go` — completion_state + collect output payloads
- `web/src/systems/loops/lib/codec.ts`, `lib/loop-graph.ts` (`fanOutSummary`)

### Dependent Files

- `internal/loop/coordinator_generation_succession.go` — partial join on the terminal path feeds completion_state
- `internal/cli/loop_runs.go` status output; native `loop_status`/`loop_runs` output schemas
- `packages/site/content/docs/loops/{dsl-reference,failure-handling}.mdx`, `skills/compozy/references/loops.md`

### Related ADRs

- [ADR-004: Fan-out v2](adrs/adr-004.md) — strategies, quorum legitimacy, partial, cancellation-first ordering

### References

Read before implementing (evidence catalog: `analysis/compozy-v0.md`; barrier contract: `analysis/01_analysis_graph-eng-concepts.md` C6/C7):

- `../compozy-v0/engine/task/tasks/shared/parent_status_manager.go:259-353` — `wait_all | fail_fast | best_effort | race` implemented as **pure recomputation over persisted child states**, re-derived on every child terminal, written only on change — the direct source for our settlement function's shape
- `../compozy-v0/engine/task/progress.go:29-122` — the mirrored strategy arithmetic
- `../compozy-v0/engine/task/progress.go:128-201` + `../compozy-v0/engine/task/tasks/shared/progress_context_builder.go:37-52` + `../compozy-v0/engine/task/tasks/collection/context_builder.go:98` — the `.progress` field set (`total_children, success_count, failure_rate, completion_rate, …`) injected into the expression context — our `progress.*` namespace precedent
- `../compozy-v0/engine/task/config.go:1587-1626` + `../compozy-v0/engine/task/tasks/collection/expander.go:162-199` — `item_var`/`index_var` custom iteration names + parent-input back-fill — the `bind_as`/`index_as` precedent
- `../compozy-v0/examples/code-reviewer/workflows/review_per_file.yaml:96-111` — real authored usage (`item_var: file`, `batch: 3`) — the authoring feel to match
- Anti-pattern to avoid: `../compozy-v0/engine/worker/executors/container_helpers.go:256-276` — v0's `fail_fast`/`race` exited the first `Await` then a second `Await` waited for all children anyway (anti-lesson 9): our settlement MUST deliver per-lane cancels via task_04's primitive

## Web/Docs Impact

- `web/`: contract regeneration; S3 progress panel consumes in task_08 (fixtures possible here); codec round-trips new grammar.
- `packages/site`: strategy + partial + progress + naming docs; `configure.mdx` untouched (no config).
- QA impact: new user-visible behavior → add `untested` scenarios (best_effort partial visible end to end; fail_fast cancels) in `docs/qa/scenarios/`; walk before completion.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — no hook/manifest/bridge change (checked); effects triggers unchanged.
- Agent manageability: partiality + progress + causes readable via `loop status`/`loop runs`/SSE structured outputs.
- Config lifecycle: none — checked `[loops.*]` (strategy is authored-only; `fan_out_width` changes land in task_06).

## Deliverables

- Four strategies with sound settlement + real cancellation, first-class partiality run-wide, progress namespace, iteration naming — surfaced and documented
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] UT-008, UT-009 — iteration-name lint (collision, reserved, scope)
- [x] UT-012, UT-013 — strategy lint (coverage declaration, thresholds)
- [x] UT-030, UT-031, UT-032, UT-033, UT-034, UT-035, UT-036, UT-037, UT-037b, UT-038, UT-039, UT-040, UT-041, UT-042, UT-043, UT-044 — settlement semantics + completion_state projection
- [x] UT-050, UT-051, UT-052, UT-053, UT-054 — progress values, scope rule, nested naming
- [x] UT-113 — `StrategyThreshold` codec round-trip + rejections
- [x] IT-007 — fail_fast post-commit lane cancellation (acpmock observes; completed lane intact)
- [x] E2E-005 — fail_fast cancels running siblings, failure path taken
- [x] E2E-006 — best_effort partial: collect output + downstream branch + `completion_state: partial` at terminal
- [x] E2E-007 — race winner settles, losers canceled

## Success Criteria

- Every assigned test case implemented and passing
- A partial run shows `partial` on every surface (CLI status, HTTP payload, SSE, native tool output) — never plain complete
- Settlement is provably monotonic under the late-arrival tests
- `make gate` passes

## Completion Notes

- Focused race verification passed across the Loop runtime, GlobalDB, daemon, API, CLI, native-tool, task, and schema-codegen packages. The final focused runs passed 1,781 tests, plus the fresh/reopen migration subtest and the editor codec's 9 tests.
- Turbo `codegen-check`, web typecheck, the full web suite except its stale in-flight codec read, the fresh targeted codec rerun, and `git diff --check` passed. The batch `make gate`, live QA walks, screenshots, and full E2E reruns remain deferred by the controlling task-loop plan.
- QA tracker: `LP-best-effort-partial.md` and `LP-fail-fast-lane-cancel.md` are flagged `untested` for the batch QA phase.

Compozy Impact Audit:

- Native tools: extended Loop run schemas with `completion_state`; regenerated descriptors, OpenAPI types, enum catalogs, and native-tool fixtures. Tool IDs and capability gates are unchanged.
- Extensibility and hooks: strategy settlement reuses the coordinator intent and post-commit cancellation seams; checked extensions, hooks, skills/capabilities, registries, bridge SDKs, MCP sidecars, and `config.toml`. No new hook, resource, capability, or config key is required.
- Workspace data isolation: partial state, coverage outputs, progress projections, and `branch_pruned` events remain run/generation/node/item scoped under the owning workspace. CLI, HTTP, UDS, SSE, cache, and store reads continue to resolve through workspace-owned runs.
- Official Compozy skill: updated `skills/compozy/references/loops.md` for strategy grammar, progress, iteration names, and partial completion.
