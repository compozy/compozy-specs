---
status: pending
title: Profiles Schema and Stamps (Migration 00079 + Memory 00002)
type: backend
complexity: critical
---

# Task 2: Profiles Schema and Stamps (Migration 00079 + Memory 00002)

## Overview

Creates the durable substrate: the `profiles` catalog seeded with the permanent `default` (fixed zero-ULID), the `profile_id` stamp on the 17 creation roots with immutability + active-owner triggers, the PK/identity rebuilds, the scope-word hard cut (`global`→`user`; memory tier →`profile`), every support table (selections, lifecycle journal + seed snapshots, delivery permits, credential requirements, enablement/marker/mute rows, env-bindings rebuild), and the memory stream's own migration with the crash-safe directory move. Threads `profile_id` through every root INSERT/UPSERT site so the NOT NULL contract holds from day one.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

**Skills**: `eng-code-guidelines`, `golang-master`, `eng-schema-migration`, `eng-test-conventions`, `testing-boss`.

<requirements>
- MUST author `internal/store/globaldb/schema/definitions/05_profiles.sql` (new definition file; prefix `05_` is unique in the 27-file set) and append `internal/store/globaldb/schema/migrations/00079_schema.sql` implementing every item in `_spec.md` Data Models, in migration-safe order: seed `default` (zero ULID `00000000000000000000000000`, `#8E8EB5`, `circle`) **before** any stamp backfill (ADR-012).
- MUST stamp the 17 roots exactly (nullable → backfill to `default` → NOT NULL rebuild), with the `BEFORE UPDATE OF profile_id` ABORT trigger and the `BEFORE INSERT` active-owner + availability trigger per root; children get no column.
- MUST rebuild `token_usage_daily` and `notification_cursors` PKs and stamp **both** `InsertBridgeInstance` and `UpsertBridgeInstance` (identical column lists — path-map §10).
- MUST execute the enablement hard cut: `extensions.enabled` and `notification_presets.enabled` columns dropped via rebuild, prior disabled state migrated to `default`-profile exception rows (delete target 11).
- MUST rebuild `extension_env_bindings` with the profile dimension and cleanup/existence triggers (delete target 13).
- MUST create `profile_selections`, `profile_lifecycle_ops` + `_steps` + `_seed` + `_credential_asks`, `profile_credential_requirements`, `notification_delivery_permits`, `extension_profile_enablement`, `extension_profile_markers`, `notification_preset_enablement`, `attention_workspace_mutes` — typed columns, zero JSON blobs.
- MUST rewrite stored scope values and refs in `00079`: `resource_records` `'global'`→`'user'` (CHECK widened to the four-value set), MCP auth/registration scopes, `vault:mcp/global/…` and `vault:extensions/global/…` → `…/user/…` (equivalence-tested).
- MUST hard-cut the Go readers in the same change: config `WriteScope` and `settings.ScopeKind` drop `global`, accept `user` (the shipped `agent` scope untouched); memory scope enum accepts `profile|workspace|agent` (ADR-013). No dual acceptance anywhere.
- MUST ship the memory stream migration `internal/memory/schema/migrations/00002_*.sql` (head today: `00001_baseline.sql`): scope `global`→`profile`, durable `profile_id` in catalog/consolidation/decision/event/recall identities and FTS indexes, and the pending `memory_maintenance_ops` row; the store's open path executes the idempotent same-volume rename `$COMPOZY_HOME/memory` → `profiles/default/memory/` and refuses profile-tier reads while pending (fail-closed; old path is a delete target, never a fallback read).
- MUST thread `profile_id` as an explicit parameter through every root INSERT/UPSERT path (store methods never default the stamp) and join it to `SessionCreationProfile`'s canonical digest input (delete target 14).
- MUST ensure boot verifies the `default` row and creates its directory layout idempotently (verify-only — `00079` owns the seed).
- MUST NOT touch `loop_runs.origin_creation_profile_ref` (a session-creation digest — naming trap, path-map correction 16) and MUST NOT confuse the new catalog with `71_task_profiles.sql` ("task profiles" is a distinct shipped concept, D8).
- Append-only identity: never edit existing migrations, versions, or `atlas.sum` history; `make codegen` refreshes, `make codegen-check` gates.
</requirements>

