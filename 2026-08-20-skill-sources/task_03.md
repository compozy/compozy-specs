---
status: pending
title: "Exposure records: schema, store, and manager"
type: backend
complexity: critical
---

# Task 3: Exposure records: schema, store, and manager

## Overview

Delivers the expose primitive's foundation: the `skill_exposures` side table (the one SQLite change in this feature) with its store layer, and the `ExposeManager` that owns per-skill symlinks into enabled preset roots — preflight/commit/rollback state machines, ownership proof via `LinkTarget`, filesystem-reconciled four-state health, and lifecycle integration so skill remove/update never leaves broken links behind. Public surfaces (API/CLI/web) consume this manager in task_04.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — no compat shims, no fallbacks, no placeholders (greenfield hard cuts)
</critical>

<requirements>
1. MUST add `skill_exposures` to the global stream following `eng-schema-migration`: new ordered declarative fragment (e.g. `internal/store/globaldb/schema/definitions/43_skill_exposures.sql`, mirroring the `41_marketplace.sql` precedent), the next gap-free Goose migration (head was `00077` at exploration time — use the actual next number), `atlas.sum` + sqlc refresh via `make codegen`. Append-only: never touch existing migration bytes.
2. MUST implement the exact column/integrity contract from `_spec.md` Data Models: columns incl. `link_target` (ownership proof) and `owner_scope`/`workspace_id`; `CHECK ((owner_scope = 'global' AND workspace_id IS NULL) OR (owner_scope = 'workspace' AND workspace_id IS NOT NULL))`; `UNIQUE (link_path)`; `UNIQUE (skill_name, owner_scope, workspace_id, target_slug)`; indexes on `(skill_name)` and `(workspace_id)`.
3. MUST implement `ExposeManager` per the Core Interfaces block: `Expose`/`Unexpose`/`Exposures`/`Reconcile` returning structured `TargetResult`s with deterministic per-target error codes (`skill_not_exposable`, `expose_target_disabled`, `expose_target_invalid`, `expose_name_conflict`, `expose_link_unsupported`, `expose_foreign_link`, `unsafe_skill_name`).
4. MUST follow the expose lifecycle state machines exactly (spec table, Safety Invariant 3): expose preflights ALL targets before any mutation, commits per target record→link, rolls back completed targets on mid-sequence failure (links, records, and only self-created still-empty directories); unexpose removes link→record with per-target independent idempotent results (no rollback concept); skill removal deletes the canonical dir only after every owned link is cleaned, aborting with retryable `skill_remove_blocked` naming the failing link when cleanup fails.
5. MUST enforce ownership authority (Safety Invariant 4): a link is ours iff a record matches its path AND `Readlink(link) == record.LinkTarget`; health reconciled live (`Lstat`+`EvalSymlinks` vs record, never fs-only inference, never record-as-current-state): `healthy` / `missing` / `broken` (ours, unresolvable — repair allowed) / `foreign_conflict` (report only, never mutated, never adopted).
6. MUST enforce destination path safety (Safety Invariant 5): `validateExposeName` (single normalized segment, skill-name grammar; rejects absolute paths, separators, `..`, NUL, encoded traversal, Unicode tricks, over-length) and `resolveExposeDest` (deepest-existing-parent realpath containment inside the preset root) before ANY write — including preset-root creation. Absent enabled preset root: expose is the one operation allowed to create the exact root path, recorded in the operation's rollback set.
7. MUST implement target rules: enabled presets only (`expose_target_invalid` for custom sources); links land at the skill's scope level (workspace skill → workspace-level folder, global → global-level); relative symlinks when target and canonical share the workspace root scope, absolute otherwise; re-expose over a healthy link is idempotent; symlink syscall failure surfaces `expose_link_unsupported` with NO copy fallback (ADR-014); Windows follows the same error path.
8. MUST integrate lifecycle: marketplace/skill remove runs reconcile-and-clean before canonical deletion; marketplace update preserves the path and verifies `healthy` after; CompozyOS never removes or overwrites an entry it cannot prove ours.
9. MUST emit the exposure observe events with base correlation keys: `skills.exposure.created`/`removed` (per-target commit), `skills.exposure.operation_failed` (preflight/commit failure), `skills.exposure.broken_detected` (reconcile, incl. foreign_conflict), `skills.exposure.cleanup_failed` (rollback/reconcile failure).
10. MUST activate `eng-schema-migration`, `eng-cleanup-failure-paths`, `eng-code-guidelines`, `golang-master` before writing code; the manager package split respects the 500-line cap (state machines, path safety, reconcile, store adapter as separate files).
</requirements>

## Subtasks

- [ ] 3.1 Declarative schema fragment + next gap-free Goose migration + `atlas.sum`/sqlc refresh (`make codegen` + `codegen-check`)
- [ ] 3.2 Store layer: queries file, sqlc mapping, repository wiring, domain types (mirror the marketplace store pattern)
- [ ] 3.3 `ExposureRecord`/`ExposureState`/`TargetResult` types + `validateExposeName` + `resolveExposeDest`
- [ ] 3.4 Expose state machine: preflight-all → per-target record→link commit → rollback (incl. self-created empty dirs)
- [ ] 3.5 Unexpose state machine: ownership proof → link→record removal, per-target independent, idempotent convergence
- [ ] 3.6 Reconcile: four-state health from fs vs records; retryable, idempotent
- [ ] 3.7 Skill remove/update lifecycle integration: cleanup-before-canonical-delete, `skill_remove_blocked` abort, update preserves links
- [ ] 3.8 Exposure observe events at every lifecycle call site
- [ ] 3.9 Extend the canonical store suites (fresh/reopen/ahead/integrity/equivalence) for the new table
- [ ] 3.10 Full assigned test suite (17 UT + 4 IT) on real temp filesystems

