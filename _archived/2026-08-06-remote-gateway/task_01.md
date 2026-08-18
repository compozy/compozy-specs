---
status: completed
title: Gateway foundation: config, schema, and exposure state machine
type: backend
complexity: critical
---

# Task 1: Gateway foundation: config, schema, and exposure state machine

## Overview

Delivers the durable substrate every other gateway slice builds on: the `[gateway]` configuration section, the Global DB tables that own exposure intent, and the `internal/gateway` policy plus reconciler that drive desired state into observed state with generation fencing and compensation. Nothing is reachable yet — this task makes exposure *decidable and crash-safe* so the listeners in task 02 have a contract to bind to.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- The daemon MUST treat the database as the sole authority for exposure intent; configuration MUST NOT carry any key that can enable a surface (`gateway.public_ui.enabled` must not exist).
- `gateway.enabled` MUST behave as an immutable ceiling: with it false every transition refuses, and flipping it false disables all tiers on reload.
- Every exposure transition MUST write desired state and bump `generation` in a single SQLite transaction; any effect completing against a stale generation MUST be discarded, never applied.
- The reconciler MUST apply effects in the fixed order persist → bind → establish → verify → advertise and unwind in reverse on failure, leaving no externally reachable intermediate state.
- `Reconcile` MUST run at boot before any endpoint is advertised, so a crash restores the operator's last desired state and never re-exposes a disabled surface.
- Provider activation MUST be modeled per `(provider_name, tier)` and surface exposure per `(surface, tier)`, with a partial unique index enforcing at most one active provider per tier.
- The migration MUST be append-only: append the next gap-free Goose file, update the declarative fragment, and refresh `atlas.sum` — never edit an applied migration.
- New gateway tables MUST NOT carry `workspace_id` (they are operator-global); the ingress-binding table arrives in task 04 and is the deliberate workspace-aware exception.
- `internal/gateway` MUST NOT import `internal/daemon`, `internal/api/*`, or `internal/cli`; boundary rules MUST be added in this task.
- Files MUST respect the 500-line cap using the split decided in the TechSpec (`types.go`, `policy.go`, `policy_transition.go`, `reconciler.go`, `store.go`, `status.go`).
</requirements>

## Subtasks

- [x] 1.1 Add the `[gateway]` config section with struct, defaults, validation, and overlay, plus the home-path entries for gateway state
- [x] 1.2 Register gateway keys in the CLI settable-key map and classify them for live-apply vs restart-required
- [x] 1.3 Author the declarative schema fragment for device sessions, provider identity, per-tier activation, and surface exposure
- [x] 1.4 Append the next Goose migration, refresh `atlas.sum`, add `queries/gateway.sql`, and regenerate sqlc output
- [x] 1.5 Wire a `GatewayRepo` into the Global DB repository set
- [x] 1.6 Implement the exposure policy: transition authority, generation fencing, ceiling enforcement, and refusal diagnostics that name cause and fix
- [x] 1.7 Implement the reconciler with the fixed effect order, compensation on failure, and boot reconciliation ahead of advertisement
- [x] 1.8 Implement the status projection (tiers, surfaces, addresses, counts, provider health) including desired-vs-observed during transitions
- [x] 1.9 Add `internal/gateway` import-boundary rules and update the boundaries test
- [x] 1.10 Wire `bootGateway` into the daemon boot sequence ahead of `bootServers`, with its shutdown phase and runtime-state publication

## Implementation Details

Follow the TechSpec sections *Data Models*, *Safety Invariants* (1–7, 15), *Config Lifecycle*, and *Architectural Boundaries*. Copy the section pattern from `internal/config/daemon.go` (a section that consumes `HomePaths`) rather than the flat `http.go`. Copy the boot-step pattern from an existing `internal/daemon/boot_*.go` and the repository pattern from an existing `globaldb` domain repo.

### Relevant Files

