---
status: pending
title: Source config foundation and policy write path
type: backend
complexity: high
---

# Task 1: Source config foundation and policy write path

## Overview

Delivers the configuration substrate every other task builds on: the daemon-owned preset table, typed root resolution (`SkillRootSpec` + opaque `RootID`), the two config keys with full lifecycle (defaults, validation, tri-state overlay, Live classification, tool_surface, CLI parsing), and the settings write path at both scopes — including the presence-aware workspace override wire shape and lifting the hard workspace-scope rejection on the skills section. After this task, source policy is fully writable and inspectable as configuration; discovery consumers arrive in task_02.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — no compat shims, no fallbacks, no placeholders (greenfield hard cuts)
</critical>

<requirements>
1. MUST implement the preset table in `internal/config` exactly per ADR-002/ADR-003 and `_spec.md` Core Interfaces: `agents` (WorkspaceRel `.agents/skills`, GlobalPath `~/.agents/skills`, WorkspaceNativeReaders `[openclaw, hermes]`, GlobalNativeReaders `[openclaw]`, DefaultOn) and `claude` (both roots, both reader lists `[claude]`), with `compozy` as the implicit builtin (`AlwaysOn`, empty reader lists). Native-reader knowledge is per root, never per preset.
2. MUST implement `SkillRootSpec{Dir, SourceSlug, Kind, OwnerScope, WorkspaceID}` and `RootID()` derived ONLY from (OwnerScope, canonical WorkspaceID, Kind, canonical Dir) — never display slugs or list position (ADR-016). Precedence tier is derived in `internal/skills` later, never stored on the spec.
3. MUST implement `CustomSourceSlugs(paths)` as a batch, order-independent allocator: sanitized lowercase basename; collisions get a deterministic short hash of the canonical path — never positional counters.
4. MUST replace `WorkspaceDiscoveryRoot.SkillsDir()` with `SkillsDirs(cfg) []SkillRootSpec` and add `ResolveGlobalSkillRoots(cfg, home)` — delete the old method in this change (no alias, no dual path). Compozy roots always resolve last intra-tier.
5. MUST add `skills.sources = ["agents"]` and `skills.custom_sources = []` to `SkillsConfig` + defaults, with validation: unknown slug → error naming valid slugs + closest match; duplicate resolved root (cross-key, including preset overlap) → `duplicate_skill_source` naming the owner; workspace-relative custom path at global scope → `invalid_source_path`. Path expansion reuses `expandUserPath`.
6. MUST wire the tri-state overlay: pointer-typed `*[]string` fields in `skillsOverlay` with replace-on-present semantics; workspace overlay carries the same fields; per-key independence in both directions.
7. MUST classify both keys `Live` in the lifecycle rules BEFORE the `skills.*` RestartRequired catch-all (`lifecycle.go:134`), and register both keys in `tool_surface` as agent-writable string slices at global+workspace scope under the same trust gate class as `disabled_skills`.
8. MUST extend CLI `config set/get/unset` for both keys: comma-separated and JSON-array forms parse to the same slice; scope via existing `--scope`/`--workspace`; daemon-routed like other `skills.*` writes; failure output exactly per `_dx.md` (exit 1, `-o json` error body with `valid` + `suggestion`).
9. MUST open the settings skills section to workspace scope for the two source keys ONLY: remove the hard rejection (`section_skills_update.go:25-29`, `sections.go` scope wiring); any other skills field at workspace scope → 400 `workspace_scope_field_forbidden` naming the field; agent scope stays read-only for the source keys while `disabled_skills` keeps its current agent-scope behavior.
10. MUST implement the workspace PATCH wire shape as the presence-aware `{"override": {...}}` object with the `OptionalStringList{Present, Null, Value}` wrapper (custom `UnmarshalJSON`: absent → untouched, `null` → clear override/inherit, array → set), decoded by ONE shared decoder used by both HTTP and UDS. Global scope keeps today's full-config body plus the two new list fields. Plain `*[]string` stays only in the TOML overlay model.
11. MUST co-ship contract changes: additive `config.sources`/`config.custom_sources` on the settings skills payloads, the workspace override request shape, and validation error contracts (`unknown_skill_source`, `duplicate_skill_source`, `invalid_source_path`, `workspace_scope_field_forbidden`) — then `make codegen` + `make codegen-check` green.
12. SHOULD keep every new file under the 500-line cap by splitting up front: preset table/types, resolution, slug allocator, and validation land as separate files in `internal/config`.
</requirements>

