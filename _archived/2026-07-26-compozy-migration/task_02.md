---
status: completed
title: Rebrand namespaces — environment variables and native tool IDs
type: refactor
complexity: high
---

# Task 02: Rebrand namespaces — environment variables and native tool IDs

## Overview

Second rebrand class: the `AGH_*` environment namespace becomes `COMPOZY_*`, native tool and toolset IDs (`agh__*`) become `compozy__*`, and the MCP projection identity (`agh_host__*`, hosted-server names) moves with them. These are agent-visible contracts — tool IDs and MCP façade names address the runtime, while env names configure it — so descriptor catalogs, guards, hosted projections, UI metadata, and provider policies co-ship.

<critical>
- ALWAYS READ `_brief.md`, `_techspec.md`, `_content-plan.md`, `_tests.md`, every ADR, and any per-task memory before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — do not retain an old environment read, identifier alias, dual manifest key, or compatibility projection
</critical>

<requirements>
- MUST rename every `AGH_*` environment variable to `COMPOZY_*`, including home/identity, QA/build/test variables, `AGH_MANAGED`, `AGH_MCP_SERVE_TOKEN`, web distribution/assets/proxy variables, hosted-MCP helpers, bridge adapter templates, and every name used only by fixtures or scaffolding.
- MUST rename native ToolIDs, ToolsetIDs, reserved-prefix guards, descriptor globs, tool metadata, UI tool-label/palette contracts, and every fixture that serializes an `agh__*` identifier.
- MUST hard-cut the MCP façade identity: `agh_host__*` → `compozy_host__*`, `agh-host-api`/`agh-hosted-tools` → their Compozy names, and every `mcp__...` fixture or ledger matcher. These are not native ToolIDs, but they are separately agent-addressable identifiers and cannot retain a dead brand.
- MUST regenerate the native tool catalog owned by `cmd/compozy-codegen`; schema digests remain unchanged when their JSON schemas are unchanged. Do not claim that an ID rename changes a schema-only digest or edit unrelated digest assertions.
- MUST run `make codegen` and `make codegen-check` after the ToolID/toolset/descriptor rename so the native catalog and every dependent generated family are updated by their owners rather than hand-edited.
- MUST update provider env-policy tables, process-env allowlists, bridge manifests/templates, and hosted-MCP env injection so provider auth/home/env resolution keeps working with renamed variables.
- MUST rename the loader-read manifest key `min_agh_version` to `min_compozy_version` across structs, JSON/TOML tags, validation/error text, real manifests, fixtures, and canonical manifest suites. The loader reads this field today, so leaving it for another task would create an invalid hard cut.
- MUST NOT keep dual-read fallbacks for old env names — a missing `COMPOZY_*` variable resolves to its default, never to a legacy name.
- MUST NOT add `t.Parallel()` to tests that use `t.Setenv` while updating env-touching suites (L-002 — Go testing contract).
- MUST update both the official `skills/agh/` bodies and every affected `.agents/skills/**` body for live environment, native-tool, and MCP identifiers in this same change. Keep the development-skill dispatch path names unchanged here; task 04 later owns the official `skills/agh/` directory/name hard cut. (Round 9, 2026-07-26: those paths were renamed to `.agents/skills/eng/eng-*` after this task shipped.)
</requirements>

## Subtasks

- [x] 2.1 Rename the environment namespace across production code, defaults, validation, process-env allowlists, manifests, and templates
- [x] 2.2 Rename env names in test scaffolding and harness helpers without introducing `t.Parallel`/`t.Setenv` conflicts
- [x] 2.3 Update provider env-policy tables, bridge templates, hosted-MCP injection, and provider home/auth resolution paths
- [x] 2.4 Rename native ToolIDs, ToolsetIDs, metadata, UI contracts, descriptors, and every matching glob
- [x] 2.5 Rename MCP façade prefixes/server names and update its existing server/projection/ledger suites
- [x] 2.6 Update the reserved-prefix and `min_compozy_version` manifest guards in their existing canonical suites
- [x] 2.7 Run the full codegen graph, verify the regenerated native tool catalog, and update schema digests only where a schema actually changed
- [x] 2.8 Sweep live env and identifier references inside `skills/agh/**` and `.agents/skills/**` bodies while retaining the then-locked development-skill dispatch path names
- [x] 2.9 Implement UT-057, UT-058 and IT-016
- [x] 2.10 Run executable-source env, native-tool, and MCP-identity grep gates to zero and `make verify`

