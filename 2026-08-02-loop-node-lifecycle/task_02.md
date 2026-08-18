---
status: completed
title: "Precedence core: retry, scheduling, routes and escalation"
type: backend
complexity: critical
---

# Task 2: Precedence core — retry, scheduling, routes and escalation

## Overview

Implements the contract's first rule end-to-end at the coordinator boundary: classification-driven
automatic retry (durable `retrying` cells, epoch-fenced due-scan on the 15s cycle, backoff +
retry-after, authored `timeout`/`deadline` anchors), forward-only error routes with skip-cascade,
explicit absorption, and escalation into generation succession with repair context — each step
recorded as an attempt-ledger disposition. This is the safety-primitive heart of the spec.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- R1: MUST enforce the fixed precedence chain retry → route → effects-slot → escalation
  identically across node families, recording each step's outcome in `loop_node_attempts`
  (PRD Rule 1; Safety Invariant 1). Effects themselves land in task_03 — this task emits the
  trigger lifecycle events and dispositions they consume.
- R2: MUST implement durable coordinator-mediated retries: `retrying` status +
  `next_attempt_at` + issuing `epoch` on the outputs cell, attempt-suffixed deterministic task
  runs, decorrelated-jitter backoff within [base, max], `retry_after` honored within bounds,
  and classification gating (only `transport`/`attempt_timeout`; expensive families only via
  node-level declaration — Safety Invariant 4).
- R3: MUST implement the due-scan joined to `RunLoopCoordinatorBackstop` (paged, epoch-checked,
  paused/terminal excluded) plus the opportunistic in-memory timer converging on the same
  idempotency key (ADR-012); every superseding mutation bumps affected cell epochs in the same
  transaction; stale fires drop with `stale_schedule_dropped`.
- R4: MUST implement `timeout` (attempt, `attempt_timeout` class) and `deadline` (total,
  `budget_exhausted` class) with `first_scheduled_at` anchoring, parked-clock suspension hooks,
  and deterministic completion/timeout race resolution (US-010).
- R5: MUST implement `on_error.route` as a forward edge activating on terminal node failure
  (handled disposition; success-path skip-cascade reusing branch skip semantics) and
  `allow_fail` absorption — handled failures never feed target health, always count toward
  caps/budgets (PRD Rules 2–4); delete the `explicitDependencyBlocker` magic strings in favor of
  classification-driven blocked mapping (TechSpec Delete Targets).
- R6: MUST keep single-writer discipline: all mutations ride the completion plan / claim-fenced
  transaction; both outputs writers CAS on epoch (Safety Invariants 5–6).
- R7: MUST preserve every existing bound unchanged (iteration_cap, budgets, max_revisions,
  stall window, watch-arm breaker) — attempts and handled failures count toward them.
</requirements>

## Subtasks

- [x] 2.1 Attempt-ledger writes + disposition recording at the boundary (classifier from task_01)
- [x] 2.2 Retry planning: eligibility, backoff, retry-after, attempt-suffixed run ids, config resolution captured at admission
- [x] 2.3 Epoch fence: bump-on-supersede in plans, CAS in both outputs writers; the SINGLE parked-state classifier function lands here (states populated by tasks 04/05/06 — one owner, round-3 B-007)
- [x] 2.4 Due-scan page in the scheduler backstop + opportunistic timer fast path
- [x] 2.5 `timeout`/`deadline` anchors, classes, race resolution, clock-suspension seams
- [x] 2.6 Error-route compilation (forward edge + handle), route activation, skip-cascade, absorption
- [x] 2.7 Escalation integration: unhandled failures → succession with classified repair context; magic-string blocker deletion
- [x] 2.8 E2E-runtime journeys (acpmock) for retry-heals, route-fallback, escalation

## Implementation Details

Follow TechSpec "Component Overview" (lifecycle planner, due-scan), "Core Interfaces", Safety
Invariants 1–6. New files: `coordinator_lifecycle.go`, `coordinator_retry.go`,
`internal/daemon/scheduler_loop_due_scan.go`. Extend (headroom): `coordinator_outputs.go`
(status mapping + epoch CAS), `coordinator_generation_reattempt.go` (route-aware rerun
exclusion). Frozen files are extended by new files only.

### Relevant Files