## Subtasks

- [ ] 2.1 Author `05_profiles.sql` (profiles, selections, journal + seed snapshot tables, permits, credential requirements, enablement/marker/mute tables) with CHECKs, FKs, and cleanup triggers per Data Models.
- [ ] 2.2 Author `00079_schema.sql`: default seed → 17-root stamp columns + backfills + NOT NULL rebuilds → immutability + active-owner/availability triggers → PK rebuilds → enablement hard cut with exception-row migration → env-bindings rebuild → scope value + vault ref rewrites.
- [ ] 2.3 Update every affected declarative definition to the end state; `make codegen` + `make codegen-check` green.
- [ ] 2.4 Author memory migration `00002` (scope values, durable profile identity in all indexes/identities, `memory_maintenance_ops` row) and the store open-path move executor with the idempotency guard and pending-refusal.
- [ ] 2.5 Sweep the Go scope enums/validators/payloads for the `global`→`user` cut (config, settings, resources, MCP, vault ref readers) — fail-closed on the old value.
- [ ] 2.6 Thread `profile_id` through the 17 roots' INSERT/UPSERT sites and their service-layer creation paths (explicit params; callers pass the default profile id until task_04 wires resolution).
- [ ] 2.7 Join the profile id to the session-creation witness digest (creation builders, reseed, goal-binding); delete the old digest-input shape.
- [ ] 2.8 Boot: default-row verify + `EnsureHomeLayout`-style profile directory creation (0o700), idempotent.
- [ ] 2.9 Extend the canonical migration suites for `00079` and the memory stream (fresh/reopen/ahead/integrity/equivalence + crash-interrupted move convergence).

## Implementation Details

Everything schema-shaped is specified field-by-field in `_spec.md` Data Models; the lifecycle journal is consumed by task_03 (this task only creates the tables). The stamping order inside `00079` is load-bearing: seed first, backfill second, NOT NULL third (both fresh and upgrade paths must pass IT-005).

### Relevant Files

- `internal/store/globaldb/schema/definitions/00_core.sql` (`app_metadata`/`workspaces` neighborhood), `20_sessions.sql:89-204` (+ `token_usage_daily` PK at 188-204), `33_extensions.sql` (enablement + env-bindings + trigger template), `34_resources.sql:1-19` (scope CHECK pair), `37_notifications.sql` (cursors PK, presets), `50_loops.sql`, `70_tasks.sql`, `72_task_runs.sql` — stamped roots + house idioms.
- `internal/store/globaldb/schema/migrations/` — append `00079_schema.sql` after task_01's `00078`.
- `internal/store/globaldb/queries/session_core.sql:56-88`, `task_core.sql:1-18`, `loop_core.sql:1-27`, `automation_core.sql:77-127`, `bridge_core.sql:1-40`, `worktrees.sql:1-25`, `observe_overview.sql:2-20` — the INSERT/UPSERT stamping sites.
- `internal/memory/schema/` — `schema.sql`, `migrations/00001_baseline.sql` (head), `atlas.sum`, `embed.go`; `internal/memory/migration_stream.go:11-24` — the separate stream this task appends `00002` to.
- `internal/memory/store_scope.go:21-46` — `dirForScope` switch (D14 landing site).
- `internal/config/persistence.go:21-53` + `internal/settings/models.go:15-41` — `WriteScope`/`ScopeKind` enum cut sites.
- `internal/resources/types.go:61-120` — `ResourceScopeKind` CHECK counterpart in Go.
- `internal/vault/types.go:29-48` — namespace regex + map (the `user` segment rename readers).
- `internal/config/home.go:71-98,217-283` — `HomePaths` + `ResolveHomePathsFrom` + `EnsureHomeLayout` (profile dir derivation; single constructor).
- `internal/daemon/` boot wiring — default-profile verify step ordering (before traffic).