## Subtasks

- [ ] 1.1 Preset table + `SkillSourcePreset` type + `SkillSourcePresets()` + `ValidateSkillSources` (did-you-mean suggestion)
- [ ] 1.2 `SkillRootSpec`, `RootKind`, `RootOwnerScope`, `RootID()` (opaque, stable, slug-free)
- [ ] 1.3 `CustomSourceSlugs` batch allocator with deterministic hash suffixing
- [ ] 1.4 `SkillsDirs(cfg)` + `ResolveGlobalSkillRoots` + delete `SkillsDir()` and update its callers' signatures (callers' behavior rewired in task_02)
- [ ] 1.5 `SkillsConfig` fields + defaults + `Validate()` (slug membership, path shape per scope, cross-key duplicate-root rejection)
- [ ] 1.6 Overlay tri-state (`merge.go`, `merge_features.go`, `merge_overlay.go`) + lifecycle `Live` rules + `tool_surface` registration
- [ ] 1.7 CLI `config set/get/unset` support for both keys (parse forms, scope flags, daemon routing, exact error rendering)
- [ ] 1.8 Settings section workspace scope: lift rejection, scope-specific request shapes, `OptionalStringList` + single shared decoder, validation error codes
- [ ] 1.9 Contract payload additions + `make codegen`; OpenAPI spec ops updated for the PATCH shapes
- [ ] 1.10 Full assigned test suite (20 UT + IT-011) in the owning canonical suites

## Implementation Details

Follow `_spec.md` Part II — Core Interfaces (`internal/config/skill_sources.go` block), Data Models (config keys table), API Endpoints (scope-specific PATCH shapes), and Config Lifecycle. The `disabled_skills` key is the end-to-end template to mirror for key plumbing. Prior art for presence-aware lists: `internal/api/contract/workspace_payloads.go:17` (`AddDirs *[]string`) — note it does NOT distinguish null from absent; the new `OptionalStringList` wrapper is required for the tri-state.

### Relevant Files

- `internal/config/agent.go:155-211` — `WorkspaceDiscoveryRoots` + `SkillsDir()` (replacement seam; `AgentsDir()`/`LoopsDir()` siblings stay)
- `internal/config/home.go:18-20,169-250,317-355` — dir-name constants, home resolution, `expandUserPath` (reuse)
- `internal/config/config_extensions_sandbox.go:9-17` — `SkillsConfig` field site; `internal/config/defaults.go:75-78` — defaults
- `internal/config/merge.go:346-353` + `merge_features.go:12-29` + `merge_overlay.go:26,69` — overlay template
- `internal/config/lifecycle/lifecycle.go:82,134,180-182` — `skills.disabled_skills` Live rule + `skills.*` RestartRequired catch-all (new rules go before it)
- `internal/config/tool_surface.go:154-156,299` + `tool_surface_security.go:102` — key registration + trust gate
- `internal/config/config_marketplace_validation.go:14-103` — `SkillsConfig.Validate()` entry
- `internal/cli/config_value_parse.go:173-192` + `internal/cli/config.go:271-274` — list parsing + set-kind table
- `internal/cli/config_value_commands.go:35-77,205-213` — `--scope`/`--workspace` flags on set/unset
- `internal/cli/config_daemon_mutation.go:72-86` — daemon-routed `skills.*` writes
- `internal/settings/section_skills_update.go:16-178` — section update; `:25-29` is the workspace hard-reject to lift; `:123-130` agent-scope allowance for `disabled_skills`
- `internal/settings/sections.go:76,117,122,175` — section scope wiring
- `internal/settings/section_diff.go:28-59` — dotted-name diffs (`skills.sources`, `skills.custom_sources` join here)
- `internal/settings/section_feature_apply.go:15-44` — overlay apply path
- `internal/api/contract/settings_config_payloads.go:295-302` — `SettingsSkillsConfigPayload`
- `internal/api/core/settings_section_requests.go:205-225` + `settings_feature_config.go:24` + `conversions_settings_runtime.go:207-215` — request decode/convert seams (shared decoder home)
- `internal/api/spec/registry_settings_features.go:251,273` — GET/PATCH `/api/settings/skills` op specs

### Dependent Files

