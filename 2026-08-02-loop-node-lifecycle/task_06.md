---
status: completed
title: Pause, auto-pause, durable waits and admission dedupe
type: backend
complexity: high
---

# Task 6: Pause, auto-pause, durable waits and admission dedupe

## Overview

Delivers the parked-state half of the contract: node-level pause with provenance and resume
variants, config-driven auto-pause rules, the `wait` control kind with restart-surviving rows and
the atomic single-claim `ResumeWait`, authored expiry/escalation ladders, the uniform
parked-state clock/stall suspension classifier, and loud durable watch-admission dedupe in the
loop-start transaction.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- R1: MUST implement node pause/resume on `loop_node_controls` (drain-or-cancel choice via the
  task_05 cancel machinery, provenance manual-vs-rule, resume plain/reset_attempts/immediate,
  idempotent answers), excluding paused nodes from scheduling, rerun sets, stall signatures, and
  due-scans, with epoch bumps parking pending schedules (US-019/US-020; PRD Rules 26, 28).
- R2: MUST implement auto-pause rules from `[[loops.defaults.*.autopause]]` (task_01 keys):
  ordered, first-match-wins, named provenance, no re-fire without recurrence, evaluated over
  `{node_id, family, target, class, attempt}`.
- R3: MUST implement the `wait` runtime: parking cells `waiting` with `loop_node_waits` rows
  (timer `resume_at`, event subscriptions via watch-events, gate expiry ladders sharing the
  row), ahead-arrival policy, and the atomic `ResumeWait` (validate → claim → transitions →
  provenance/event → coordinator reservation in ONE transaction; 3×60s admission failures →
  `intervention_required`) (ADR-017; Safety Invariant 12; round-2 B-003 analog for waits).
- R4: MUST integrate the parked states into the SINGLE classifier owned by task_02: `paused |
  waiting | awaiting-approval | quarantined` suspend node clocks and the run wall-clock budget,
  are excluded from stall/no-progress arithmetic and due-scans (except their own escalation),
  and stay visible in progress reporting; token spend always counts (Safety Invariant 11;
  single owner per round-3 B-007). This task registers + emits its event kinds (`node_paused`,
  `node_resumed`, `node_wait_started`, `node_wait_resumed`, `duplicate_suppressed`).
- R5: MUST implement watch-admission dedupe: suppression key `(workspace, loop, source_key,
  event_key)` claimed `INSERT OR IGNORE` inside `CreateLoopRunForStart`'s transaction, loud
  structured suppression answers + `duplicate_suppressed` diagnostics + counters, horizon sweep,
  and the runtime `event_key` fail-closed validation (Safety Invariant 15; ADR-017.5) —
  including the `watch-source` `PollResponse.event_key` protocol change + scaffold update.
- R6: Escalation-ladder steps fire as effects through the task_03 relay; a decision arriving
  mid-ladder cancels remaining steps.
</requirements>

## Subtasks

- [x] 6.1 Node pause/resume verbs (service/store CAS) + scheduling/stall/due-scan exclusion
- [x] 6.2 Auto-pause rule evaluation + provenance + episode no-re-fire
- [x] 6.3 `wait` runtime: parking, timer wakes, event subscription bridge, ahead arrival
- [x] 6.4 Atomic `ResumeWait` + admission-failure ladder → intervention_required
- [x] 6.5 Gate expiry + escalation ladders on the wait row (steps via effect relay)
- [x] 6.6 Parked-state classifier + budget/stall/clock suspension wiring
- [x] 6.7 Admission claims in start tx + suppression answers + horizon sweep + runtime event_key validation + SDK scaffold
- [x] 6.8 Restart/concurrency suites + acpmock E2E journeys (approval link, live repair, durable wait, ladder, dedupe)

## Implementation Details

Follow TechSpec `WaitAdmission`/`NodeControlStore` interfaces, ADR-017, Safety Invariants 11–12,
15. New files: `coordinator_wait.go`, `node_waits.go`, `admission_claims.go`, wait/pause store
files; `dsl/wait.go` grammar shipped in task_01.

### Relevant Files

