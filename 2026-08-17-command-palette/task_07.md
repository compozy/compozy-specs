---
status: completed
title: "P6 — Extension Contribution: Manifest Family, Projection, Declarative Tier"
type: backend
complexity: high
---

# Task 7: P6 — Extension Contribution: Manifest Family, Projection, Declarative Tier

## Overview

Opens the palette to extensions: the `resources.cmd_palette` manifest family (commands + views in both tiers' declared shapes) with build/validate-time enforcement, the storage-free health-gated projection `Manager.CmdPalette(workspaceID)`, the extension-default shortcut tier (bind-only-if-free, dormant + visible), dev-mode hot reload, and the per-extension Palette panel in settings (S16). Ships the Go fixture extension (commands + default chord + declarative view) that closes the all-Go integration blind spot, and completes the P4 journeys (E2E-016/E2E-025) whose transcripts require this fixture.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting; read every ADR in `adrs/` and any `memory/task_NN.md` present
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — fix root causes; frozen `_dx.md`/Part II contracts are binding; surface gaps as clarifications, never invent
</critical>

<requirements>
1. MUST add the `cmd_palette` family to the manifest exactly per `_dx.md` (`CmdPaletteConfig` owned by `internal/extension`): commands (id/title/section/icon/keywords/arguments/action union without `client_op`/destructive+confirmation/default_shortcut/optional `execution` policy) and views (id/title/kind + `source:{tool}` | `program:true`); ids namespace to `ext.<name>.<id>`; validation fails with field-naming actionable messages (duplicate ids, unknown tool refs, icon token rules, length/charset caps, chord grammar); Tier-1 `source` tools MUST be read-only risk class (validate-time rejection, SI-18).
2. MUST implement `Manager.CmdPalette(workspaceID)` as a storage-free, enable-scoped, health-gated, dev-overlay-aware projection (the `Manager.Commands` pattern) — NOT a ResourceKind (Surface table untouched; checked `surfaces/registry.go`, `extensions.resources.allowed_kinds`); membership by enablement, availability by health: an unhealthy instance keeps last-known validated descriptors `available:false` with source status + reason; every membership/health transition produces one atomic catalog revision (SI-5).
3. MUST wire the projection into the `cmdpalette.Provider` seam at the composition root (no import cycle: extension owns manifest types; cmdpalette consumes via the interface).
4. MUST implement the extension-default shortcut tier in the keymap: binds only when entirely free (no core default, no user override, no earlier extension claim; enable order breaks ties deterministically); conflicts stay dormant + visible (`conflict_with` owner); disable deactivates ext bindings while user overrides persist dormant and reactivate (BR-5, US-029).
5. MUST make dev mode first-class: dev-linked instances shadow published per workspace; manifest edits hot-reload the projection within the watch interval emitting `cmd_palette.catalog.changed`; broken edits keep last-good + dev diagnostics (US-031).
6. MUST ship the S16 settings panel: per-extension Palette panel listing contributed commands (effective bindings + dormant defaults) and views; disabled/unhealthy contributions grayed with the health reason.
7. MUST ship the Go fixture extension with a `cmd_palette` block (commands incl. destructive `purge`, default chord, Tier-1 declarative view) under `internal/extension/testdata/` for the integration suites, and program the `ext.notes` shapes the `_dx.md` transcripts use.
8. MUST complete the deferred P4 journeys on the fixture: E2E-016 (record/conflict/overwrite/alias/cheatsheet) and E2E-025 (CLI personalization + bind/alias parity transcripts).
9. MUST co-ship: `make codegen` (manifest contract types propagate to `sdk/typescript` `contracts.ts` + `sdk/go/contracts` automatically), docs (`packages/site/content/docs/extensions/cmd-palette.mdx` NEW + `manifest.mdx` update + `meta.json` navigation), `skills/compozy/` (extension palette family), transport-parity rows for any new route, and the settings payload extension (`extension_defaults` now populated).
10. MUST block and surface if a cited artboard is missing at execution time.
</requirements>

## Visual Contract

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | -------- | ---------------------------------- |
| VC-01 | `docs/design/opendesign/command-palette/command-palette-settings-palette.html` — S16 populated panel (commands + bindings + views) | Extensions settings detail with the fixture enabled | 1440×900 | normative | None |
| VC-02 | `command-palette-settings-palette.html` — dormant-default row ("default unavailable — conflicts with X") | fixture default chord colliding with a core binding | 1440×900 | normative | None |
| VC-03 | `command-palette-settings-palette.html` — unhealthy-extension gray state with health reason | crash-looping fixture instance | 1440×900 | normative | None |

Evidence for each row: `.compozy/tasks/command-palette/evidence/visual/task_07/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 7.1 Manifest family: `CmdPaletteConfig` types + key constants + normalize + encode + validate (field-naming errors; read-only source-tool enforcement; chord grammar; contract types for codegen propagation)
- [x] 7.2 `Manager.CmdPalette(workspaceID)` projection (enable-scoped, health-gated, dev-overlay) + `cmdpalette.Provider` wiring at the composition root
- [x] 7.3 Membership-vs-health semantics through the catalog: last-known descriptors on unhealthy, atomic revision per transition, source statuses in `sources`
- [x] 7.4 Extension-default shortcut tier in the keymap (bind-only-if-free, deterministic tie-break, dormant visibility, disable/re-enable semantics) + settings payload `extension_defaults`
- [x] 7.5 Dev reload path: watch → projection refresh → `catalog.changed`; broken-edit last-good + diagnostics
- [x] 7.6 S16 `ExtensionPalettePanel` in the extensions settings detail
- [x] 7.7 Go fixture extension (`ext.notes` shapes: capture/recent/purge + default chord + Tier-1 view) in `internal/extension/testdata/`
- [ ] 7.8 Deferred P4 journeys: E2E-016 + E2E-025 on the fixture — execution remains in the accepted task_12 QA tail
- [x] 7.9 Docs (`extensions/cmd-palette.mdx` new + `manifest.mdx` + navigation) + `skills/compozy/` + codegen co-ship
- [ ] 7.10 Visual Contract evidence bundles VC-01..03 — capture remains in the accepted task_12 QA tail

## Implementation Details

Reference `_spec.md` Part II: Extensibility Integration Plan (the authoritative checklist), Core Interfaces (`CmdPaletteConfig`), Business Rules 4–5, Safety Invariants 5, 11, 18 (Tier-1 read-only).

### Skills

`golang-master` · `eng-code-guidelines` · `eng-contract-codegen-coship` · `eng-test-conventions` · `testing-boss` · `eng-design` · `ui-craft` · `impeccable` · `eng-ui-screenshot` · `documentation-writer`

### Relevant Files

- `internal/extension/manifest.go` (`ResourcesConfig` L108 — the family lands here) + `manifest_normalize.go` + `manifest_encode.go` + `manifest_validate.go` + `static_resources_validation.go` + `manifest_errors.go` — the manifest pipeline
- `internal/extension/command.go` — `Manager.Commands(workspaceID)` — the exact projection pattern (health gate via `commandExtensionAvailable`)
- `internal/extension/contract/` — contract types that auto-propagate to both SDK `contracts` files via `make codegen`
- `internal/extension/surfaces/registry.go` — explicitly untouched (not a ResourceKind); cite as checked evidence
- `internal/extension/dev.go` + `manager_dev_lifecycle.go` + `manager_dev_admission.go` + `dev_generation.go` — dev overlay + reload machinery
- `internal/extension/build.go` + `build_describe.go` + `extension_validation.go` — build/validate pipeline the new validation joins
- `internal/extension/testdata/command-fixture-go/` — fixture extension layout precedent
- `internal/windowmanager/shortcuts.go` + task_05's `BindableIDs` seam — where the ext tier slots
- `web/src/routes/_app/settings/-extensions-settings-page.tsx` — S16 host
- `extensions/spec-cycle/extension.json` + `embed_test.go` — richest manifest exemplar + embedded-resource suite pattern
- `internal/extension/manager_views.go` — **false friend**: instance snapshot helpers, NOT UI views; put nothing view-related there

### Dependent Files

- `sdk/typescript/src/generated/contracts.ts` + `sdk/go/contracts/*_gen.go` — regenerate (manifest family typings)
- `internal/api/core/settings_window_manager_payload.go` + settings contract — `extension_defaults` population
- `packages/site/content/docs/extensions/{cmd-palette.mdx,manifest.mdx,meta.json}` — new page + updates + navigation registration (nav completeness is test-enforced)
- `skills/compozy/` — extension palette family entry
- `web/src/systems/os/` projection — extension source chips/attribution render from catalog data (no new web seam)

### Competitor References

- (No `.resources/` subset for this task — the manifest/projection design follows in-repo patterns; Vicinae/Raycast references are cited by concept in the spec's Design and Analysis Sources, not vendored.)

### Related ADRs

- [ADR-002](adrs/adr-002.md) — the three contribution surfaces (commands/views/default shortcuts); storage-free live projection, not a ResourceKind
- [ADR-005](adrs/adr-005.md) — `resources.cmd_palette` family naming
- [ADR-009](adrs/adr-009.md) — Tier-1 declarative available to both languages; `program: true` field shape (its program-specific validation completes in task_08)

### Web/Docs Impact

- `web/`: `ExtensionPalettePanel` (S16) + extension-source rendering in the projection; MSW fixtures for contributed catalogs; stories for the three VC states.
- `packages/site`: `content/docs/extensions/cmd-palette.mdx` (new, navigation-registered), `extensions/manifest.mdx` (family), generated contract typings notes; turbo `inputs` check if truth tests read new fixture files.
- QA impact: flag only (walks in task_12): add content-addressed `untested` scenario **extension palette contributions** (enable → contribute → disable/health → dormant defaults → dev reload); existing `ET-discover-extension-command-tree.md` NOT reset (CLI command projection unchanged — checked).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: FULL — new manifest family + validation + projection + default-chord tier + dev reload + docs + SDK contract propagation; hooks catalog untouched (checked `internal/hooks/events_catalog.go`); Surface table untouched; MCP/bridge SDKs unaffected (no protocol change this task).
- Agent manageability: contributions surface through the existing task_01 verbs/routes (`list --source ext.<name>`, invoke, catalog `sources`); `compozy extension build|validate|dev` carry the new validation errors; no new verb needed — checked against `_dx.md`.
- Config lifecycle: none — checked: extension defaults live in the manifest + keymap tier, not `config.toml`; user overrides continue through `[window_manager.shortcuts]` (task_05).

## Deliverables

- `resources.cmd_palette` family end-to-end: manifest → validate → projection → catalog → palette/settings render
- Extension-default shortcut tier with dormant-conflict visibility and override persistence
- Dev hot reload of palette contributions with last-good safety
- Go fixture extension powering the integration suites + the completed E2E-016/E2E-025 journeys
- S16 panel + docs + skills + codegen co-ship
- Visual Contract evidence bundles VC-01..03 **(REQUIRED)**
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-055, UT-056, UT-057, UT-058, UT-059, UT-060, UT-061, UT-062, UT-063 — manifest validation (ids, execution policy round-trip, tool refs, hostile strings, chord grammar), projection membership-vs-health, dev overlay/shadow, broken-reload last-good
- [x] UT-075, UT-076, UT-077 — ext-default tier: bind-when-free, deterministic dormant conflicts, disable/override persistence
- [x] IT-016 — enable/disable atomicity, collision rejection, dormant default with owner
- [ ] IT-017 — real crash-loop/recovery and trust-reason walk is retained by `ET-extension-palette-contributions` for task_12; manager/catalog automated coverage is green
- [x] IT-018 — dev overlay + reload + broken-edit diagnostics
- [x] IT-019 — Tier-1 extension view path: validated fixture envelope plus shared invalid-payload and patch-fence suites
- [ ] IT-023 — real SSE enable/revision-convergence walk is retained by `ET-extension-palette-contributions` for task_12; event registration and notifier coverage are green
- [x] IT-024 — patch-stream replay guards (`after` without epoch → 400; epoch mismatch → resync)
- [ ] E2E-016 — task_12 QA tail
- [ ] E2E-017 — task_12 QA tail
- [ ] E2E-020 — task_12 QA tail
- [ ] E2E-021 — task_12 QA tail
- [ ] E2E-025 — task_12 QA tail

## Checkpoint

Implementation is closed at the task checkpoint. The accepted loop exception keeps the flagged QA
scenario, E2E walks, live SSE/crash-loop observations, and VC-01..03 bundles in task_12; this
checkpoint does not claim real-user or visual parity before that tail runs.

Verification evidence: `make gate` escalated to the full monorepo gate and passed at fingerprint
`6ac860448ca26e07301dd0542d4a434de527b462`; log `.cache/gate/logs/full-1787177604.log`.

Compozy Impact Audit:

- Native tools: no new IDs or schemas — checked the existing `compozy__cmd_palette_list|invoke` descriptors, capability gates, and CLI fallbacks; extension contributions flow through their existing catalog contract.
- Extensibility and hooks: added `resources.cmd_palette`, generated Go/TypeScript SDK contracts, build/validate/install/dev-reload enforcement, storage-free extension projection, declarative read-only views, extension shortcut defaults, docs, and the official Compozy skill. Hook events, MCP sidecars, bridge SDKs, and the ResourceKind registry are unchanged.
- Workspace data isolation: manifest declarations are instance-owned; live commands, views, defaults, source health, catalog revisions, view calls, and dev overlays are projected per workspace. The daemon resolves workspace identity before catalog/view access, and the fixture integration proves enable/view/disable through public surfaces.
- Official Compozy skill: `skills/compozy/SKILL.md` now routes command palette contribution authoring to `references/extension-authoring.md`, which documents the family, validation, shortcuts, health, dev reload, and structured inspection paths.

## Success Criteria

- Every assigned test case implemented and passing
- Membership never changes on health transitions — only on enable/disable/remove — and every transition is one atomic revision (IT-016/IT-017 pin SI-5)
- An extension can never steal an operator key: conflicts dormant, overrides always win, disable never deletes overrides
- `compozy extension validate` output matches `_dx.md` error examples byte-for-byte for the covered cases
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
- `make gate` green including codegen-check (SDK contract propagation) and site navigation tests (new docs page registered)
