---
status: pending
title: "Calls domain core: schema, admission/activation, settlement"
type: backend
complexity: critical
---

# Task 3: Calls domain core: schema, admission/activation, settlement

## Overview

Build the call primitive's heart: the full feature schema in one migration (00078), the store layer, and `internal/calls.Service` covering create (spawn-or-address, batch, idempotency), the admission/activation split over `call_activation` task_runs, single-writer settlement with the return/repair/extraction pipeline, await, cancel, the deadline authority, fence-first subtree drain, the operator-caller fence, and the daemon-side activation dispatcher + `SessionInvoker`. This is the deepest-risk task of the feature — every safety invariant 1-9 and 3a-3e lands here.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Migration `00078` MUST carry ALL schema of this feature in one gap-free Goose migration, with the declarative sources updated in the same change: new fragment `73_agent_calls.sql` (contract_schemas, calls, call_activation_runs, operator_caller_sessions, call_messages, call_deliveries, call_publications, payload_blobs), the `72_task_runs.sql` CHECK rebuild admitting `call_activation` (+ `task_runs.expect_digest`/`result_budget_bytes`/`result_overflow` snapshot columns in the same table rebuild), `sessions.parked_at`/`idle_expires_at`/`draining_at`, and `tasks.expect_digest`. Append-only identity: existing migration bytes, versions, order, and `atlas.sum` history are immutable (`eng-schema-migration`).
2. Every matchable field is a column — no JSON metadata blobs; the only JSON at rest is schema bytes and content-addressed payloads (Data Models table is the authoritative column list).
3. Admission and activation are separate phases (invariant 3a): the admission transaction commits contract pin + prompt blob + call row + idempotency fence + one `run_kind="call_activation"` task_run for every child-starting call; NO subprocess work inside a store transaction; the post-commit fast path claims that exact run via `task.Service.ClaimNextRun`.
4. Queued execution lives only in `task_runs` (invariant 3b): exact-kind claim selection, the governed-root execution budget (`max_active_per_root`) evaluated INSIDE the claim transaction, no claim/lease columns on `calls`, no component scanning call rows to start work; crash recovery flows through the queue.
5. `calls.Service` settlement methods are the ONLY writers of terminal states; `RequireCallSettlementActor` fences everything else (invariant 1); sanitize→validate→blob→terminal→delivery-row happen in one transaction (invariants 2-3, 10) — the completion delivery ROW is written here; delivery draining/notify is task_04.
6. Terminalizing always fences the activation run first via `ActivationRunCanceler` CAS (invariant 3c); `SweepDeadlines` is the only `timeout` authority (invariant 3e); `DrainSubtree` is fence-first via `sessions.draining_at`, idempotent, boot-resumable (invariant 3d); boot reconciliation repairs every crash window.
7. `max_children` is a per-parent admission wall that REJECTS (typed `call_children_cap`); `max_active_per_root` QUEUES; depth is enforced at admission from durable lineage (invariant 8); cross-workspace targets are denied before any side effect (invariant 9); narrowing is validated pre- and post-hook (invariant 7, existing spawn hooks keep firing).
8. Idempotency is a DB UNIQUE fence on (workspace, caller, key); budget snapshot AND deadline participate in payload identity; replay returns the original marked `replayed`; conflicts are typed.
9. Operator-originated calls resolve to the workspace's durable operator-caller session through `operator_caller_sessions` with conflict-winner semantics inside the admission transaction; the bound session is excluded from targeting, liveness caps, revival, and the reaper; `calls.actor_*` records the authenticated creator separately.
10. Return admission implements the full pipeline: sanitize-first, contract validation, one repair round (delivered to the live child; infrastructure failures don't consume it), single-key unwrap, extraction fallback with `extracted` verdict, strict mode, completed-without-result, budget enforcement against the call's immutable snapshot.
11. `Await` is durable-backed (register-before-snapshot, resume tokens, 30-minute clamp) extending the session-wait canonical machinery; `Cancel` propagates a real stop (managed-stop escalation) and records superseded evidence for late outcomes (invariant 5).
12. The daemon implements `calls.SessionInvoker` over the retained internal spawn engine (the agent-facing spawn surface is deleted in task_05 — the ENGINE stays); the hardcoded `DefaultSpawnMaxChildren`/`MaxActivePerWorkspace` consts are superseded by `[calls]` keys (delete targets).
</requirements>

## Subtasks

- [ ] 3.1 Author fragment `73_agent_calls.sql` + the `72_task_runs.sql` rebuild + sessions/tasks columns; append migration `00078`; refresh `atlas.sum` + sqlc via `make codegen`; extend the canonical fresh/reopen/ahead/integrity/equivalence suites
- [ ] 3.2 Write sqlc queries + repo files for calls, activation side table, operator-caller, registry, payload blobs (workspace+ref keyed, digest re-verified on read)
- [ ] 3.3 Implement `internal/calls` create path: target resolution, roster-aware unknown-agent errors, contract pin, budget/deadline validation + snapshot, narrowing validation, depth admission, child-cap wall, batch with per-item outcomes, idempotency fence, operator-caller resolution, `draining_at` re-validation
- [ ] 3.4 Implement `task.RunKindCallActivation` + the `72` CHECK contract in `internal/task` (validation, exact-kind claim selection, in-claim-tx root budget) + the daemon call-activation dispatcher (network-wake executor precedent) + failed-activation settlement
- [ ] 3.5 Implement settlement: `Return` transaction (sanitize/validate/blob/terminal/delivery-row), repair round, extraction, strict mode, completed-without-result, `RequireCallSettlementActor`, post-terminal rejection + superseded evidence
- [ ] 3.6 Implement `Await` (durable waiter registry, resume, clamp) and `Cancel` (activation-run CAS → managed stop → terminal write)
- [ ] 3.7 Implement `SweepDeadlines` + the daemon deadline ticker; `DrainSubtree` fence-first + the boot recovery that resumes drains and repairs activation/claim crash windows
- [ ] 3.8 Implement `calls.SessionInvoker` in the daemon over the retained spawn engine; delete the superseded spawn consts; add `internal/calls` boundary rules
- [ ] 3.9 Implement every assigned unit + integration case (real SQLite, `-race`); close on `make gate`

## Implementation Details

New package `internal/calls` (interfaces defined where consumed; daemon implements bridges). Store work follows the globaldb conventions exactly: declarative fragments under `internal/store/globaldb/schema/definitions/`, migrations + `atlas.sum` under `schema/migrations/`, per-store `sqlc.yaml` at `internal/store/globaldb/sqlc.yaml`, repos as `global_db_<domain>*.go` with `global_db_<domain>_integration_test.go` suites (`//go:build integration`). The commit-then-notify acceptance transaction and named-skip-reason patterns come from the network store files. `_spec.md` Part II › Data Models is the authoritative column list; System Architecture › Data flow is the authoritative happy path.

### Relevant Files

- `internal/store/globaldb/schema/definitions/72_task_runs.sql:147,158-167,294-323` — the `run_kind` CHECK + taskless invariants + partial indexes being rebuilt (NOT in 60_network.sql)
- `internal/store/globaldb/schema/definitions/60_network.sql` + `20_sessions.sql:140` — fragment style; lineage columns
- `internal/store/globaldb/schema/migrations/00077_schema.sql` + `schema/migrations/atlas.sum` + `internal/store/globaldb/sqlc.yaml` — the migration chain (next: 00078); atlas.sum lives INSIDE migrations/
- `internal/store/globaldb/global_db_network_accept.go:37-97` — the commit-then-notify acceptance transaction to mirror
- `internal/store/globaldb/global_db_network_admission.go:39,150-161` — named skip reasons; owner-key accounting
- `internal/task/lease_manager_claim.go:6` + `internal/store/globaldb/global_db_task_lease_settlement_tx.go:36` — `ClaimNextRun` chain gaining exact-kind selection + in-claim-tx budget
- `internal/task/network_wake_settlement.go` + `internal/task/wake.go` — the taskless-run precedent (`network_wake`) for `call_activation`
- `internal/task/lease_settlement_authority.go:7` — `RequireLeaseSettlementActor`, the single-writer fence being generalized to calls
- `internal/session/spawn.go:17-20,84,131` + `spawn_permissions.go:15,51` + `spawn_governance.go:32` — the retained internal engine + narrowing validation + governance hooks (consts superseded by `[calls]`)
- `internal/session/session_wait.go:12-18,79-127` + `session_wait_registry.go:236-249` — the await skeleton (register-before-snapshot, resume grace)
- `internal/network/participation/owner.go:9` + `types.go:34` — OwnerRef/OwnerKey (consumed, not modified)
- `magefiles/boundaries.go` — gains `internal/calls` rules

### Dependent Files

- `internal/task/**` — run-kind validation, claim selector, canceler implementation (`ActivationRunCanceler`)
- `internal/session/**` — spawn engine consumed via `SessionInvoker`; session columns read by drain/park queries (park behavior itself is task_04)
- `internal/daemon/**` — activation dispatcher, deadline ticker, boot reconciliation, `SessionInvoker` wiring
- `internal/api/testutil/task_stub_run_queue.go:20` — test stub tracking the claim-selector contract change

### Competitor References

- `.resources/omp/docs/tools/task.md:1-176` — spawn-carries-prompt, batch shape, lifecycle end-states, derived-messaging note
- `.resources/omp/packages/coding-agent/src/tools/yield.ts:1-140` + `task/yield-assembly.ts:1-90` — forced terminal return, `schemaOverridden` provenance (our `verdict`)
- `.resources/hermes/tools/delegate_tool.py:4597-4730` — batch schema + control-plane multiplexing; the 4,793-line file is the layout anti-pattern
- `.resources/codex/codex-rs/core/src/agent/control/residency.rs:17-232` + `spawn.rs:257-387` — LRU park/lazy reopen (our parked/revive; park orchestration is task_04, the schema and revive-on-create seam land here)
- `.resources/herdr/src/api/wait.rs:216-248,611-640` — identity-pinned waits + typed stall errors (await semantics)

### Related ADRs

- [ADR-001: Typed call result lives in a first-class durable call record](adrs/adr-001.md) — the record this task creates
- [ADR-002: One agent-facing call verb; the spawn surface is deleted](adrs/adr-002.md) — the engine is retained here; surface deletion is task_05
- [ADR-003: Finished children are parked and revivable; TTL is an idle ceiling](adrs/adr-003.md) — schema columns + revive-on-create seam (orchestration in task_04)
- [ADR-008: Recursive delegation, default max depth 3, budget-based containment](adrs/adr-008.md) — depth admission from durable lineage
- [ADR-009: Async-by-default calls; result-carrying wake; explicit bounded await](adrs/adr-009.md) — create returns immediately; await is explicit
- [ADR-011: Accounting-only call activations in v1 (no admission ceilings)](adrs/adr-011.md) — a committed completion is never admission-denied
- [ADR-013: Digest-keyed contract registry with compiled-schema cache](adrs/adr-013.md) — the registry store lands here (IT-061)

### Web/Docs Impact

- `web/`: none in this task — checked surfaces: no public API contract yet (handlers/routes/types are task_05; generated TS refreshes there); reason: domain + store only.
- `packages/site`: none in this task — docs ship in task_07 against the real surfaces.
- QA impact: none directly user-visible — no CLI verb, route, or UI exists for calls until task_05; the call-journey scenarios are flagged by task_05/task_06 and walked in task_09.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: spawn governance hooks (`spawn.pre_create` etc.) keep firing inside the call spawn path with post-hook re-validation (IT coverage lands with the `call` hook family in task_05); no extension contract changes here — checked: hooks catalog, Host API, bridge SDKs, MCP sidecars.
- Agent manageability: none exposed yet — the service is internal until task_05 lands tools/CLI/HTTP; deterministic error codes are minted here (`call_*` vocabulary) so surfaces map 1:1 later.
- Config lifecycle: consumes `[calls]` keys from task_01 (`max_depth`, `max_batch`, `max_children`, `max_active_per_root`, `idle_ttl`, results.*); deletes the hardcoded `DefaultSpawnMaxChildren`/`MaxActivePerWorkspace` consts (superseded — checked: no `config.toml` key references them; `[roles.coordinator].max_children` stays coordinator-only).

## Deliverables

- Migration `00078` + fragment 73 + rebuilt fragment 72 + sessions/tasks columns; `make codegen-check` green; canonical schema suites extended and green
- `internal/calls.Service` with create/batch/idempotency/return/await/cancel/deadline/drain complete and fenced per invariants 1-9 + 3a-3e
- Daemon activation dispatcher, deadline ticker, boot reconciliation, `SessionInvoker`
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-029, UT-030, UT-031, UT-032, UT-033 — create happy paths (queued→running via claim, uncontracted, prompt blob, runtime override, TTL default)
- [ ] UT-034, UT-035, UT-036, UT-037, UT-038, UT-039 — create errors (roster error, invalid expect pre-spawn, empty prompt, child cap, widening atoms, keyless duplicates)
- [ ] UT-040, UT-041, UT-042 — session-target follow-up, expired-vs-unknown, lineage denial
- [ ] UT-043, UT-044, UT-045, UT-046, UT-047, UT-048, UT-049 — batch semantics (per-item outcomes, distinct digests, over-cap, empty, isolation, mid-batch cap, fast rejection)
- [ ] UT-050, UT-051 — idempotent replay, conflict
- [ ] UT-052, UT-053, UT-054 — cancel running/terminal/queued-item
- [ ] UT-055, UT-056, UT-057, UT-058 — await settled/partial/unknown/clamp
- [ ] UT-059, UT-060, UT-061, UT-062 — return settlement, double return, unbound return, repair round
- [ ] UT-063, UT-064, UT-065, UT-066, UT-067, UT-068 — completed-without-result, prose-only, uncontracted budget, extraction verdict, extracted-invalid repair, strict mode
- [ ] UT-110, UT-111 — drain planner + subtree walk with cycle guard
- [ ] UT-122 — `max_depth` raise forward-only
- [ ] UT-137, UT-138, UT-139 — narrowing composition, post-hook widening rejection, current-effective-set validation
- [ ] UT-140 — cross-workspace denial at the service boundary
- [ ] UT-149 — `SweepDeadlines` selection, fencing, race interleavings
- [ ] IT-001 — migration 00078 through the canonical fresh/reopen/ahead/integrity/equivalence suite
- [ ] IT-002, IT-003, IT-004, IT-005, IT-006, IT-007 — admission tx atomicity, child-cap wall, crash recovery via queue, idempotency race, follow-up, expired-vs-missing
- [ ] IT-010, IT-011, IT-012, IT-013, IT-014 — batch persistence isolation, execution-budget queueing, conflict byte-identity, replay across restart, retention rebirth
- [ ] IT-015, IT-016, IT-017 — cancel propagation, cancel-vs-return race, managed-stop escalation
- [ ] IT-018, IT-019, IT-020 — await edge, await restart, await cap (extends the session-wait canonical suite)
- [ ] IT-021, IT-022, IT-023, IT-024, IT-025 — settlement tx atomicity, double return, return-vs-crash, repair→invalid-result, silence→completed-without-result
- [ ] IT-027, IT-028 — store-overflow blob integrity, extraction verdict persistence
- [ ] IT-048 — depth × batch fan-out against the governed-root cap (visible queue/reject reasons)
- [ ] IT-058 — workspace deletion terminalizes calls/children with typed reasons
- [ ] IT-061 — contracts registry against real store (pin/resolve/dedup, cache across instances)
- [ ] IT-068, IT-069, IT-070, IT-071, IT-072 — CHECK rebuild + exact-kind claim + in-claim-tx budget, deadline races, operator-caller fence, claim-vs-terminalization race, one-schema-two-budgets

## Success Criteria

- Every assigned test case implemented and passing (`-race`; integration on real SQLite through the migration chain)
- All safety invariants 1-9 and 3a-3e hold under the race suites (IT-005, IT-016, IT-069, IT-071 loop clean)
- `make codegen-check` green (schema, sqlc, atlas.sum); append-only migration identity preserved
- File split respected up front (no god files; 500-line cap); `make gate` green at close
