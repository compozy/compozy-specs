---
status: completed
title: Liveness, death-resume, cancel ≠ kill and the canceled terminal
type: backend
complexity: critical
---

# Task 5: Liveness, death-resume, cancel ≠ kill and the canceled terminal

## Overview

Makes long-running work first-class and stoppable on purpose only: the 7m30s hidden kill dies,
liveness becomes evidence-only (silence flags, never actions), confirmed session death resumes as
continuation through the atomic `ResumeDeadNode` authority with a progress-reset streak, the
durable node cancellation machine and run-level cancel/kill land with the new `canceled` terminal
outcome swept across the domain, parent-close policy governs spawned work, and the `stop` verb's
backend semantics are deleted.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- R1: MUST delete the duration-kill machinery (`loopActionNoProgressTimeout`,
  `node_timeout`/`no_progress` reasons and their lease-fail paths) and launch loop-bound
  sessions with supervision `inactivity_warning_after = 0` / `inactivity_timeout = 0`
  (ADR-016.1; TechSpec Delete Targets).
- R2: Liveness prober MUST be evidence-only (fresh activity, in-flight tool, transport presence
  = life), driving only `last_evidence_at` and the silence attention flag (30m default, 0
  disables, self-clearing) via the idempotent attention CAS (Safety Invariants 9–11 slice).
- R3: Confirmed death MUST route through `ResumeDeadNode` — one BEGIN IMMEDIATE transaction
  validating live + `cancel_state == ''` (raced cancel wins with deterministic no-op), bumping
  the cell epoch, incrementing the progress-reset streak (3 → `resume_exhausted` attention),
  rotating the managed binding, and reserving exactly one deterministic continuation run
  (round-2 B-007; Safety Invariant 10). Death while parked never resumes.
- R4: MUST implement the node cancellation machine on `loop_node_controls`
  (`'' → requested → delivering → draining → canceled`) with provenance, epoch bumps, delivery
  via the existing prompt-cancel + scoped-interrupt path, `on_cancel` trigger events, and
  idempotent repeat-verb answers; kill = immediate session stop-with-cause + interrupt ladder,
  no node-trigger effects (round-1 B-005/B-006; Safety Invariant 17).
