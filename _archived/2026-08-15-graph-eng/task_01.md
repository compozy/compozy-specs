---
status: completed
title: P0 cleanup: hash_fields deletion, predicate policy, per-gate counters
type: backend
complexity: medium
---

# Task 1: P0 cleanup: hash_fields deletion, predicate policy, per-gate counters

## Overview

Delivers the spec's P0 cleanup batch: the dead `no_progress.hash_fields` knob is deleted everywhere (zero-legacy, B-1), the fully-written-but-unwired predicate failure policy gains its grammar (`on_eval_error` on nodes, `StopWhenSpec` string-or-object on the contract), its three call sites, and CEL cost observability (B-2/B-3), and gate revision budgets become per-gate/per-lane persisted counters instead of generation arithmetic (B-5). This unblocks the router (task_02) whose conditions rely on the wired fail-closed semantics.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. `hash_fields` MUST be deleted from the DSL, the public API contract, and every fixture/test — grep-to-zero across `internal/` and regenerated artifacts; lint MUST report `unknown_parameter` when authored (Part II Delete Targets lists exact paths, including the site `dsl-reference.mdx` contract block and any `skills/compozy/references/loops.md` mention — co-ship in this task).
2. `ApplyPredicateFailurePolicy` MUST gain its three call sites (branch condition, stop_when, watch-events/fan-out filter): continuation predicates fail open (broken `stop_when` exits with diagnostic `predicate_evaluation_failed`, never blocks succession), routing predicates fail closed (routable `authoring` failure).
3. `contract.stop_when` MUST become the strict string-or-object `StopWhenSpec{expr, on_eval_error}` (scalar keeps today's shape + fail-open default); nodes with conditions MUST accept `on_eval_error: fail | exit`. Both codecs are strict closed decoders (RuntimeSpec pattern) and round-trip through the editor codec.
4. CEL cost MUST be observable: the ≥80% warning and cost-limit exhaustion surface in run diagnostics instead of being discarded at the three eval sites.
5. Gate `max_revisions` MUST count per `(gate, item_index)` from persisted counters in `loop_node_controls.gate_revisions_json` (P0-owned migration: column add + declarative fragment + `atlas.sum`/sqlc via `make codegen` + canonical fresh/reopen/ahead/integrity/equivalence extensions); operator-origin generations MUST NOT consume authored budgets.
6. The removed generation-derived revision computation (`GateInput.Revision = generation-1`) MUST be deleted, not bypassed.
</requirements>

## Subtasks

- [x] 1.1 Delete `hash_fields` end to end (grammar, contract mirror, fixtures, docs rows, codegen) with the `unknown_parameter` lint proof
- [x] 1.2 Add `StopWhenSpec` + node `on_eval_error` grammar with strict dual-form codecs and linter coverage
- [x] 1.3 Wire `ApplyPredicateFailurePolicy` at the three eval sites with the fail-open/fail-closed defaults and per-site overrides
- [x] 1.4 Surface predicate cost warnings and cost-limit failures into run diagnostics
- [x] 1.5 Author the P0 migration (`gate_revisions_json`) with canonical suite extensions
- [x] 1.6 Replace the revision computation with per-gate/per-lane counters and the exhaustion behavior
- [x] 1.7 Update `packages/site/content/docs/loops/dsl-reference.mdx` + `skills/compozy/references/loops.md` for the grammar deltas (co-ship)
- [x] 1.8 Implement all assigned tests; run focused checkpoint validation (`make gate` deferred by the accepted loop instructions)

## Implementation Details

Reference `_spec.md` Part II — Implementation Design (StopWhenSpec block), Data Models (gate counters), Impact Analysis (Delete Targets with exact paths), ADR-005.

### Relevant Files

- `internal/loop/dsl/contract.go` — `NoProgress.HashFields` deletion; `StopWhenSpec` home
- `internal/api/contract/loops.go` — `NoProgress` mirror deletion
- `internal/loop/predicate_policy.go` — the dead policy to wire (both defaults already implemented)
- `internal/loop/dsl/refs/condition.go` — cost tracking (`CostWarningThresholdReached`) currently discarded
- `internal/loop/control_plan_branch.go:46`, `internal/loop/control_stop_when.go:41`, `internal/loop/coordinator_watch_events.go:255` — the three eval sites calling `Program.Eval` directly
- `internal/loop/control_gate.go:156` — the generation-derived revision to delete
- `internal/store/globaldb/schema/definitions/50_loops.sql` — `loop_node_controls` fragment for the counter column
- `internal/loop/linter_lifecycle.go`, `internal/loop/linter.go` — lint tables for the new/removed grammar

### Dependent Files

- `internal/loop/coordinator_generation_succession.go:33-38` — broken `stop_when` currently blocks succession; behavior inverts here
- `internal/loop/gate/routing.go`, `internal/loop/coordinator_gate_evaluations.go` — consume the new counters
- `web/src/systems/loops/lib/loop-dsl.ts` + editor codec — round-trip the new `stop_when` object form
- `openapi/compozy.json`, `web/src/generated/compozy-openapi.d.ts` — regenerated

### Related ADRs

- [ADR-005: Bug absorption](adrs/adr-005.md) — B-1/B-2/B-3/B-5 decisions, delete-not-wire for `hash_fields`

### References

Read before implementing (evidence catalog: `analysis/compozy-v0.md`, `analysis/sim.md`):

- `../compozy-v0/engine/task/cel_evaluator.go:42-176` — the bounded-cost CEL pattern this task completes: `cel.CostLimit` + `OptTrackCost`, 80% warning threshold, compiled-program cache, `ValidateExpression()` (idea 36's source; our B-3 fix surfaces exactly this warning)
- `.resources/sim/apps/sim/executor/orchestrators/loop.ts:39,41-56,704-781` — loop conditions evaluated in a sandbox where a broken predicate → `false` → **the loop exits rather than hangs** (the fail-open precedent for `stop_when`)
- `.resources/sim/apps/sim/executor/handlers/condition/condition-handler.ts:19,26-79,204-243` — condition blocks deliberately **throw** so the error port catches a bad predicate (the fail-closed precedent for routing conditions); two opposite policies, both intentional — exactly our split default
- `../compozy-v0/engine/core/global.go:156-235` — hierarchical timeout/retry merge with named defaults (the per-node override merge style `on_eval_error` follows)

## Web/Docs Impact

- `web/`: editor DSL view round-trips `stop_when` object form (codec tables); no new UI surface.
- `packages/site`: `docs/loops/dsl-reference.mdx` (contract block loses `hash_fields`, gains `stop_when` object + `on_eval_error`); generated CLI docs unaffected.
- QA impact: user-visible behavior change (broken `stop_when` now exits; lint errors change) → add one `untested` content-addressed scenario for the predicate-failure behavior in `docs/qa/scenarios/`; walk before completion.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked extension manifests, hooks, bridge SDKs (grammar-internal change).
- Agent manageability: `compozy loop validate` surfaces the new lint codes (structured output already).
- Config lifecycle: none — checked `[loops.*]` (no key touched in P0).

## Deliverables

- Zero `hash_fields` references repo-wide; wired predicate policy with correct defaults; observable CEL cost; per-gate/lane revision counters on a P0-owned migration; docs rows updated
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-015 — `hash_fields` → `unknown_parameter` + fixture sweep
- [x] UT-090, UT-091, UT-092, UT-093 — per-gate/per-lane revision counters, exhaustion, operator-origin exclusion
- [x] UT-094, UT-095, UT-096, UT-097, UT-098 — predicate fail-open/fail-closed, `StopWhenSpec` codec, overrides, cost warning + limit
- [x] E2E-009 — broken `stop_when` exits the run with diagnostic instead of iterating

## Success Criteria

- Every assigned test case implemented and passing
- `grep -r hash_fields internal/ web/ packages/ skills/` returns zero production hits; `make codegen-check` clean
- A run with a broken `stop_when` reaches its completion path with the diagnostic recorded (no wedge)
- Two gates in one loop consume independent revision budgets in the counter tests
- `make gate` passes

## Completion Notes

- Deleted `hash_fields` from runtime, contracts, fixtures, docs, examples, and generated artifacts; the only remaining authored occurrence is the UT-015 negative lint proof.
- Wired strict predicate failure policy and durable cost diagnostics across branch routing, `stop_when`, and watch-event filters. Broken continuation predicates exit successfully by default; broken routing predicates remain fail-closed and honor `on_error` routing.
- Replaced generation-derived gate revisions with persisted counters scoped by gate and item lane. Migration `00065_schema.sql`, declarative schema, sqlc output, and fresh/reopen/upgrade tests co-ship.
- Web/Docs Impact: the existing web codec and inspect view understand object-form `stop_when`; loop DSL, guardrail, example, and official skill references document the contract. No new UI surface was introduced.
- QA tracker impact: added `docs/qa/scenarios/LP-broken-stop-when-policy.md` as `untested`; execution is deferred to the loop QA tail by the accepted workflow instructions.
- Verification: focused Go race tests for UT-015 and UT-090–UT-098 passed; E2E-009 and the migration fresh/reopen/upgrade suites passed with integration tags and race detection; focused web tests passed (19/19); web/site typechecks and `make codegen-check` passed. The full GlobalDB package previously exceeded its ten-minute suite timeout without a task assertion failure; the task-owning GlobalDB tests pass. `make gate` remains intentionally deferred.

Compozy Impact Audit:

- Native tools: no impact — checked loop tool IDs, toolsets, descriptors, schemas, capability gates, and CLI/API fallback surfaces; this task changes the loop DSL/runtime contract, not native-tool availability.
- Extensibility and hooks: loop extension fixtures were updated to the strict grammar; hooks, bridge SDKs, MCP sidecars, registries, capabilities, and `config.toml` loop settings were checked and remain unchanged. The official loop skill reference documents the new public grammar.
- Workspace data isolation: `gate_revisions_json` is session-scoped through the existing workspace-owned loop run and node-control records; reads and mutations retain the existing `workspace_id`-scoped run lookup, and the E2E SSE assertion proves foreign workspaces receive no events. No global cache or cross-workspace list path was added.
- Official Compozy skill: updated `skills/compozy/references/loops.md` for `StopWhenSpec`, `on_eval_error`, predicate diagnostics, and per-gate revision semantics; the skill entry point and unrelated references remain unchanged.
