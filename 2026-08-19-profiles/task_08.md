---
status: pending
title: Resource, Config, and Credential Layers
type: backend
complexity: high
---

# Task 8: Resource, Config, and Credential Layers

## Overview

Delivers the layers capability end to end: the four additive resource layers with most-specific-wins and inspectable shadowing, the profile config layers with context-owned write targets + denylist + "saved but not applied" feedback, the `general` settings split (persona defaults), the vault `profiles` namespace with owner-prefix containment and the per-profile credential journey, provider prestart + bridge-SDK cache keys, MCP sidecar profile layers, and the memory read-path isolation rows. Includes the grant-ceiling **lattice** (single owner, scalar rank deleted) pinned by tests, the Settings/shadow-inspect UI touches, and this capability's docs/skill content.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

**Skills**: `eng-code-guidelines`, `golang-master`, `eng-test-conventions`, `testing-boss`, `eng-cleanup-failure-paths`, `documentation-writer`; `eng-design`+`ui-craft` for the Settings touches.

<requirements>
- MUST insert the two discovery roots so the effective order is project-profile → project base → profile → user (+ existing additional/bundled slots), with the precedence rank decoupled from enum iota (appending never renumbers persisted comparisons).
- MUST implement name-binding for the repo per-name layer (`workspace_profile`, scope_id `<ws>@pf:<name>`): binds only while the active profile's name matches; absent name → dormant with the create-hint payload; rename/create to the name wakes it; personal roots inside a registered workspace are rejected.
- MUST keep shadowing inspectable (`agent list` LAYER/SHADOWS columns) via the existing shadow-evidence mechanism.
- MUST make the ceiling lattice ONE owner (`meetScopeCeilings` + closure table) consumed by both capability narrowing and the surface-registry expansion; the scalar `scopeRank` helper and the duplicate `scopesThrough` are delete targets (12); closures + pairwise meets pinned in the same change (D24); `ExtensionsConfig.Resources.MaxScope` accepts the new values.
- MUST add the config layers at the only overlay site (`loadWithHome` + `WithProfile` option), with write targets context-owned (user file under `default`, profile file otherwise), `--scope user|profile|workspace` override, the denylist rejected fail-closed with allowed-prefix guidance, `OkOverridden` "saved but not applied — <layer> wins" feedback, and orphan profile-config diagnostics; repo profile layer read-only to the CLI.
- MUST split persona defaults (`defaults.agent|provider|sandbox`) out of `general` into a profile-layerable section (D15); the six hard `ScopeGlobal` gates at `internal/settings/collections.go:102-175` are delete target 6 (sandboxes stay user-level; hooks gain the profile layer).
- MUST implement the vault capability: `profiles` namespace in pattern + map, grammar `vault:profiles/<name>/providers/<provider>/<slot>` and `…/extensions/<ext>/<key>`, owner-prefix containment, `env:` refusal for profile scope (`profile_secret_env_forbidden`), rename rewrite-list enumeration, and the `_dx.md` secret journey (`secret set/rm` under a profile, `provider inspect` source lines, removal warning with owned work).
- MUST add `PreStartScope.ProfileID` to the provider prestart cache key and profile to `bridgesdk.InstanceCache`'s key (no cross-profile credential reuse).
- MUST add the MCP sidecar layers (`profiles/<p>/mcp.json` both sides) merged in their owning config layer's slot, same write-target rule, orphan diagnostics; auth/registration scope coupling already widened in `00079`.
- MUST enforce the memory read-path rows: recall/FTS/catalog/cache identity includes the profile; aggregate memory reads refused with a typed error; native-tool + situation memory projections scoped.
- MUST extend the IT-038 fixture with config-layer file, MCP sidecar entries, and credential-override rows, and the IT-077 fixture with the resource-discovery and provider-prestart rejection arms (cumulative fixture contract in `_tasks.md`).
- MUST extract instead of appending: `internal/providers/prestart_cache.go` (488) and `internal/settings/collections.go` (406) are near the cap.
- MUST ship the capability's docs (layer precedence, file locations, denylist, credentials) and official-skill updates in the same task.
</requirements>

