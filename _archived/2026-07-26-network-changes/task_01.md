---
status: pending
title: "Contract, resolver, schema hard-cut & config lifecycle"
type: backend
complexity: high
---

# Task 1: Contract, resolver, schema hard-cut & config lifecycle

## Overview

Land the leaf participation contract, the only Resolver, the destructive migration v82+ hard cut, and the `[network.live.*]` config lifecycle. After this task every retained row carries a canonical Local snapshot, removed participation-selecting config keys fail validation, and no owner domain still writes through the old channel columns — wiring of resolve-and-persist into session/task/loop/automation lands in task_02.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. `internal/network/participation` MUST be a leaf package (zero imports of task/session/loop/network/daemon/api) exporting `Mode`, `ChannelStrategy`, `Request`, `Spec`, `Source`, `Bounds`, `Resolver`, `ResolveInput`, and sentinel diagnostics (`ErrUnavailable`, `ErrChannelUnknown`, `ErrStrategyInvalid`, `ErrStrategyChannelConflict`, `ErrAuthorityDenied`, `ErrBoundsExceedCeiling`, `ErrLoopRequiresLive`, `ErrLiveUnsupported`) matched with `errors.Is`.
2. Persisted `Spec.Version` MUST be the brand-neutral atom `network-participation/v1`; modes are only `local` and `live` (no `mailbox` enum, ADR-004/006).
3. `Resolver.Resolve` MUST be the ONLY owner of precedence and channel derivation (request > definition/profile > workspace coordination for coordinated owners > `built_in_local`); Local resolutions MUST perform zero channel lookups.
4. Migration v82+ (append-only after current tail, currently v81 `add_goal_adoption_attempt`) MUST add snapshot + projection columns on sessions/task_runs/loop_runs, wake generalization columns/CHECKs, workspace coordination + invitation + availability + disposition + wake ledger + wake sources + budget tables, and session-keyed relation rebuilds exactly as TechSpec "Data Models"; `EnsureSchema` is forbidden.
5. Every retained pre-cut session/task_run/loop_run row MUST backfill the canonical Local Spec JSON plus matching projections (B-012); empty/undecodable snapshots MUST be impossible by construction.
6. Delete targets in this task's schema layer MUST be removed in the same migration: `sessions.channel`, `tasks.network_channel`, `task_runs.network_channel`/`coordination_channel_id` (+ indexes), peer-keyed relation columns, `network_delivery_guidance_state`.
7. Config MUST add `[network.live.defaults]` + `[network.live.limits]` with the TechSpec normative table; keep `network.enabled` + `network.greet_interval`; remove `network.default_channel`, `port`, `max_payload`, `activation_top_k`, digest/guidance keys with strict rejection (no compat read).
8. Config apply/reload for `network.enabled` MUST write the persisted `network_availability` row transactionally (B-013) so admission later shares one serialization domain.
9. `mage Boundaries()` MUST gain the leaf-package forbidden-import rows in this change.
10. Workspace qualification (C-01) MUST hold for every new/rebuilt network identity path — same channel name across workspaces never leaks.
</requirements>

## Subtasks

- [ ] 1.1 Create `internal/network/participation` with planned file split (types, validation, errors, resolver, bounds) — every file under 500 lines.
- [ ] 1.2 Implement table-driven Validate + Resolve covering precedence, strategy/channel conflicts, bounds ceilings, and zero-lookup Local path.
- [ ] 1.3 Author migration v82+ (registry tail append only) for all TechSpec Data Models columns/tables/rebuilds + Local Spec backfill.
- [ ] 1.4 Swap store read/write projections onto snapshot + typed columns; delete obsolete channel column consumers that block compilation in store layer.
- [ ] 1.5 Ship `[network.live.*]` + remove deleted keys through structs/defaults/overlay/validation/tool_surface/lifecycle matrix/tests.
- [ ] 1.6 Wire `network_availability` persistence on config apply/reload; append Boundaries() rows.
- [ ] 1.7 Land assigned UT/IT cases in canonical suites (extend, never duplicate).

