---
status: completed
title: "P4 — Keyboard: Open-ID Keymap, Aliases, Config, Settings Surfaces"
type: backend
complexity: high
---

# Task 5: P4 — Keyboard: Open-ID Keymap, Aliases, Config, Settings Surfaces

## Overview

Opens the daemon keymap to the full registry id space (closing the "bindable but invisible / visible but unbindable" wall), adds aliases as first-class daemon state, and lands the `[cmd_palette]` config lifecycle with its settings section. Ships the mutation agent surface — CLI `bind|unbind|alias|bindings|pin|unpin` atomic through the settings PATCH path — plus the whole-registry settings shortcut table with alias editing, and the extended cheatsheet. This closes the MVP line (P1–P4).

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting; read every ADR in `adrs/` and any `memory/task_NN.md` present
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — fix root causes; frozen `_dx.md`/Part II contracts are binding; surface gaps as clarifications, never invent
</critical>

<requirements>
1. MUST open the bindable-id space: `internal/windowmanager` known-id validation moves behind a `BindableIDs` source supplied by `cmdpalette` at composition time (core + `ext.*`); the closed-set shortcut validation in `internal/config/window_manager.go` is a delete target; chord grammar/canonicalization/collision semantics preserved; unknown-id overrides tolerated on read with a diagnostic (never boot failure — US-022.EC-3).
2. MUST implement aliases per the frozen surface: stored in `[cmd_palette.aliases]`, mutated through the window-manager settings PATCH (sections are presentation, roots are storage — Key Decisions), grammar 1–32 chars no whitespace, unique per workspace, `alias_conflict` naming the owner with explicit-overwrite transfer; alias pruning on dead command ids.
3. MUST land the `[cmd_palette]` config lifecycle complete: `internal/config/cmd_palette.go` (`FallbackTargets`, `Personalization`, `Aliases` + defaults + `Validate()`), overlay merge + clone registration, `tool_surface.go` registration for the two scalar keys, docs + examples — structs/defaults/merge/validation/docs/tests move together (SD-011). The fallback BEHAVIOR stays out (task_10); the key exists and validates now.
4. MUST add `SectionCmdPalette` (build/diff/apply per the section machinery, lifecycle `Live`, contract `SettingsSectionName` + apply-target consts) carrying personalization controls only — the fallback toggle ships with its behavior in task_10 (no control before its runtime exists, B-008); and extend `window_manager_section.go` with aliases + open-id shortcut diff paths.
5. MUST ship the mutation verbs per `_dx.md` transcripts exactly: `compozy cmd-palette bind|unbind` (`--overwrite` semantics naming the loser), `alias set|clear`, `bindings -o json` (effective + aliases + dormant_defaults + conflicts), `pin|unpin` — atomic through their daemon paths, structured errors identical to HTTP (US-034 parity).
6. MUST extend `GET|PATCH /api/settings/window-manager` per `_dx.md` (every registry command in `effective_shortcuts`, `aliases`, `extension_defaults` shape ready — populated by task_07) and `GET|PATCH /api/settings/cmd-palette` for the personalization flag; concurrent PATCHes converge last-write-wins (US-022.EC-4); keymap applies Live without restart (IT-025).
7. MUST deliver the settings shortcut table per S12 (minus the global-hotkey section — task_09): whole registry, source filter (Core areas / per-extension), recorder with conflict block naming the culprit + explicit overwrite flagging the loser, reset one/all, inline alias cell with grammar validation; palette rows render "Title (alias)"; cheatsheet gains registry-wide + source grouping (extension groups populate when task_07 lands).
8. MUST keep binding truth daemon-owned end-to-end: zero chord literals in TS; badges/cheatsheet/menus re-render from the effective keymap after PATCH.
9. MUST block and surface if a cited artboard is missing at execution time.
</requirements>

## Visual Contract

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | -------- | ---------------------------------- |
| VC-01 | `docs/design/opendesign/command-palette/command-palette-settings.html` — source-filtered whole-registry table | Settings → Shortcuts with source filter active | 1440×900 | normative | Extension rows/dormant defaults absent until task_07 (build order); global-hotkey section absent until task_09 |
| VC-02 | `command-palette-settings.html` — alias cell editing + invalid alias | inline alias edit with grammar violation message | 1440×900 | normative | None |
| VC-03 | `command-palette-settings.html` — conflict block naming culprit + explicit overwrite | recorder conflict flow | 1440×900 | normative | None |
| VC-04 | `command-palette-settings.html` — overwritten-loser flagged row | table after `--overwrite`/UI overwrite | 1440×900 | normative | None |

