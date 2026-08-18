---
status: completed
title: "Attention truth core: canonical records, badges, presence leases, discovery + CLI"
type: backend
complexity: critical
---

# Task 1: Attention truth core: canonical records, badges, presence leases, discovery + CLI

## Overview

Delivers the attention foundation every other task consumes: the two new badge tokens (`waiting-for-input`, `done`) with their total precedence order and attention classes, the canonical `session_pending_interactions` store + `sessions` projection/fence columns in one global-catalog transaction, per-client presence leases, the widened catalog wake filter, the `session_attention_changed` event, operator-only presence and attention-summary routes, and same-workspace interaction discovery with their CLI surfaces (`list --attention/--badge/--all-workspaces/--summary`, `session interactions`). This is P1 of `_spec.md` Build Order plus the CLI truth surfaces of P2 — it closes its own public loop (contracts, OpenAPI specs, HTTP/UDS parity, codegen, docs) per round-2 B-112.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Badge vocabulary MUST gain `waiting-for-input` and `done` under the exact precedence order in `_spec.md` Core Interfaces; `BadgeInputs` gains `PendingClarify` and `Unseen`; `ClassForBadge` maps the needs-you/finished/none classes. One derivation authority — no surface computes attention independently.
2. Schema MUST follow `eng-schema-migration`: declarative source updates + next gap-free Goose migration for `session_pending_interactions` (daemon-owned `interaction_id` PK, unique `(session_id, kind, provider_request_id)` among pending) and the `sessions` columns (`pending_permission_count`, `pending_clarify_count`, `attention_revision`, `last_settled_revision`, `last_seen_revision`, `last_seen_at`, `attention_changed_at`); `atlas.sum` + sqlc via `make codegen`; append-only identity; no boot repair.
3. Canonical writes MUST commit in ONE global-catalog transaction (side-table + projections + revisions + `attention_changed_at`); transcript events append after as observational projections; publishes (catalog wake + attention event) only after the canonical commit — Safety Invariants 2, 3, 16.
4. `Unseen` MUST derive from `last_settled_revision > last_seen_revision` — transcript sequences are never consulted (round-2 B-102).
5. Presence MUST be a per-client lease: acquire returns opaque `lease_id`; renew/release require the caller's own id; settle evaluation linearized against the lease map; a settle under any live lease marks seen and never derives `done` (Safety Invariant 12).
6. Pending-interaction content MUST pass the secret redactor and bounds (title ≤200, `payload_json` ≤4KB allowlisted, resolution ≤240) before every persistence and publication boundary (Safety Invariant 11).
7. Orphaned-interaction resolution MUST commit resolution + `session_input_queue` insert atomically; queue-full aborts everything with the retryable `queue-full` outcome; duplicate resolution is idempotent on `interaction_id` (Safety Invariant 15).
8. The catalog wake filter MUST widen to `{permission, clarify, done, error}`; seen-clears publish the normal catalog wake before the attention event.
9. New routes MUST land on HTTP AND UDS with `internal/api/spec` OperationSpecs and updated route-inventory parity tests: `POST …/presence`, `GET …/interactions`, `GET /api/sessions/attention-summary`. Presence and attention-summary MUST deny agent identity with the deterministic `403 agent_scope_denied` shape on both transports; interactions MUST accept a validated agent identity only in the caller's workspace (Safety Invariant 14).
10. CLI MUST ship `session list --attention/--badge/--all-workspaces/--summary` and `session interactions <id>` with human + toon formatters and the documented exit codes; `SessionPayload` gains `attention_changed_at`; detail/status payloads embed sanitized pending interactions.
11. The `session.attention.changed` hook event family MUST follow the window-manager exemplar (definitions, payload with `at`, typed dispatch post-commit/cloned/fail-open, introspection, async clone).
12. MUST NOT grow near-cap files — new siblings per `_spec.md` Architectural Boundaries (e.g. `session_attention.go`, `session_pending.go`, `session_presence.go`, `session_seen.go`).
13. Docs MUST ship in this task: `skills/compozy/references/runtime-operations.md` (attention vocabulary, interactions, summary), `configuration.md` if touched, and the `packages/site` session reference pages for the new routes/verbs (generated CLI/API refs regenerated).
</requirements>

## Subtasks

