---
status: completed
title: "Hard cut: delete the Bundle product across every surface (Phase D)"
type: backend
complexity: high
---

# Task 2: Hard cut — delete the Bundle product across every surface (Phase D)

## Overview

With every replacement live (task_01), this task hard-deletes the public Bundle concept: the `internal/bundles` package, the bundle schema/loaders in `internal/extension`, all `/api/bundles/*` routes and OpenAPI operations, the `compozy bundle` CLI, the `compozy__bundles_*` native tools/toolset/capabilities, marketplace kind `bundle`, the resource kinds and the mixed-kind projector seam, the skills `installed_from_bundle` field, and the orphaned data — leaving greps for product Bundle APIs with only historical/homonym hits. Homonyms (support bundles, SourceBundled, ContextBundle, crash bundles, dev-lane "bundle generation") must survive byte-identical.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST delete every item in the TechSpec **Delete Targets** section (Go runtime, generated artifacts, store, boundaries rule) — whole-file deletes and ⚠-marked shared-file strips exactly as enumerated; no fallback, no compat shim, no `@deprecated` stub, no alias.
- MUST ship migration `00030`: `DELETE FROM resource_records WHERE kind IN ('bundle','bundle.activation')` + `DELETE ... WHERE owner_kind = 'bundle.activation'`; append-only Goose, no boot repair (`eng-schema-migration`).
- MUST remove `ErrExtensionHasActiveBundles` and the registry's direct `resource_records` SQL scan — disable/remove gate only on extension state.
- MUST delete the `PackageOwned` bundle arm (task_01 left it dual) so `PackageOwned == (Owner.Kind == "extension")`.
- MUST delete `InstalledFromBundle` end to end (skills types/spec/payload/CLI) and the `extension_blocked_by_bundle` diagnostic code; support-bundle diagnostic codes stay.
- MUST delete the `internal/resources` mixed-kind escape hatch (`BundleActivationProjector` + registration + validation) and rename `"bundle"`/`"bundle.activation"` fixture kinds in the resources suites to neutral kinds (rename, not delete).
- MUST relocate/rename shared helpers instead of deleting them: `cloneBundleTaskConfig` (used by the automation host API), the `compozy__resources_*` descriptors co-located in `bundles_resources.go`, CLI client shared files (strip methods only).
- MUST reword — not delete — homonym-adjacent doc comments (`JobSourcePackage`, `BridgeInstanceSourcePackage`, extension-archive "bundle" wording).
- MUST regenerate every generated artifact clean (`openapi/compozy.json` −tag/−8 ops, `web/src/generated/compozy-openapi.d.ts`, `routeTree.gen.ts` is task_03, sqlc −`ListBundleActivationResourceSpecs`, native-tool catalog −5 rows, CLI docs −`bundle` group) and remove the `magefiles/boundaries.go` marketplace↛bundles rule in the same change.
- MUST keep every homonym surface untouched and prove it (homonym smoke: support bundle create/status/download, `--source bundled` skills, ContextBundle, crash bundles, `outputBundle`, sigstore/JS bundling, dev-lane generation vocabulary).
- MUST leave the marketplace kind set at `{extension, skill, mcp}` across contract/spec/handlers (web strips are task_03).
- MUST strip product-bundle cases from shared Go suites (extensions/payload/marketplace/network/resources/parity/native-tools/toolmeta) in the same change that deletes the behavior.
- Skills to activate: `eng-code-guidelines`, `golang-master`, `eng-schema-migration` (00030), `eng-contract-codegen-coship`, `eng-consolidate-test-suites`, `deslop` before completion.
</requirements>

## Subtasks

- [x] 2.1 Delete `internal/bundles/**` + daemon bundle wiring/publishers/native tools + error maps; re-file `bootBundles` siblings by responsibility
- [x] 2.2 Delete extension bundle schema/loaders/snapshot fields/surface families; drop `resources.bundles`; delete `ErrExtensionHasActiveBundles` + activation scan; collapse `PackageOwned` to extension-only
- [x] 2.3 Delete contract/spec/routes: `contract/bundles.go`, marketplace bundle kind/payloads, `registry_bundles.go` + registrations, HTTP/UDS route blocks + plumbing, network-observability bundle merge, resource mutate block
- [x] 2.4 Delete CLI `bundle*` files + root registration; strip client methods + fakes
- [x] 2.5 Delete native tools/toolset/capabilities + toolmeta rows; split `bundles_resources.go` keeping `compozy__resources_*`
- [x] 2.6 Delete `internal/resources` mixed-kind seam; rename fixture kinds in resources suites
- [x] 2.7 Delete skills `InstalledFromBundle` + diagnostic code; delete store `global_db_bundles.go` + sqlc query; migration `00030`
- [x] 2.8 Boundaries rule removal; doc-comment rewording; regen all artifacts; strip shared Go suites
- [x] 2.9 Grep gate + homonym smoke evidence

## Implementation Details

The TechSpec **Delete Targets** section is the authoritative inventory (file:line-verified); execute it top to bottom. Order within the task: replacements are already live, so deletion order is compile-driven (delete package → fix importers → regen). Migration `00030` lands only after the projector code is gone from the build.

### Relevant Files