Evidence for each row: `.compozy/tasks/command-palette/evidence/visual/task_05/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 5.1 `BindableIDs` seam in `internal/windowmanager` + composition-root wiring; delete the closed-set validation in `internal/config/window_manager.go`; tolerant unknown-id reads
- [x] 5.2 Alias storage + validation + pruning in the daemon path (`[cmd_palette.aliases]` root, WM settings PATCH ride-along)
- [x] 5.3 `internal/config/cmd_palette.go` + defaults + validation + overlay/merge/clone + `tool_surface.go` registration + `config.toml` docs/examples
- [x] 5.4 `SectionCmdPalette` (personalization controls) + `window_manager_section.go` extension (aliases + open-id diffs) + contract consts + OpenAPI + payload projections
- [x] 5.5 CLI mutation verbs (`bind|unbind|alias|bindings|pin|unpin`) with transcript-exact output/errors/exit codes
- [x] 5.6 Settings shortcut table UI: whole-registry listing + source filter + recorder conflict/overwrite/reset + alias cell
- [x] 5.7 Cheatsheet derivation extension (registry-wide, source grouping) + "Title (alias)" row rendering + badge re-render on PATCH
- [x] 5.8 Transport parity rows + `make codegen` co-ship + skills/compozy (P4 verbs) + site docs (`configuration/shortcuts.mdx`, config reference)
- [ ] 5.9 Visual Contract evidence bundles VC-01..04 — deferred to task_12 by the accepted tail-only QA policy

## Implementation Details

Reference `_spec.md` Part II: Config Lifecycle (the authoritative checklist for this task), Key Decisions (aliases storage-vs-section), API Endpoints (settings sections), Safety Invariants 4, 10.

### Skills

`eng-code-guidelines` · `golang-master` · `eng-contract-codegen-coship` · `eng-test-conventions` · `testing-boss` · `eng-design` · `ui-craft` · `impeccable` · `eng-ui-screenshot` · `vitest`

### Relevant Files

- `internal/windowmanager/shortcuts.go` + `defaults.go` + `shortcut_binding.go` + `normalize.go` — keymap authority: action consts, `CanonicalShortcutsV2`, `EffectiveKeymap`, collision rejection; the `BindableIDs` cut lands here
- `internal/config/window_manager.go` (closed-set validation delete target) + `merge_window_manager.go` + `window_manager_overlay.go` + `config_clone.go` + `merge_overlay.go:9` + `config_extensions_sandbox.go:75-81` (root `Config` struct — watch the 500-line cap; new field, new file for the struct) — config plumbing to mirror for `[cmd_palette]`
- `internal/config/tool_surface.go` + `write_scope_policy.go` — scalar-surface registration for agent-manageable keys
- `internal/settings/window_manager_section.go` + `models.go:80` + `sections.go` + `section_config_update.go` + `config_apply_helpers.go:48` + `config_apply_service.go:42` + `internal/api/core/settings_window_manager_payload.go` + `internal/api/spec/registry_settings_window_manager.go` — the exact section machinery to clone as `cmd_palette_section.go`
- `internal/api/contract/settings.go:31-69` — `SettingsSectionName` + `SettingsApplyTargetName` closed sets gaining `cmd-palette`
- `internal/cli/window_manager_common.go` + `format.go` — CLI family/output patterns for the mutation verbs
- `web/src/systems/settings/components/layouts/window-manager-shortcut-table.tsx` + `use-window-manager-shortcut-recorder.ts` — the S12 table/recorder this task extends
- `web/src/systems/os/lib/window-manager-shortcuts.ts` + `os-shortcuts-dialog.tsx` — chord labels/conflicts/cheatsheet derivation reused
- `internal/store/globaldb` — no schema change here (aliases/bindings are config-backed; pins landed task_03) — checked

### Dependent Files

- `internal/daemon/boot_resource_graph.go` — `BindableIDs` wiring
- `openapi/compozy.json` + generated TS — settings payload extensions
- `packages/site/content/docs/configuration/shortcuts.mdx` + `config-toml.mdx` (hand-written) + generated CLI docs — `[cmd_palette]`, aliases, mutation verbs
- `skills/compozy/` — P4 verb surface
- `config.toml` (shipped example) — `[cmd_palette]` block per `_dx.md`

### Competitor References

- `.resources/supercmd/src/renderer/src/settings/HotkeyRecorder.tsx` — e.code capture rules for the recorder
- Legacy commander (cited by path in `analysis/01_legacy_commander.md`; repo `/Users/pedronauck/Dev/compozy/compozy-code` — not vendored under `.resources/`): conflict logic `src-tauri/src/plugins/commander/shortcuts.rs`, formatter `renderer/lib/commander/shortcut-formatter.ts`.

### Related ADRs

- [ADR-005](adrs/adr-005.md) — `[cmd_palette]` section naming; `palette.*` action-id family preserved
- [ADR-006](adrs/adr-006.md) — daemon owns binding truth; zero chord literals in TS

### Web/Docs Impact

- `web/`: `window-manager-shortcut-table.tsx` + recorder extension, alias cell, cheatsheet extension, settings hooks/fixtures/stories; palette "(alias)" row rendering in the projection.
- `packages/site`: `content/docs/configuration/shortcuts.mdx` (rebinding + aliases), `configuration/config-toml.mdx` (`[cmd_palette]` — hand-written page), generated CLI docs for the new verbs, generated config lifecycle matrix.
- QA impact: flag only (walks in task_12): reset `docs/qa/scenarios/ET-web-command-palette-shortcuts.md` to `untested` (rebind surface extended to the whole registry + aliases).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: the extension-default binding TIER shape is prepared in payloads (`extension_defaults`) but populates in task_07 — checked: no manifest/hook/SDK change here.
- Agent manageability: CLI `bind|unbind|alias set|alias clear|bindings|pin|unpin` with structured output + `-o json`; settings GET/PATCH on both transports; `compozy config get|set cmd_palette.personalization`/`fallback_targets` via the scalar surface; deterministic `shortcut_conflict`/`alias_conflict`/`invalid_alias` errors naming owners.
- Config lifecycle: FULL — new `[cmd_palette]` (`fallback_targets`, `personalization`, `aliases` map) + structs/defaults/merge/overlay/clone/validation/examples/docs/tests in this change; `[window_manager.shortcuts]` semantics extended (open id space), never broken; no key removed.

## Deliverables

- Open-id keymap + aliases live daemon-side with Live apply; closed-set validation deleted
- `[cmd_palette]` config lifecycle complete (structs → docs → tests) + `SectionCmdPalette` + extended WM section
- Mutation CLI verbs matching `_dx.md` transcripts byte-for-byte
- Whole-registry shortcut table + alias cell + extended cheatsheet
- Visual Contract evidence bundles VC-01..04 **(REQUIRED)**
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-070, UT-071, UT-072, UT-073, UT-074 — open-id acceptance, conflict naming + overwrite, shadow classification, bare-letter guard, tolerant unknown-id reads
- [x] UT-080, UT-081, UT-082, UT-083 — config defaults/validation (alias grammar, fallback target values), section build/diff/apply + Live lifecycle
- [x] UT-148, UT-149 — table + recorder + conflict/overwrite/loser flow; alias cell + "(alias)" rendering
- [x] IT-013 — WM settings PATCH bindings: effective map, 409 conflict naming owner, tolerated unknown ids, concurrent convergence
- [x] IT-014 — aliases via PATCH: round-trip, 422 grammar, visibility in commands GET
- [x] IT-025 — keymap Live apply without restart

Note: E2E-016 and E2E-025 are owned by task_07 — their frozen transcripts require the extension fixture that first exists in P6; this task closes on its settings/keymap suites per the P4 gate.

## Success Criteria

- Every assigned test case implemented and passing
- Any registry id — core or (once task_07 lands) `ext.*` — binds through one validation path; the closed set is gone
- CLI mutation verbs and HTTP PATCH produce byte-identical structured errors (IT-013/IT-014 assert both transports)
- `make gate` green including `make codegen-check` (settings contract + config docs regen)
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
