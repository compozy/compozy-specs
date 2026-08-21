---
status: completed
title: P3 per-lane addressing, review gate, amend overlay
type: backend
complexity: critical
---

# Task 4: P3 per-lane addressing, review gate, amend overlay

## Overview

Delivers the P3 slice in its strict internal order: first the complete per-lane `(node_id, item_index)` mutation primitive (store identity through cancel/pause/resume/kill, post-commit session delivery, every operator/native surface), then the `review:` block on action nodes (pre-execution gating with approve/edit/reject/respond and exactly-once execution of the admitted params), then the `amend` verb as an **append-only overlay** over immutable generation outputs with the `amendments[]` public projection. The strategies task (05) consumes the per-lane primitive.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Per-lane addressing MUST land first and completely: `ItemIndex` through `CancellationMutation`/pause/resume/kill mutations + store rows + post-commit session cancellation delivery + `--item`/`item_index` on `loop node cancel|kill|pause|resume` across CLI/HTTP/UDS/native tools with codegen/docs — no surface before its primitive.
2. `review:` MUST gate at enqueue: the coordinator materializes resolved params, opens a `review` request (reusing task_03's request plane: previews + private `proposed_ref`), holds the `NodeRun` until admission, and executes exactly once with the admitted snapshot (original or validated edit); `when:` CEL gates review conditionally; `on_reject.route` or default `quality_rejection`; `respond` requires a declared output shape (lint).
3. Amend MUST be the overlay model: settled `loop_generation_outputs` rows are never mutated; one `loop_node_amendments` row per amendment (`amendment_seq` insert CAS); effective output = newest amendment else recorded, resolved at every read (namespace, resume, downstream); eligibility = settled output + paused/parked (`amend_not_parked`, `amend_no_output`, `amend_schema_missing`); `AmendInput` carries trusted Actor only (no request_id).
4. P3's owned migration MUST land `loop_node_amendments` + the `node_amended` event kind; amendment refs join the orphan-sweep roots; `amendments[]` ships in run detail across all read surfaces (status/HTTP/UDS/native) with inline redacted values bounded + size/hash fallback (no raw ref-read surface — stated).
5. `compozy loop node amend` + route + `compozy__loop_node_amend` (gated `loops.respond`) MUST co-ship with codegen, skill rows, and site docs.
</requirements>

## Subtasks

- [x] 4.1 Per-lane mutation primitive end to end (store identity, service inputs, post-commit delivery, all surfaces + docs)
- [x] 4.2 `review:` grammar + lint (decisions/schema pairing, `when`, on_reject reachability) on action nodes
- [x] 4.3 Coordinator review gating: resolved-params snapshot, request open, held enqueue, admitted-params execution, review-clock rule (pause never consumes the action's own clock)
- [x] 4.4 Respond decision plumbing for review (approve/edit/reject/respond) over task_03's dispatch
- [x] 4.5 P3 migration (`loop_node_amendments` + `node_amended`) + sweep-root extension + canonical suites
- [x] 4.6 Amend service (overlay resolution at every effective-output read) + surfaces + `amendments[]` projection
- [x] 4.7 Docs co-ship (review authoring section, amend operating section, CLI pages, skill rows)
- [x] 4.8 Implement all assigned tests; run focused task verification (the batch gate is deferred by the execution directive)

## Implementation Details

Reference `_spec.md` Part II — Request plane + Data Models (amendments, blob roots), Core Interfaces (ReviewSpec, AmendInput note), Safety Invariants 3/5/9, ADR-001.

### Relevant Files

- `internal/loop/cancel_control.go:12-22` — `CancellationMutation` gains `ItemIndex`
- `internal/loop/service_node_pause.go`, `node_pause.go`, `node_requeue.go`, `coordinator_cancellation.go` — per-lane plumbing + delivery
- `internal/loop/dsl/graph.go` — `Review *ReviewSpec` envelope field; `dsl/node_params.go` — `ReviewSpec`/`RejectPolicy`
- `internal/loop/control_readiness.go:190-256` — the enqueue point review holds
- `internal/loop/coordinator_outputs.go` — effective-output resolution seam
- `internal/loop/control_namespace.go`, `control_namespace_history.go` — overlay-aware reads
- `internal/api/contract/loop_nodes.go` (`item_index` on requests), `loop_runs.go` (`amendments[]`)
- `internal/cli/loop_node.go`, `internal/tools/builtin/loops.go` + `loops_lifecycle_schemas.go`
- `internal/store/globaldb/schema/definitions/50_loops.sql`, sweep query

### Dependent Files

- `internal/loop/action.go` / `action_runagent.go` — reviewed action consumes the admitted snapshot
- `internal/loop/quarantine.go` history rendering — amended cells visible
- `openapi/compozy.json` + generated TS + CLI docs

### Related ADRs

- [ADR-001: Human interaction model](adrs/adr-001.md) — edit-before-execute, respond-as-result, amend as operator repair verb

### References

Read before implementing (evidence catalog: `analysis/temporal.md`, `analysis/sim.md`; the four-decision gate: `analysis/02_analysis_graph-frameworks.md` §3.2 Gen 3):

- `.resources/temporal/service/history/workflow/activity.go:245-520` — per-activity pause (running attempt drains, scheduled one stamped out), reset variants, unpause modes, **pause provenance (who/why, manual or rule)** — the per-lane verb + provenance precedent
- `.resources/temporal/service/history/workflow/activity.go:267-276,315-350` + `.resources/temporal/service/history/hsm/tasks.go:24-62` — monotonic stamps invalidating stale scheduled work (`ValidateNotTransitioned`) — the fencing style behind the amendment-seq CAS and per-lane cancel delivery
- `.resources/temporal/common/effect/buffer.go` (+ `docs/architecture/effect-package.md`) — commit-gated effect buffer: success callbacks only after persistence — the post-commit cancellation-delivery discipline
- `.resources/sim/apps/sim/executor/human-in-the-loop/utils.ts:12-73` — per-lane pause contexts (review requests inside fan-out lanes)
- `../compozy-v0/engine/llm/orchestrator/response_handler.go:58-165` + `../compozy-v0/engine/llm/orchestrator/loop_state.go:27-35` — validation-feedback re-ask with a bounded finalize budget (the reject-with-feedback loop shape for review)
- Anti-pattern to avoid: `../compozy-v0/engine/worker/executors/container_helpers.go:256-276` — early-exit that waits for everyone anyway (anti-lesson 9); our per-lane primitive exists precisely so task_05's strategies genuinely cancel

## Web/Docs Impact

- `web/`: contract regeneration (`amendments[]`, `item_index`); S1 review card + S5 amend dialog land in task_08 (fixtures become possible here).
- `packages/site`: review authoring section, amend operating section, `node/amend` CLI page, `running.mdx` row.
- QA impact: new user-visible behavior → add `untested` scenarios (review edit-then-execute; amend-then-rerun pairing) in `docs/qa/scenarios/`; walk before completion.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none new — request plane reused; no hook/manifest/bridge change (checked).
- Agent manageability: amend verb ×3 surfaces gated `loops.respond`; review decisions via the task_03 respond surface; per-lane `--item` on all node verbs.
- Config lifecycle: none — checked `[loops.*]` (review/amend carry no config).

## Deliverables

- Per-lane primitive, review gating with exactly-once admitted execution, append-only amend overlay with public history — all surfaced and documented
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] UT-011 — review `respond` without output shape lint
- [x] UT-057, UT-058, UT-059, UT-060, UT-061 — review admission (approve/edit/reject/respond), clock rule
- [x] UT-099, UT-100, UT-101, UT-102, UT-103, UT-112 — amend overlay, guards, seq CAS, append-only provenance
- [x] IT-006 — review edit admission → enqueue carries the edited snapshot; executed = admitted
- [x] IT-011 — amend → resume → downstream consumes effective output; `amendments[]` parity across surfaces
- [x] IT-017 — blob durability roots (request + amendment refs survive the sweep; true orphans reclaimed)
- [x] E2E-002 — review flow with edit, executes exactly once with edited args
- [x] E2E-003 — reject routes via `on_reject.route` without executing
- [x] E2E-004 — respond-as-result substitutes execution

## Success Criteria

- Every assigned test case implemented and passing
- No `--item`/`item_index` surface exists without the complete primitive behind it (per-lane suite proves delivery)
- `loop_generation_outputs` rows are byte-identical before/after amend in the overlay tests
- `make gate` passes

## Completion Notes

- Focused race verification passed across the Loop runtime, GlobalDB, daemon, API, CLI, and native-tool packages. A fresh affected-package rerun passed 2,265 tests after the final fixes.
- `make codegen-check` and `git diff --check` passed. The batch `make gate`, live QA walks, and screenshots remain deferred by the controlling task-loop plan.
- QA tracker: `LP-review-edit-execute.md` and `LP-amend-rerun.md` are flagged `untested` for the batch QA phase.

Compozy Impact Audit:

- Native tools: added `compozy__loop_node_amend`; extended lifecycle tools with `item_index`; extended `compozy__loop_respond` with review decisions; regenerated descriptors, catalog, schemas, capability gates, and native-tool tests.
- Extensibility and hooks: reused the request plane, action registry, output-blob store, and `loops.respond` capability. Checked extension manifests, hooks, bridge SDKs, MCP sidecars, and `config.toml`; no new hook, extension resource, bridge contract, or config key is required.
- Workspace data isolation: amendments and review requests are workspace-scoped, with `workspace_id` enforced through run joins on list/read/write and overlay paths. Lane mutations remain run/generation/node/item scoped; HTTP, UDS, CLI, native tools, events, and output reads preserve that identity.
- Official Compozy skill: updated `skills/compozy/references/loops.md` for review decisions, per-lane lifecycle control, and amendments.
