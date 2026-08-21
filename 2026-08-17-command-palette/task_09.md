---
status: completed
title: "P7 — Desktop Global Hotkeys: Preload Bridge, globalShortcut, Reconciler"
type: desktop
complexity: high
---

# Task 9: P7 — Desktop Global Hotkeys: Preload Bridge, globalShortcut, Reconciler

## Overview

Takes the palette to the OS level under the Electron shell: the first product-window preload bridge (`window.compozyShell`, modeled on the boot contract pair), a `globalShortcut` module with typed per-chord registration statuses and restore-on-failure, and the daemon-owned global-binding model — `[window_manager.global_shortcuts]` intended state (any registry command; `palette.summon.global` = `meta+shift+Space` default) reconciled by the shell over the client channel with per-shell registration status projected on catalog and settings. Global summon focuses/restores the window with the palette open; the S12 global-hotkey section ships its honest states; browser mode renders the requires-desktop-shell reason.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting; read every ADR in `adrs/` and any `memory/task_NN.md` present
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — fix root causes; frozen `_dx.md`/Part II contracts are binding; surface gaps as clarifications, never invent
</critical>

<requirements>
1. MUST build the product preload bridge on the boot-contract discipline: `desktop/src/product/product-contract.ts` (single IPC contract: method allowlist + params/response validation) + `product-preload.ts` (`contextBridge` → `window.compozyShell = { platform, on(event, cb), globalShortcuts: { sync(bindings), status() } }`); new esbuild entry in `scripts/build-main.ts` + `files:` entry in `electron-builder.yml`; `sandbox:true`/`contextIsolation:true` and the default-deny posture unchanged; shell detection = bridge presence.
2. MUST implement the `globalShortcut` module (`desktop/src/shortcuts/`, policy pure with an injected `GlobalShortcutLike` seam for vitest): chord → Electron accelerator conversion (tested table; unconvertible → typed error), registration returning per-id `{registered|failed_in_use|failed_permission|unsupported}`, **restore-previous-binding on failure**, unregister-all on quit; macOS Accessibility detection surfaces once at bridge init with the settings deep-link payload.
3. MUST land the `[window_manager.global_shortcuts]` config lifecycle complete: map field on `WindowManagerConfig` (command_id → chord; ANY registry command valid via the task_05 `BindableIDs` seam) + defaults (`palette.summon.global = "meta+shift+Space"`) + validation + overlay merge/clone + settings PATCH path with the same atomic conflict semantics + `config.toml` example + docs — `internal/config/*` edits escalate the gate to full (expected).
4. MUST implement the reconciliation loop: web pushes the effective global set (from the daemon keymap) over `sync`; the shell registers and acknowledges per-chord status; the daemon stores that ephemeral registration status **per shell client** and projects both intended and registration state on catalog + settings GET; a chord renders active only when confirmed registered (SI-10, SD-007); failure restores the previous working binding and reports it.
5. MUST ship the mutation surface per `_dx.md`: `compozy cmd-palette bind <id> <chord> --global` / `unbind <id> --global` (extending the task_05 verbs) + the settings PATCH; browser mode: the section is absent from runtime state and Settings renders the requires-desktop-shell reason (US-024.AC-4).
6. MUST implement summon: the global chord fires `shell:summon` → web opens the palette overlay; `ProductWindow.focus()` restores; summon over a modal focuses without executing through it; an argument-command global hotkey opens directly in argument mode (US-015.EC-3 parity — task_04's args mode).
7. MUST deliver the S12 global-hotkey section states (in-use failure with previous binding still effective, browser-mode disabled-with-reason, Accessibility callout with system deep-link) and emit `global_hotkey.registration_failed{workspace, client_id, command_id, chord, reason}`.
8. MUST run the `_electron` E2E lane explicitly: `make test-e2e-desktop` is NOT part of the default verify lanes — cite its green run as completion evidence alongside `make gate`.
9. MUST block and surface if the cited artboard is missing at execution time.
</requirements>

## Visual Contract

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | -------- | ---------------------------------- |
| VC-01 | `docs/design/opendesign/command-palette/command-palette-settings.html` — global section: "unavailable — in use by another application" with previous binding effective | Settings → Shortcuts global section, harness-preregistered accelerator | 1440×900 | normative | None |
| VC-02 | `command-palette-settings.html` — browser-mode disabled with requires-desktop-shell reason | plain-browser session, global section | 1440×900 | normative | None |
| VC-03 | `command-palette-settings.html` — macOS Accessibility callout with system deep-link | shell reporting `failed_permission` | 1440×900 | normative | None |

Evidence for each row: `.compozy/tasks/command-palette/evidence/visual/task_09/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 9.1 `product-contract.ts` + `product-preload.ts` bridge (allowlist validator, boot-contract discipline) + build/packaging entries
- [x] 9.2 `desktop/src/shortcuts/` module: accelerator conversion table, `GlobalShortcutLike` DI, register/status/restore-previous/unregister policies, Accessibility detection
- [x] 9.3 `[window_manager.global_shortcuts]` config lifecycle (struct/defaults/validate/merge/clone/example/docs) + settings PATCH conflict semantics
- [x] 9.4 Daemon reconciliation: effective-set push over the client channel, per-shell-client registration-status storage, catalog + settings projection (intended vs registered)
- [x] 9.5 CLI `bind|unbind --global` extension + transcripts + structured errors
- [x] 9.6 Summon path: `shell:summon` → palette overlay + window focus/restore; modal + argument-mode variants
- [x] 9.7 S12 global section states + browser-mode gating + `global_hotkey.registration_failed` event
- [x] 9.8 `_electron` E2E specs in `desktop/e2e/_electron/__tests__/` (summon, args-mode hotkey, failure surface, browser-mode) + docs page `packages/site/content/docs/desktop/global-hotkeys.mdx` (new) + `configuration/shortcuts.mdx` cross-link
- [ ] 9.9 Visual Contract evidence bundles VC-01..03 — capture remains in the accepted task_12 QA tail

## Implementation Details

Reference `_spec.md` Part II: Integration Points (Electron shell + global-hotkey binding model — the authoritative two paragraphs), Monitoring (`global_hotkey.registration_failed` fields), Safety Invariant 10.

### Skills

`eng-code-guidelines` · `golang-master` · `eng-contract-codegen-coship` · `eng-test-conventions` · `vitest` · `eng-design` · `ui-craft` · `impeccable` · `eng-ui-screenshot` · `documentation-writer`

### Relevant Files

- `desktop/src/boot/boot-preload.ts` + `boot-contract.ts` — the contextBridge + hand-written validator template the product pair mirrors
- `desktop/src/main.ts` — entry wiring point (beside `installApplicationMenu`); `cleanup()` owns unregister-all
- `desktop/src/window/product-window.ts` (no-preload window gaining the bridge; `before-input-event` precedent) + `security.ts` + `navigation-policy.ts` — default-deny posture to respect
- `desktop/src/window/application-menu.ts` — today's only accelerator surface (local menu, NOT globalShortcut — greenfield confirmation)
- `desktop/scripts/build-main.ts` + `electron-builder.yml` — esbuild entries + `files:` + fuses for the new preload
- `desktop/e2e/_electron/__tests__/shell.spec.ts` + `fixtures.ts` + `desktop/playwright.config.ts` — the desktop E2E harness
- `internal/config/window_manager.go` + `merge_window_manager.go` + `window_manager_overlay.go` + `config_clone.go` — config plumbing for the new map (zero Go presence today — greenfield keys)
- `internal/windowmanager` + task_05's `BindableIDs`/settings PATCH path — conflict semantics + id validation reused
- `internal/api/core/window_manager_*` + settings payloads — registration-status projection surfaces
- `web/src/systems/os/hooks/use-desktop-overlays.ts` + `use-desktop-shell-body.ts` — summon → palette overlay wiring; bridge-presence shell detection
- `web/src/systems/settings/components/layouts/window-manager-shortcut-table.tsx` — S12 host gaining the global section
- `internal/events/names.go` — `global_hotkey.registration_failed` registration

### Dependent Files

- `internal/cli/cmd_palette*.go` — `--global` flag surface on bind/unbind
- `openapi/compozy.json` + generated TS — settings/catalog projection extensions
- `config.toml` (shipped example) + `packages/site/content/docs/configuration/config-toml.mdx` — `[window_manager.global_shortcuts]` documentation (hand-written page)
- `packages/site/content/docs/desktop/global-hotkeys.mdx` (new) + owning `meta.json` — navigation-registered (test-enforced)
- `skills/compozy/` — global-hotkey operation path

### Competitor References

- `.resources/supercmd/src/main/main.ts:~14839` — typed duplicate-conflict + restore-previous registration (the exact policy to mirror).

### Related ADRs

- [ADR-001](adrs/adr-001.md) — any registry command may carry a global hotkey (one id space)
- [ADR-006](adrs/adr-006.md) — daemon owns intended binding truth; shell reports registration; SD-007 active-only-when-confirmed

### Web/Docs Impact

- `web/`: `GlobalHotkeySection` in the shortcut table (three states), summon overlay wiring, bridge-presence detection, MSW/story fixtures for the states.
- `packages/site`: `desktop/global-hotkeys.mdx` (new, registered), `configuration/shortcuts.mdx` cross-link, `configuration/config-toml.mdx` section, generated CLI docs (`--global` flags).
- QA impact: flag only (walks in task_12): add content-addressed `untested` scenario **global summon** (shell summon + failure surface + browser gating).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: no manifest/hook/SDK change (extension commands participate via the open id space automatically; their global binding is operator-authored config).
- Agent manageability: `bind|unbind --global` CLI + settings PATCH + registration-status projection on catalog/settings GET (both transports) + deterministic errors; `compozy config get window_manager.global_shortcuts` via existing config surfaces.
- Config lifecycle: FULL — new `[window_manager.global_shortcuts]` map + defaults + validation + merge/overlay/clone + example + docs + tests in this change; existing `[window_manager]` keys untouched.

## Deliverables

- Product preload bridge + shortcuts module with typed statuses and restore-on-failure
- `[window_manager.global_shortcuts]` lifecycle + daemon reconciliation + per-shell status projection
- Global summon (incl. argument-mode) + S12 global section honest states + browser gating
- `_electron` E2E lane green (cited explicitly) + new docs page
- Visual Contract evidence bundles VC-01..03 **(REQUIRED)**
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-155 — chord→accelerator conversion table incl. typed unconvertible error
- [x] UT-156 — sync policy: failed registration → `failed_in_use` + restore-previous; quit unregisters all
- [x] UT-157 — Accessibility detection → `failed_permission` + deep-link payload
- [x] UT-158 — bridge contract validator (allowlist + params; unknown method rejected)
- [x] UT-150 — settings global section states (browser-mode reason; in-use with previous binding effective)
- [ ] E2E-027 — global summon focuses/restores with palette open; modal variant — authored; execution remains in task_12
- [ ] E2E-028 — argument-command global hotkey → summon in argument mode — authored; execution remains in task_12
- [ ] E2E-029 — registration failure surface + previous binding works + relaunch re-registers/re-reports — authored; execution remains in task_12
- [ ] E2E-030 — browser mode: global settings disabled-with-reason; in-app ⌘K unaffected — authored; execution remains in task_12

## Success Criteria

- Every assigned test case implemented and passing
- A chord never renders active without confirmed registration; a failed registration never leaves the command silently unbound (UT-156/E2E-029 pin SI-10)
- Renderer security posture unchanged (sandbox/contextIsolation/default-deny — asserted by the existing desktop suites)
- `make gate` green (full — config trigger) AND `make test-e2e-desktop` green, both cited as evidence
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
