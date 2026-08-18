---
status: pending
title: P7+P8 time travel: diff, rerun, fork + cross-cutting suites
type: backend
complexity: critical
---

# Task 7: P7+P8 time travel: diff, rerun, fork + cross-cutting suites

## Overview

Delivers operator time travel end to end: the daemon-side diff service (generation↔generation, run↔run with the changed/rerun/skipped/carried/verdict vocabulary), `rerun` from a healthy node (new generation, origin `operator_rerun`), and `fork` as a new linked run through the canonical start transaction with the pre-settled seed generation 1 (`fork_seed`) and first executing generation 2 — plus the `loop_timetravel_ops` intent-keyed idempotency ledger, the `loops.timetravel` capability, and lineage projections. As the last backend slice it also closes the cross-cutting suites: seven-operation transport parity, the capability matrix, workspace-isolation negatives, eight-event SSE replay, and the full migration equivalence pass.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. **Two owned migrations, in order**: P7's (`loop_timetravel_ops` + origins CHECK adding `operator_rerun`) then P8's (`loop_runs` fork-lineage columns + origins CHECK adding `fork_seed` + events CHECK adding `run_forked`) — each with declarative fragments, `atlas.sum`/sqlc, canonical suites; never merged or reordered.
2. Diff MUST be a daemon-side read (no capability): change-kind rows matching `_dx.md` exactly, carried marking, size+hash summaries for oversized outputs, run↔run with input rows + definition-divergence banner + live-side as-of labeling, same-loop and same-workspace only (`diff_cross_loop`).
3. Rerun MUST generalize the requeue planner: rerun set = node + transitive dependents (parked excluded), parent = latest settled generation, origin `operator_rerun`, guards `rerun_node_unsettled`/`rerun_busy`, per-lane `--item`.
4. Fork MUST implement the exact start transition from Part II Key Decisions: child inserted at `generation=1` with pre-settled seed rows + blob copies in the same transaction, `(1,0,fork_seed)` intent, generation-2 coordinator reservation `(2,1,initial)`, concurrency policy as a fresh start (queued forks seed at insert, reserve on promotion), both-or-neither best projection, boot-reconcile compatibility, fork-of-fork, missing-blob whole-transaction abort, source resolved strictly in the caller workspace.
5. Idempotency MUST identify intent: explicit `request_id` → replay identity (partial unique index; digest mismatch → `timetravel_key_reuse`); omitted → fresh operation with its own ledger row; `rerun_busy`/concurrency policy absorb keyless rapid duplicates.
6. `ResponderPolicy` MUST gate rerun/fork self-operation only while the source run executes; `loops.timetravel` gates agent rerun/fork on all three surfaces; operators ungated.
7. All surfaces co-ship: `loop diff|rerun|fork` CLI (+`--request-id`), HTTP/UDS routes, `compozy__loop_diff|loop_rerun|loop_fork` (+`request_id?`), lineage (`forked_from`/`forks[]`) in run payloads, codegen, skill rows, site docs (`running.mdx` rows + a time-travel docs page).
8. The cross-cutting suites MUST pass here: IT-001 (full migration set), IT-004 (7-op parity), IT-012 (8 events), IT-013 (capability matrix incl. diff-ungated), IT-016 (workspace negatives incl. lineage/diff/SSE/bell counts).
</requirements>

## Subtasks

- [ ] 7.1 P7 migration + `loop_timetravel_ops` ledger with the intent-keyed idempotency rules
- [ ] 7.2 Diff service (`timetravel_diff.go`) + payloads + CLI/HTTP/UDS/tool surfaces
- [ ] 7.3 Rerun planner (`coordinator_rerun.go`) generalizing requeue + guards + provenance + surfaces
- [ ] 7.4 P8 migration + fork seed/start transition through the canonical start path + lineage projections
- [ ] 7.5 Capability `loops.timetravel` + `ResponderPolicy` executing-only rule for rerun/fork
- [ ] 7.6 Docs co-ship (time-travel page, CLI pages, skill rows, triple-table rows)
- [ ] 7.7 Cross-cutting suites: IT-001/IT-004/IT-012/IT-013/IT-016 green over the whole backend
- [ ] 7.8 Implement all remaining assigned tests; run `make gate`

## Implementation Details

Reference `_spec.md` Part II — Time travel component, Data Models (ledger, fork transition, origins), Core Interfaces (ForkInput/RerunInput/ResponderPolicy), Safety Invariants 7/8, Key Decisions (fork start transition, diff daemon-side), ADR-002.

### Relevant Files

- `internal/loop/service_start.go`, `internal/store/globaldb/global_db_loop.go` — the canonical start transaction fork extends
- `internal/loop/coordinator_requeue.go:11-88` — the requeue planner rerun generalizes
- `internal/loop/coordinator_succession.go:242-324` — ratchet-seed machinery informing the seed write
- `internal/loop/generation_intent.go` — origins + intent validation (`initial` parenting the seed)
- `internal/loop/control_namespace_history.go:69-139` — unmodified reader the seed satisfies (prove with tests)
- `internal/loop/loop_api_run_history.go` (daemon) — lineage + diff read substrate
- `internal/api/contract/loops.go`/`loop_runs.go`, `internal/api/core/loops.go`, `httpapi/loops_routes.go` + UDS
- `internal/cli/loop.go` (+ `loop_timetravel.go` new), `internal/tools/builtin/loops.go`
- `internal/store/globaldb/schema/definitions/50_loops.sql` + reconcile queries (boot compatibility)

### Dependent Files

