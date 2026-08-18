---
status: pending
title: "Orchestration kernel O1–O5: lease attempts, loop breaker, ClaimRun CAS, timeouts, wake-cap"
type: backend
complexity: critical
---

# Task 11: Orchestration kernel O1–O5: lease attempts, loop breaker, ClaimRun CAS, timeouts, wake-cap

## Overview

One critical slice absorbing former merge-time couplings (31↔33 guarded-claim+cap; 29↔32 attempt
budget on timeout). Fixes O1–O5 in the loop/lease/scheduler kernel — five evidence-linked
correctness defects in the durable orchestration surface (`analysis/09`; TechSpec §3.10). This
eliminates the two "must coordinate at merge time" hazards: (1) the per-workspace concurrent-run
cap MUST share the same immediate transaction as the guarded-claim CAS (N-401 / N-302), and
(2) action-node timeout terminalization MUST consume the O1 attempt budget rather than reclaiming
forever. Landing them as one coherent change removes the cross-PR coordination tax.

**Authoritative sources:** this suite has no `_prd.md` / `_user_stories.md` / `_tests.md`.
`_techspec.md` §3.10 and `analysis/09_analysis_task-orchestration.md` are authoritative (no
separate ADR — correctness fixes to existing behavior). Concrete test cases are inline below
(exact input/condition/expected).

Merges former tasks 29+30+31+32+33. Independent of W1–W5 (W6); feeds `11→12`.

**CRITICAL subtask order:**
1. Shared guarded-claim helper (ex-31) WITH per-workspace concurrent-run cap in SAME transaction
   (ex-33) — N-401
2. Lease-expiry consumes attempt (ex-29)
3. Action-node timeout + progress liveness that rides O1 attempt budget (ex-32)
4. Loop circuit-breaker per-node accounting (ex-30)
5. Indexed wake-dedup (ex-33 remainder)

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
### Shared guarded-claim + concurrent-run cap (ex-31 + ex-33 cap half) — MUST land first
1. MUST make the manual `ClaimRun` transition atomic: the ownership write applies only when the row
   is still `queued`, inside the same immediate transaction (status CAS,
   `WHERE id=? AND status='queued'`).
2. MUST return a typed `ErrNoClaimableRun`/`ErrInvalidStatusTransition` on a 0-row update so the
   caller learns the run was already claimed, instead of receiving a false success.
3. MUST NOT weaken the existing `ErrSessionAlreadyBound` guard or the general `UpdateTaskRun` path
   used for non-claim mutations — the generic blind update never becomes status-aware for all
   callers.
4. MUST implement the guarded claim as ONE shared store-level helper: extract `ClaimNextRun`'s
   inline guarded `status='queued'` UPDATE into a single helper that BOTH `ClaimNextRun` and
   `ClaimRun` invoke — never a parallel `ClaimRunCAS`/second CAS statement (Safety Invariant 24;
   peer-review round 2 B-001; L-004/L-005).
5. MUST add a per-workspace concurrent-active-run cap consulted at claim/admission time —
   dependents remain durably enqueued in `task_runs` (the single queue); the cap defers *claiming*
   only, draining as capacity frees (Safety Invariant 26; never an enqueue-time drop or reject —
   round-2 N-003). The cap check MUST execute inside the same immediate transaction as the
   guarded-claim CAS — a hard bound, not an advisory read — so two concurrent claims cannot both
   observe `count < cap` and over-admit (round-3 N-302; round-4 N-401).
6. MUST make the cap configurable with a safe default and expose the saturation state on a
   status/diagnostics surface (agent-manageable) rather than failing opaquely.
7. MUST preserve dependency-completion semantics — deferred dependents still run once capacity
   frees, never dropped.

### Lease-expiry attempt budget (ex-29 / O1)
8. MUST make a lease-expiry recovery consume the attempt budget: either increment `attempt` on
   requeue or track a durable `lease_recovery_count`, so repeated crash-reclaims are bounded.
9. MUST terminalize the run to a non-retryable state (`failed`/`needs_attention`) once the budget
   is exhausted, with a recorded reason distinguishing "crash-looped" from an ordinary failure.