- `internal/config/network_config.go` — the section pattern to copy for a new `gateway_config.go` (struct + defaults + `Validate`)
- `internal/config/merge_network.go` — the per-section merger pattern for a new `merge_gateway.go`
- `internal/config/window_manager_overlay.go` — the overlay-type pattern for a new `gateway_overlay.go`
- `internal/config/config_extensions_sandbox.go` — holds `type Config struct`; the new section field is added here
- `internal/config/defaults.go` — `DefaultWithHome` registers the section default
- `internal/config/merge.go`, `internal/config/merge_overlay.go` — merge dispatcher and root overlay struct
- `internal/config/config_validation.go` — section validation dispatch
- `internal/config/config.go` — `ValidationError{Code,Path,Message}` for deterministic field errors
- `internal/config/home.go` — home-path constants, `HomePaths`, `EnsureHomeLayout` for gateway state dirs
- `internal/config/persistence.go` — `ResolveConfigWriteTarget` and `OverlayEditor` used by the write path
- `internal/config/lifecycle/lifecycle.go` — the restart-versus-live classification matrix
- `internal/cli/config.go`, `internal/cli/config_path_classification.go` — settable-key allowlist and path classification
- `internal/cli/config_network.go` — the section CLI surface pattern for a new `config_gateway.go`
- `internal/settings/config_apply_classification.go`, `internal/settings/config_apply_service.go`, `internal/settings/config_reload_diff.go` — live-apply plumbing
- `internal/daemon/settings.go`, `internal/daemon/settings_runtime_applier.go` — runtime application of a config section
- `internal/store/globaldb/schema/definitions/` — declarative fragments are **domain-clustered**, not strictly sequential (`50_loops`, `60_network`, `70_tasks`); use the free network-adjacent slot `61_gateway.sql`
- `internal/store/globaldb/schema/migrations/` — highest applied is `00055_schema.sql`, so append `00056_schema.sql`; `atlas.sum` lives here
- `internal/store/globaldb/queries/` — add `gateway.sql`; siblings are `network_channels.sql`, `bridge_core.sql`
- `internal/store/globaldb/global_db_bridge.go` (+ `global_db_bridge_scan.go`, `bridge_sqlc_mapping.go`) — the repository implementation trio to mirror for `global_db_gateway.go`
- `internal/store/globaldb/repositories.go` — `repoBase` + `initializeRepositories` wiring
- `internal/store/globaldb/migration_stream.go`, `internal/store/globaldb/sqlc.yaml` — stream registration and codegen configuration
- `internal/store/migrate_test.go`, `internal/store/migrate_streams_test.go` — canonical fresh-apply, ahead-of-head, atlas drift, and schema-equivalence suites to extend
- `internal/store/globaldb/global_db_app_metadata_test.go` — the reopen-after-restart pattern for the reopen case
- `internal/daemon/boot_network.go` — the boot-step pattern to copy for `boot_gateway.go`
- `internal/daemon/boot_components.go` — ordered boot steps; insert `bootGateway` before `bootServers`
- `internal/daemon/boot.go`, `internal/daemon/daemon_runtime_state.go`, `internal/daemon/boot_publish.go` — boot state, published runtime state, and publication
- `internal/daemon/daemon_shutdown_targets.go`, `internal/daemon/boot_cleanup.go` — shutdown snapshot and cleanup registration
- `magefiles/boundaries.go` — forbidden-import rules; mirror the `internal/marketplace` block
- `magefiles/magefile_test.go` — boundary assertions plus the production source line-limit check

### Dependent Files

- `internal/daemon/daemon_shutdown_*.go` — gateway teardown joins the shutdown ordering
- `packages/site/content/docs/configuration/config-toml.mdx` — `[gateway]` key reference ships with the keys
- `openapi/compozy.json`, `web/src/generated/compozy-openapi.d.ts` — regenerate if any contract type lands here

### Related ADRs