- [x] 1.1 Schema: declarative sources + Goose migration for the side-table and seven `sessions` columns; extend the canonical fresh/reopen/ahead/integrity/equivalence suites; `make codegen`.
- [x] 1.2 Badge derivation: new tokens, `BadgeInputs` fields, total precedence, `ClassForBadge`; wire `Info()` assembly from the counts and revisions.
- [x] 1.3 Canonical pending-interaction lifecycle: create/resolve/timeout/cancel/orphan transitions in one-transaction coupling with projections; redaction + bounds at every boundary; orphan resolution with atomic queue insert and `queue-full`/`already-resolved` outcomes.
- [x] 1.4 Presence leases: per-client lease map with TTL/renewal, `MarkSessionSeen` (revision-based), linearized settle evaluation, seen-clear wake+event publication.
- [x] 1.5 Attention transitions: recompute at the lifecycle choke point + event-storage paths; `attention_revision`/`attention_changed_at` bumps; `session_attention_changed` publish after canonical commit; widen the wake filter.
- [x] 1.6 Hook family `session.attention.changed` per the exemplar file set.
- [x] 1.7 Routes: presence / interactions / attention-summary on HTTP+UDS with specs and parity-test updates; operator-scope denial for presence/summary and same-workspace agent authorization for interactions.
- [x] 1.8 Contracts + codegen: payload additions (`attention_changed_at`, attention event, presence, interactions, summary), `make codegen`, web generated types, MSW/acpmock fixture updates for the new badges/events.
- [x] 1.9 CLI: list filters (`--attention`, `--badge`, `--all-workspaces`, `--summary`) + `session interactions` verb with formatters and exit codes.
- [x] 1.10 Docs: skill references + site pages for everything this task ships.

## Implementation Details

Reference `_spec.md` Part II (Core Interfaces, Data Models, API Endpoints, Safety Invariants 1–4, 11–16) and ADR-005. Verified anchors from this session's exploration:

### Relevant Files

- `internal/session/badge.go:11-99` — current 8-token vocabulary, `BadgeInputs`, `CanonicalBadge` precedence, `BadgeForInfo`/`BadgeForHealth` entry points (161 lines — extend in place, keep under cap).
- `internal/session/session_info.go:12-69` — `Info()` assembly; already computes `pendingPermission`; new counts/revisions surface here.
- `internal/session/manager_transition.go:16-54` — `persistSessionLifecycleStateLocked`, the lifecycle choke point all durable attention writes ride.
- `internal/session/session_catalog_stream.go:22-27,119-124` — `CatalogEvent` shape + the permission-only wake filter to widen; 64-slot broadcaster with slow-subscriber drop.
- `internal/session/manager_clarify_event.go:19-68` — current clarify append+publish flow to re-route through the canonical store.
- `internal/acp/permission_pending.go:15-92` — in-memory pending permission map to project into the canonical store.
- `internal/store/globaldb/schema/definitions/20_sessions.sql` — sessions + `session_input_queue` declarative source (same physical DB — the atomicity boundary).
- `internal/api/core/session_catalog_stream.go:12-65` — SSE handler, `session_catalog_changed` event name, `PrepareSSE`/`WriteSSE` helpers.
- `internal/api/contract/session_runtime_payloads.go:61-92` + `session_catalog.go:11-15` — payload homes.
- `internal/api/spec/session_catalog.go` — OperationSpec pattern; `internal/api/httpapi/handlers_test.go:240` + `udsapi/handlers_test.go:236` route-inventory parity tests.
- `internal/hooks/events_window_manager.go` + `dispatch_window_manager.go:5-60` + `payloads_window_manager.go` + `introspection_window_manager.go` + `async_clone_window_manager.go` — the exemplar file set for the new event family.
- `internal/cli/session_list.go:36,149` + `internal/cli/format.go:24-154` (`writeCommandOutput`, `listBundle`) + `internal/cli/session_clarify.go:135-155` (small-object bundle exemplar) — CLI conventions.
- `internal/tools/dispatch_workspace_access.go:21-72` + `internal/daemon/native_tool_scope.go:217` — trusted-scope classification the operator-denial mirrors.
- `internal/agentidentity/errors.go:20-35` — exit-code constants (65/69/75/78 conventions).

### Dependent Files

- `internal/daemon/clarify_bridge.go:23-33` — boot-scoped clarify broker becomes a consumer of the canonical store.
- `web/src/generated/` types + `web/src/systems/session/mocks/{handlers,fixtures}.ts` + `internal/testutil/acpmock/` — co-ship per `eng-contract-codegen-coship`.
- `web/src/systems/os/lib/attention-model.ts:26` — single-criterion predicate task_03 rewrites against this task's badge classes.
- `skills/compozy/references/runtime-operations.md` (§ session verbs, line ~164 area) + `packages/site/content/` session reference pages.