- Everything under **Delete Targets** in `_techspec.md` — the enumerated list is the file set; do not re-derive it
- `internal/extension/host_api_workspace_automation_mapping.go:113,139` — `cloneBundleTaskConfig` rename target
- `internal/tools/builtin/bundles_resources.go` — split point (`compozy__resources_*` stays)
- `internal/daemon/agent_skill_catalog.go:171` — PackageOwned collapse
- `internal/resources/{projector.go,reconcile_test.go,reconcile_integration_test.go,kernel_test.go,typed_test.go,typed_integration_test.go,perf_bench_test.go}` — seam delete + fixture-kind renames
- `internal/store/globaldb/{global_db_bundles.go,queries/global_small.sql,repositories.go}` + `schema/migrations/` (next: `00030`)
- `magefiles/boundaries.go:156`

### Dependent Files

- `openapi/compozy.json`, `web/src/generated/compozy-openapi.d.ts`, `internal/store/globaldb/sqlcgen/**`, `internal/tools/builtin/testdata/native-tool-catalog.json`, generated CLI docs — all regenerate
- Shared Go suites listed in Delete Targets §tests — strip bundle cases
- `internal/api/{httpapi,udsapi}` parity suites — route tables shrink

### Related ADRs

- [ADR-001](adrs/adr-001.md) — the cut itself (scope, homonym keep-list, no-zombie rule); [ADR-004](adrs/adr-004.md) — PackageOwned collapse.

### Web/Docs Impact

- `web/`: generated types refresh only (`make codegen`); the web source deletes/strips are task_03 (the web build may carry dead bundle code until then — acceptable, it compiles against its own copies of types until task_03 lands; do not hand-edit web source here).
- `packages/site`: none this task — docs deletions/rewrites are task_04.
- QA impact: user-visible surface removed (`compozy bundle`, `/api/bundles/*`, 5 native tools, marketplace kind) — scenario deletions/strips (`ET-024..030`, `NB-023`, `ET-web-bundle-*`, `ET-020`/`ET-033`/`ET-cli-marketplace-refresh`/`RT-reserved-builtin-agent-names` strips) are authored in task_04 with the QA tracker pass.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: surface families `bundles`/`bundle_activations` and resource kinds `bundle`/`bundle.activation` deleted; marketplace kinds = `{extension, skill, mcp}`; hooks/MCP/bridges/registries otherwise untouched (checked: no hook events existed; `internal/registry` has no bundle coupling).
- Agent manageability: `compozy bundle *` group, 8 routes, 5 native tools + toolset + `bundles.read|write` capabilities removed; no replacement needed — kit management shipped in task_01 (inventory/preview/secrets/extension lifecycle).
- Config lifecycle: none — no bundle config key ever existed (checked `internal/config` + site `config-toml.mdx`; only homonyms).

## Deliverables

- Zero living references to product Bundle APIs (`/api/bundles`, `compozy bundle `, `compozy__bundles_`, `MarketplaceKindBundle`) outside historical/homonym hits — grep evidence in completion notes
- Migration `00030` applied with clean fresh/reopen + cleanup semantics
- All generated artifacts regenerated; boundaries gate green
- Homonym smoke evidence (support bundles, bundled skills, ContextBundle)
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-055 — `ExtensionPayload` final shape (new fields present, `bundles` gone)
- [x] UT-056 — skills payload/spec without `installed_from_bundle`
- [x] UT-057 — surfaces registry without bundle families; no activation gate on disable/remove
- [x] UT-058 — native-tool catalog golden (new tools present, bundle tools/capabilities absent)
- [x] IT-002 — migration `00030` cleanup on seeded DB (bundle kinds + activation-owned rows deleted; homonyms preserved)
- [x] IT-011 — transport parity: bundle routes absent from both routers + spec registry
- [x] IT-012 — native tools: inventory/preview parity, bundle tools absent from availability/bindings
- [x] IT-016 — homonym smoke (support bundles, bundled skills, ContextBundle suites green)
- [x] E2E-006 — `compozy support bundle` create → status → download post-cut

## Success Criteria

- Every assigned test case implemented and passing
- Scoped Go gates green (`make lint`, `go test -race` over touched packages); `make codegen-check` green
- Grep gate output attached to completion notes showing only historical/homonym hits
- No file named or shaped around product bundles remains in `internal/`, `cmd/`, `magefiles/`

## Completion Notes

- Deleted the Bundle runtime, routes, CLI, native tools, resource/store seams, marketplace kind, extension schema arms, and generated contract/catalog entries without compatibility aliases.
- Added the gap-free append-only migration as `00040_schema.sql`; the authored `00030` label predated migrations already present on this branch.
- Product-surface greps are empty for `/api/bundles`, `compozy bundle`, `compozy__bundles_*`, `MarketplaceKindBundle`, `InstalledFromBundle`, and the deleted web activation APIs. Remaining `bundle.activation` hits are limited to the cleanup migration and its preservation test.
- Homonyms remain covered: support bundles, `SourceBundled`, `ContextBundle`, CLI `outputBundle`, crash bundles, and build bundling vocabulary were not removed.
- Compozy Impact Audit: native Bundle IDs/toolset/capabilities were deleted while extension inventory/preview remain; bundle extension/resource/registry seams were deleted with no config keys affected; removed records were global and the surviving extension instance paths preserve explicit workspace scope; official `skills/compozy/` documentation updates remain owned by task 04.
