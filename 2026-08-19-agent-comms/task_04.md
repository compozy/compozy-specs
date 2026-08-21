---
status: pending
title: "Mailbox, delivery, and lifecycle: park/revive, loop-breakers"
type: backend
complexity: high
---

# Task 4: Mailbox, delivery, and lifecycle: park/revive, loop-breakers

## Overview

Make communication move: the lineage mailbox (send, receipts, loop-breakers), the durable delivery machinery that drains completion and message rows to recipients at turn boundaries (or wakes idle ones) with crash-safe exactly-once semantics, the exact completion-wake texts, and the park/revive lifecycle with the idle-TTL reaper changes. This turns task_03's committed rows into delivered outcomes while honoring non-interruption and commit-then-notify everywhere.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. `SendMessage` MUST commit the durable row before any notification (commit-then-notify); bodies pass through `contracts.SanitizeText` BEFORE dedup hashing, hooks, receipts, events, and persistence (invariant 10, IT-059c); receipts come from the closed projection of the ONE delivery state machine (`pending→queued`, `injected→delivered-into-turn`, `woken→woke`, `failed→failed`) — no second delivery authority.
2. Loop-breakers run inside the message-accept transaction (invariant 13): per-sender rate limit (`message_rate_limited`), identical-repeat dedup (typed `message_duplicate` rejection carrying the original id), pending-cap on queued-undelivered transport backlog (no read/seen state anywhere), size cap. Every engagement is typed and observable, never silent.
3. Delivery is non-interrupting: boundaries only via the synthetic-prompt seam gaining durable backing; a busy recipient's rows drain at its next boundary in stable order; an idle (parked) recipient is revived and woken — waking consumes a turn on the shared owner-key accounting substrate (ADR-004, ADR-011: a committed completion is NEVER admission-denied).
4. `wake_event_id` dedupe is durable (invariant 12): restart with pending deliveries delivers exactly once; the in-memory LRU is replaced, not wrapped.
5. Completion wakes carry call identity + terminal state + exactly the applicable payload (invariant 4), rendering the exact `_dx.md` wake texts for result-carrying AND resultless terminals.
6. Park/revive: settle with an empty queue parks the child (runtime stopped, `parked_at` set, idle clock armed); any call or message revives it; the idle clock is NULL while a call is in flight; revival never rejects for capacity — over the root budget it queues its activation run (US-018.EC-1, UT-106).
7. The reaper gains idle-clock semantics and NEVER reaps a session with an open call (invariant 11); operator-caller sessions are excluded entirely; queued messages for an expired/drained target terminalize `failed` with that reason (no retry knob).
8. Delivery-failure policy is bounded: repeated failures terminalize `failed` with a named reason — no infinite retry loop.
</requirements>

## Subtasks

- [ ] 4.1 Implement `SendMessage` with sanitize-first, loop-breakers in the accept transaction, and receipt projections
- [ ] 4.2 Implement the delivery drainer: boundary injection through the synthetic-prompt seam (durable backing), stable ordering, busy-vs-idle paths, durable `wake_event_id` dedupe, boot drain
- [ ] 4.3 Implement revive-and-wake for idle recipients on the shared accounting substrate (`owner_key` rows; no double-booking)
- [ ] 4.4 Render the exact completion-wake and resultless-wake texts + structured metadata from `_dx.md`
- [ ] 4.5 Implement park transition on settle-with-empty-queue and revive orchestration on contact (call and message paths), including queued-revival over the root budget
- [ ] 4.6 Rework the reaper: idle-clock candidate query, open-call exclusion, operator-caller exclusion, queued-message terminalization on expiry/drain
- [ ] 4.7 Wire the mailbox/delivery/park events into the canonical event stream (names per Monitoring and Observability; the coverage-matrix extension lands in task_05 with the full family)
- [ ] 4.8 Implement every assigned case including the IT-059 sanitize sweep and the acpmock boundary-delivery suites; close on `make gate`

## Implementation Details