### Related ADRs

- [ADR-001: Attention state model](adrs/adr-001.md) — vocabulary, precedence, done/seen rules this task implements.
- [ADR-005: Attention truth pipeline](adrs/adr-005.md) — canonical store, presence leases, revision fence, event publication (as amended by both review rounds).

### Competitor References

- `.resources/herdr/src/detect/mod.rs:10` + `src/pane/state.rs:10` + `src/app/api_helpers.rs:99` — AgentState + `seen` bit → derived status mapping (`done` server-derived, never reportable).
- `.resources/herdr/src/app/actions.rs:3096` — seen-clearing semantics ("CLI reads don't mark seen"; completion-transition rule).

### Web/Docs Impact

- `web/`: `web/src/generated/compozy-openapi.d.ts` (regenerates), `web/src/systems/session/mocks/handlers.ts` + `fixtures.ts` (new badges/events/payload fields), `web/src/systems/os/mocks/*` (catalog-stream fixtures). Consuming UI lands in task_03/task_06.
- `packages/site`: session reference pages for `POST …/presence`, `GET …/interactions`, `GET /api/sessions/attention-summary`; generated CLI docs for `session list` flags + `session interactions`; `skills/compozy/references/runtime-operations.md`.
- QA impact: new scenarios — add content-addressed `untested` files under `docs/qa/scenarios/` for: badge honesty for question/permission across restart, done/seen via presence, `session list --attention/--all-workspaces/--summary`, `session interactions` discovery + orphan resolution. Flag only — task_08 walks them.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: new hook event family `session.attention.changed` (introspection-discoverable); no manifest/provide/permission surface changes (checked: extension manifests, bridge SDKs, MCP sidecars — unaffected).
- Agent manageability: badge vocabulary + `attention_changed_at` + embedded pending interactions on `session_list`/`session_status` structured outputs; CLI filters/verbs above; deterministic errors per `_dx.md`; operator-scope surfaces deny agent identity (`agent_scope_denied`).
- Config lifecycle: none — checked surfaces: `internal/config/` (no new keys in this task; `[attention]` lands in task_02); reason: attention truth is state, not configuration.

## Deliverables

- Migration + canonical store + badge derivation + presence leases + transitions live behind `make gate` (Go lanes green).
- All three new routes on both transports with parity tests updated; codegen artifacts current (`make codegen-check`).
- CLI surfaces shipped with formatters and exit codes exactly as `_dx.md`.
- Hook family dispatchable and introspectable.
- Skill + site docs for every surface above.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-001..UT-011 — badge derivation, precedence, classes, transition publisher
- [x] UT-024..UT-026 — seen path (revision-based, monotonic, idempotent)
- [x] UT-080 — revision fence + `attention_changed_at` ordering incl. seen-clears
- [x] UT-081 — per-client lease races and settle linearization
- [x] UT-082 — pending-interaction redaction and bounds
- [x] UT-086 — hook contract (post-commit, cloned, fail-open, no event-tailing)
- [x] IT-001..IT-006 — migration suites, transactional coupling, wake filter, event-after-commit
- [x] IT-009 — presence lease lifecycle over the route
- [x] IT-018 — CLI list filters (attention/badge/all-workspaces)
- [x] IT-022, IT-023 — clarify timeout clearing; done/seen restart durability (IT-021 is withdrawn — codegen drift is gate-owned; cite `make codegen-check` as evidence)
- [x] IT-029 — pending lifecycle across restart, orphan resolution, queue-full, idempotency
- [x] IT-030 — operator-scope authorization denials (both transports)
- [x] IT-032 — interactions discovery (route/CLI/embedded projection)
- [x] IT-033 — attention-summary exactness at 100+ sessions + `--summary`
- [x] E2E-001..E2E-003 — clarify/permission/done journeys via acpmock + CLI transcripts

## Success Criteria

- Every assigned test case implemented and passing.
- A session asking a question shows `waiting-for-input` within the latency target, survives daemon restart, and clears everywhere on answer (US-001 ACs).
- `done` derives only daemon-side; a settle under a live lease never produces `done`; CLI reads never clear it (US-002 ACs).
- Route inventories identical on HTTP and UDS including the three new routes; `make gate` green; no near-cap file grew.
