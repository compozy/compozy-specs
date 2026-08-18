---
status: completed
title: "Effects: outbox, relay and delivery identity"
type: backend
complexity: high
---

# Task 3: Effects — outbox, relay and delivery identity

## Overview

Delivers the authored reaction system: trigger firings expand into same-transaction
`loop_effect_outbox` rows with deterministic `delivery_id`s, and a boot-started daemon-singleton
relay drains pending rows post-commit — executing `emit` and `tool` entries in isolation,
fail-open, observe-only, idempotent on the delivery identity. Includes the 15 new lifecycle event
kinds, the pre-bound `effect.*` template context, and the daemon-scope tool execution identity.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- R1: MUST expand declared effects into outbox rows inside the same transaction as the trigger's
  state change, persisting the FULLY rendered, sanitized execution payload and
  `delivery_id = sha256(loop_run_id, source_event_id, trigger, entry_index)` (ADR-015; Safety
  Invariant 7).
- R2: MUST implement the relay as boot-started + cycle-driven + nudge-hinted, paging ALL pending
  rows, acking each in one transaction: idempotent result-event appends via
  `loop_run_events.delivery_key = "<delivery_id>:<kind>"` INSERT OR IGNORE, row →
  delivered/failed, attempts++ (round-2 B-004). Tool execution is at-least-once with
  delivery_id in the tool correlation.
- R3: The relay MUST NOT fire hooks and MUST NOT derive work from `loop_run_events` — hooks stay
  at owning call sites (round-1 B-001); `emit` kinds never enter the hook taxonomy.
- R4: MUST execute `tool` entries daemon-side via `tools.Call` in the run's workspace scope,
  actor `daemon:loop-effect`; approval-requiring tools fail deterministically
  (`effect_tool_approval_required`); entries are isolated — one failure never stops the next
  (PRD Rules 20–22; grill decision).
- R5: MUST pre-bind the `effect.*` context (identity, classified failure + hint, attempt info,
  resume/approval links) before input resolution, sanitized once (Safety Invariant 16).
- R6: MUST register and emit ONLY the event kinds whose transitions exist by this task
  (`node_retry_scheduled`, `effect_results`, `custom_event`) in `loopRunEventKindValid`; each
  later behavior-owning task registers its own kinds (04: quarantine/requeue/breaker; 05:
  cancel/kill/attention; 06: pause/wait/suppression), and the aggregate contract
  `LoopRunEventKind` enum + codegen + SSE parity ship in task_07 (round-3 B-007). Kill-scope
  effect suppression is proven in task_05 where kill exists.
</requirements>

## Subtasks

- [x] 3.1 Outbox expansion inside completion-plan/verb transactions + delivery_id derivation
- [x] 3.2 Relay component: boot start, cycle page, nudge, per-entry isolation, ack transaction
- [x] 3.3 `emit` execution (custom_event, bounded, idempotent) + `effect_results` recording
- [x] 3.4 `tool` execution via daemon-scope `tools.Call` + approval-required deterministic failure
- [x] 3.5 Pre-bound `effect.*` context + link building + sanitization pass
- [x] 3.6 This task's event kinds registered + emitted (retry/effects family); relay tolerant of later-task kinds
- [x] 3.7 Crash/replay + idempotency integration suite; acpmock E2E for on_error notification

## Implementation Details

Follow TechSpec "Effect relay" component, `EffectDispatcher` interface, `loop_effect_outbox` DDL,
and ADR-015. New files: `effects_dispatch.go`, `effect_context.go`,
`internal/daemon/loop_effect_relay.go`, store `global_db_loop_effect_outbox.go`.

### Relevant Files

- `internal/store/globaldb/global_db_task_coordinator_intents.go:16-57` — in-tx intent application (outbox rows join here)
- `internal/store/globaldb/global_db_loop_events.go:114-166` — event append + kind validation
- `internal/loop/gate/evaluator_extension.go:14-70` — daemon-side `tools.CallRequest` precedent
- `internal/bridges/task_notifier.go:62-135` — sweep/ack pattern reference
- `internal/loop/goal/types_outbox.go` + `internal/daemon/loop_goal_outbox_relay.go` — existing outbox relay precedent
- `internal/tools/` dispatch + policy (approval classification for R4)
- `internal/api/contract/loop_runs.go:123-131` — `LoopRunEventPayload` + kind enum

### Dependent Files

- `internal/daemon/loop_hook_observer.go` — post-commit nudge hook-in
- `web/src/systems/loops/lib/loop-events.ts` — consumes kinds (task_08)
- `internal/hooks/payloads_task_loop.go` — untouched (verify no hook coupling)

### Related ADRs

- [ADR-015](adrs/adr-015.md) — outbox + relay protocol · PRD [ADR-006](adrs/adr-006.md) — reaction contract

## Web/Docs Impact

Backend-only here; event kinds reach the web reducer in task_08 and docs in task_11.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: effect `tool:` entries reach extension/MCP/native tools through the standard
  tool surface — no new extension kind; `emit` kinds are author-namespaced, outside the hook
  taxonomy (checked `internal/hooks/events.go`).
- Agent manageability: effect results and deliveries are observable via SSE events; no new verbs.
- Config lifecycle: none — no keys (checked `internal/config/loops.go` scope from task_01).

## QA impact

Flag new `untested` scenarios: on-error-notification-with-context (US-011),
terminal-outcome-notification (US-013).

## Skills

`eng-code-guidelines`, `golang-pro`, `eng-test-conventions`, `testing-boss`,
`eng-consolidate-test-suites`, `eng-cleanup-failure-paths`.

## References

- `.resources/temporal/common/effect/buffer.go` + `.resources/temporal/docs/architecture/effect-package.md` — commit-gated effects
- `.resources/sim/apps/sim/executor/handlers/human-in-the-loop/human-in-the-loop-handler.ts:154-519` — pre-binding + swallowed effect failures
- `.compozy/tasks/loop-ideas/analysis/sim.md` §14, `analysis/temporal.md` §6

## Deliverables

- Outbox + relay live end-to-end with at-least-once, identity-stable delivery
- This task's event kinds observable on SSE with truthful ordering (aggregate enum parity in task_07)
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] UT-070..UT-073, UT-075..UT-076, UT-080, UT-082..UT-085 — trigger firing, isolation, fail-open, scopes, terminal outcomes, and emit contract (UT-077..UT-079 move to task_06 with pause decisions; kill suppression and UT-081 canceled-terminal behavior move to task_05)
- [x] UT-180..UT-182 — pre-binding, approval-required failure, sanitization
- [x] UT-194 — delivery_key idempotent append
- [x] UT-196 — render failure → error-as-data delivery (trigger commits, sibling delivers)
- [x] IT-014 — relay end-to-end (same-tx rows, drain, results, no hook dispatch)
- [x] IT-015 — crash/replay protocol (lost nudge, post-tool crash, duplicate drains, cross-run seq)
- [x] E2E-004 — on_error notification journey

## Success Criteria

- Every assigned test case implemented and passing
- Crash windows never lose or duplicate result events; render failure never blocks a state transition
- A dead webhook costs one recorded `effect_results` failure and nothing else