10. MUST keep the token-fenced snapshot-CAS in `requeueExpiredLease` intact — only the attempt
    accounting changes; no weakening of the claim fence.
11. MUST cover the accounting with an append-only migration if a new column is introduced (L-021).
12. MUST leave the authoritative `ClaimNextRun` path and normal `ReleaseRunLease` requeue
    semantics unchanged (those already consume/track attempts correctly).

### Action-node timeout + progress liveness (ex-32 / O4)
13. MUST give action-node runs a wall-clock timeout (the dual of the existing
    `awaitingChildTimedOut` child-loop timeout) so a wedged action node terminalizes instead of
    hanging the generation.
14. MUST add a progress signal to the lease so a heartbeat that carries no progress can be
    distinguished from genuine work — a heartbeat-healthy-but-stalled run eventually fails rather
    than holding its lease indefinitely.
15. MUST use idle-vs-active thresholds (a longer window while a tool is actively running, a shorter
    one while idle) modeled on Hermes' dual staleness thresholds, not a single blunt cap.
16. MUST record a forensic terminal reason (e.g., `node_timeout`/`no_progress`) and free the lease
    so the run can be retried under the O1 attempt budget.
17. MUST NOT shorten leases for legitimately long, progressing work — the cap is on
    inactivity/wall clock, not on total runtime of a healthy run.

### Loop circuit-breaker (ex-30 / O2)
18. MUST make failure accounting resistant to sibling success: track consecutive failures per node
    (or as a whole-generation outcome), not a single loop-wide counter zeroed by any node success.
19. MUST trip `StatusStalled`/`TransitionCauseNoProgress` when a node's own failure streak reaches
    `LoopFailureBreakerLimit`, independent of other nodes succeeding in the same generation.
20. MUST give an unbounded watch loop (`IterationCap <= 0`) a hard generation backstop so a
    defeated breaker cannot produce an infinite regenerate loop.
21. MUST preserve the existing no-progress stall detector and reattempt-strategy semantics; this
    changes only the failure-counter accounting and its backstop.
22. If a schema column changes (e.g., per-node failure counts), the migration appends at the tail
    (L-021). Delete target: the sibling-resetting global `CASE … ELSE 0` accounting.

### Indexed wake-dedup (ex-33 remainder / O5)
23. MUST replace the unbounded `ListTaskEvents` wake-dedup scan with an indexed point-lookup (or a
    dedicated dedup table) so dedup is O(1)-ish regardless of a task's event count; keep the
    in-memory cache as a fast path only.
24. MUST preserve wake correctness (the DB audit stays authoritative).
25. Any new index/table migration appends at the registry tail (L-021).
</requirements>

## Subtasks (CRITICAL order — absorbs former 31↔33 and 29↔32 couplings)

- [ ] 11.1 Extract `ClaimNextRun`'s inline guarded claim into one shared store helper; wire both
      `ClaimNextRun` and `ClaimRun` through it; compose the per-workspace concurrent-active-run
      cap check into the SAME immediate transaction (N-401 / N-302); typed 0-row error; config
      default + saturation observability (CLI/HTTP/UDS).
- [ ] 11.2 Lease-expiry requeue consumes attempt/recovery budget; terminalize on exhaustion with
      forensic reason; append-only migration if a column is added (L-021); surface terminal state
      on run listing.
- [ ] 11.3 Action-node wall-clock timeout + progress signal on `LeaseHeartbeat` + idle-vs-in-tool
      thresholds; forensic terminal reason + lease release feeding the O1 attempt budget;
      append-only migration for progress/last-activity column.
- [ ] 11.4 Per-node (or whole-generation) consecutive-failure accounting replacing the global
      reset; breaker trip keyed on a node's own streak; hard generation backstop for
      `IterationCap <= 0`; append-only migration if needed; delete sibling-resetting accounting.
- [ ] 11.5 Indexed wake-dedup lookup replacing the unbounded `ListTaskEvents` scan; keep
      in-memory cache as fast path; append-only migration for the index/dedup table (L-021).

## Implementation Details

