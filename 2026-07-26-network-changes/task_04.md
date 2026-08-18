---
status: pending
title: "Delivery restructure, live admission & projection gating"
type: backend
complexity: critical
---

# Task 4: Delivery restructure, live admission & projection gating

## Overview

Replace the embedded NATS broker with durable-commit-then-in-process dispatch, implement the atomic AcceptNetworkMessage admission transaction (dispositions + budgets + wake enqueue), run wakes via NetworkWakeRunner on ClaimNextRun, settle usage truthfully, and gate env/prompts/situation/coordination tools from the immutable Spec so Local sessions pay zero network context tax. This merges TechSpec Build Order steps 5–7 because broker deletion, acceptance, and projection share one delivery authority.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Embedded broker MUST be fully deleted (ADR-010/N-001): `transport.go`, every NATS import, manager publish/subscribe/flush paths, broker tests, `go.mod` module via `go get` (never hand-edit), broker config keys already removed in task_01, and protocol pages flagged for task_06.
2. `GlobalDB.AcceptNetworkMessage` MUST be the single BEGIN IMMEDIATE acceptance transaction covering conversation row, recipient dispositions from pre-message participant snapshot, admission checks (availability/budgets/wake sources/open-wake unique), wake `task_runs` enqueue, and ledger/sources writes (B-010/B-018); callers fire Notify only after commit.
3. Live admission MUST wake only on direct/mention (never control/status/greet/whois/receipt/trace); coalesce within window into one open wake per owner; enforce depth/budget ceilings; same-envelope idempotency forever via `network_wake_sources`.
4. `NetworkWakeRunner` MUST claim `network_wake` runs for the target session via ClaimNextRun, execute PromptNetwork once, heartbeat with claim token, and Settle terminal outcome; never touch task/kanban state.
5. Settle MUST be open→terminal CAS (B-011): duplicate terminal no-ops; failed turns never `delivered`; missing usage → `usage_unavailable` keeping reserved quantities consumed.
6. Session-keyed durable relations MUST replace peer-id keys (ADR-007); `network_work` transitions only through store repository (C-04).
7. Security non-regression: raw `agh_claim_*` rejected pre-persistence with hash-only diagnostics; verified-format identity without proof → `rejected` not `unverified` (B-005).
8. Projection gating MUST key off Spec: Local sessions get no `AGH_SESSION_CHANNEL`/`AGH_PEER_ID`, no network prompt section, no coordination toolset, no peer join; live gets compact wake header + policy-limited tools.
9. God-file hard cap: split over-cap `manager.go`/`delivery.go`/`router.go`/`validate.go` into single-responsibility files in this change (dispatch, admission, budget, presence, usage) — do not grow them.
10. Availability disable MUST linearize against admission via `network_availability` epoch (IT-039); re-enable must not re-admit settled/accumulated envelopes.
</requirements>

## Subtasks

- [ ] 4.1 Delete broker/NATS transport and sweep all manager/router/delivery call sites + tests + module dependency.
- [ ] 4.2 Implement AcceptNetworkMessage + disposition persistence + post-commit notify-only dispatcher (C-02/C-05).
- [ ] 4.3 Implement admission/budget/ledger/sources inside the acceptance transaction; enqueue network_wake runs.
- [ ] 4.4 Implement NetworkWakeRunner + WakeSettler CAS settlement + usage ledger reads.
- [ ] 4.5 Rebuild session-keyed relations; confine work authority to SQLite.
- [ ] 4.6 Gate harness/env/situation/toolset/peer-join on Spec; delete ChannelBound inference and digest/guidance machinery.
- [ ] 4.7 Split over-cap network files; land adversarial concurrency + security suites.
- [ ] 4.8 Land assigned UT/IT/E2E cases (runtime E2E against acpmock).

## Implementation Details