## Implementation Details

The environment, native-tool, and MCP-projection identities ship together because each is namespace-wide and agent-visible. Follow TechSpec §Delete Targets (identity hard cut) and §Development Sequencing step 1.

Tool ID rename has no single mint site. Sweep descriptor, toolset, metadata, UI, hosted-MCP, and fixture surfaces, then regenerate the catalog. Do not introduce a new prefix abstraction as part of this task; that would be a design change, not a rename.

Each grep gate is explicit and executable: include production source, extension manifests/templates, shell/Mage/workflow inputs, generated-code source headers, runtime fixtures, and tests; exclude only authored/generated documentation deferred to tasks 03–05, `.compozy/tasks/**`, and the development-skill directory names, which were still branded when this task ran. A separate skill-body check permits those directory names but rejects old live env/tool/MCP identifiers in their contents. (Round 9 lifted that carve-out by renaming the tree to `.agents/skills/eng/eng-*`.)

### Relevant Files

- `internal/config/home.go:141` — `AGH_HOME` env read (plus the workspace `.env` variant at :160)
- `internal/agentidentity` — resolves caller identity from `AGH_SESSION_ID` / `AGH_AGENT`
- `magefiles/{verifylock,gotest_lane,defaults}.go`, `scripts/{run-mage,dev}.sh`, and `.github/workflows/release.yml` — build, test, web-assets, and release environment inputs
- `internal/api/httpapi/static.go`, `internal/update/types.go`, `internal/procutil/env.go` — web-dist, managed-install, and child-process environment owners
- `internal/extension/resource_publication.go:156` — reserved tool-prefix guard
- `internal/tools/builtin_ids.go`, `internal/toolmeta/native_entries.go`, `internal/tools/builtin/testdata/native-tool-catalog.json` — ToolIDs, ToolsetIDs, UI metadata, and generated catalog
- `cmd/agh-codegen/main.go` — current catalog generator; it is renamed in task 01 before this task invokes it
- `internal/mcp/{serve,serve_projection,hosted}.go`, `internal/cli/mcp_serve.go` — host-MCP prefix, server names, and token environment default
- `internal/extension/{manifest,manifest_normalize,manifest_load,manifest_errors}.go` and `extensions/**/extension.{json,toml}` — loader-read `min_agh_version` key and manifests
- `internal/config/provider_effective.go`, `provider_builtin.go` — provider env/home policy tables
- `extensions/bridges/**/extension.toml`, `extensions/bridges/**/provider.go`, `internal/providerauth/native_cli.go` — subprocess environment templates and provider launch behavior
- `web/src/systems/session/lib/tool-labels.ts`, `web/src/systems/agent/components/agent-create-access-step.tsx`, `web/src/systems/loops/lib/loop-palette.ts` — live UI contracts for native IDs
- `skills/agh/**` and `.agents/skills/**` — official and development-skill guidance containing live environment and identifier references

### Dependent Files

- `internal/testutil/e2e/**` and `internal/testutil/acpmock/**` — harness env plumbing
- `web/e2e/**`, `web/e2e/fixtures/hosted-mcp.ts`, `internal/mcp/*_test.go`, `internal/daemon/*mcp*test.go`, and ACP/session tests — env/MCP names and agent-callable fixture contracts
- `web/CLAUDE.md`, root `CLAUDE.md` — env lines in governance docs; full prose sweep is task 05, but any line naming a renamed env MUST move in this change (co-ship rule)
- `packages/site/public/install.sh` — hard-cut its `AGH_*` variable names here; task 10 owns only the installer trust/channel behavior and release identity that use the already-renamed variables

### Related ADRs

- [ADR-006: Flat `.compozy/` Namespace and Explicit Config Migration](adrs/adr-006.md) — home resolution and the first-boot legacy-state posture that `COMPOZY_HOME` feeds

## Deliverables