See `_techspec.md` §3.10 and `analysis/09_analysis_task-orchestration.md` §4–6. Deliberate
divergence: adopt Hermes' *policies* (spawn pause, activity-liveness, reject-at-cap), never its
in-memory persistence — AGH's SQLite durable-lease model stays everywhere. The concurrency cap is
a defer gate, not a hard reject. Fencing semantics (claim-token issuance, `lease_until`,
owner-tuple WHERE columns) live in the one shared helper so the two claim entry points
structurally cannot diverge.

### Relevant Files

- `internal/store/globaldb/global_db_task_claim.go` — `requeueExpiredLease`, `RecoverExpiredRunLeases`,
  guarded-claim extraction, loop node-terminal failure accounting
- `internal/store/globaldb/global_db_task.go` — claim update path
- `internal/task/manager.go` / `internal/task/lease_manager.go` / `internal/task/lease.go` —
  `ClaimRun`, heartbeat progress, exhaustion predicate, active-run cap gate
- `internal/task/wake.go` — replace `ListTaskEvents` scan with indexed lookup
- `internal/task/limits.go` — concurrency cap + `DefaultTaskMaxAttempts`
- `internal/loop/coordinator.go` — breaker check, action-node timeout, watch-loop backstop
- `internal/loop/types.go` / `internal/loop/config.go` — breaker limit + iteration-cap backstop
- `internal/session/` — activity signal fed into the heartbeat
- `internal/store/globaldb/` — append-only migrations (L-021)

### Dependent Files

- `internal/api/contract/` + TS codegen — if new run/heartbeat/saturation fields surface
- `internal/api/core` / CLI/UDS — claim errors, saturation diagnostics, terminal-state listing
- `internal/scheduler/` — recovery sweep + starvation escalation respects the cap
- `internal/task/errors.go` — typed already-claimed error if not already present

### Related ADRs

- TechSpec §3.10 (O1–O5) — authoritative design source; no separate ADR (correctness fixes to
  existing behavior, not contested designs). See also Safety Invariants 24 + 26; peer-review
  N-401 / N-302 / B-001.

### Competitor References

- `.resources/hermes/cron/scheduler.py:3182` — `claim_dispatch` at-most-once discipline (policy only)
- `.resources/hermes/tools/delegate_tool.py:1767,1788-1815` — activity-aware staleness (policy only)
- `.resources/hermes/tools/async_delegation.py:96-117` — reject-not-queue concurrency cap (policy
  only; AGH defers durably instead)

## Deliverables

- Atomic status-CAS manual claim + typed already-claimed error; single shared guarded-claim helper
- Per-workspace concurrent-active-run cap in the same claim transaction (defer, never drop)
- Lease-expiry requeue that consumes attempt budget and terminalizes on exhaustion
- Action-node wall-clock timeout + progress-aware heartbeat with idle/in-tool thresholds
- Sibling-resistant loop failure accounting + watch-loop generation backstop
- Indexed wake-dedup replacing the unbounded scan
- Append-only migrations as needed (L-021 trio)
- Every test case in `## Tests` implemented and passing **(REQUIRED)**

## Tests

No `_tests.md` for this suite — concrete inline cases below. Coverage target ≥80% on touched
packages; all Go tests under `-race`.

### Guarded-claim CAS + concurrent-run cap (ex-31 + ex-33 cap / O3+O5)