- `internal/loop/executed_definition_snapshot.go` — fork pins the source digest
- `internal/loop/coordinator_generation_reattempt.go` — rerun-set arithmetic reuse
- `openapi/compozy.json`, generated TS/CLI docs, `skills/compozy/references/loops.md`, `packages/site/content/docs/loops/`

### Related ADRs

- [ADR-002: Operator time travel](adrs/adr-002.md) — split rerun/fork, seed model (3b), phase-owned origin rebuilds
- [ADR-006: Bell integration](adrs/adr-006.md) — cross-workspace bell-count isolation proven in IT-016

### References

Read before implementing (evidence catalog: `analysis/sim.md` mechanisms 3/11, `analysis/temporal.md` mechanisms 8/14; append-only fork semantics: `analysis/02_analysis_graph-frameworks.md` §1.1.13):

- `.resources/sim/apps/sim/executor/utils/run-from-block.ts:76-139,156-239` — `computeExecutionSets`: BFS-downstream dirty set + preserved reachable-upstream set, and `validateRunFromBlock` refusal rules (sentinels, in-loop nodes, unexecuted upstreams) — the rerun-set + guard precedent
- `.resources/sim/apps/sim/executor/execution/executor.ts:128-266` + `.resources/sim/apps/sim/executor/orchestrators/node.ts:57-75` — snapshot filtering, severed non-dirty incoming edges so joins don't wait forever, non-dirty short-circuit to cached output — carried-vs-rerun arithmetic
- `.resources/sim/apps/sim/lib/logs/execution/snapshot/service.ts:27-103` + `.resources/sim/packages/db/schema.ts:373-391,393-484` — content-addressed graph snapshots keyed `(workflow_id, state_hash)` per run — the pinning/replay-read precedent behind diff and fork-pins-source-digest
- `.resources/sim/apps/sim/app/api/logs/execution/[executionId]/route.ts:107-173` — the replay endpoint returning the graph as-it-was with run spans overlaid (diff/inspection read shape)
- `.resources/temporal/service/history/workflow/retry.go:156-369` — continue-as-new: fresh run inheriting input, policy, parent/root linkage, `LastCompletionResult` — the carry-contract precedent for the fork seed
- `.resources/temporal/service/history/api/resetworkflow/api.go:233-255` + `.resources/temporal/service/history/ndc/events_reapplier.go:52-80` + `.resources/temporal/service/history/workflow/mutable_state_impl.go:4102-4114` — reset-to-checkpoint with reapply policy and up-front resettability validation — fork guards + whole-transaction-abort precedent
- `.resources/temporal/service/history/workflow/mutable_state_impl.go:2867-2889` — `ContinueAsNewMinBackoff` (tight-loop prevention; informs keyless rapid-duplicate posture)
- `.resources/sim/apps/sim/executor/errors/boundary.ts:6-54` — typed fail-closed error boundary with opaque `ref` (the deterministic-rejection style for missing-blob/foreign-workspace)

## Web/Docs Impact

- `web/`: contract regeneration (diff payloads, lineage, ledger errors); S6/S7/S8 surfaces land in task_08 with these fixtures.
- `packages/site`: new time-travel docs page (Operate section), CLI pages `diff|rerun|fork`, `running.mdx` rows.
- QA impact: major new behavior → add `untested` scenarios (rerun-from-node; fork with overrides; diff read) in `docs/qa/scenarios/`; walk before completion.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none new — no hook (start dispatches existing `loop.started`), no manifest/bridge change (checked); skill rows updated.
- Agent manageability: three verbs ×3 surfaces; diff ungated read; rerun/fork gated `loops.timetravel` with the executing-only self rule; deterministic errors per `_dx.md`.
- Config lifecycle: none — checked `[loops.*]` (time travel carries no config by decision).

## Deliverables

- Diff/rerun/fork shipping end to end with the ledger, capability, lineage, and docs; the five cross-cutting suites green over the complete backend
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [ ] UT-070, UT-071, UT-072, UT-073, UT-074, UT-075, UT-076, UT-077 — rerun planner, guards, intent-keyed idempotency
- [ ] UT-078, UT-079, UT-080, UT-081, UT-082, UT-082b — fork seed, overrides, lineage, blob abort, canonical start transition
- [ ] UT-083, UT-084, UT-085, UT-086, UT-087, UT-088, UT-089 — diff kinds, carried, summaries, run-diff, guards
- [ ] UT-109 — origins (`operator_rerun`, `fork_seed`) + eight event kinds enum parity
- [ ] IT-001 — full migration set: fresh/reopen/ahead/integrity/equivalence incl. live-run reopen
- [ ] IT-004 — seven-operation HTTP=UDS byte parity
- [ ] IT-009 — fork snapshot isolation while source live; concurrency policy
- [ ] IT-010 — keyed replay vs keyless fresh-operation semantics
- [ ] IT-012 — eight event kinds append + SSE replay/resume + contract/listener lockstep
- [ ] IT-013 — capability matrix (respond/timetravel grants, diff ungated, self-denials via shared policy)
- [ ] IT-016 — cross-workspace negatives (seven ops, lineage, diff, SSE, `aggregates.pending`)
- [ ] E2E-010 — rerun on a terminal run: `operator_rerun` generation, carried nodes intact, completes
- [ ] E2E-011 — fork with overrides: linked run, two-way lineage, source byte-identical
- [ ] E2E-014 — agent native-tools journey: requests→respond granted; self-response and ungated timetravel denied

## Success Criteria

- Every assigned test case implemented and passing
- A fork's generation 2 reads `previous.*`/`best.*` from the seed via the **unmodified** history reader (asserted, not assumed)
- Source runs are byte-identical after any fork (hash comparison in E2E-011)
- `make gate` passes (final backend gate)