- `COMPOZY_*` environment namespace across production, manifests/templates, tests, and harness code with no dual-read paths
- `compozy__*` native ToolIDs/ToolsetIDs and `compozy_host__*` MCP façade IDs, with updated guards, metadata, catalogs, and server names
- `min_compozy_version` as the sole loader-read extension manifest key
- Dev-skill bodies executable against the renamed identifiers
- Both class grep gates at zero, `make codegen-check` diff-clean, and `make verify` green
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from [`_tests.md`](_tests.md), the test contract — read each ID's full definition there before writing tests.

- [x] UT-057 — home resolution: `DirName = ".compozy"`, `COMPOZY_HOME` override honored, database filename `compozy.db` (update the existing home/config suites; do not duplicate)
- [x] UT-058 — reserved tool-prefix guard rejects `compozy__` collisions from extensions (update the existing guard test)
- [x] IT-016 — post-rename smoke: daemon boots with a tempdir `COMPOZY_HOME`, creates `compozy.db`, listens on the renamed socket, and `compozy status -o json` reports the new paths
- [x] Existing MCP projection and manifest suites — host-MCP advertises only the Compozy façade prefix/server names, and `min_compozy_version` parses and validates with no old-key reader

### Web/Docs Impact

- `web/`: build/runtime env references, native-tool labels, agent access controls, loop palette, hosted-MCP fixtures, and E2E harnesses MUST follow the rename. Checked surfaces: `web/src/lib/vite-api-proxy-target.ts`, `web/src/systems/{session,agent,loops}/**`, and `web/e2e/**`; no route or query-key semantics change.
- `packages/site`: `public/install.sh` receives only the environment-namespace hard cut here; generated/authored reference prose still regenerates or sweeps in tasks 04–05. Installer trust/channel behavior remains task 10.
- QA impact: new scenarios — add content-addressed `untested` files at completion for `COMPOZY_HOME` isolation and native tool invocation under the new IDs. Reset any existing scenario whose `entry_points` cite `AGH_*` env or `agh__*` tool IDs.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: native tool descriptors/toolsets, tool metadata, capability gates, reserved-prefix guard, MCP sidecars, bridge templates, and the loader-read manifest key all carry the new identities. No manifest shape is retained under an old key.
- Agent manageability: `compozy tool list/search/info/invoke` resolves new native IDs, while hosted MCP and `compozy mcp serve` advertise the new façade/server identities. Structured output shapes are unchanged; deterministic unavailable/unknown diagnostics must not name stale identifiers.
- Config lifecycle: no `config.toml` key changes. Environment variables that *override* config keys are renamed; defaults, merge/overlay, and validation behavior are unchanged. Provider env-policy tables move with the rename.

### AGH Impact Audit

- Native tools: ToolIDs, ToolsetIDs, descriptor globs, tool metadata, reserved-prefix guards, generated native-tool catalog, and web contracts move from `agh__*` to `compozy__*`; schema digests are preserved unless a schema changes.
- Extensibility and hooks: bridge process templates, provider env policy, extension manifest `min_compozy_version`, hosted MCP sidecars, and `.agents/skills/**` live bodies move together; checked hooks, skills/capabilities, tools/resources, bundles, registries, bridge SDKs, and manifests.
- Workspace data isolation: environment identity and tool/MCP names do not add data. `COMPOZY_HOME` continues to select the same global/workspace ownership paths after task 01; no workspace_id payload, cache, SSE, or event routing changes.
- Official AGH skill: keep the official directory/name at `skills/agh/` until task 04, but update its live environment, native-tool, and MCP references here. Separately update the development-skill bodies without renaming those dispatch paths; the round-9 rename to `.agents/skills/eng/eng-*` happened afterwards and changed no public contract.

## Success Criteria

- Every assigned test case implemented and passing
- Zero `AGH_` env names, `agh__` ToolIDs/ToolsetIDs, and `agh_host__`/old hosted-MCP names in their executable-source gate scopes
- Regenerated native-tool catalog is current, schema-only digests are unchanged where schemas are unchanged, and `make verify` is green
- A managed session can invoke a renamed native tool and hosted-MCP façade end-to-end with no legacy-identifier fallback in the resolution path
