---
status: pending
title: Discovery engine and live apply with sources read model
type: backend
complexity: critical
---

# Task 2: Discovery engine and live apply with sources read model

## Overview

Rewires the three discovery paths (workspace resolver, registry global load, registry workspace load) onto resolved root lists, gives the scanner first-level symlink following with per-projection containment and realpath dedup, and makes source changes apply live through one generation-fenced apply coordinator — including the highest-risk seam, resource-authority republication. It also ships the measurement surface of discovery: the daemon-computed `sources[]` read model on the settings envelope and the `compozy skill sources` verb, so IT-001/E2E-006 complete inside this slice. This is the hard-cut task: the single-root fields die here.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — no compat shims, no fallbacks, no placeholders (greenfield hard cuts)
</critical>

<requirements>
1. MUST make all three scan paths consume `[]SkillRootSpec` from task_01's resolution: registry global load (`GlobalSkillRoots` replaces `UserSkillsDir`), registry workspace load (resolved specs replace hardcoded joins), and the workspace resolver loop. Tier derivation via `sourceTierFor` (workspace preset/builtin → `SourceWorkspace`, global preset/builtin → `SourceUser`, custom → `SourceAdditional`); compozy loads last intra-tier so existing overlay semantics keep compozy winning ties.
2. MUST execute the delete targets in this change: `RegistryConfig.UserSkillsDir` (readers: `registry_load.go:25`, `registry_agent.go:387-395`, `watcher.go:60`, `internal/cli/skill_workspace.go:43`, `internal/daemon/boot_finalize.go:79-93`), `WorkspaceDiscoveryRoot.SkillsDir()` callers (`internal/workspace/scanner.go:162`, `boot_finalize.go:142`), hardcoded joins in `registry_workspace_cache.go:271-293`, and the `globalAgentsDir()` `filepath.Dir` derivation (explicit `GlobalAgentsDir` config field instead).
3. MUST keep daemon and CLI in agreement: the CLI-local daemonless `RegistryConfig` construction (`internal/cli/skill_workspace.go:33-67`) resolves roots through the same `internal/config` functions — no second resolution algorithm.
4. MUST implement scanner symlink semantics per ADR-012 and Safety Invariant 1: first-level links followed only when the resolved target lies inside the projection's trusted set (owning workspace's configured roots + visible global roots, never another workspace's roots); escaping/dangling/cyclic links skipped with per-entry diagnostics (`SkippedLink{Path, Reason}`); realpath identity dedups one physical skill to exactly one catalog entry attributed to the highest-precedence root; second-level links stay unfollowed; containment canonicalizes the source root (macOS `/private/var`).
5. MUST report per-root scan stats: `scanned_count` (SKILL.md candidates pre-dedup/verification) vs `skill_count` (winning contributions), `truncated` (caps stay 300 candidates / 20k entries PER ROOT — do not add aggregate caps), `readable: false` for permission-denied roots (counts omitted entirely, sibling roots unaffected), skipped links, collisions with `winner_root_id` + `qualified_form`, and `verification{blocked, warned}`.
6. MUST keep every governance rule uniform on absorbed skills: `VerifyContent` on every non-bundled load, shadow/overlay audit unchanged, enable/disable, budgets; `origin` (source slug, empty for compozy) attributed on every registry entry; frontmatter allowlist extended per `_spec.md` Data Models (ecosystem fields accepted without warnings, never honored; truly unknown fields still warn).
7. MUST implement the live-apply coordinator (Safety Invariant 9): one owner of the monotonic config generation; watcher roots provider re-derives from current config (fix the `watcherSnapshotRoot` basename branch for custom roots ending in `agents`); every asynchronous commit point (registry swap, resource publication, diagnostics replacement, session-command revision broadcast) re-compares the generation immediately before committing and discards stale work; workspace cache keys incorporate workspace id + generation (Safety Invariant 12); `workspaceSkillPathsMatchRoots` accepts resolved-spec paths and rejects stale ones.
8. MUST prove resource-authority republication survives source changes (spec risk #1): `DiscoverGlobal`/`DiscoverWorkspace` → `ApplyResourceRecords` carries configured roots; enabling/disabling a source adds/removes its skills from the resource-backed projection (IT-003). Extension-kit skills (`internal/extension/agentplugin/skills.go:24` `<root>/skills` convention) are explicitly NOT preset sources — they keep flowing through the resource store unchanged.
9. MUST emit the canonical observe events for this slice with base correlation keys `{workspace_id?, config_generation, actor_kind, actor_id}`: `skills.sources.applied` (successful commit only), `skills.sources.superseded` (stale discard — never recorded as applied), `skills.sources.apply_failed`, `skills.scan.truncated`, `skills.scan.link_skipped`; durable append precedes any live revision/SSE broadcast derived from the same change. Follow the existing `internal/skills/observe_events.go` shape; pick the `skills.` prefix deliberately (existing names mix `skill.`/`skills.`).
10. MUST build the sources read model: settings envelope gains daemon-computed `sources[]{slug, label, kind, enabled, always_on, default, workspace_path?, global_path?, path?, roots[]}` with the full per-root diagnostic schema on every root regardless of kind, plus `inherits{sources, custom_sources}` at workspace scope; runtime unavailable → counts omitted, never zeroed.
11. MUST ship `compozy skill sources` (human table incl. `always on` row, truncation notes, scope footer; `-o json` exactly per `_dx.md`; `--workspace` view with `workspace_id` + `inherits`, both omitted at global scope) reading the settings envelope — no new read route.
12. MUST co-ship contract additions for the envelope read model (`make codegen` + `codegen-check` green) and keep prompt-catalog behavior byte-stable for a default config with no ecosystem folders present (no regression for existing users).
</requirements>

## Subtasks

- [ ] 2.1 `RegistryConfig{GlobalSkillRoots, GlobalAgentsDir}` + global load fan-out over N roots; delete `UserSkillsDir` + `globalAgentsDir()` derivation
- [ ] 2.2 Workspace load + resolver loop on resolved specs; delete hardcoded joins + `SkillsDir()` callers; CLI-local mirror through the same resolution
- [ ] 2.3 Scanner: first-level follow, per-projection containment (trusted-set parameter), realpath dedup, skip diagnostics, per-root caps + `readable:false`
- [ ] 2.4 Registry: `sourceTierFor`, `origin` attribution, per-root contribution stats, generation-keyed workspace cache + `workspaceSkillPathsMatchRoots` update
- [ ] 2.5 Frontmatter allowlist extension (accept-without-warning list; unknown fields still warn)
- [ ] 2.6 Watcher roots provider re-derivation + `watcherSnapshotRoot` basename-branch fix + daemon watcher root wiring
- [ ] 2.7 Apply coordinator: monotonic generation, commit fence at all four commit points, workspace-cache invalidation, republication trigger
- [ ] 2.8 Resource-authority republication with configured roots (IT-003) + explicit agentplugin non-impact
- [ ] 2.9 Observe events (applied/superseded/apply_failed/truncated/link_skipped) + durable-before-broadcast ordering
- [ ] 2.10 Settings envelope `sources[]` + `inherits` read model + contract additions + codegen
- [ ] 2.11 `compozy skill sources` verb (human/JSON/`--workspace`)
- [ ] 2.12 Full assigned test suite (29 UT + 6 IT + E2E-006) in the owning canonical suites

## Implementation Details

Follow `_spec.md` Part II — System Architecture (Scanner/Registry/Live apply rows), Core Interfaces (`RegistryConfig`, `RootScanStats`), Data Models (per-root diagnostic schema, count semantics), Monitoring (event table), Safety Invariants 1, 2, 8, 9, 11, 12. The registry's existing `GlobalVersion() int64` atomic counter (`internal/skills/registry.go:135-232`) is the natural seed for the generation fence — extend, don't duplicate.

### Relevant Files

- `internal/skills/types.go:96-111,227-233` — tier enum (iota = precedence; presets map into existing tiers, no new slots) + `RegistryConfig`
- `internal/skills/registry_load.go:14-38,99-123,157-218` — global load loop, per-root scan seam, verification pipeline, marketplace provenance flip
- `internal/skills/registry_workspace_cache.go:225-293,295-339,382-416` — workspace roots, overlay order, consistency guard, cache keys (TTL at `registry.go:30`)
- `internal/skills/registry_agent.go:101-121,264-321,387-395` — agent-local overlay (unchanged semantics) + `globalAgentsDir` delete target
- `internal/skills/registry_override_log.go:52-62` + `shadows.go:8-40` + `registry_snapshot.go:51-96` — overlay/shadow machinery absorbed unchanged
- `internal/skills/registry.go:135-232` — `LoadAll`/`RefreshGlobal`/`GlobalVersion` (generation hook)
- `internal/skillscan/scan.go:14-53` + `scan_directory.go:18-33,57-63,86-129` + `scan_fs.go:75-108` — walker, caps, dot-dir skip, symlink posture
- `internal/skills/path_security.go:10-23` — `ensurePathWithinRoot` (containment primitive to generalize per-projection)
- `internal/skills/loader.go:150-168` + `loader_values.go:178` — frontmatter decode + unknown-field warning site
- `internal/skills/watcher.go:26,55-100,138-234,235-273,313-359` — polling, `SetRootsProvider`, snapshot branching on `agents` basename
- `internal/daemon/boot_finalize.go:79-148` — `skillsRegistryConfig` + `startSkillsWatcher` + `workspaceSkillWatcherRoots`
- `internal/cli/skill_workspace.go:33-67` — CLI-local `RegistryConfig` (mirror seam)
- `internal/skills/registry_discovery.go:76-137,155-214` + `internal/daemon/agent_skill_publisher.go:138-230` — resource-authority handoff (risk #1)
- `internal/skills/observe_events.go` + `internal/events/names.go:82-83` + `internal/skills/registry.go:87-133` — event content shapes, names, summary-store injection
- `internal/skills/diagnostics.go:15-165` + `registry_diagnostics.go:13-90` — per-root diagnostics extension point
- `internal/settings/section_build.go:39-90` + `models_runtime.go:39-47` + `internal/api/core/conversions_settings_sections.go:82-93,461-464` — envelope build seams
- `internal/cli/skill_commands.go:21-43` + `skill.go:12-98` + `skill_output.go` — verb registration + DTO/rendering patterns
- `internal/extension/agentplugin/skills.go:24-60` — parallel `<root>/skills` convention (explicit non-impact)

### Dependent Files

- `internal/skills/catalog.go` + `internal/daemon/prompt_skills.go` — consume the filtered registry later (task_05 adds the filter; this task must not pre-filter)
- `internal/skills/registry_command.go` — command candidates read origins from this task's attribution (task_05)
- `internal/api/contract/*settings*` + `web/src/generated/compozy-openapi.d.ts` — envelope contract additions regenerate
- `internal/skills/expose.go` — task_03 resolves enabled preset roots through this task's resolution
- `internal/tools/builtin/skills.go` — origin surfaces on native tools in task_04 from registry attribution

### Related ADRs

- [ADR-003](adrs/adr-003.md) — implicit compozy contributes existing roots exactly · [ADR-005](adrs/adr-005.md) — live apply · [ADR-007](adrs/adr-007.md) — six tiers with root lists · [ADR-008](adrs/adr-008.md) — frontmatter recognized, not honored · [ADR-012](adrs/adr-012.md) — first-level symlink following + realpath dedup · [ADR-013](adrs/adr-013.md) — origin in read models · [ADR-016](adrs/adr-016.md) — RootID as cache/identity key

### Web/Docs Impact

- `web/`: no component changes in this task; `web/src/generated/compozy-openapi.d.ts` regenerates (envelope `sources[]`/`inherits`). S1 binds to this read model in task_06 (`web/src/systems/settings/…`).
- `packages/site`: no edits in this task — discovery-order and diagnostics docs land in task_07 (`content/docs/skills/index.mdx`, `configuration/config-toml.mdx`).
- QA impact: flag (walk deferred to task_09) two new content-addressed `untested` scenarios: (a) live source toggling — enable/disable preset and custom roots, skills appear/leave every surface within two poll intervals, truncation flag set/clear; (b) `compozy skill sources` diagnostics — absent dir, unreadable root, truncated root, collision with qualified form, `--workspace` inherits view.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: resource-authority republication is the one extension-adjacent seam — `DiscoverGlobal`/`DiscoverWorkspace` → `ApplyResourceRecords` must carry configured roots (IT-003). Extension-published skills and the agent-plugin `<root>/skills` lane are unaffected (checked: `internal/extension/agentplugin/skills.go`, resource store handoff). Extensions cannot register presets (ADR-014).
- Agent manageability: `compozy skill sources` human + `-o json` + `--workspace` (reads the settings envelope — one read model); per-root `readable`/`truncated`/skips/collisions make "skill missing" diagnosable from structured output alone.
- Config lifecycle: consumes task_01 keys; no new keys. Live-apply behavior (`applied live` semantics) becomes real here — visibility budget: every read surface within two poll intervals (owned by IT-001).

## Deliverables

- Three scan paths on resolved root lists; single-root fields and methods deleted (grep-clean)
- Scanner symlink matrix (follow/containment/dedup/skip/caps/readable) with per-root stats
- Live apply with generation fence at all four commit points; republication proven by IT-003
- Canonical observe events for sources/scan lifecycle
- Settings envelope `sources[]` + `inherits` read model; `compozy skill sources` verb per `_dx.md`
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there. Owning suites: `internal/skillscan/scan_test.go`, `internal/skills/registry_test.go` (+ `loader_test.go`, `watcher_test.go`, `registry_diagnostics_test.go`, `registry_concurrency_test.go`, `registry_integration_test.go`), `internal/settings/service_test.go`, `internal/cli/skill_test.go`, `internal/daemon/agent_skill_resources_integration_test.go`.

- [ ] UT-016, UT-017, UT-018, UT-019, UT-020, UT-021, UT-022, UT-023, UT-074, UT-083, UT-097 — scanner symlink matrix, caps, containment, unreadable root
- [ ] UT-024, UT-025, UT-026, UT-027, UT-028, UT-029, UT-030, UT-031, UT-032, UT-075 — registry tiers/origin/shadows/verification/frontmatter/caches
- [ ] UT-033, UT-034 — watcher roots provider + snapshot basename branch
- [ ] UT-056, UT-058, UT-084 — envelope `sources[]`, `inherits`, per-root diagnostic fields (readable/counts-omitted semantics)
- [ ] UT-063, UT-064, UT-089 — `skill sources` human golden, JSON schema, `--workspace` view
- [ ] IT-001 — live apply end-to-end (PATCH → watcher → registry → envelope counts; truncation clears)
- [ ] IT-002 — two-workspace override isolation + inherits reporting
- [ ] IT-003 — resource republication carries configured roots (spec risk #1)
- [ ] IT-006 — vercel-labs layout (canonical + link, both presets) → one entry, correct origin
- [ ] IT-007 — override change → generation-keyed cache invalidation, other workspace untouched
- [ ] IT-012 — deterministic live-apply race: generation N never overwrites N+1 on any read surface
- [ ] E2E-006 — managing-agent journey (sources → replace truncated custom root → confirm), `_dx.md` transcripts verbatim

## Success Criteria

- Every assigned test case implemented and passing; `make gate` green (task close)
- Default config with no ecosystem folders produces a byte-identical prompt catalog to today (no regression)
- `rg 'UserSkillsDir|SkillsDir\('` returns zero production references
- A source toggle is visible in registry, resource projection, envelope counts, and watcher roots without restart — and never mixes two generations