- `internal/loop/coordinator.go:175` — `buildCoordinatorPlan` branch point
- `internal/loop/coordinator_outputs.go:112-145` — node-status mapping (failure entry point)
- `internal/loop/coordinator_generation_succession.go:134` — `buildFailedGenerationPlan`
- `internal/loop/coordinator_generation_reattempt.go:111-171` — rerun sets + transitive dependents
- `internal/loop/coordinator_terminal_helpers.go:249` — magic-string blockers (delete target)
- `internal/daemon/scheduler_loop_coordinator.go:12` — backstop the due-scan joins
- `internal/scheduler/types.go:14` — 15s cycle
- `internal/retry/backoff.go:36-53` — decorrelated jitter (reused inside attempts)
- `internal/store/globaldb/global_db_task_coordinator.go:18` — completion transaction
- `internal/store/globaldb/global_db_task_claim_complete.go:349` — task-terminal outputs writer

### Dependent Files

- `internal/loop/generation_snapshot.go` — snapshot writer (epoch column)
- `internal/loop/control_namespace_history.go` — repair context gains classified failure summary
- `internal/testutil/acpmock/` + `internal/testutil/e2e/` — E2E fixtures/matchers co-ship (L-007)

### Related ADRs

- [ADR-012](adrs/adr-012.md) — scheduling/epochs · [ADR-013](adrs/adr-013.md) — classification consumption
- PRD [ADR-002](adrs/adr-002.md) — escalate-by-default · PRD [ADR-003](adrs/adr-003.md) — family asymmetry

## Web/Docs Impact

Backend-only here; the `retrying` status and attempt fields reach web/docs via task_07 payloads +
task_08 UI + task_11 docs.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — no hook/manifest changes (checked: hooks payloads unchanged this task).
- Agent manageability: attempt/next-attempt visibility lands in payloads in task_07; this task
  guarantees the durable rows exist.
- Config lifecycle: consumes task_01 keys (admission-time resolution, IT-031); no new keys.

## QA impact

Flag new `untested` content-addressed scenarios (walked in task_13): transient-blip-heals
(US-008), error-route-fallback (US-005), unannotated-escalation (US-007).

## Skills

`eng-code-guidelines`, `golang-pro`, `eng-test-conventions`, `testing-boss`,
`eng-consolidate-test-suites`, `eng-cleanup-failure-paths`, `systematic-debugging` (concurrency).

## References

- `.resources/temporal/common/retrypolicy/retry_policy.go:25-97`, `.resources/temporal/service/history/workflow/retry.go:115-152` — classification-gated retry
- `.resources/temporal/service/history/workflow/timer_sequence.go:176-362` — attempt vs total timers
- `.resources/temporal/service/history/hsm/tasks.go:24-62` — stamp/epoch validation pattern
- `.resources/sim/apps/sim/executor/execution/edge-manager.ts:252-341` — error port + skip cascade
- `.compozy/tasks/loop-ideas/analysis/behavior-defaults.md` — decision-grade defaults

## Deliverables

- Precedence chain live for every family with durable attempt visibility
- Due-scan + epoch fence + timers operational across restart
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] UT-029..UT-035, UT-037..UT-043 — routes, absorption, escalation (the cancel-preemption case moves to task_05 with the cancel verbs; the quarantined-producer rerun case lives in task_04)
- [x] UT-045..UT-050, UT-053..UT-056 — retry planner, backoff, caps, deadline interplay (cancel/pause-during-backoff cases move to tasks 05/06 with their verbs)
- [x] UT-057..UT-061, UT-197 — expensive-family DSL-only opt-in (both families, config layers ignored) + checkpoint continuation + resume distinction + authored sub-loop retry
- [x] UT-062..UT-065, UT-067..UT-069 — timeout/deadline classes, races, stale-epoch drops (the parked-clock suspension case moves to task_06 with the wait/pause states)
- [x] UT-177..UT-179 — due-scan selection, epoch bumps, timer/idempotency convergence
- [x] IT-001..IT-012 — boundary walks (classification, hints, predicates, routes, retry cycle, timeouts); the pause-driven epoch-fencing integration case moves to task_06
- [x] IT-031 — config admission-time pin + global breaker source labels
- [x] IT-033 — two-writer epoch race
- [x] E2E-001, E2E-002, E2E-003 — retry heals / route fallback / escalation journeys

## Success Criteria

- Every assigned test case implemented and passing; `-race` clean
- A transient failure costs one backoff delay (never a generation); an unannotated failure escalates with classified context from generation 1
- No ghost fires: induced pause/cancel between schedule and fire always drops with the diagnostic