- `internal/loop/goal_reactivation.go` + `internal/loop/coordinator_reactivation.go:11-33` — atomic grant/reactivation precedent for `ResumeWait`
- `internal/store/globaldb/global_db_loop.go:32-66` — `CreateLoopRunForStart` transaction (claims join here)
- `internal/loop/coordinator_terminal_helpers.go:170-197` — stall signature (exclusion seam)
- `internal/loop/watch_events_registry*.go` + `internal/extension/watch_source.go:20-57` — event wait + `event_key` protocol
- `internal/loop/watch/types.go:10-34` — `PollResponse` (gains `event_key`)
- `internal/extension/scaffold_templates/loop-watch-source-go/main.go` — SDK scaffold update
- `internal/loop/service_control.go:70-127` — Pause/Resume run-level patterns
- `internal/loop/goal/budget.go` — budget accounting seam for wall-clock suspension

### Dependent Files

- `internal/daemon/scheduler_loop_due_scan.go` — wait/ladder due pages (task_02 component)
- `internal/loop/coordinator_lifecycle.go` — parked-state classifier consumption
- `internal/extensionprotocol/` docs — `event_key` contract note (docs finalized task_11)

### Related ADRs

- [ADR-017](adrs/adr-017.md) — parked states + dedupe · PRD [ADR-005](adrs/adr-005.md) / [ADR-009](adrs/adr-009.md)

## Web/Docs Impact

Backend-only here; waiting inventory, pause controls, and suppression diagnostics reach web in
task_08, docs (incl. extension `event_key` breaking change) in task_11.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: **breaking** `loop.watch_source` provide-surface change (`PollResponse.event_key`
  required) — protocol docs + scaffold + dispatch updated together, hard cut, no compat.
- Agent manageability: pause/resume verbs + waiting inventory surface in task_07; suppression
  answers are structured for the deliverer.
- Config lifecycle: consumes autopause/waits/admission keys from task_01; no new keys.

## QA impact

Flag new `untested` scenarios: live-pause-repair-resume (US-019), approval-link-journey (US-012),
durable-wait-restart (US-021), waiting-inventory-escalation (US-022), duplicate-event-suppressed
(US-025).

## Skills

`eng-code-guidelines`, `golang-pro`, `eng-test-conventions`, `testing-boss`,
`eng-consolidate-test-suites`, `eng-cleanup-failure-paths`, `systematic-debugging` (claims).

## References

- `.resources/temporal/service/history/workflow/activity.go:245-520` — pause/unpause variants + rule-driven auto-pause
- `.resources/sim/apps/sim/lib/workflows/executor/resume-policy.ts:3-62` — 3×60s → intervention_required
- `.resources/sim/apps/sim/lib/workflows/executor/execution-id-claim.ts:33-90` — claim + tombstones
- `.resources/sim/apps/sim/executor/human-in-the-loop/utils.ts:12-73` — iteration-scoped pause ids
- `.compozy/tasks/loop-ideas/analysis/behavior-defaults.md` §Sim pause/dedupe rows

## Deliverables

- Parked states durable, governed, and uniformly excluded from work accounting; one event = one run
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] UT-052 — pause parks the pending retry; resume decides attempt reset (moved here with the pause verbs)
- [x] UT-066 — parked states suspend timeout/deadline clocks (moved here with the wait/pause states)
- [x] UT-109..UT-121 — pause verbs, exclusion, variants, auto-pause rules
- [x] UT-122..UT-132 — wait rows, atomic resume, lanes, ahead arrival, ladders, inventory truth
- [x] UT-146..UT-152 — dedupe key/claims/tombstones/counters
- [x] UT-192 — runtime event_key fail-closed validation
- [x] IT-013 — pause-driven epoch fencing (moved here: needs the pause verb); IT-018 — pause/resume + budget suspension; IT-019 — approval link flow; IT-020 — concurrent claims; IT-023 — auto-pause; IT-027 — wait lifecycle + restart; IT-028 — dedupe across restart
- [x] E2E-005 — approval where people live; E2E-008 — live repair; E2E-009 — durable wait restart; E2E-010 — inventory + ladder; E2E-013 — watch redelivery

## Success Criteria

- Every assigned test case implemented and passing
- Engine restart mid-wait resumes exactly once; concurrent resumes yield one winner with provenance in loser answers
- Redelivering one event N times yields one run and N−1 loud suppressions