## Subtasks

- [ ] 8.1 Discovery roots + rank function + per-name binding/dormancy + hint payloads + shadow inspection.
- [ ] 8.2 Resource scope widening in Go (`user|workspace|profile|workspace_profile` + scope_id validation) and the lattice owner + pins; delete `scopeRank` + duplicate `scopesThrough`.
- [ ] 8.3 Config: `WithProfile` overlay, write-target arms, denylist validator, `OkOverridden` feedback, orphan diagnostics, `--scope` on config verbs, apply-records naming layers.
- [ ] 8.4 Settings: `general` split (persona defaults section), scope-gate deletions, settings-UI write path hitting the same targets.
- [ ] 8.5 Vault namespace + containment + env-refusal + rewrite-list enumeration; secret CLI journey + `provider inspect` sources.
- [ ] 8.6 Prestart cache + bridge-SDK instance cache keys.
- [ ] 8.7 MCP sidecar layers (merge slot, write target, orphan diagnostic, rename move with the profile folder).
- [ ] 8.8 Memory read paths: recall/FTS/cache identity, aggregate refusal, projection scoping.
- [ ] 8.9 IT-038 + IT-077 fixture extensions; Settings shadow-inspect UI touches (LAYER/SHADOWS surfacing).
- [ ] 8.10 Docs (`configuration/index.mdx` precedence table, `file-locations.mdx`, credentials page) + skill updates + QA scenario flags.

## Implementation Details

Three existing seams — discovery roots, the overlay chain, the vault ref grammar — never new parallel mechanisms. The lattice semantics (closures, meets, forbidden scalar minimums) are specified in `_spec.md` Data Models (`resource_records` bullet).

### Relevant Files

- `internal/config/agent.go:154-211` — `WorkspaceDiscoveryRoots` + per-root dir mapping (the one slice to extend).
- `internal/skills/types.go:95-111` + `registry_source.go:18-65` + `internal/skills/shadows.go:9-40` — precedence enum hazard + tier names + shadow evidence.
- `internal/workspace/scanner.go:281-315` — `mergeSkillPaths` most-specific-wins rule.
- `internal/resources/types.go:61-120` + `projector.go:16-116` — scope enum + generic record seam.
- `internal/extension/capability_resource_policy.go:115-136` — `scopeRank`/`scopesThrough` delete sites; `surfaces/registry.go:42-100` — `LegalScopes` + the duplicate closure to delete.
- `internal/config/config_load.go:39-52,137-174` + `config_paths.go:36-61` — overlay site + path builders.
- `internal/config/persistence.go:21-53,87-148` + `write_scope_policy.go:9-25` + `workspace_overlay.go:41-55` — write scopes, target mapper, denylist validators.
- `internal/settings/models.go:15-104` + `sections_daemon.go:10-93` + `collections.go:26-177` + `collection_provider_write.go:311-325` — scope kinds, `general` split, gates to cut, secret owner prefix.
- `internal/cli/config_value_commands.go:14-48` + `config_mutation_scope.go:9-14` — `--scope` per subcommand + the CLI choke point.
- `internal/vault/types.go:29-48` + `mcp_refs.go:11-125` + `extension_refs.go:6-19` — pattern/map + containment templates.
- `internal/providers/probe_env.go:81-88` + `prestart_cache.go:187-205` + `internal/providerenv/env.go:227-245` — prestart scope, cache key, segment grammar.
- `internal/memory/` recall/FTS/cache query paths (durable identity landed in task_02).
- `internal/config/home_extension_data.go:15-54` — `@pf-<id>` segment template.

### Dependent Files