- [ADR-007: Daemon-owned tier listeners with a durable desired/observed state machine](adrs/adr-007.md) — the state machine, fencing, and reconciliation contract
- [ADR-010: Gateway state ownership](adrs/adr-010.md) — table shapes, keys, and global scope
- [ADR-011: `internal/gateway` domain package](adrs/adr-011.md) — package placement and boundaries
- [ADR-006: Fail-closed exposure](adrs/adr-006.md) — refusal semantics and diagnostics

## Deliverables

- `[gateway]` config section with defaults, validation, overlay, home paths, settable keys, and live-apply classification
- Global DB gateway tables via declarative fragment + appended Goose migration + refreshed `atlas.sum` + generated sqlc output + `GatewayRepo`
- `internal/gateway` policy, reconciler, and status projection with the decided file split
- `bootGateway` wired into daemon boot ahead of `bootServers`, with shutdown and runtime-state publication
- Import-boundary rules for `internal/gateway` enforced by `mage Boundaries`
- Config key documentation shipped in the same change
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-001, UT-002, UT-003, UT-004, UT-005, UT-006 — config defaults, validation, absence of any surface-enabling key, overlay unset semantics, live-apply classification
- [x] UT-010, UT-011, UT-012, UT-013, UT-014 — DB-only intent authority, ceiling enforcement, fail-closed refusals with cause and fix, ordering and legacy-semantics rejection
- [x] UT-015, UT-016, UT-017, UT-018 — generation fencing under concurrent transitions and the effect-order/compensation contract
- [x] UT-019, UT-020, UT-021 — immediate disable, mid-delivery behavior, idempotent double disable
- [x] UT-022, UT-023, UT-024, UT-025, UT-026, UT-027 — status projection, desired-vs-observed, empty states, degraded reporting, independent tiers, abandoned consent
- [x] UT-160, UT-161 — migration fresh/reopen/ahead/integrity and both-tier key survival with the partial unique index
- [x] IT-002 — boot with a forbidden `[gateway]` combination starts local-only and reports the refusal without crash-looping
- [x] IT-004 — fault injection at each effect boundary leaves no externally reachable intermediate
- [x] IT-005 — crash mid-transition then restart: reconcile completes before advertisement and a disabled surface stays disabled

## Success Criteria

- Every assigned test case implemented and passing
- `make gate` passes with zero lint findings; `make codegen-check` reports no drift after sqlc/atlas regeneration
- `mage Boundaries` fails if `internal/gateway` imports a shell package
- A fresh home boots to fully local-only posture with no reachable surface
- No configuration value can enable any surface — asserted by test, not by review

## Completion Notes

- Focused verification passed for gateway config/lifecycle, policy/status/reconciliation, daemon boot and live apply, Global DB/migration behavior, and package boundaries. The workstream-wide gate is deferred to the final close by the active goal.
- QA tracker impact is flagged in `MS-gateway-config-ceiling` and `RT-gateway-local-only-boot`; planning and execution are owned exclusively by Tasks 08 and 09.
- Contract parity checked against `_prd.md`, `_techspec.md`, `_user_stories.md`, `_tests.md`, and ADR-006/007/010/011.

Compozy Impact Audit:

- Native tools: no tool IDs, toolsets, descriptors, schemas, digests, risk flags, or capability gates change in Task 01; checked `internal/tools`, `internal/toolmeta`, `internal/api/contract`, and `openapi`. Agent-facing gateway tools arrive in Task 06.
- Extensibility and hooks: adds the `gateway.Store`, `Policy`, `EffectDriver`, and reconciler seams plus the `[gateway]` config lifecycle. No extension hook, capability, resource, bridge SDK, MCP sidecar, or registry contract changes in this foundation slice.
- Workspace data isolation: device sessions, provider identity/activation, and surface exposure are operator-global in Global DB and intentionally omit `workspace_id`; schema and reopen/index tests prove the scope. Workspace-aware ingress bindings arrive in Task 04.
- Official Compozy skill: updated `skills/compozy/` configuration routing and reference with gateway defaults, live/restart lifecycle, and the rule that config cannot enable a surface.