## Implementation Details

Follow `_spec.md` Part II — Core Interfaces (`internal/skills/expose.go` block), the expose lifecycle state-machine table, Data Models (`skill_exposures` columns + integrity contract), Safety Invariants 3-5, and ADR-015's reconcile rules. Enabled-preset root resolution comes from task_01/02 config functions; skill eligibility = physical directory (bundled skills have no on-disk home → `skill_not_exposable`).

### Relevant Files

- `internal/store/globaldb/schema/definitions/41_marketplace.sql:1,20` — fragment precedent (ordering + shape)
- `internal/store/globaldb/schema/migrations/` — Goose head (`00077_schema.sql:1-13` format) + `atlas.sum`
- `internal/store/globaldb/sqlc.yaml:1-12` + `queries/marketplace.sql` — sqlc config + query-file pattern
- `internal/store/globaldb/global_db_marketplace_skill.go` + `*_sqlc_mapping.go` + `repositories.go` — store-layer pattern to mirror
- `internal/store/globaldb/migration_stream.go:9-25` + `internal/store/migrate.go` — stream registration/engine
- `internal/store/migrate_streams_test.go` + `internal/store/globaldb/schema_introspection_test.go` — canonical fresh/reopen/ahead/integrity suites to extend (IT-015)
- `internal/skills/path_security.go:10-23` — containment primitive (`ResolveExistingPathWithinRoot`)
- `internal/skills/provenance.go` — sidecar/hash patterns (canonical-dir realpath at creation)
- `internal/skills/marketplace/` — install/remove/update seams for lifecycle integration
- `internal/skills/registry_toggle.go` + `registry_disabled.go` — adjacent mutation patterns (registry refresh after mutation)
- `internal/skills/observe_events.go` + `internal/events/names.go:82-83` — event emission shape
- `internal/config/agent.go` (post-task_01 `SkillsDirs`) + `ResolveGlobalSkillRoots` — enabled preset root resolution

### Dependent Files

- `internal/api/core/skills.go` — expose/unexpose handlers consume the manager (task_04)
- `internal/cli/skill_commands_mutation.go` — CLI verbs + `create --expose` (task_04)
- `internal/api/core/conversions_skills.go` — `exposures[]` payload projection (task_04)
- `internal/skills/registry_discovery.go` — scanned expose links must dedup to canonical (IT-009 guards with task_02 scanner)

### Related ADRs

- [ADR-011](adrs/adr-011.md) — canonical-home + symlink expose, preset targets only · [ADR-014](adrs/adr-014.md) — symlinks never copies · [ADR-015](adrs/adr-015.md) — persisted ownership records, fs-reconciled health

### Web/Docs Impact

- `web/`: none in this task — the Exposures panel binds to task_04's API (`web/src/systems/skill/**` lands in task_06). Checked surfaces: no contract change ships here beyond the migration (payloads land in task_04).
- `packages/site`: none in this task — exposure docs land in task_07 (`content/docs/skills/` sources/exposures page).
- QA impact: none — no user-visible surface ships in this task (manager + schema are internal; the expose behavior becomes user-visible in task_04, which flags the scenario).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: marketplace remove/update lifecycle is the only touched extension-adjacent surface (cleanup integration); extension manifests, hooks, MCP sidecars, bridge SDKs unaffected (checked — no contact).
- Agent manageability: defines the deterministic per-target error vocabulary agents will match on (surfaced via API/CLI in task_04); no public route ships here.
- Config lifecycle: none — no new keys; enabled-target resolution consumes task_01 keys. Checked: no `config.toml` surface changes.

## Deliverables

- `skill_exposures` schema + migration + sqlc + store layer, with the canonical store suites extended
- `ExposeManager` with exact state machines, ownership proof, four-state reconcile, path safety
- Skill remove/update lifecycle integration (`skill_remove_blocked`, update-preserves-links)
- Exposure observe events wired at every lifecycle call site
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there. Owning suites: new canonical `internal/skills/expose_test.go` (new component — real temp filesystems, no fs fakes); the canonical store suites own the migration case; `internal/skills/registry_integration_test.go` owns the self-shadow case.

- [ ] UT-045, UT-046, UT-047, UT-048, UT-049, UT-050, UT-051, UT-052, UT-053, UT-054, UT-055 — expose/unexpose semantics, four-state matrix, error codes, removal cleanup
- [ ] UT-077, UT-078, UT-079, UT-080 — record-side reconcile, `validateExposeName` matrix, `resolveExposeDest` containment, multi-target rollback + cleanup_failed
- [ ] UT-087, UT-088 — absent-root creation + rollback of self-created empty dirs; unexpose ordering + crash convergence
- [ ] IT-004 — marketplace install → expose → update preserves → remove cleans
- [ ] IT-009 — exposed skill scanned from both sources → one entry, zero self-shadow
- [ ] IT-013 — reconcile lifecycle: missing → repair; partial cleanup; `skill_remove_blocked` abort + retry completes
- [ ] IT-015 — store suites extended: fresh apply, reopen with data, ahead detection, `atlas.sum` integrity

## Success Criteria

- Every assigned test case implemented and passing; `make gate` green (task close); `make codegen-check` green
- Crash injected between any two state-machine steps converges to the documented state on re-run (UT-088, UT-077)
- A foreign entry at a link path is never mutated by any operation, proven by tests asserting fs before == after
- Migration suite proves append-only identity (no edits to existing migrations, `atlas.sum` consistent)
