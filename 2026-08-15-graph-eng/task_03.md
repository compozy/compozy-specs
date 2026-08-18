---
status: pending
title: P2 requests core: ask node, respond plane, ResponderPolicy
type: backend
complexity: critical
---

# Task 3: P2 requests core: ask node, respond plane, ResponderPolicy

## Overview

Delivers the human-request plane on the shipped wait-resume machinery: the `ask` control node (prompt/context/expect/responders/expires), the `loop_requests` side table with its private-ref vs public-preview split, the full read/write operation set — list, **detail (full-context read)**, respond — across CLI/HTTP/UDS/native tools, the reminder→escalation→expiry ladder, atomic request cancellation, and the shared `ResponderPolicy` trust boundary with the `loops.respond` capability. This is the substrate every later HITL surface (review, amend, bell) builds on.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. `ask` MUST join the closed control enum with `AskParams{prompt, context, expect (required — lint), responders, expires}`; the answer becomes the node output, compile-checked via `produces` derivation from `expect`.
2. P2's owned migration MUST land `loop_requests` (exact columns per Part II Data Models: private `*_ref` vs public `*_preview_json` split, per-decision schema columns, both preview redaction/bounds applied at open), the `request` wait kind in the waits CHECK, and the four request event kinds.
3. The operation set MUST ship complete on all three surfaces in this task: `compozy loop requests` (+pinned ordering/cursors, `aggregates.pending`), `compozy loop request` (detail, full redacted context), `compozy loop respond` (all decision plumbing for ask; review decisions arrive in task_04 but the decision vocabulary/validation dispatch is built here), plus HTTP/UDS routes and `compozy__loop_requests`/`compozy__loop_request`/`compozy__loop_respond` — with `make codegen`, official-skill rows, and site docs in the same change (co-ship rule).
4. Admission MUST be single-winner epoch-CAS on the parked cell; expiry MUST fire exactly once (scheduler + boot backstops, shared idempotency key); cancellation of the cell (node/run cancel/kill, route skip, later strategy cancel) MUST transition the request to `canceled` in the same transaction with the `request_canceled` event; answer/expiry/cancel tiebreak = transaction order.
5. `ResponderPolicy` MUST be implemented in `daemon/` over trusted `task.ActorContext` + durable spawn lineage (fail-closed), injected into `internal/loop`, and consumed by respond AND the existing approve path (one policy); `responders.agents: allow|deny` per node; `loops.respond` capability registered.
6. Request blob refs MUST become orphan-sweep roots; `loops.defaults.<kind>.requests.expire_after` MUST land with full config lifecycle (struct, defaults, overlay, clone, validation, example, site config docs, round-trip tests).
7. Deterministic errors per `_dx.md`: `request_validation_failed` (422, field details), `request_already_answered` (409 + recorded decision), `request_expired`/`request_canceled` (410), `respond_not_permitted`/`respond_self_denied` (403).
</requirements>

## Subtasks

- [ ] 3.1 DSL `ask` grammar + lint (expect required, responders enum, expiry reachability/durations) + produces derivation
- [ ] 3.2 P2 migration (`loop_requests` + wait-kind + 4 event kinds) + canonical suite extensions + sweep-root extension for request refs
- [ ] 3.3 Coordinator: ask open (render+freeze prompt/context previews + refs), park as `request` wait, escalation/expiry ladder, atomic cancel coupling
- [ ] 3.4 Service: `ListRequests` (ordering/cursor/aggregates), `GetRequest`, `Respond` (per-decision schema dispatch, single-winner admission, provenance with `answered_at`)
- [ ] 3.5 `ResponderPolicy` (daemon adapter + injection) unified across approve + respond; self-denial over the spawn chain
- [ ] 3.6 Surfaces ×3 + capability `loops.respond` + codegen + `loops.md` tool-table rows + site docs (`running.mdx` triple-table rows, new authoring section)
- [ ] 3.7 Config lifecycle for `requests.expire_after`
- [ ] 3.8 Implement all assigned tests; run `make gate`

## Implementation Details

Reference `_spec.md` Part II — Request plane component, Data Models (`loop_requests`, blob roots), API Endpoints (ordering, error mapping), Core Interfaces (`RespondInput`, `ResponderPolicy`), Safety Invariants 1/2/12/13, ADR-001.

### Relevant Files

- `internal/loop/dsl/types_nodes.go`, `dsl/node_params.go`, `dsl/wait.go` — grammar homes (`AskParams`, `ResponderSpec`, reuse `WaitExpiry`)
- `internal/loop/coordinator_wait.go`, `node_waits.go`, `coordinator_gate_wait.go` — the wait plane to extend (`request` kind; escalation ladder pattern)
- `internal/loop/service_node_requeue.go` (`ResumeNodeWait`) — the schema-validated admission pattern to generalize
- `internal/loop/action_schema.go` — JSON-Schema validation reuse
- `internal/daemon/scheduler_loop_due_scan.go` — expiry/escalation backstops
- `internal/store/globaldb/schema/definitions/50_loops.sql` + `global_db_loop_events.go` — migration + kinds
- `internal/api/contract/loop_runs.go`, `internal/api/core/loops.go`, `internal/api/httpapi/loops_routes.go` + UDS registration — payloads/handlers/routes
- `internal/cli/loop.go` (+ new `loop_requests.go`), `internal/tools/builtin/loops.go` + `builtin_ids.go` — verbs/tools/capability
- `internal/config/loops.go`, `merge_loops.go`, `defaults.go`, `config_clone_loops.go` — config lifecycle
- Orphan sweep query (`SweepOrphanedLoopOutputBlobs` in `internal/store/globaldb/`) — root-set extension

