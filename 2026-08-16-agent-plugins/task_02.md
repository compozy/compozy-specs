---
status: completed
title: Ingestion core: detection, synthesis, persistence, lifecycle
type: backend
complexity: critical
---

# Task 2: Ingestion core: detection, synthesis, persistence, lifecycle

## Overview

Turns a validated Agent Plugins package into a first-class extension: the installer recognizes the third root layout, the manifest loader synthesizes a resource-only manifest in memory (ADR-005), persistence gains the six instance-scoped columns, and every mutating lifecycle path routes through the per-`InstanceKey` lifecycle coordinator with fixed commit order, boot reconciliation, data-directory quarantine semantics, and dev-link parity. This is the critical-path task: it owns the schema migration, the concurrency primitive, and the kit-publication integration.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — repair root ownership and failure semantics; do not patch symptoms
</critical>

<requirements>
1. MUST extend installer root detection with `plugin.json` as the third accepted root under the fixed precedence matrix (Part I Business Rule 4 / Safety Invariant 9): both-native → existing hard error byte-identical (with or without plugin.json); one native root beats plugin.json; portable only when no native root; client-specific dirs never participate; a selected invalid root never falls through. Detection uses `agentplugin.ClassifyManifest` triage; existing archive hardening is untouched.
2. MUST implement `SynthesizeAgentPluginManifest` + the `LoadManifest` branch per ADR-005: in-memory only, explicit per-skill paths, MCP entries mapped onto the widened `MCPServerConfig` shape, empty provides/permissions/subprocess, `min_compozy_version` gate waived for `format=agent-plugin` (native gate unchanged), deterministic output.
3. MUST land ONE appended gap-free Goose migration on the global stream adding all six columns (`extensions.format`/`.ingest_diagnostics_json`, `extension_dev_links.format`/`.ingest_diagnostics_json`, `extension_env_bindings.mcp_server`/`.header_name`) with the both-or-neither `CHECK` on the binding mapping; declarative schema + `atlas.sum` + sqlc refresh via `make codegen`; applied-migration bytes immutable (activate `eng-schema-migration`).
4. MUST route every mutating path (CLI, HTTP/UDS, marketplace, dev reload, native tools) through the per-`InstanceKey` lifecycle coordinator with the fixed commit order stage → validate → final move → registry transaction → post-commit cleanup; boot reconciliation removes row-less staged/final directories before extension traffic; retries idempotent (Safety Invariant 10).
5. MUST implement the data-directory contract: dedicated root `$COMPOZY_HOME/extension-data/<name>` (global) / `<name>@ws-<workspace_id>` (dev links) via a `HomePaths`-owned accessor; path derived at synthesis, directory NOT created at install/update/enable; preserved bit-for-bit across updates; on remove, delete → quarantine-rename on failure → deterministic remove failure if quarantine also fails (Safety Invariant 11; a completed remove never leaves reachable data under the key).
6. MUST persist `format` + ingest diagnostics on the owning instance row (global vs dev link), replace the dev link's diagnostics atomically with its `bundle_generation` on link/reload, and convert `agentplugin.Diagnostic` → `contract.DiagnosticItem` (code `extension_agent_plugin_component_skipped`, category `extension`, `evidence.scope`) with the total order (source set → scope/server → code → message).
7. MUST re-run the full ladder on update (diagnostics replaced with content), keep enable/disable kit publication unchanged (ingested skills + `mcp_server` resources publish through existing paths), and keep trust/provenance/scan behavior identical for the new layout.
8. MUST NOT create any parallel queue, second registry, or `EnsureSchema`-style repair; MUST NOT materialize a synthesized manifest to disk.
</requirements>

## Subtasks

- [x] 2.1 Installer detection: third root + precedence matrix + updated missing-manifest message naming all three roots
- [x] 2.2 Synthesis + `LoadManifest` branch + `ExtensionFormat` + min-version waiver
- [x] 2.3 Goose migration (six columns + CHECK) + declarative schema + `make codegen` artifacts
- [x] 2.4 Lifecycle coordinator: commit order, per-boundary failure injection seams, boot reconciliation, idempotent retry
- [x] 2.5 Data-directory accessor (dedicated root, instance encoding incl. the `data`-named-package proof) + quarantine-on-remove semantics
- [x] 2.6 Registry/row plumbing: format + diagnostics persistence (global + dev link), update-replaces-diagnostics, provenance untouched
- [x] 2.7 Kit publication integration: enabled portable instance publishes skills + mcp_server resources; disable unpublishes; running sessions untouched
- [x] 2.8 Dev-link flow: link/reload with per-generation diagnostics, last-good retention on fatal edit, workspace/global isolation
- [x] 2.9 Full assigned unit + integration suites, including concurrency races through the coordinator and the interruption injections

## Implementation Details

Follow `_spec.md` Part II → System Architecture (data flow + coordinator), Implementation Design (interfaces + Data Models + migration contract), Safety Invariants 8–12. Key existing anchors: `internal/registry/installer_metadata.go:149-172` (`manifestNameAtRoot`, wrapper descent), `internal/extension/manifest_load.go` (branch point), `internal/extension/install_managed.go:45-65,132-183` (containment + checksum + move), `internal/extension/marketplace_update.go` (update flow), `internal/extension/dev.go` + `manager_dev_*.go` (dev links, generations), `internal/extension/registry_install.go` / `registry_types.go` / `registry_scan.go` (row plumbing), `internal/store/globaldb/schema/definitions/33_extensions.sql` + `34_extension_dev_links.sql` (schema sources), `internal/config/home.go:57` (`HomePaths` — gains the extension-data accessor), `internal/daemon/extension_kit_resources.go` + `agent_skill_publisher.go` (kit publication).