- `internal/cli/` config/secret/provider command files — journeys + feedback copy.
- `web/src/systems/settings/` — settings write path + shadow-inspect touches; generated types regen.
- `skills/compozy/references/configuration.md` — layer/denylist/credentials updates.

### Competitor References

- `.resources/codex/` profile-v2 sources per `analysis/09` — sibling-config write-target + `OkOverridden` feedback + name newtype + denylist footgun (D16 lineage).

### Related ADRs

- [ADR-006](adrs/adr-006.md) — four additive layers, name-binding, dormancy.
- [ADR-009](adrs/adr-009.md) — user-default credentials + per-provider vault overrides.
- [ADR-013](adrs/adr-013.md) — `user` scope vocabulary in every surface this task touches.
- [ADR-012](adrs/adr-012.md) — names in vault refs; ids in extension-data segments.

## Deliverables

- Four-layer discovery with dormancy/hints/shadowing; lattice single-owner with pins; scalar rank deleted.
- Config layers + write targets + denylist + feedback + `general` split, CLI and settings-UI paths converged.
- Vault namespace + containment + secret journey; caches keyed by profile; MCP sidecar layers.
- Memory read-path isolation enforced.
- Docs + skill content for layers/config/credentials.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-025..UT-032 — overlay merge, write targets, context-owned default, persona defaults, denylist, overridden feedback, orphan diagnostics, layered hooks.
- [ ] UT-035..UT-042 — vault parse/containment/env-refusal/resolution-source/rewrite-list; prestart + instance cache keys; segment identities.
- [ ] UT-043..UT-047 — root order, empty-layer composition, shadow evidence, name-binding/dormancy/wake, personal-path-in-repo rejection.
- [ ] UT-049..UT-052 — scope enum + scope_id rules; closure pins; pairwise meet pins; rank decoupled from iota.
- [ ] UT-070 — recall query scoped by profile tier.
- [ ] UT-092 — MCP sidecar merge/write/orphan.
- [ ] IT-021 — usage attribution with mixed credentials (extends task_06's usage fixtures).
- [ ] IT-040, IT-041, IT-042 — layer composition; repo-layer dormancy/refresh/validity; team adoption binding + hint lifecycle.
- [ ] IT-043..IT-048 — write targeting end-to-end; concurrent layer writes; profile-layer hooks + sandbox refusal; single apply timeline; provider override in real run path; `secret rm` warning + fallback.
- [ ] IT-058, IT-059, IT-074 — memory tier isolation; consolidation boundary; FTS/cache/aggregate-refusal + projection scoping.
- [ ] E2E-006, E2E-007 — config trio; credentials journey.

### Web/Docs Impact

- `web/`: `web/src/systems/settings/` (write path + persona-defaults section + shadow inspection), generated types regen; no new system module.
- `packages/site`: `content/docs/configuration/index.mdx` (precedence table), `content/docs/configuration/file-locations.mdx` (profile dirs), profiles docs credentials page, `content/docs/configuration/mcp-json.mdx` (sidecar layers); generated CLI docs regen (`config`, `secret`, `provider`).
- QA impact: new scenarios — add content-addressed untested files for layered config write/feedback/denylist, per-profile secret set/rm, and repo-layer adoption dormancy; reset existing config-write scenarios (write target changed). Walk owned by task_13.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: lattice + `LegalScopes` widening + `MaxScope` values are the extension capability-grant surface (D24 pins); hooks gain the profile layer; MCP sidecar layers; bridge SDK cache key. Manifest placement/enablement stays task_09.
- Agent manageability: `compozy config set --scope user|profile|workspace` with overridden-write feedback, `compozy secret`/`provider inspect` resolution visibility, `agent list` shadow columns — all structured, all deterministic errors.
- Config lifecycle: no new keys; two new **layers** enter the documented precedence with validation, examples, docs, and merge/write/feedback tests (`config_apply_records` entries name the layer, D26).