### Dependent Files

- `internal/loop/coordinator_cancellation.go`, `coordinator_error_routes.go` — atomic request-cancel coupling
- `internal/loop/hooks.go` consumers — none (no new hook; verified)
- `openapi/compozy.json`, `web/src/generated/compozy-openapi.d.ts`, generated CLI docs

### Related ADRs

- [ADR-001: Human interaction model](adrs/adr-001.md) — one plane, responder rules, escalation, bridges notify+deep-link
- [ADR-006: Bell integration](adrs/adr-006.md) — `aggregates.pending` is the count the bell will compose (task_09)

### References

Read before implementing (evidence catalog: `analysis/sim.md`, `analysis/temporal.md`; framework mechanics: `analysis/02_analysis_graph-frameworks.md` §3 HITL):

- `.resources/temporal/service/history/workflow/update/state.go:12-25` — validated-update lifecycle (`Admitted → Sent → Accepted → Completed` + `Aborted`) — the request-state model precedent
- `.resources/temporal/service/history/workflow/update/abort_reason.go:35-104` — the hand-written (abort-reason × state) outcome matrix so retries can't duplicate delivery — the answered/expired/canceled tiebreak + idempotent-echo precedent
- `.resources/sim/apps/sim/executor/human-in-the-loop/utils.ts:12-73` — iteration-scoped pause context ids (`blockId + branchIndex + loop iteration`) — per-lane requests addressed independently
- `.resources/sim/apps/sim/lib/workflows/executor/human-in-the-loop-manager.ts:390-501,503-640` — N concurrent pauses independently addressable + concurrent resumes serialized through a claim row (`SELECT ... FOR UPDATE`) — the single-winner admission precedent
- `.resources/sim/apps/sim/executor/handlers/human-in-the-loop/human-in-the-loop-handler.ts:154-169,384-519` — notification side-effects fired at pause, failures swallowed into results, **never failing or delaying the pause** — the bridge-notify-via-effects pattern
- `.resources/sim/apps/sim/lib/workflows/executor/resume-policy.ts:3-62` + `.resources/sim/apps/sim/app/api/resume/poll/route.ts:90-101,331-355` — bounded admission retries → terminal `intervention_required` — the escalation-ladder shape
- `.resources/sim/apps/sim/lib/workflows/executor/execution-id-claim.ts:33-90` — idempotent claim with durable tombstone (the `INSERT OR IGNORE` + token-matched release style)
- Anti-pattern to avoid: `../compozy-v0/engine/task/activities/exec_signal.go:135-150` vs `../compozy-v0/engine/worker/executors/task_wait.go:40` — v0's two disconnected signal planes (anti-lesson 7); our requests ride the ONE wait-resume plane

## Web/Docs Impact

- `web/`: contract types regenerate; run-page consumption lands in task_08 (`requests[]` in run detail ships here so fixtures can be built).
- `packages/site`: new "Human requests" authoring/operating docs section under `docs/loops/`, CLI pages for `requests|request|respond`, `running.mdx` triple-table rows, config page for `expire_after`.
- QA impact: major new user-visible behavior → add `untested` scenarios (ask-answer flow via CLI; expiry route; agent self-denial) in `docs/qa/scenarios/`; walk before completion.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: bridge notification = authored `expires.escalate` effects (existing grammar; deep link via `web_url`) — no bridge SDK change; no new hook; skill rows updated (checked: manifests, MCP, registries unaffected).
- Agent manageability: three new verbs ×3 surfaces with structured output + deterministic errors; `loops.respond` capability; discovery via `compozy__tool_info`.
- Config lifecycle: `loops.defaults.{delivery,watch}.requests.expire_after` — full same-change lifecycle.

## Deliverables

- Ask/request plane complete end to end (grammar → storage → admission → surfaces → docs), `ResponderPolicy` unified with approve, sweep-roots + config lifecycle landed
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [ ] UT-010, UT-014 — ask/expiry lint
- [ ] UT-055, UT-056, UT-062, UT-063, UT-064, UT-065, UT-066, UT-067, UT-068, UT-069 — open/freeze, per-decision validation, allowlist, tiebreaks, idempotent echo, lanes, responder policy rules
- [ ] UT-104, UT-105 — `expire_after` config lifecycle
- [ ] UT-107, UT-108 — payload shapes (`answered_at`, no raw refs), redaction/truncation
- [ ] UT-111 — atomic cancel transitions + `request_canceled`
- [ ] UT-114 — `ResponderPolicy` chain evaluation (starter/child/unrelated/human/cross-workspace/stale)
- [ ] IT-002 — admission races (respond×respond, respond×expiry, respond×cancel legs; amend legs land in task_04)
- [ ] IT-003 — expiry exactly-once across timer/scan/restart
- [ ] IT-005 — respond → namespace exposes the answer downstream
- [ ] IT-014 — list filtering/ordering/cursors/aggregates under concurrency
- [ ] IT-015 — run cancel → atomic request cancel → bell count drop → respond returns `request_canceled`
- [ ] E2E-001 — golden-path ask flow (publish → park → requests → respond → resume → done)
- [ ] E2E-012 — expiry across daemon restart fires escalation + route exactly once

## Success Criteria

- Every assigned test case implemented and passing
- The seven-operation surface set for this task's trio is byte-parity across HTTP/UDS and mirrored in CLI/native tools
- No raw `proposed_ref`/`answered_payload_ref` payload reachable through any API/event/log projection (UT-108 proves)
- `make gate` passes
