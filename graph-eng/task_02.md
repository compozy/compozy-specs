---
status: pending
title: P1 router: route node + gate verdict routing
type: backend
complexity: high
---

# Task 2: P1 router: route node + gate verdict routing

## Overview

Delivers exclusive multi-way routing — the biggest structural gap the research found: the new `route` control node (ordered `routes: [{when, to}]` + mandatory `default`, first-match-wins, dominated-subgraph skip) and real gate-verdict routing via the `on_result` `{route: <node>}` object form, deleting the dead `branch` route action (B-4). Every routing decision lands in history with its cause and as a `route_taken` event.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. The `route` control kind MUST join the closed control enum with node-envelope fields `Routes []RouteSpec` + `Default NodeID`; lint MUST enforce `route_default_missing`, `route_target_invalid` (forward-only, declared nodes), condition compile/reference integrity, and duplicate-mapping errors.
2. Selection MUST be deterministic: declaration order breaks ties; `default` fires only on no-match; a broken condition is a fail-closed authoring failure (`predicate_evaluation_failed`) — never the default route (rides task_01's wired policy).
3. Non-selected routes MUST skip their dominated subgraphs via the existing skip machinery, recorded as `route_not_taken` (never failure), with per-lane scoping inside fan-out bodies.
4. Gates MUST route: `on_result: { <outcome>: { route: <node_id> } }` selects a pre-declared forward route; string actions keep working; the `branch` action string and its plumbing MUST be deleted (lint `route_action_removed`); approval outcomes keep their constrained-override legality.
5. `route_taken` MUST be a durable run event (`{route, cause, matched_when|default}`) in P1's owned migration (events CHECK rebuild), with the route cause visible in generation payloads (`route_causes`).
6. CLI/HTTP/native-tool surfaces are read-side only here (history/status/SSE) — no new verb; codegen + docs rows co-ship (`dsl-reference.mdx` control-node table + `on_result` vocabulary, `loops.md` grammar prose + route vocabulary, editor `on_result` option list).
</requirements>

## Subtasks

- [ ] 2.1 DSL grammar + linter table for `route` (default, targets, conditions, duplicates) and the `route_action_removed` code
- [ ] 2.2 Coordinator route planner (`control_plan_route.go`): ordered evaluation, cause recording, skip intents, lane scoping
- [ ] 2.3 Gate `{route:}` object form: parse, legality (incl. approval constraints), coordinator handling; delete `RouteBranch` + verdict `Branch` plumbing
- [ ] 2.4 P1 migration: events CHECK + `route_taken`; event payload builder + SSE typed frame
- [ ] 2.5 Editor codec + inspector option list updates (bijective round-trip of `routes`/`default`/`on_result` object)
- [ ] 2.6 Docs co-ship: site DSL reference + skill grammar rows
- [ ] 2.7 Implement all assigned tests; run `make gate`

## Implementation Details

Reference `_spec.md` Part II — System Architecture (Router row), Data Models (events), Delete Targets (branch action exact paths), ADR-003.

### Relevant Files

- `internal/loop/dsl/types_nodes.go`, `internal/loop/dsl/graph.go` — control enum + envelope fields
- `internal/loop/linter.go`, `internal/loop/linter_references.go` — new codes + route condition references
- `internal/loop/control_plan_branch.go` — the skip machinery to reuse (`skipBranchDependents`, `allBranchPathDependenciesSkipped`)
- `internal/loop/control_gate.go:202-204` — the no-op `RouteBranch` case to replace
- `internal/loop/gate/types.go:209-227`, `internal/loop/gate/routing.go` — action vocabulary + legality rows
- `internal/store/globaldb/global_db_loop_events.go` — event kind registration
- `internal/api/contract/loop_runs.go` — `route_causes` + typed event payload
- `web/src/systems/loops/lib/codec.ts`, `components/editor/loop-editor-field-effects.tsx` (on_result options)

### Dependent Files

- `internal/loop/control_readiness.go`, `control_topology.go` — route destinations as dependencies
- `internal/loop/generation_snapshot.go` — route causes in snapshots
- `packages/site/content/docs/loops/dsl-reference.mdx`, `skills/compozy/references/loops.md`

### Related ADRs

- [ADR-003: Router node and gate-verdict routing](adrs/adr-003.md) — mandatory default, object form, branch-action deletion

### References

Read before implementing (evidence catalog: `analysis/sim.md`, `analysis/compozy-v0.md`):

- `.resources/sim/apps/sim/executor/execution/edge-manager.ts:252-299` — `shouldActivateEdge`: success/error ports mutually exclusive — the activate-exactly-one discipline a route selection needs
- `.resources/sim/apps/sim/executor/execution/edge-manager.ts:15-108,199-230,301-388` — cascading edge deactivation guarded by `hasActiveIncomingEdges` so **a join reachable via a live path is never starved** (the exact rule our route-skip propagation must preserve, mirroring `allBranchPathDependenciesSkipped`)
- `../compozy-v0/engine/core/global.go:16-57` + `../compozy-v0/engine/task/tasks/shared/response_handler.go:296-400` — per-node `on_success/on_error {next, with}` transitions rendered at transition time (the routing-grammar precedent; note v0's mandatory-error-edge rule at `:248-253`)
- `../compozy-v0/engine/task/validators.go:14-79` — v0's cycle/type validators (the lint-table style for the new route codes)

## Web/Docs Impact

- `web/`: editor palette/inspector for `route`, on_result option list (S10 slice lands fully in task_08; this task keeps the codec bijective so the DSL view round-trips).
- `packages/site`: control-node table row, `on_result` vocabulary, route-cause prose in `running.mdx` history section.
- QA impact: new authoring behavior → add one `untested` scenario (author a router loop, observe route cause) in `docs/qa/scenarios/`; walk before completion.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked hooks (no new hook), extension manifests, bridge SDKs.
- Agent manageability: route causes readable via `compozy loop status -o json` / `compozy__loop_status`; new lint codes via `loop validate`.
- Config lifecycle: none — checked `[loops.*]`.

## Deliverables

- `route` node + gate `{route:}` shipping end to end with lint, planner, events, docs; `branch` action extinct repo-wide
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [ ] UT-001, UT-002, UT-003, UT-004, UT-005, UT-006, UT-007 — router/gate-route lint table
- [ ] UT-020, UT-021, UT-022, UT-023, UT-024, UT-025, UT-026, UT-027, UT-028, UT-029 — route planner semantics, gate object form, causes/events
- [ ] E2E-008 — router by classifier output + gate verdict reroute, causes in `loop status`

## Success Criteria

- Every assigned test case implemented and passing
- `grep -rn "RouteBranch\|\"branch\"" internal/loop/gate internal/loop/control_gate.go` shows no live branch-action path
- A no-match router provably cannot fail at runtime (lint blocks missing default; planner tests cover default cause)
- `make gate` passes