### Dependent Files

- `internal/session`, `internal/task`, `internal/loop`, `internal/automation`, `internal/bridges`, `internal/worktree`, `internal/network`, `internal/notifications`, `internal/observe` — creation paths gain the explicit `profileID` parameter (data only; none imports `internal/profile`).
- `internal/store/globaldb/migration_stream.go` — untouched (verify).
- `web/src/generated/compozy-openapi.d.ts` — regen if payload types shift.

### Competitor References

- `.resources/hermes/hermes_cli/profiles.py:1-58` — profile anatomy; the `default`-asymmetry to reject (uniform `default` here, seeded by migration).

### Related ADRs

- [ADR-012](adrs/adr-012.md) — identity: zero-ULID `default`, stable stamp, name handle.
- [ADR-013](adrs/adr-013.md) — scope vocabulary hard cut; per-stream migrations; row-driven memory move.
- [ADR-015](adrs/adr-015.md) — stamp 17 roots, inherit the rest.
- [ADR-004](adrs/adr-004.md) — one runtime, one database; stamped work.

## Deliverables

- `05_profiles.sql` + `00079_schema.sql` + memory `00002` passing all canonical migration suites on fresh and upgrade paths.
- 17 roots stamped with triggers; PK rebuilds done; enablement hard cut done; env-bindings rebuilt.
- Scope-word cut complete in storage and Go readers (no reader accepts `global`).
- Memory directory move row-driven, crash-convergent, fail-closed while pending.
- Every root creation path threading `profile_id` explicitly; witness digest updated.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-017 — per-suite stamp assertions at each creation boundary's canonical suite (session/task/loop/automation).
- [ ] UT-033, UT-034 — scope enum cuts (config+settings; memory).
- [ ] UT-069, UT-071 — `dirForScope(profile)`; legacy content resolves post-move.
- [ ] IT-001, IT-002 — full-equivalence fresh; pre-profiles reopen backfill.
- [ ] IT-004, IT-005 — PK rebuilds; default seed before backfill + boot verify idempotent + zero-workspace fresh start.
- [ ] IT-006 — stored scope/ref rewrites, no reader accepts old values.
- [ ] IT-007, IT-008, IT-009 — stamp immutability ABORT; FK RESTRICT; workspace-delete cascade on selections.
- [ ] IT-010, IT-073 — memory dir move + recall finds moved content; crash-interrupted move converges, pending refusal enforced.
- [ ] IT-076 — archived-owner creation guard at each of the 17 roots (per-root trigger).
- [ ] IT-083 — session creation witness digests differ across profiles; `origin_creation_profile_ref` untouched.
- [ ] IT-084 — memory durable ownership: identities + FTS carry `profile_id`; no cross-read, no collision.

### Web/Docs Impact

- `web/`: none directly — generated types regen only (`web/src/generated/compozy-openapi.d.ts`); no component consumes the new tables yet.
- `packages/site`: none in this task — the scope-word (`--scope user`) and layer docs ship with tasks 04/08 per the vertical-slice rule.
- QA impact: none — no user-visible behavior change yet (schema + internal threading; surfaces land in later tasks).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: schema-side substrate for enablement/markers/placement/env-bindings lands here; manifest/publish behavior unchanged until task_09 (checked: `internal/extension` manifest chain untouched).
- Agent manageability: none yet — no new verbs/routes; the scope-word cut surfaces in task_04/08 CLI.
- Config lifecycle: `WriteScope`/`ScopeKind` value rename is a contract change shipped with its validators and tests here; no `config.toml` keys added; docs for `--scope user` ship with task_04/08 slices.
