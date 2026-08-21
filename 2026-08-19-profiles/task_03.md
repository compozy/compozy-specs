---
status: pending
title: Profile Domain and Lifecycle Operation Protocol
type: backend
complexity: critical
---

# Task 3: Profile Domain and Lifecycle Operation Protocol

## Overview

Builds `internal/profile` — the concrete `Manager` with the full struct set from Part II Core Interfaces, the revisioned plans, the prepare → apply → finalize lifecycle protocol (durable journal, seed snapshots, name/path reservations, availability gate, forward-only recovery, explicit `RetryOp`), the selection store and resolver, and the archive-correctness primitives that live outside the package: the `ClaimNextRun` owner-active predicate and the notification owner-active permit protocol. Boot reconciliation runs before the daemon accepts traffic.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

**Skills**: `eng-code-guidelines`, `golang-master`, `eng-test-conventions`, `testing-boss`, `eng-cleanup-failure-paths`.

<requirements>
- MUST implement `*profile.Manager` with the exact method set and struct definitions in `_spec.md` Core Interfaces (signatures final); interfaces are defined where consumed, with compile-time assertions added at daemon wiring.
- MUST decide the file split BEFORE writing (500-line production cap): contract/types, selection store, resolver, plans, one lifecycle operation concern per file — never a god file.
- MUST implement the Lifecycle Operation Protocol exactly: read-only prepare with `Revision` fingerprint; `BEGIN IMMEDIATE` apply that re-validates revision (`profile_plan_stale`), commits rows + journal (+ seed snapshot for declared creations) atomically; forward-only, idempotent, containment-checked finalize; `failed` ops stay reserved and unavailable until explicit `RetryOp`; boot auto-resumes only `applied`/`finalizing`, before traffic.
- MUST enforce name/path reservations from non-`done` journal rows (`profile_name_taken` naming the holding operation) and the availability gate (`profile_unavailable`) in the resolver, selection, and creation paths (Safety Invariants 17, 19).
- MUST implement archive per ADR-001 in one immediate transaction: running-session block, leased-run block, queued-run freeze count, automation pause list, permit-row refusal (`profile_deliveries_in_flight`, retryable), selections preserved; unarchive lists paused automations for explicit reactivation.
- MUST extend the single claim query so `ClaimNextRun` additionally requires the owning profile to be `active` — signature unchanged, sole claimer preserved, no peer gate, no second queue (L-003/L-005).
- MUST implement the notification owner-active permit protocol (Safety Invariant 18): acquire (INSERT verifying owner active) before any external send, hold through acknowledgement and cursor advance, clear in the advancing transaction; crash recovery replays surviving permits by deterministic delivery id — no duplicates.
- MUST implement delete with the zero-owned-work gate across all 17 roots in one transaction, plus the total `RemovalSummary` enumeration; archive leaves selection rows (fallback provenance `archived_remembered_fallback`), delete sweeps them (Safety Invariants 7, 10, 16).
- MUST implement the resolver rules verbatim: daemon-validated `SessionProfileID` is authoritative (differing flag/env on an acting path → `profile_session_conflict`); otherwise flag → env → remembered(lens) → default; unknown/archived/unavailable explicit selection is a typed error, never a fallback.
- MUST define the lifecycle-op retention policy here (round-8 deferred note, L-019 posture): bounded pruning of `done` ops/steps that NEVER erases pending/failed ops nor the needs-setup authority (`profile_credential_requirements` answers needs-setup after `done` — IT-085 pins this in task_09); retention behavior documented on the ops surface.
- MUST add the `internal/profile` boundary rules in `magefiles/boundaries.go` in the same commit that creates the package (imports allowed: `internal/store/globaldb`, `internal/config` paths, `internal/logger`; forbidden: daemon, api/*, cli, session, task, workspace, extension) — add the package before the rule (`os.Stat` skip).
- IT-078 MUST exercise deterministic-delivery replay through the real restartable bridge delivery substrate, not only an in-memory fake (round-8 deferred note, L-026).
</requirements>

## Subtasks

- [ ] 3.1 Create `internal/profile` with the typed contract files: `Profile`/inputs/plans/results/support types, name grammar + reserved names, state machine.
- [ ] 3.2 Implement catalog CRUD + identity updates (symbol exclusivity swap, auto-assign) over the task_02 tables.
- [ ] 3.3 Implement the selection store (`Get/List/Put/SweepProfile`; real profiles only) and the resolver (`Resolve` with source + note taxonomy).
- [ ] 3.4 Implement prepare/plan builders (rename/archive/delete) with revision fingerprints feeding the frozen dialog payloads.
- [ ] 3.5 Implement apply + journal + reservations + seed-snapshot persistence; finalize executor with idempotent, containment-checked steps; `ListOps`/`RetryOp`; retention policy.
- [ ] 3.6 Implement boot reconciliation ordered before traffic acceptance; emit `profile.lifecycle_op_recovered/failed` events.
- [ ] 3.7 Extend `ClaimNextRun`'s claim query with the owner-active predicate (task service suite owns UT-087).
- [ ] 3.8 Implement the notification delivery permit acquire/hold/clear protocol against `notification_delivery_permits`, wired into the dispatch and cursor-advance paths, with archive's refusal arm.
- [ ] 3.9 Wire daemon composition: Manager construction, consumer-side interface assertions, boot ordering; boundaries entries.
- [ ] 3.10 Land the race and recovery suites: rename-vs-create/delete, archive-vs-claim, concurrent same-name create, crash between apply and finalize, permit crash replay on the real bridge substrate.

## Implementation Details

The protocol section of `_spec.md` ("Lifecycle Operation Protocol") is the normative algorithm; the workspace two-phase CRUD is the house skeleton. IT-078's home: extend the restartable-delivery canonical suites rather than a new harness — `internal/extension/bridge_delivery_integration_test.go` (`TestBridgeDeliveryIntegrationShouldReconcileFreshBrokerOverSameStore` is the restart/reconcile path) and the durability cluster in `internal/bridges/delivery_broker_test.go`; `internal/store/globaldb/global_db_bridge_delivery_test.go` (`TestGlobalDBBridgeDeliveryMetricsSurviveRestart`) anchors store-level restart.

### Relevant Files

- `internal/workspace/workspace.go:14-80` + `internal/workspace/resolver_crud.go:13-122` — sentinel-error set + two-phase CRUD templates.
- `internal/store/globaldb/` — task_02's tables (journal, permits, selections, requirements) + new query files for the domain.
- `internal/cli/workspace_resolution.go:15-24,84-146,338-393` — source taxonomy + session-identity derivation the resolver mirrors (consumed in task_04; the domain type ships here).
- `internal/notifications/scope.go:19-45` + `internal/notifications/` dispatch/cursor-advance paths — permit protocol wiring.
- `internal/bridges/task_notifier_delivery.go`, `internal/bridges/task_notifier_resolution.go`, `internal/extension/manager_bridge_delivery.go` — the real delivery substrate IT-078 rides.
- `internal/notifications/delivery_id_test.go` + `internal/store/globaldb/global_db_notification_cursor_test.go` — deterministic delivery-id + cursor canonical suites.
- Task service claim path (single claim query; `ClaimNextRun`) — owner-active predicate site; canonical suite `internal/task` service tests.
- `internal/daemon/` — composition root wiring + boot ordering.
- `magefiles/boundaries.go:21-148` — rule shape (`directImportBoundary`); add the 4–6 entry fan for `internal/profile`.

### Dependent Files

- `internal/api/core` (task_04) — consumer-side `profileLifecycle`/`profilePlans` seams assert against `*profile.Manager`.
- `internal/automation` — pause/reactivation hooks invoked by archive/unarchive results.
- `internal/vault` — rename apply invokes the `vault:profiles/<old>/…` rewrite step (full namespace capability in task_08; the rewrite step ships behind the Manager's explicit rewrite list).

### Competitor References

- `.resources/hermes/hermes_cli/profiles.py:1-58` — lifecycle verb anatomy; the `default`-asymmetry to reject (uniform `default`, permanence enforced here).

### Related ADRs

- [ADR-001](adrs/adr-001.md) — archive semantics, claim predicate, queued freeze.
- [ADR-002](adrs/adr-002.md) — `CreateDeclared` + seed snapshot (consumed by task_09).
- [ADR-003](adrs/adr-003.md) / [ADR-014](adrs/adr-014.md) — selection semantics; daemon-owned remembered choice.
- [ADR-012](adrs/adr-012.md) — id/name identity; rename rewrite list.
- [ADR-015](adrs/adr-015.md) — ReadScope contract (type ships here; sweep in task_06).

## Deliverables

- `internal/profile` package complete per Core Interfaces, split per responsibility, with boundaries rules.
- Lifecycle protocol end-to-end: plans, journal, reservations, availability, forward-only recovery, explicit retry, retention policy.
- Claim predicate and permit protocol live in their owning packages.
- Boot reconciliation ordered before traffic; lifecycle events emitted.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-001..UT-016 — domain state machine: create/identity/rename/archive/unarchive/delete, grammar, reserved names, permanence, idempotency, total enumeration, list boundary.
- [ ] UT-018, UT-019, UT-020, UT-021, UT-023 — resolver chain, selection-store hygiene, fallback notes, typed selection errors, env-no-persist.
- [ ] UT-086, UT-088, UT-089 — plan-revision staleness; symbol exclusivity; `work_items` contract.
- [ ] UT-087 — claim eligibility predicate (canonical suite: task service).
- [ ] UT-091 — session-identity conflict vs flag/env.
- [ ] UT-093 — availability predicate on pending ops.
- [ ] IT-067, IT-068 — archive-vs-claim serialization; queued-run freeze/unfreeze.
- [ ] IT-072 — crash between apply and finalize converges deterministically at boot.
- [ ] IT-078 — owner-active permit protocol at all three barriers, on the real restartable bridge delivery substrate (L-026).
- [ ] IT-080 — name/path reservations under racing create/rename/delete; recovery never touches a replacement profile.

### Web/Docs Impact

- `web/`: none — domain-only; no route or component exists yet (surfaces in tasks 04/05).
- `packages/site`: none in this task — lifecycle docs ship with task_04's capability slice.
- QA impact: none — no user-visible behavior change (no surface exposes the domain yet).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: `CreateDeclared` + seed snapshot + markers contract defined here for task_09's install pipeline; bridge SDKs/MCP unaffected (checked: no manifest/publish surface touched).
- Agent manageability: domain errors are the deterministic error catalog (`profile_*` codes) that task_04 maps to CLI/API payloads; `ListOps`/`RetryOp` back the ops surface.
- Config lifecycle: none — no keys; retention policy is code + ops-surface documentation (task_04 docs slice), not a config knob.