Reference implementations:

- Data-root collision lesson (hash-keyed, NUL separator — ours is simpler because the root is dedicated, but read why theirs exists): `.resources/codex/codex-rs/core-plugins/src/store.rs:141-156`
- Format-gated behavior forks at load: `.resources/codex/codex-rs/core-plugins/src/loader.rs:880-882,1570-1585`
- Degenerate-native-manifest synthesis + shared probe/scanner discipline: `.resources/hermes/hermes_cli/plugins.py:4208-4248,4099-4153`
- Native-wins detection predicate in ONE function: `.resources/hermes/hermes_cli/plugins_cmd.py:1638-1648`
- Real `git init` + `file://` install test: `.resources/hermes/tests/hermes_cli/test_plugins_cmd.py:761-796`

Skills to activate: `eng-code-guidelines`, `golang-master`, `eng-schema-migration`, `eng-test-conventions`, `testing-boss`, `eng-consolidate-test-suites`, `eng-cleanup-failure-paths`.

### Relevant Files

- `internal/registry/installer.go` + `installer_metadata.go` — root detection + hardening (unchanged hardening)
- `internal/extension/manifest.go` / `manifest_load.go` / `manifest_compatibility.go` — manifest shape, load branch, min-version gate
- `internal/extension/install_managed.go`, `marketplace_lifecycle.go`, `marketplace_update.go`, `marketplace_update_apply.go` — install/update flows the coordinator wraps
- `internal/extension/registry.go` / `registry_install.go` / `registry_types.go` / `registry_scan.go` — persistence
- `internal/extension/dev.go`, `internal/daemon/extension_dev_lifecycle.go` — dev links + rollback
- `internal/store/` migration streams + `atlas.sum` — owning stream for the migration
- `internal/testutil/` + `internal/extension/testdata/` — fixture homes (new conformant package fixture set lands here)

### Dependent Files

- `internal/extension/describe.go` — task_04 projects `format`/diagnostics from the row this task persists
- `internal/api/contract/extensions.go` — task_04 widens payloads over this task's data
- `internal/mcp/` executor — task_03 consumes synthesized `ServerSpec`-derived config + data-dir path
- `internal/daemon/boot_*extensions*.go` — boot reconciliation wiring

### Related ADRs

- [ADR-001: Third install layout](adrs/adr-001.md) — the install-flow contract
- [ADR-004: Verbatim names](adrs/adr-004.md) — identity through registry + containment
- [ADR-005: In-memory synthesis](adrs/adr-005.md) — the load-branch contract this task implements

### Web/Docs Impact

- `web/`: none in this task — payload/inventory contract changes land in task_04; web consumption in task_05 (checked: no `internal/api/contract` change here).
- `packages/site`: none — docs land in task_06.
- QA impact: new behavior (portable install/update/remove/dev-link lifecycle) — add content-addressed `untested` scenario files covering install-from-dir/git, dual-manifest note, remove-with-data, dev-link reload (files added in task_06's QA-flag sweep; walked in the loop's QA phase by task_08).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: installer third layout IS the extension-surface change (ADR-001); kit publication reused unchanged; hooks/provides/permissions/bridge SDKs/MCP sidecars untouched (checked: `capabilities.go`, surface registry, bridge contract).
- Agent manageability: none new in this task — behavior reaches agents via task_04's surfaces; lifecycle coordinator serves all existing verbs identically.
- Config lifecycle: zero `config.toml` keys (checked: `ExtensionsConfig` sections; trust key reused verbatim). Schema migration follows the append-only contract with `make codegen-check` evidence.

## Deliverables

- Third-layout install/update/remove/dev-link working end-to-end against the committed conformant fixture set (incl. `data`-named, 30-skill, fully-degraded, git-source variants)
- One appended migration + refreshed declarative schema/`atlas.sum`/sqlc output; `make codegen-check` clean
- Lifecycle coordinator with boot reconciliation and injection seams
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**
- `make gate` green for the affected lanes

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-027, UT-028, UT-029, UT-030, UT-031, UT-032, UT-033 — synthesis mapping, load branch, precedence, no-fallback, waiver, determinism, diagnostics conversion
- [x] UT-038, UT-039, UT-040, UT-041 — registry detection: third root, three-roots error, both-native regression, client-dir non-participation
- [x] UT-053 — data-dir encoding table (incl. `data`-named package disjointness)
- [x] UT-057 — diagnostic total-order repeatability
- [x] IT-001, IT-002 — install happy variants (no data dir at install) + fatal atomicity
- [x] IT-004, IT-005 — kit publication + session merge (env baked, host-env literal, name-collision precedence)
- [x] IT-007, IT-008 — update ladder re-run + data preservation; drifted-layout update failure
- [x] IT-009 — remove + quarantine semantics + double-failure remove error
- [x] IT-010 — dev-link per-generation diagnostics + isolation matrix
- [x] IT-014, IT-016 — cross-surface races through the coordinator; per-boundary interruption injection + boot reconcile
- [x] IT-018 — migration suite: six columns, backfills, CHECK shapes, ahead/integrity/equivalence

## Success Criteria

- Every assigned test case implemented and passing
- A package legally named `data` installs/updates/removes without touching any other instance's data (B-001 regression proven)
- Interruption injection at every commit boundary leaves no partial state visible on any read surface; re-run succeeds
- `make codegen-check` clean; `mage Boundaries` clean
