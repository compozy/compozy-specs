---
status: completed
title: Manifest v2, permissions, config consolidation + hook source (Phase B)
type: backend
complexity: high
---

# Task 2: Manifest v2, permissions, config consolidation + hook source (Phase B)

## Overview

Cut the data and policy foundation: manifest v2 collapses the triple capability declaration into one closed-set-validated `permissions` list with derived consent areas (R5/R9), the extensions registry migrates hard (new columns + the `extension_dev_links` side table), extension-declared hooks get their own truthful source tier, and the extension config surface consolidates under `[extensions.trust]`/`[extensions.sources]` (ADR-007). Phantom capabilities and silent no-op loads die here (R11).

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST replace `Manifest.Actions`/`Manifest.Security` with `PermissionsConfig{Requires []string}` and delete `ActionsConfig`/`SecurityConfig` plus the SDK-side duplications (delete targets in the TechSpec); a manifest containing `[actions]`/`[security]` fails load naming `[permissions]` as the replacement.
- MUST validate `capabilities.provides` and `permissions.requires` closed-set at load/validate/install (invariant 6): unknown values are load errors, never no-ops; `bridge.adapter` on installed manifests rejects deterministically (ADR-006); the public provide set is exactly four.
- MUST derive consent areas from the method list via the generated derivation table (`internal/extension/contract` — fixes the `resources.write` mismap, ADR-004); enforcement stays in `CapabilityChecker.CheckHostAPI`.
- MUST migrate the global catalog stream per `eng-schema-migration`: `extensions.provides_json`/`permissions_json` columns, actions/security persistence deleted, new `extension_dev_links` side table (`extension_name`, `workspace_id`, `origin_path`, `bundle_generation`, `linked_at`; unique on name+workspace) — declarative source + next gap-free Goose migration + `make codegen` refresh; append-only identity.
- MUST add `HookSourceExtension` (`"extension"`, priority 300; Config 500 > Extension 300 > AgentDefinition 100) and delete the `extensionHookSource = hookspkg.HookSourceConfig` alias (`internal/extension/manager.go:48`); introspection reports the source truthfully.
- MUST restructure config per ADR-007: `[extensions.trust]`/`[extensions.sources.github]`/`[extensions.sources.git]`/`[extensions.dev]` with the TechSpec's defaults; `[extensions.marketplace]` deleted with validation naming the replacement key; structs, defaults, merge/overlay, `compozy config set` registry, examples, and generated docs move in the same change.
- MUST regenerate/re-author all in-repo manifests (`extensions/dev-cycle`, `extensions/bridges/*`) to v2 in the same change.
- MUST keep every production file under the 500-line cap — plan the file split (manifest types / permissions validation / consent derivation) before writing.
</requirements>

## Subtasks

- [x] 2.1 Manifest v2 types + load/validate closed sets + deterministic replacement errors (delete Actions/Security everywhere)
- [x] 2.2 Consent-area derivation from the generated table; `EffectiveGrant.Security` becomes derived-only
- [x] 2.3 Global-stream schema change: columns + `extension_dev_links` side table, Goose migration, `atlas.sum`/sqlc via `make codegen`
- [x] 2.4 `HookSourceExtension` value + tier + manager stamping; alias deleted
- [x] 2.5 ADR-007 config cut: structs/defaults/merge/validation/`config set` registry/examples
- [x] 2.6 In-repo manifests regenerated to v2 (dev-cycle + 8 bridges)
- [x] 2.7 Implement every assigned test case; extend the canonical migration suites (no parallel suite)

## Implementation Details

TechSpec: Core Interfaces (Manifest v2, `DeriveConsentAreas`, `HookSourceExtension`), Data Models (columns + side-table rationale), Delete Targets, Safety Invariant 6, Config Lifecycle. Phase B gate: migration suites + scoped tests.

### Relevant Files