## Implementation Details

See TechSpec "Core Interfaces", "Data Models", "Config Lifecycle", "Delete Targets", Build Order steps 1–2, and ADR-001/006/009. Competitor caution for bounded fan-out ceilings (do not copy expensive defaults): `.resources/hermes/website/docs/user-guide/features/mixture-of-agents.md`, `.resources/hermes/hermes_cli/runtime_provider.py`.

Skills to activate: `eng-code-guidelines`, `golang-pro`, `eng-schema-migration`, `eng-test-conventions`, `testing-boss`, `eng-consolidate-test-suites`.

### Relevant Files

- `internal/network/` — current package to keep import-acyclic relative to new leaf
- `internal/config/config.go` — `NetworkConfig` / `DefaultNetworkConfig()` (`Enabled: true`, `DefaultChannel: "default"` today)
- `internal/config/tool_surface.go` — agent-manageable key paths
- `internal/store/globaldb/global_db.go` — `sessions.channel`, task/run channel columns
- `internal/store/globaldb/global_db_network_schema.go` — network conversation schema
- `internal/store/globaldb/global_schema_migrations.go` + `global_db_migrations_tail.go` — append-only registry
- `magefile.go` — Boundaries() registry

### Dependent Files

- `internal/store/globaldb/` — all channel-column readers/writers must compile against new projections
- `internal/settings/sections.go` — overlay tracks network keys
- `packages/site/content/runtime/core/configuration/config-toml.mdx` — key section (docs rewrite completes in task_06; config docs stub/flag here if lifecycle matrix requires it)
- Downstream tasks 02–05 assume snapshot columns and Resolver exist

### Related ADRs

- [ADR-001](adrs/adr-001.md) — explicit per-execution default-local participation
- [ADR-006](adrs/adr-006.md) — one contract, resolve-once immutable snapshot
- [ADR-009](adrs/adr-009.md) — coordination setting is workspace data, not config

## Deliverables

- `internal/network/participation` leaf package + Boundaries rows
- Migration v82+ + store projection swaps + Local backfill
- `[network.live.defaults|limits]` lifecycle complete; deleted keys rejected
- Every assigned test case implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [ ] UT-001, UT-002, UT-003, UT-004, UT-005, UT-006, UT-007, UT-008, UT-009, UT-010 — participation contract validation/normalization
- [ ] UT-022, UT-023, UT-024, UT-025, UT-026 — config lifecycle defaults/ceilings/removed keys/tool-surface
- [ ] IT-001 — FreshDB at v82+ columns/tables + atomic reservation snapshot
- [ ] IT-002 — ReopenAfterRestart destructive cut + Local Spec backfill (B-012)
- [ ] IT-003 — C-01 workspace isolation for same channel name

## Success Criteria

- Every assigned test case implemented and passing
- Leaf package has zero upward imports; `mage Boundaries` green
- Fresh DB and reopen-after-restart migration tests prove Local Spec backfill and deleted columns
- Removed config keys fail load; live defaults never apply to Local resolutions

### Web/Docs Impact

- `web/`: none yet — checked surfaces: no OpenAPI change in this task (contract co-ship is task_05); generated TS untouched.
- `packages/site`: config-toml / lifecycle-matrix rows for new/removed keys SHOULD land or be flagged for task_06 hard-cut; no autonomy doctrine rewrite here.
- QA impact: flag config-key rows in `docs/qa/state.csv` as `untested` only if this task alone ships user-visible config behavior; otherwise defer to task_06 tracker sweep.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none yet — checked: hooks/bundles/tools untouched until later tasks; leaf types are the shared shape they will consume.
- Agent manageability: `agh config` tool-surface paths MUST expose new live keys and drop removed ones (UT-026).
- Config lifecycle: full add/remove matrix in this task (structs, defaults, overlay, validation, examples, tests).
