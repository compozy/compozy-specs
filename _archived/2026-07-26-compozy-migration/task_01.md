---
status: completed
title: Rebrand core identity — home dir, database, binary, module path
type: refactor
complexity: high
---

# Task 01: Rebrand core identity — home dir, database, binary, module path

## Overview

First class of the zero-legacy rebrand hard cut: the single-constant identity surfaces (`.agh/` → `.compozy/`, `agh.db` → `compozy.db`, `agh.log` → `compozy.log`, cache/lock dir), every shipped helper command, and the Go module path (`github.com/compozy/agh` → `github.com/compozy/compozy`). This runs first because every later phase would otherwise double-touch the same files.

<critical>
- ALWAYS READ `_brief.md`, `_techspec.md`, `_content-plan.md`, `_tests.md`, every ADR, and any per-task memory before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — a hard-cut rename deletes the old identifier; it never adds an alias, dual-read, migration bridge, or fallback
</critical>

<requirements>
- MUST run on branch `v0.3` of `github.com/compozy/compozy` after the Stage-0 tree copy (brief round-8); `main` is untouched v0.2.x.
- MUST change `DirName` to `.compozy` in `internal/config/config.go:26`, replace all production layout literals that bypass it, and make workspace/home paths derive from that constant or its owning helper.
- MUST rename both `config.DatabaseName` and `store.GlobalDatabaseName` to `compozy.db`; `SessionDatabaseName` (`events.db`) is brand-free and MUST NOT change. Rename `LogFileName` to `compozy.log` in the same home-layout change.
- MUST rename every shipped helper command and its invocation path: `cmd/agh` → `cmd/compozy`, `cmd/agh-codegen` → `cmd/compozy-codegen`, `cmd/agh-catalog` → `cmd/compozy-catalog`, and `internal/sandbox/daytona/cmd/agh-daytona-sidecar` → `.../compozy-daytona-sidecar`. Update generated headers, asset names, CI, Mage, scripts, and release references in the same change.
- MUST change the Go module path via `go mod edit` and rewrite every import site tree-wide, including `magefiles/`, `extensions/`, and ldflags version paths.
- MUST rename the verify-lock and mage cache dir names in `magefiles/verifylock.go:18` and `scripts/run-mage.sh:14` (`agh-dev` → `compozy-dev`), keeping `AGH_VERIFY_LOCK`/`AGH_MAGE_CACHE_DIR` env names untouched — env namespace belongs to task 02.
- MUST replace the live `agh-catalog` registry/source identity and `.agh-catalog-dirty` marker with their `compozy` forms. They are persisted identity values, so clean restart is accepted and no old-state reader is permitted.
- MUST consume the externally released `github.com/compozy/compozy-web-assets` module after its companion-repository publication, using `go get`; replace the old import, Mage source repository, and public-module checks without retaining `agh-web-assets`. Publication is an external dependency, not a compatibility exception.
- MUST update the existing official `skills/agh/` bodies for the renamed binary, Go module, home/database/log paths, and helper commands in this same change. Task 04 owns only the later directory/name/metadata hard cut; it must not inherit stale Task 01 guidance.
- MUST NOT introduce aliases, dual constants, or data-migration shims for the old database file — clean restart is in policy (brief; ADR-006).
- MUST leave a dated note in the replacement commit body citing the source AGH SHA.
- SHOULD update existing home/config/store test suites in the same change rather than adding new ones — the invariants already have owning suites.
</requirements>

## Subtasks

- [x] 1.1 Flip `DirName`, eliminate layout literals that bypass it, and verify every home/workspace path follows the canonical owner
- [x] 1.2 Rename both global database owners and the home log filename while leaving brand-neutral session files unchanged
- [x] 1.3 Rename the CLI, codegen, catalog, and Daytona sidecar commands, their generated headers/assets, and every build/release/CI invocation
- [x] 1.4 Rewrite the Go module path and all import sites, including magefiles and extensions
- [x] 1.5 Rename verify-lock and mage cache directory names (env var names excluded)
- [x] 1.6 Replace catalog/dirty-marker identities and consume the externally published Compozy web-assets module
- [x] 1.7 Update the official `skills/agh/` command, module, and filesystem guidance in the same hard cut
- [x] 1.8 Update existing home/config/store/command/lifecycle test suites to the new literals (co-ship, no new suites)
- [x] 1.9 Run the executable-source class grep gate and drive its allowlist to zero
- [x] 1.10 Run `make verify` and confirm the boundaries check still passes with the new module path

## Implementation Details

Constant-first rename: every runtime layout path in this task must flow from a single constant or a mechanical module-path rewrite. Where a literal appears outside an owning constant (build scripts, generated headers, CI, Goreleaser, or packaged sidecar assets), update it in the same change. See TechSpec §Delete Targets (identity hard cut) and §Development Sequencing step 1.

The grep gate for this class runs as a task verification command, not a suite test (repo test-placement policy). Scope it to executable sources and inputs: Go, shell, Makefile/Mage, workflow YAML, Goreleaser, manifests, generated-file headers, and runtime test fixtures. Exclude only authored/generated documentation deferred to tasks 03–05 and `.compozy/tasks/**`; the executable-source allowlist is zero before closing.

### Relevant Files