- Unit (`internal/store/globaldb/*_test.go` + `internal/task/*_test.go` claim suite):
  - [ ] Two concurrent `ClaimRun` on one queued run → exactly one success, one `ErrNoClaimableRun`
        (`-race`)
  - [ ] `ClaimRun` on an already-claimed/running run → typed error, no ownership overwrite
  - [ ] Both claim entry points route through the single shared guarded-claim helper — one fencing
        implementation, structurally verified (Safety Invariant 24)
  - [ ] `ClaimNextRun` behavior unchanged after the extraction (existing claim suite stays green)
  - [ ] Generic `UpdateTaskRun` (non-claim mutations) still succeeds unchanged (no status-guard
        regression)
  - [ ] Workspace at the run cap → new admission deferred (recorded), not dropped; frees when
        capacity opens
  - [ ] Transactional cap bound: two concurrent claims with one slot left → exactly one admitted
        (cap check shares the claim's immediate transaction; `-race`) (N-302 / N-401)
  - [ ] Deferred dependent becomes claimable once an active run completes (no drop)
  - [ ] Cap is per-workspace: saturation in workspace A does not throttle workspace B (isolation)
- Integration (`make test-integration`):
  - [ ] Concurrent claim attempts through the service layer converge to a single owner under `-race`
  - [ ] Wide fan-out completion under the cap admits work in bounded waves (no unbounded pile-up /
        over-spawn)

### Lease-expiry attempt budget (ex-29 / O1)

- Unit (`internal/store/globaldb/*_test.go` task-claim suite + `internal/task/*_test.go`):
  - [ ] Claim → lease expiry → requeue increments attempt/recovery-count (was previously unchanged)
  - [ ] Repeated expiry beyond `max_attempts` → run terminalizes to `failed`/`needs_attention` with
        reason
  - [ ] Snapshot-CAS still rejects a requeue when owner tuple/lease_until no longer matches (fence
        intact)
  - [ ] Migration trio: fresh DB + upgrade/reopen + recorded-prefix (L-021), if a column is added
- Integration (`make test-integration`):
  - [ ] Simulated crash-loop (claim → kill worker → expire) across the real scheduler sweep bounds
        at `max_attempts` and stops re-queuing the same row

### Action-node timeout + progress liveness (ex-32 / O4)

- Unit (`internal/loop/coordinator_test.go` + `internal/task/*_test.go` lease suite):
  - [ ] Action node past its wall-clock window with no progress → terminalizes `node_timeout`,
        lease freed
  - [ ] Heartbeat carrying progress within the window → run stays live (no false timeout)
  - [ ] Idle threshold vs in-tool threshold: a run actively in a long tool is NOT killed at the
        idle window
  - [ ] Long progressing run keeps its lease (no shortening for healthy long work)
  - [ ] Timed-out run consumes the O1 attempt budget (does not reclaim forever)
  - [ ] Migration trio (L-021) for the progress/last-activity column
- Integration (`make test-integration`):
  - [ ] Wedged action-node fixture across the real coordinator+scheduler terminalizes and the loop
        advances

### Loop circuit-breaker (ex-30 / O2)

- Unit (`internal/loop/coordinator_test.go` + `internal/store/globaldb/*_test.go`):
  - [ ] 2-node loop, node A fails / node B succeeds each generation → breaker trips `Stalled` (not
        `Exhausted`)
  - [ ] Interleaved terminal order (B after A) → counter is NOT reset to 0 by B's success
  - [ ] Unbounded watch loop (`IterationCap=0`) with persistent failure → hard generation backstop
        terminates it
  - [ ] Healthy loop (all nodes succeed) → breaker never trips; no false stall
  - [ ] Migration trio (L-021) if a per-node counter column is added
- Integration (`make test-integration`):
  - [ ] Real coordinator over a failing/succeeding two-node loop stalls within the streak limit

### Indexed wake-dedup (ex-33 remainder / O5)

- Unit (`internal/task/wake_test.go` + `internal/store/globaldb/*_test.go`):
  - [ ] Wake dedup with a large event history + evicted cache → single indexed lookup, correct
        dedup verdict
  - [ ] Migration trio (L-021) for the index/dedup table
- Integration (`make test-integration`):
  - [ ] Wake correctness preserved under the indexed path (no false positive/negative dedup)

### E2E

- E2E: N/A for unit lanes — covered by the W7 QA orchestration scenarios (13–17)

## Success Criteria

- Every assigned test case implemented and passing
- Coverage ≥80% on touched packages
- O1–O5 reproductions each terminate with the forensic outcome named in TechSpec §3.10
- Guarded-claim helper + concurrent-run cap share one immediate transaction (N-401 proven)
- Timed-out runs ride the O1 attempt budget (29↔32 coupling closed)
- No behavioral change to non-claim `UpdateTaskRun` callers
- Sibling-resetting global `consecutive_failures` accounting deleted
- Wake dedup cost independent of a task's event count; saturated workspace back-pressures without
  dropping work; workspace isolation holds