Mailbox + delivery + park/revive live in `internal/calls` (service methods + delivery contracts) with daemon-owned draining (the `DeliverAtBoundary` seam of `SessionInvoker`) and `internal/session` park/reaper edits. Schema already exists (task_03's fragment 73). acpmock-backed integration suites (busy-child steering, wake carries result) follow the `internal/testutil/e2e` harness + `internal/testutil/acpmock` driver conventions.

### Relevant Files

- `internal/session/synthetic_prompt.go:159-177,347-365` — the boundary-delivery seam gaining durable backing
- `internal/session/spawn_wake.go:86-97,108-119,140-146` — redact-then-bound, the wake-reason hole being fixed, content-addressed event ids (the in-memory dedupe being replaced)
- `internal/daemon/spawn_reaper.go:161-193,230-261` — the reaper gaining idle-clock semantics
- `internal/store/globaldb/global_db_network_accept.go:37-97` — commit-then-notify transaction shape
- `internal/store/globaldb/global_db_network_admission.go:150-161` — owner-key accounting rows (ADR-011 substrate)
- `internal/testutil/acpmock/` + `internal/testutil/e2e/runtime_harness.go` — the mock-agent driver + harness for boundary-delivery integration suites
- `internal/hooks/dispatch_loops_spawn.go:133` — dispatch-at-transition pattern for event emission

### Dependent Files

- `internal/calls/**` — service gains message/delivery/park methods over task_03's store layer
- `internal/session/**` — park transition + reaper candidate queries
- `internal/daemon/**` — delivery drainer wiring, boot drain, reaper schedule
- `internal/config/**` — consumes `[calls].messages.*` + `idle_ttl` (read-only)

### Competitor References

- `.resources/omp/packages/coding-agent/src/irc/bus.ts:1-150` — delivery receipts vocabulary; in-memory volatility to avoid
- `.resources/codex/codex-rs/core/src/session/input_queue.rs:121-263` — mailbox delivery phases (defer after answer, reopen on steer)
- `.resources/hermes/tools/async_delegation.py:1-40` — completion as a new turn when idle; never splice; self-contained payload

### Related ADRs

- [ADR-003: Finished children are parked and revivable; TTL is an idle ceiling](adrs/adr-003.md) — park/revive + idle-clock semantics
- [ADR-004: Hybrid cost semantics — durable delivery; wake consumes a turn on the shared substrate](adrs/adr-004.md) — the delivery/accounting contract
- [ADR-009: Async-by-default calls; result-carrying wake; explicit bounded await](adrs/adr-009.md) — the wake carries the result
- [ADR-011: Accounting-only call activations in v1 (no admission ceilings)](adrs/adr-011.md) — completions are never admission-denied

### Web/Docs Impact

- `web/`: none in this task — checked surfaces: no public contract/route/type change (delivery + receipt projections become public in task_05); reason: internal machinery.
- `packages/site`: none in this task — the mailbox docs page ships in task_07.
- QA impact: none directly user-visible — messaging verbs/routes land in task_05; the mailbox scenarios are flagged there and walked in task_09.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: transition points for `call.message_sent` / `call.message_delivered` emit canonical events here; the hook-family catalog registration + dispatch assertion is task_05 — checked: hooks catalog, Host API, bridge SDKs, MCP sidecars (no contract change here).
- Agent manageability: none exposed yet — `compozy message` verbs and `/messages` routes are task_05; typed error vocabulary (`message_*`) is minted here for 1:1 surface mapping.
- Config lifecycle: consumes `[calls].messages.rate_limit_per_minute/dedup_window/pending_cap/max_bytes` and `[calls].idle_ttl` (IT-062 proves config→enforcement); no new keys; no removed keys.

## Deliverables

- Mailbox send/receipts/loop-breakers complete with sanitize-first and typed observable rejections
- Durable delivery drainer with boundary injection, stable ordering, exactly-once crash semantics, and the exact `_dx.md` wake texts
- Park/revive lifecycle + reaper idle-clock semantics with open-call and operator-caller exclusions
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-069, UT-070, UT-071 — send durable-first, lifecycle-aware receipts, over-size rejection
- [ ] UT-072, UT-073 — blocked-target refusal, other-parent lineage denial
- [ ] UT-074, UT-075 — closed receipt vocabulary, unknown message id
- [ ] UT-076, UT-077 — rate limit, dedup window drop/deliver
- [ ] UT-079, UT-080 — provenance-stamped bounded rendering, operator-claim contradiction
- [ ] UT-081, UT-082 — completion wake payload text/meta, delivery vocabulary excludes budget-denial (storm: none denied)
- [ ] UT-103, UT-104, UT-105 — reaper open-call exclusion, park transition, idle-clock clearing on contact
- [ ] UT-106, UT-107, UT-108, UT-109 — revival queues over budget, definition drift flag, TTL error payload, config forward-only
- [ ] IT-008, IT-009 — sequential boundary delivery of queued calls, create-racing-reaper determinism
- [ ] IT-026 — TTL-kill mid-work classifies `failed`, never completed-without-result
- [ ] IT-029, IT-030, IT-031, IT-032 — commit-then-notify, receipt transitions, restart exactly-once, message-vs-terminal ordering
- [ ] IT-033, IT-034 — busy-child boundary steering (acpmock, no mid-tool interruption), delivery-failure policy terminal
- [ ] IT-035, IT-036 — reply-loop brakes engage observably, burst is not misclassified
- [ ] IT-037 — capability subtraction (pending prompt untouched, embedded commands inert, no permission laundering)
- [ ] IT-038, IT-039, IT-040 — wake carries result ref (acpmock), stable multi-completion order, owner-key accounting without double-booking
- [ ] IT-041, IT-042 — park→revive context preservation, racing revivals converge with queued-over-budget activation
- [ ] IT-043, IT-044 — full drain outcomes with parked children, crash-recovery drain equivalence
- [ ] IT-059 — sanitize-before-everything sweep (payloads, errors, hooks, logs, SSE, messages, delivered frames)
- [ ] IT-062 — config overlay + `config set` → enforced pending_cap
- [ ] IT-065 — reaper sweep mixed fixture

## Success Criteria

- Every assigned test case implemented and passing (`-race`; acpmock suites through the e2e harness conventions)
- Exactly-once delivery proven across restart (IT-031); no interruption of a running tool ever observed (IT-033 event-stream assertion)
- Wake texts byte-match `_dx.md` for result-carrying and resultless terminals
- `make gate` green at close