- R5: MUST add the `canceled` terminal outcome across the domain — `dsl.TerminalState`, runtime
  `Status`, transition causes `operator_cancel`/`operator_kill`, `loop_runs.cancel_requested` +
  `cancel_kind` (columns from task_01's migration), terminal-effect selection `on_canceled`,
  and run-level semantics: ONE request transaction that also projects `requested` onto every
  live node control row with epoch bumps, carries the Goal stop cleanup (prompt-lease
  revocation + binding close) before terminalizing, kill = service-tx terminal, cancel drains
  via coordinator, no-active-node runs terminalize directly (round-2 B-001; round-3 B-001).
  `ResumeDeadNode` rejects on pending run- or node-scope cancellation. Contract/HTTP/CLI/native
  enum ripple completes in task_07. This task registers + emits its event kinds
  (`node_canceled`, `node_killed`, `node_attention_flagged`, `node_attention_cleared`).
- R6: MUST implement `on_parent_close: terminate | cancel | abandon` on run-loop nodes with
  strictly parent→child propagation and the typed, fail-closed sub-loop failure boundary
  (ADR-016.6).
- R7: Service-level `Stop` semantics are replaced — backend `stop` path deleted (surface
  deletion completes in task_07); cancel/kill/timeout collapse into the single cancellation
  path (SD-001).
</requirements>

## Subtasks

- [x] 5.1 Delete 7m30s kill + supervision bind-time override; evidence-only prober rewrite
- [x] 5.2 Silence flag lifecycle (raise/clear/aggregate) via attention CAS
- [x] 5.3 `ResumeDeadNode` atomic authority + streak + checkpoint-carrying continuation
- [x] 5.4 Node cancel machine (states, provenance, delivery, drain, close) + kill path
- [x] 5.5 `canceled` terminal sweep (dsl/status/causes/run columns/terminal-effect selection)
- [x] 5.6 Run-level cancel/kill semantics incl. no-active-node direct terminalization
- [x] 5.7 Parent-close policy + typed sub-loop failure boundary
- [x] 5.8 Race/restart suites (cancel×death, cancel×completion, double detection) + E2E journeys

## Implementation Details

Follow TechSpec run-level contract paragraph, `DeathResume`/`NodeControl` interfaces, Safety
Invariants 8–10/17, ADR-016. New files: `service_node_control.go`, `service_run_cancel.go`,
death/liveness evolution in `internal/daemon/loop_action_liveness.go` (rewrite) + a new
`internal/daemon/loop_node_liveness.go` if the cap demands.

### Relevant Files

- `internal/daemon/loop_action_liveness.go:18-21,186-212` — delete targets + `refreshActionProgress` precedent
- `internal/session/prompt_activity_heartbeat.go:11-113` — `touchWithTool` vs waiting heartbeat (evidence split)
- `internal/session/manager_prompt.go:135-192` — idempotent `CancelPrompt` + scoped interrupt
- `internal/session/prompt_activity_timeout.go:80-141` — grace-then-force template
- `internal/toolruntime/interrupt.go:15-93` — SIGTERM→SIGKILL ladder + PID-reuse guard
- `internal/loop/action_managed.go` — binding rotation (`AdvanceActionSessionRetry`)
- `internal/loop/service_control.go:62-129` — Stop/Pause/Approve verb patterns (Stop replaced)
- `internal/loop/service_types.go:34-91` — Status + TransitionCause enums
- `internal/loop/dsl/contract.go:70-85` — TerminalState enum
- `internal/loop/action_runloop.go` — parent-close attachment point
- `internal/config/config_agent_session.go:54-62` — supervision config override at bind

### Dependent Files

- `internal/loop/coordinator_goal_control.go` — goal park interactions (death-while-parked)
- `internal/api/contract/loop_enums.go` — status enum (task_07 ripple)
- `web/src/systems/loops/lib/loop-formatters.ts` — status pill (task_08)

### Related ADRs

- [ADR-016](adrs/adr-016.md) — liveness/cancel/kill/parent-close · PRD [ADR-004](adrs/adr-004.md) / [ADR-008](adrs/adr-008.md)

## Web/Docs Impact

Backend-only here; `canceled` status, cancel controls, and attention flags reach web in task_08,
docs in task_11.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: `loop.node.terminal` hook payload gains class/disposition/attempt/target fields
  (additive; matchers unchanged — verified against `internal/hooks/matcher_payload.go:184`).
- Agent manageability: verbs surface in task_07; this task owns their service/store authorities
  with deterministic invalid-state answers.
- Config lifecycle: consumes `liveness.silence_window` + `resume.death_streak_limit` from
  task_01; no new keys.

## QA impact

Flag new `untested` scenarios: crash-at-3am-death-resume (US-016), cancel-vs-kill (US-018),
days-long-node-no-clock (US-015); reset any existing scenario exercising `loop stop` semantics to
`untested` (walked in task_13 against cancel/kill).

## Skills

`eng-code-guidelines`, `golang-pro`, `eng-test-conventions`, `testing-boss`,
`eng-consolidate-test-suites`, `eng-cleanup-failure-paths`, `systematic-debugging` (races).

## References

- `.resources/temporal/service/history/api/recordactivitytaskheartbeat/api.go:102-106` — control on the liveness channel
- `.resources/temporal/common/enums/defaults.go:47-51` — parent-close default TERMINATE
- `.resources/sim/apps/sim/executor/errors/boundary.ts:6-54` — typed fail-closed child boundary
- `.compozy/tasks/loop-ideas/analysis/behavior-defaults.md` §Temporal cancel/pause, §Mastra bidirectional anti-lesson

## Deliverables

- No duration-based failure path exists; death costs one continuation; cancel/kill/`canceled`
  terminal live end-to-end at both scopes
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] UT-036 — cancel preempts route, on_cancel fires (moved here with the cancel verbs)
- [x] UT-051 — cancel during backoff epoch-invalidates the pending retry (moved here)
- [x] UT-074 — kill suppresses node-trigger effects (moved here with the kill verb)
- [x] UT-081 — operator cancel/kill terminal → contract terminal effect fires exactly once (moved here: rides the kill/cancel verbs and on_canceled selection)
- [x] UT-086..UT-104 — evidence rules, supervision override, death/streak/silence lifecycle
- [x] UT-105..UT-108 — cancel machine walk, idempotency, races, grace visibility
- [x] UT-187 — (withdrawn) no implementation; the no-duration-kill invariant is owned by task_02's timeout suite plus this task's IT-021 and E2E-006
- [x] UT-189..UT-191 — parent-close + typed sub-loop boundary
- [x] UT-193 — run-level cancel/kill terminal contract
- [x] UT-195 — `ResumeDeadNode` atomicity + loser answers
- [x] IT-016 — cancel matrix; IT-021 — liveness/no-clock regression; IT-024 — death-resume races/restart; IT-025 — kill; IT-035 — sub-loop boundary
- [x] E2E-006 — backend long-run + death-resume journey registered; Task 13 owns the isolated daemon walk
- [x] E2E-007 — backend cancel vs kill journey registered; Task 07 owns public stop absence and Task 13 the parity walk

## Success Criteria

- Every assigned test case implemented and passing; `-race` clean
- Clock-far-forward with live evidence never fails a node; a week of daemon restarts never exhausts the streak
- Cancel wins every induced race; exactly one continuation per death is representable