- `internal/daemon/boot_finalize.go:79-93,142` — `RegistryConfig` wiring consumes the new resolution (task_02)
- `internal/workspace/scanner.go:162` — `SkillsDir()` caller (rewired in task_02)
- `internal/cli/skill_workspace.go:33-67` — CLI-local `RegistryConfig` mirror (task_02)
- `internal/skills/types.go:227-233` — `RegistryConfig` gains `GlobalSkillRoots` (task_02)
- `web/src/generated/compozy-openapi.d.ts` — regenerates from contract additions

### Related ADRs

- [ADR-001](adrs/adr-001.md) — two separate keys · [ADR-002](adrs/adr-002.md) — v1 preset curation · [ADR-003](adrs/adr-003.md) — implicit always-on compozy · [ADR-004](adrs/adr-004.md) — custom_sources in v1 · [ADR-006](adrs/adr-006.md) — workspace override with full management · [ADR-007](adrs/adr-007.md) — tri-state replace merge · [ADR-016](adrs/adr-016.md) — opaque stable RootID

### Web/Docs Impact

- `web/`: no component changes in this task (S1-S3 land in task_06). `web/src/generated/compozy-openapi.d.ts` regenerates here via `make codegen`; `web/src/systems/settings/hooks/settings-skills-draft-logic.ts` consumes the new payload fields only in task_06.
- `packages/site`: no edits in this task — `configuration/config-toml.mdx`, `file-locations.mdx`, `lifecycle-matrix.mdx` rows land in task_07.
- QA impact: flag (do not walk — loop QA phase walks in task_09) one new content-addressed `untested` scenario in `docs/qa/scenarios/`: `compozy config set/get/unset skills.sources|skills.custom_sources` across scopes — validation errors (`unknown_skill_source` with suggestion, `duplicate_skill_source`, `invalid_source_path`), workspace override set/clear via PATCH, agent-scope read-only rejection.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — the preset table is curated code, closed to extension registration (ADR-014). Checked surfaces: extension manifests, hooks, registries, bridge SDKs, MCP sidecars — no contact in this task.
- Agent manageability: `compozy config set/get/unset` for both keys (daemon-routed, structured errors), `tool_surface` registration makes both keys operable through `compozy__config_*` native tools at global+workspace scope; deterministic error contract per `_dx.md` Errors.
- Config lifecycle: this task IS the lifecycle — keys, defaults, merge/overlay, validation, Live classification, tool_surface, CLI parse forms, contract payloads. Docs/examples land in task_07; tests enumerated below.

## Deliverables

- Preset table, `SkillRootSpec`/`RootID`, `CustomSourceSlugs`, `SkillsDirs`/`ResolveGlobalSkillRoots` in `internal/config` (split files, each under cap)
- Both keys end-to-end: defaults, validation, tri-state overlay, Live lifecycle, tool_surface, CLI set/get/unset
- Settings write path at both scopes with the `OptionalStringList` wire shape and one shared HTTP/UDS decoder; workspace hard-reject removed
- Contract additions + regenerated OpenAPI/TS (`make codegen-check` green)
- `SkillsDir()` deleted — no compat path left
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests. Owning suites: `internal/config` (agent/merge/persistence/lifecycle suites), `internal/settings/service_test.go` + section tests, `internal/api/core` handler suites, `internal/cli` config command suites.

- [ ] UT-001, UT-002, UT-003, UT-004, UT-005, UT-006 — preset table, validation, custom slug allocator
- [ ] UT-007, UT-008, UT-009, UT-010, UT-011, UT-086 — root resolution, ordering, OwnerScope/WorkspaceID, RootID stability
- [ ] UT-012, UT-013, UT-014, UT-015 — overlay per-key independence, Live classification, CLI parse forms, agent-scope rejection
- [ ] UT-057, UT-059, UT-082 — workspace scope acceptance rules, PATCH validation error body, tri-state wire decode (shared decoder, both transports)
- [ ] UT-067 — CLI `config set` failure rendering (exit code + JSON error body)
- [ ] IT-011 — concurrent global + workspace PATCH under the sequential-write discipline

## Success Criteria

- Every assigned test case implemented and passing; `make gate` green (task close)
- `compozy config set skills.sources agents,claude` and the workspace override PATCH behave exactly per `_dx.md` transcripts (messages, exit codes, error bodies)
- `RootID()` provably stable across restarts, list reordering, and display-slug re-allocation (UT-086)
- Grep proves `SkillsDir(` has zero remaining references