See TechSpec Core Interfaces (`AcceptNetworkMessage*`, `WakeSettler`, `NetworkWakeRunner`), Safety Invariants 9–16/20–26, Build Order steps 5–7, Delete Targets broker/digest/guidance, ADR-007/008/010/011. Competitor anti-patterns for unbounded fan-out: `.resources/hermes` MoA audit (`.compozy/tasks/network-changes/06-hermes-moa-current-source-audit.md` + `.resources/hermes/hermes_cli/runtime_provider.py`).

Skills to activate: `eng-code-guidelines`, `golang-pro`, `nats` (deletion sweep), `eng-test-conventions`, `testing-boss`, `eng-consolidate-test-suites`, `eng-cleanup-failure-paths`, `extreme-software-optimization` (hot path), `no-workarounds`.

### Relevant Files

- `internal/network/transport.go` — delete
- `internal/network/manager.go`, `delivery.go`, `router.go`, `validate.go` — split + restructure
- `internal/store/globaldb/` — AcceptNetworkMessage owner
- `internal/daemon/` — NetworkWakeRunner composition root; harness_context.go ChannelBound
- `internal/tools/builtin/toolsets.go`, `network.go` — coordination toolset gating
- `internal/situation/service.go` — coordination context injection
- `internal/session/manager_helpers.go` — joinNetworkPeer gate
- `go.mod` — NATS module removal via `go get`

### Dependent Files

- Public usage/coordination HTTP handlers finalize in task_05 but ledger queries must exist now
- Site protocol/NATS pages deleted/rewritten in task_06
- Official skill tooling section rewritten in task_06

### Related ADRs

- [ADR-007](adrs/adr-007.md) — session-keyed relations
- [ADR-008](adrs/adr-008.md) — deterministic admission + aggregate-usage budgets
- [ADR-010](adrs/adr-010.md) — in-process delivery replaces broker
- [ADR-011](adrs/adr-011.md) — wakes on task_runs

## Deliverables

- Broker-free in-process delivery + acceptance transaction
- Live admission/settlement/usage ledger
- Projection gating for Local zero-cost
- File splits under 500-line cap
- Every assigned test case implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [ ] UT-027, UT-028, UT-029, UT-030, UT-031, UT-032, UT-033, UT-034, UT-035 — live admission/coalesce/depth/budget/idempotency
- [ ] UT-036, UT-037, UT-038, UT-039, UT-040 — settlement and usage truthfulness
- [ ] UT-041, UT-042, UT-043, UT-044, UT-045, UT-046 — accept + dispatch ordering/idempotency/work authority
- [ ] UT-063, UT-064 — security non-regression
- [ ] IT-016 — local agent coordination tool → `not_participating`
- [ ] IT-023, IT-024, IT-025, IT-027 — send→wake→settle, burst/depth, cancel, restart recovery
- [ ] IT-037, IT-038, IT-039, IT-040 — adversarial concurrency, wake path, availability linearization, security sweep
- [ ] E2E-001 — light user stays fully local (runtime)
- [ ] E2E-006 — live bounds exhaustion (runtime)
- [ ] E2E-010 — explicit live collaboration journey under new dispatcher (runtime)

## Success Criteria

- Every assigned test case implemented and passing
- No NATS dependency remains in go.mod or production imports
- Local E2E shows zero network rows/env/tools/wakes/usage
- One open wake per owner under adversarial concurrency; CAS settlement never double-applies

### Web/Docs Impact

- `web/`: none required — checked: delivery internals; usage/conversation panels are task_05.
- `packages/site`: protocol/nats + delivery docs become false — flag for task_06 hard-cut; do not leave broker subjects as live truth.
- QA impact: flag network delivery/live-bounds/local-default runtime scenarios `untested`.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: extension host network API MUST return `not_participating` for Local; hook dispatch for wake events at call sites (not event-table tails).
- Agent manageability: `agh ch *` for Local sessions returns typed `not_participating`; full CLI verb polish in task_05.
- Config lifecycle: none new — checked: availability row already from task_01; live defaults applied only on live resolve.