- `internal/config/config.go:26` — `DirName = ".agh"`; the single source for home and workspace overlay paths (112 refs / 24 files)
- `internal/config/home.go` — `config.DatabaseName`, `LogFileName`, `ResolveHomeDir`, `ResolveHomePaths`, `EnsureHomeLayout`, and layout dir constants
- `internal/store/store_paths.go:7,9` — `SessionDatabaseName` (unchanged) and `GlobalDatabaseName = "agh.db"` (renamed); note `internal/store/store.go` is a package-doc stub
- `internal/bundles/service_materialize_automation.go`, `internal/memory/store_scope.go`, `internal/skills/loader_values.go`, `internal/soul/soul_path.go`, `internal/heartbeat/source_path.go` — live layout literals that must stop bypassing `DirName`
- `cmd/agh/`, `cmd/agh-codegen/`, `cmd/agh-catalog/`, `internal/sandbox/daytona/cmd/agh-daytona-sidecar/` — all command/source directories and generated identity headers being hard-cut
- `magefiles/codegen.go`, `magefiles/daytona*.go`, `scripts/catalog-digest.sh`, `.github/workflows/catalog.yml` — invocations of the helper commands
- `internal/marketplace/projection.go`, `internal/extension/marketplace_trust.go`, `internal/memory/catalog_dirty_marker.go` — catalog identity and marker values
- `magefiles/verifylock.go:17-19` — `verifyLockDirName = "agh-dev"`, lock filename, env var name
- `scripts/run-mage.sh:14` — separate mage cache dir under `agh-dev`
- `.goreleaser.yml:30-47` — build id/binary/main path and `-X github.com/compozy/agh/internal/version.*` ldflags
- `go.mod`, `internal/api/httpapi/static.go`, `magefiles/{defaults,web_assets,web_assets_publish}.go` — externally published web-assets module, embed provenance, and publish/check paths
- `magefiles/boundaries.go` — import-graph check that must stay green under the new module path

### Dependent Files

- Every Go file in `cmd/`, `internal/`, `extensions/`, `magefiles/` — module path rewrite
- `internal/config/home_test.go`, `internal/store/*_test.go`, and command-specific suites — canonical ownership for home/database/log/command path invariants
- `internal/cli/lifecycle.go` + `lifecycle_test.go` — managed-lifecycle strings reference the binary name
- `web/` and `packages/site` — untouched in this task; their branding is task 04
- `skills/agh/**` — official runtime guidance for the binary, module, and renamed filesystem identities; the directory/name rename remains task 04

### Related ADRs

- [ADR-006: Flat `.compozy/` Namespace and Explicit Config Migration](adrs/adr-006.md) — flat merge decision and the no-data-shim rule this task implements

## Deliverables

- `.compozy/` home and workspace overlay resolved from a single constant, with `compozy.db` and `compozy.log` as global home artifacts
- `cmd/compozy`, `cmd/compozy-codegen`, `cmd/compozy-catalog`, and the Compozy Daytona sidecar building through their updated paths
- Module path `github.com/compozy/compozy` across the tree with boundaries check green
- Renamed verify-lock/mage cache dirs and catalog identities
- External dependency recorded: the released `github.com/compozy/compozy-web-assets` companion module is required before its import can be switched
- Class grep gate at zero and `make verify` green
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from [`_tests.md`](_tests.md), the test contract — read each ID's full definition there before writing tests.

- [x] No `_tests.md` IDs are assigned to this task. The rename invariants are owned by existing home/config/store, codegen/catalog/Daytona, and lifecycle suites, plus `make verify` and the executable-source class grep gate. The behavior-bearing environment-override smoke remains IT-016 in task 02, once the environment namespace flips.

### Web/Docs Impact

- `web/`: none — checked surfaces: `web/src/systems/**`, `web/src/generated/**`, MSW fixtures; reason: no contract, route, or payload changes in this task; web branding strings are task 04.
- `packages/site`: none — checked surfaces: `content/runtime/cli-reference/**` (regenerated later in P1-f), `content/runtime/core/**`; reason: generated CLI docs regenerate from the cobra tree in task 04, never hand-swept here.
- QA impact: new scenarios — add content-addressed `untested` scenario files at completion for the renamed home directory, database filename, and binary name (user-visible install/runtime paths). Flag, don't retest.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: `extensions/dev-cycle` import paths follow the module rewrite; catalog registry/source identity follows this hard cut. `min_agh_version` becomes `min_compozy_version` in task 02; no hook or bundle semantics change here.
- Agent manageability: no CLI verb, HTTP endpoint, or UDS route is added, removed, or reshaped. `compozy status`/`doctor` report the new paths as a consequence of the constants; structured output shape is unchanged.
- Config lifecycle: no `config.toml` key is added, renamed, or removed. The config *file location* moves with `DirName`; the root example `config.toml` socket path updates in the same change. Legacy config translation belongs to task 09.

### AGH Impact Audit

- Native tools: no tool IDs, toolsets, descriptors, schemas, or capability gates change; checked `internal/tools/**`, hosted MCP projection, and generated catalog ownership. Binary invocation paths change before task 02 renames identifiers.
- Extensibility and hooks: module imports, catalog registry/source identity, helper commands, and packaged Daytona assets change; checked extension manifests, hooks, skills/capabilities, tools/resources, bundles, registries, bridge SDKs, and MCP sidecars. Manifest version-key work is explicitly owned by task 02.
- Workspace data isolation: home/workspace layout and database/log paths are global and workspace-scoped filesystem identities; `DirName`/home-path suites must prove workspace overlays cannot resolve an old `.agh` path. No workspace_id payload, cache, SSE, or event schema changes.
- Official AGH skill: keep the `skills/agh/` directory/name until task 04, but update its live binary, module, helper-command, home, database, and log references in this task so the public contract co-ships with the hard cut.

## Success Criteria

- Every assigned test case implemented and passing
- `make verify` green, including `mage Boundaries` under the new module path
- Zero occurrences of the old module path, `cmd/agh`, `.agh` `DirName`, and `agh.db` in the class grep gate scope
- A fresh daemon boot creates `~/.compozy/` and `compozy.db` with no legacy-path fallback code anywhere in the tree
