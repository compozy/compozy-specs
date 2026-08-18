---
status: completed
title: "Shortcuts v2: grammar, daemon-owned keymap, new actions, migrations, preset"
type: backend
complexity: high
---

# Task 5: Shortcuts v2: grammar, daemon-owned keymap, new actions, migrations, preset

## Overview

Delivers P5 as one hard cut across both sides of the mirrored shortcut system: the binding grammar widens to arrays + range strings (empty disables), chord defaults move into the daemon (ADR-006) and are served as the effective keymap, the frozen default keymap from `_dx.md` Keyboard Defaults lands (new actions, four revisions, four shell-chord migrations, tiles completed), the Settings table gains the Tabs group + array/range rendering, the cheatsheet derives from the live registry behind `?`/⌘/, and the validator-clean Terminal preset ships with preview/revert. No dependencies — runs in parallel with tasks 01–04.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. `[window_manager].shortcuts` MUST widen to string-or-array (range strings expand; array members may be ranges; empty disables) as a hard cut: the scalar-only `map[string]string` is a delete target across config struct, overlay, contract payload, CLI classifier, and web parser — no dual-shape reading (greenfield rule; delete targets in `_spec.md` Impact Analysis).
2. Chord defaults MUST move to `internal/windowmanager` (`DefaultKeymap`/`EffectiveKeymap`/`CanonicalShortcutsV2`): full-map conflict validation (blocked AND shadowed, arrays and expansions included), effective keymap served on `GET /api/settings/window-manager` (`defaults` + `effective`), TS registry keeps UI metadata only — a parity fixture asserts zero chord literals remain in TS (ADR-006).
3. The Go action allowlist MUST grow to the complete v2 set (new actions + migrated `palette.open`, `session.new`, `scope.global.toggle`, `window.nav.back`) with a distinct `desktop.switch.1..9` range rule (never widening `window.tab.jump.1..8`).
4. The frozen keymap MUST land exactly as `_dx.md` Keyboard Defaults: ⌘E sessions view (action registered; the view itself is task_06), ⌃⇧↑↓ session cycle (frozen-order burst), ⌃⌥A attention jump, ⌘⇧O picker (opens the existing `OsWorkspacesOverview`), ⌘⇧[/] workspace cycle (menubar's visible order, switch barrier), ⌃1..9 desktop switch (comparator shared with `switchDesktopDirection`), ⌘⇧N create, ⌘⇧D overview, ⌃⇧←→ desktop prev/next, ⌘B sidebar, ⌃⌥O focus-last (daemon `focusOrder` MRU), ⌃⌥Z zoom, ⌃⌥↑↓ tiles top/bottom, ?+⌘/ cheatsheet — old chords are delete targets, no aliases.
5. Shell-chord migration MUST preserve behavior: migrated actions exempt from the availability gate, ⌘K/⌘N whitelisted inside editables, ⌘[ becomes rebindable, Esc stays hardcoded, `isPlainMod` palette/new-session branches + `SHELL_ROWS` are delete targets (K5 semantics).
6. Settings MUST gain the Tabs group, array/range row rendering, and expansion-aware conflict diagnostics; the recorder handles array alternates.
7. The cheatsheet MUST derive every row from the effective keymap (`?` outside editables, ⌘/ always) with the surface-local read-only reference section and ONE shared modifier-glyph helper (the 3× maps are delete targets).
8. The Terminal preset MUST ship exactly as the `_dx.md` block (validator-clean: bottom tiles → ⌃⌥⇧J/K), applied from Settings with old→new preview incl. hazard flags, atomic apply, idempotent re-apply, one-step revert; documented as a pasteable block on the new shortcuts docs page.
9. Docs MUST ship in this task: the new `packages/site` shortcuts page (default keymap + preset + platform notes), `config-toml.mdx` value-shape update, `skills/compozy/references/window-management.md` (§ shortcuts, line ~320) + `configuration.md`.
10. MUST implement S9/S10/S11 from the locked visual contracts in `docs/design/opendesign/herdr-parity/` (`DESIGN-NOTES.md` + the boards named in `## Visual Contract`). Activate `eng-design` + `ui-craft` + `impeccable` and `eng-ui-screenshot` Visual Contract Mode before the Settings/cheatsheet UI work. Artboard CSS is a contract, never a stylesheet to import. MUST produce the `eng-ui-screenshot` evidence bundle for every Visual Contract row — implementation-only captures are invalid.
</requirements>

## Visual Contract

Reference artboards: `docs/design/opendesign/herdr-parity/` (iterate, never regenerate). All rows viewport 1440×900, fidelity normative. Authorized differences: `DESIGN-NOTES.md` lossiness rules (runtime truth + `COPY.md` own content/copy; `@compozy/ui` owns component identity; live Settings/cheatsheet host chrome owns placement context; keymap chords render `_dx.md` Keyboard Defaults, never a local copy) — record each divergence as an authorized delta in `review.md`.

| ID | Reference artifact + state | Implementation target + state | Viewport | Fidelity |
| --- | --- | --- | --- | --- |
| VC-01 | `herdr-parity-settings-shortcuts.html` — §01 table (Tabs group, solid primary + dashed alternates, compact ranges) | Settings → Shortcuts table | 1440×900 | normative |
| VC-02 | `herdr-parity-settings-shortcuts.html` — §02 blocked conflict names the member digit | Shortcut row `blocked` diagnostic | 1440×900 | normative |
| VC-03 | `herdr-parity-settings-shortcuts.html` — §02 shadowed names the winning surface | Shortcut row `shadowed` diagnostic | 1440×900 | normative |
| VC-04 | `herdr-parity-settings-shortcuts.html` — §02 partial range override (diverged digit marked accent) | Range-family row partial override | 1440×900 | normative |
| VC-05 | `herdr-parity-settings-shortcuts.html` — §03 Terminal preset preview (displaced defaults + hazard flags) | `ShortcutPresetCard` preview | 1440×900 | normative |
| VC-06 | `herdr-parity-settings-shortcuts.html` — §03 preset applied (revert affordance) | `ShortcutPresetCard` applied | 1440×900 | normative |
| VC-07 | `herdr-parity-cheatsheet.html` — §01 two-column sheet + lock-marked surface-local section | Cheatsheet overlay populated | 1440×900 | normative |
| VC-08 | `herdr-parity-cheatsheet.html` — §02 override rows marked after rebind | Cheatsheet post-rebind freshness | 1440×900 | normative |

Evidence for each row: `.compozy/tasks/herdr-parity/evidence/visual/task_05/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 5.1 Go grammar v2: `ShortcutBinding`, array/range parsing, full-map validation (blocked + shadowed), allowlist growth, `DefaultKeymap`/`EffectiveKeymap`.
- [x] 5.2 Config hard cut: struct/overlay/contract/CLI-classifier value-type change with atomic array literals in `config set`.
- [x] 5.3 Settings payload + GET/PATCH serving defaults/effective; parity tests.
- [x] 5.4 TS mirror: parser arrays/ranges, chord literals removed from the registry (daemon-fed), platform mapping intact; parity fixture Go↔TS.
- [x] 5.5 New action dispatchers: session cycle (frozen burst order), attention jump (shared land-on-session behavior), workspace picker/cycle (ordering authority + barrier), desktop switch/create, sidebar toggle, focus-last (MRU), zoom/tile revisions.
- [x] 5.6 Shell-chord migrations with gate-exemption + editable whitelist; delete `isPlainMod` branches and `SHELL_ROWS`.
- [x] 5.7 Settings table (Tabs group, arrays/ranges, diagnostics) + recorder updates.
- [x] 5.8 Cheatsheet: registry-derived rows, `?`/⌘/ bindings, read-only section, shared glyph helper (delete the 3 maps + hardcoded JSX labels).
- [x] 5.9 Terminal preset: data + preview/apply/revert flow in Settings.
- [x] 5.10 Docs: shortcuts site page + config reference + skill references.
- [x] 5.11 Visual-contract capture bundles for VC-01..VC-08 — completed in task_08; all 8 bundles validate with zero blocking divergences.

## Implementation Details

Reference `_spec.md` Part II (Core Interfaces windowmanager block, Impact Analysis delete targets) and `_dx.md` Keyboard Defaults (frozen). Keymap-verify audit anchors:

### Relevant Files

- `internal/windowmanager/shortcuts.go:15-171` — grammar, `CanonicalShortcuts`, allowlist (`validShortcutAction:108-147`, tab-jump 1..8 rule at 109-111), code allowlist (Slash present).
- `internal/config/window_manager.go:72,126,300-305` + `merge_window_manager.go:22` + `internal/api/contract/settings_window_manager.go:74` + `internal/cli/config_daemon_window_manager.go:12` — the five-file value-type cut.
- `web/src/systems/os/lib/window-manager-shortcuts.ts:35-153` (mirror grammar, `shortcutMatches` portable-primary at 131-153) + `window-manager-command-registry.ts:87-211` (defaults to strip) + `window-manager-action-dispatch.ts:56` (context to widen with shell callbacks).
- `web/src/systems/os/hooks/use-os-shortcuts.ts:27-152` — guard order, `isPlainMod` (:27-29), editable rules (:55-57,100-104), migration surface.
- `web/src/systems/settings/components/layouts/window-manager-shortcut-table.tsx:14-50` + `window-manager-shortcut-row.tsx:19` + `use-window-manager-shortcut-recorder.ts:81` — Settings surface.
- `web/src/systems/os/components/os-shortcuts-dialog.tsx:30-41` — `SHELL_ROWS` delete target; `menubar/help-menu.tsx:45`.
- `docs/design/opendesign/herdr-parity/DESIGN-NOTES.md` + `herdr-parity-settings-shortcuts.html` + `herdr-parity-cheatsheet.html` — locked visual contracts for S9–S11.
- Dispatch targets: `os-workspaces-overview.tsx:109` + `use-desktop-overlays.ts:21` (picker), `use-active-workspace.ts:41` + `workspace-menu.tsx:91-94` (cycle ordering authority) + `workspace-switch-barrier.ts`, `window-manager-runtime.ts:56,71,88-98` (desktop create/switch + comparator), `window-instance-lookup.ts:44-57` (MRU), `use-session-sidebar-state.ts:84-95` (sidebar toggle), `use-os-attention.ts` (jump target from task_03's model — ships with a workspace-scoped fallback if 03 hasn't landed; the actions are registry entries either way).
- cmdk precedence: `packages/ui/src/components/command.tsx` (vim bindings stay ON — focused-UI-wins documented).

### Dependent Files

- `web/src/systems/os/components/os-command-palette-shell-actions.tsx:38,50` + `go-menu.tsx:43` + `session-menu.tsx:32` + `os-win-layer.tsx:113` + `os-window-deck.tsx:76,184` + `shell-shortcuts.ts` — hardcoded chord labels → shared helper.
- `packages/site/content/docs/configuration/config-toml.mdx:937-943` + new shortcuts page; `skills/compozy/references/window-management.md`.
- `web/e2e/` — E2E-017/018.

### Related ADRs

- [ADR-004: Shortcut strategy](adrs/adr-004.md) — keymap posture + preset; [ADR-006: Daemon-owned keymap defaults](adrs/adr-006.md) — ownership move this task executes.

### Competitor References

- `.resources/herdr/src/config/keybinds.rs:19-24,1017-1328` — untagged `One(String)|Many(Vec<String>)` binding shape, empty-disables, user-beats-defaults two-pass resolution, duplicate diagnostics.
- `.resources/herdr/src/config/model.rs:1010-1077` — default keymap + the three range/indexed actions (`switch_tab`, `switch_workspace`, `focus_agent` with `1..9`).
- `evidence/herdr-live-cli.md §5` — Pedro's verbatim personal bindings the Terminal preset transposes.

### Web/Docs Impact

- `web/`: all files above (os lib/hooks/components, settings layouts, ui command precedence unchanged).
- `packages/site`: NEW shortcuts docs page (default keymap table + preset block + platform notes: ⌃digits browser-tab collision on non-Apple, AltGr aliasing, `?` on non-US layouts); `config-toml.mdx` shortcuts value shape; `skills/compozy/references/window-management.md` + `configuration.md`.
- QA impact: new scenarios — add content-addressed `untested` files for: array/range rebinding via Settings and `config set`, migrated ⌘K/⌘N behavior in inputs, cheatsheet truth after rebind, preset apply/revert, new navigation actions. Flag only — task_08 walks them.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: registry action IDs are a public contract (palette/menubar/cheatsheet derive); no extension surfaces change (checked: manifests, hooks, tools).
- Agent manageability: effective-keymap discovery via `GET /api/settings/window-manager` + `compozy config get window_manager` (defaults + effective); `config set` accepts scalar/array/range literals with full-truth conflict errors.
- Config lifecycle: `[window_manager].shortcuts` value-type change (breaking, hard cut) with validation/examples/docs/tests in this task; no keys added/removed (checked: `internal/config/window_manager.go` inventory).

## Deliverables

- Grammar v2 live on both sides with the parity fixture proving single-source chords.
- Frozen keymap + migrations + Settings/cheatsheet truth + validator-clean preset.
- New shortcuts docs page; config reference updated.
- Every Visual Contract row has a durable passing evidence bundle **(REQUIRED)**.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

- [x] UT-035..UT-046 — grammar parse/expand/conflicts, defaults/effective, disable, shadowed detection
- [x] UT-062 — TS parser mirror parity
- [x] UT-063, UT-064 — preset diff + idempotent apply/revert
- [x] UT-065 — session-cycle frozen order
- [x] UT-072..UT-074 — focus-last MRU, desktop-switch comparator, attention-jump no-op honesty
- [x] UT-075 — cheatsheet derivation (arrays, migrated rows, read-only section)
- [x] IT-015 — settings PATCH validation + defaults/effective serving + TS-no-literals fixture
- [x] IT-016 — `config set` attention + shortcuts array/range literals (atomic rejection)
- [x] E2E-017, E2E-018 — passed in task_08's full Web E2E run.

## Success Criteria

- Every assigned test case implemented and passing.
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence.
- Every action bindable and truthfully listed; `?` and ⌘/ open the cheatsheet; ⌘K/⌘N behavior unchanged inside inputs post-migration (US-024..US-028 ACs).
- The published preset block applies verbatim through `config set` with zero validation errors.
- `make gate` green on Go + web lanes; no near-cap file grew.