- `internal/extension/{manifest.go,manifest_load.go,manifest_compatibility.go,manifest_errors.go,capability.go,capability_host_api.go}` — manifest v2 + closed-set validation + mismap fix
- `internal/extension/manager.go:48` — hook-source alias delete; manager stamping
- `internal/hooks/{types.go,ordering.go}` — new source value + tier (suite: `ordering_test.go`). Fan-out: const block + `hookSourceNames` + `Validate` (types.go), `DefaultHookPriority` (ordering.go:12), `hookSourceValues()` in `internal/codegen/sdkts/generate_maps.go:157`, and regenerated `sdk/typescript/src/generated/contracts.ts`
- `internal/store/globaldb/schema/definitions/33_extensions.sql` — the extensions-table fragment to reshape (current columns incl. `actions`, `capabilities`); new `extension_dev_links` fragment; `internal/store/globaldb/schema/migrations/00028_*.sql` (next after `00027_schema.sql`) + `atlas.sum` via `make codegen` (canonical suites `internal/store/migrate_test.go` + `migrate_streams_test.go`)
- `internal/config/{extensions_marketplace.go,marketplace.go,defaults.go,config_marketplace_validation.go,merge_extensions_marketplace.go}` — ADR-007 cut (suites: `config_test.go`, `merge_test.go`)
- `internal/cli/config_extensions.go` — key registry (suite: `internal/cli/config_test.go`)
- `extensions/{dev-cycle,bridges/*}/extension.toml` — v2 regeneration

### Dependent Files

- `sdk/go` + `sdk/typescript` definition types — `ExtensionDefinition.Actions/Security` deletions ride the same change (contract from task_01)
- `internal/api/contract` — `ExtensionPayload.permissions_json` projection consumers (task_05 completes payload additions)

### Related ADRs

- [ADR-004: Single permissions list](adrs/adr-004.md) — primary
- [ADR-007: Extension config consolidation](adrs/adr-007.md) — primary
- [ADR-006: Closed-surface positioning](adrs/adr-006.md) — `bridge.adapter` install rejection
- [ADR-002: First-class dev lane](adrs/adr-002.md) — `extension_dev_links` schema (consumed by task_04)

### Competitor References

- `.resources/openclaw/docs/plugins/manifest.md:29-31` — single-list permission declaration precedent

## Web/Docs Impact

none in this task — `config-toml.mdx` `[extensions.*]` table, permissions/consent reference, and manifest v2 reference are owned by task_09; no `web/` change (consent-area display strings reach the web through existing payloads).

**QA impact**: reset the `docs/qa/scenarios/` extension-install/config scenarios that exercise `extensions.marketplace.*` keys or `[actions]`/`[security]` manifests (ET-015..ET-023 family — reset the affected files' `qa_status` to `untested`); add one content-addressed `untested` scenario for the legacy-key rejection UX (validation error naming the replacement).

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: manifest schema v2 is the authored extension contract cut; hook source tier changes extension-declared hook ordering; in-repo extension manifests co-migrate.
- Agent Manageability: `compozy config set` registry covers every new `[extensions.*]` key and rejects deleted ones naming the replacement (deterministic errors); `compozy hooks list -o json` reports `source: "extension"`.
- Config Lifecycle: full ADR-007 table (trust/sources/dev keys + defaults + merge/overlay + validation + examples + generated docs + tests) — this task owns it end to end.

## Deliverables

- Manifest v2 with closed sets + derived consent live; phantom capabilities impossible
- Migration applied with green canonical suites and `make codegen-check`
- `HookSourceExtension` truthful in introspection
- ADR-007 config surface complete (structs → docs examples)
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-001, UT-002, UT-003, UT-004, UT-005 — manifest v2 load, closed sets, replacement errors, compat floor
- [x] UT-006, UT-007 — consent derivation happy/error
- [x] UT-035, UT-036, UT-037 — hook source string/priority, ordering, manager stamping
- [x] UT-056, UT-057, UT-058, UT-059 — config defaults, legacy-key rejection, overlay merge, `config set` registry
- [x] IT-001 — extensions-table v2 migration in the canonical fresh/reopen/ahead/integrity suites
- [x] IT-012 — extension-declared hook fires with `source: "extension"` priority 300

## Success Criteria

- Every assigned test case implemented and passing
- Zero references to `Actions`/`Security`/`extensions.marketplace` remain outside migration history (repo-wide grep clean)
- `compozy config set extensions.trust.allow_unverified true` round-trips; legacy key errors name it
- All in-repo manifests load under v2 on a stamped daemon
