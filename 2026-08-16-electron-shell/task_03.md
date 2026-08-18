---
status: completed
title: Electron shell at parity
type: desktop
complexity: critical
---

# Task 3: Electron shell at parity

## Overview

Builds the new `desktop/` Electron app to full behavioral parity with the Tauri shell (minus the decided exceptions): windowing, boot window with non-interactive phases, single instance, deep links, external-link guard, zoom, window-state, renderer crash recovery, the bundled runtime with the shell-owned pre-spawn trust check, Go-bootstrap launch + typed-progress rendering, the app-track updater bound to the Update Operation, and the hardened token-authenticated control server. Also lands the `make test-e2e-desktop` Playwright `_electron` lane — the migration's testability payoff.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST create `desktop/` as a Bun-workspace Electron app with the t3code-style module layout — one responsibility per file, 500-line hard cap, main entry as pure composition; Electron and every dependency pinned exact.
2. MUST load the product window via `loadURL(daemonOrigin)` with **no preload**; the boot window is a packaged local page (port of `pages/boot.*`) with a minimal shape-validated bridge and its own meta CSP per Security Invariant 16.
3. MUST enforce Security Invariants 13–15 on every window: `contextIsolation: true`, `sandbox: true`, `nodeIntegration: false`, no `webviewTag`; default-deny permission request+check handlers; packaged devtools disabled with no env override; per-window navigation policy (same-origin `will-navigate`, deny `setWindowOpenHandler`, http/https-only `openExternal`).
4. MUST bundle the lockstep per-arch `compozy` binary and verify its digest against the build-embedded manifest **before first spawn** (B-027 trust check); provision is install-from-bundle driven by the Go bootstrap — the shell launches `compozy daemon bootstrap -o jsonl` and renders typed phases, deciding nothing (ADR-002).
5. MUST implement the boot ladder UX at parity: phase progress, explain-the-failure (retry/open-logs/quit), skew and error states with frozen strings; the interactive update offer does NOT exist (ADR-006).
6. MUST implement single instance (lock after userData), deep links (`compozyos://` — macOS `open-url` registered pre-ready, Win/Linux argv + second-instance, queue-until-renderer-ready exactly-once, hostile-payload rejection to default view), window-state persistence with display clamping, bounded zoom with re-assert on reload, and renderer crash recovery (3 per rolling 60s, then dialog).
7. MUST implement the app-track updater bound to the Update Operation: watch the record (fs watch + poll fallback), transition phases only via the installed `compozy` binary (never writing the journal), install only the recorded digest, honor the non-cancelable `installer-handoff` phase, write post-restart verification, and resolve staged operations on launch.
8. MUST implement the control server wire-compatible subset (`navigate`, `retry`, `diagnose`, `export_diagnostics`, `copy_diagnostics`) hardened per Security Invariants 17–18: dir/socket modes, capability-token auth (generate/rotate `~/.compozy/app.token` 0600, timing-safe compare), bounds/backpressure, stale-socket safety — plus the lockstep `token` field in the Go CLI client.
9. MUST write `app.json` (schema v2) as the app-state writer with atomic replace semantics, preserving `compozy app status/open/retry/diagnose` behavior byte-for-byte.
10. MUST quit leaving the runtime running (detached spawn, no tree-kill), with quiesce coordination during runtime updates.
11. MUST land `make test-e2e-desktop` (Playwright `_electron`, real daemon, mock feed) in CI, plus the dev loop (`scripts/desktop-dev.sh` swapping cargo build/exec for the Electron dev flow) and Makefile targets (`desktop-dev/build/test/lint` re-pointed).
12. MUST read no update configuration in the shell (daemon-only cadence, B-015) and add no telemetry.
</requirements>

## Subtasks

- [x] 3.1 Scaffold `desktop/` (workspace, pinned deps, main/preload bundling, static builder config + minimal channel/signing generator, module layout)
- [x] 3.2 Windowing: product window (no preload), boot window port with meta CSP, reveal fallback, window-state clamp, zoom, crash recovery
- [x] 3.3 Security baseline: window flags, permission handlers, packaged devtools ban, navigation guards (UT-064, E2E-010, E2E-034)
- [x] 3.4 Bundled runtime + pre-spawn digest check (UT-055) + bootstrap launch/typed-progress rendering (E2E-001..006)
- [x] 3.5 Single instance + deep links (queue, hostile rejection) (UT-066/067, E2E-004/013/014/015)
- [x] 3.6 App-track updater: operation consumer via `compozy` transitions, digest-bound install, installer-handoff, staged-on-launch, silent-failure watchdog (UT-068, E2E-016/017)
- [x] 3.7 Control server hardened + token; Go CLI client token field (UT-078, IT-017, IT-021, E2E-024/025)
- [x] 3.8 Quit contract + quiesce coordination (E2E-007)
- [x] 3.9 `make test-e2e-desktop` lane (fixtures: mock feed, temp homes, per-test isolation) wired in CI; engine-parity run of the daemon-served journey suite (E2E-011)
- [x] 3.10 Dev loop + Makefile target swap; local unsigned packaged build for e2e
- [ ] 3.11 Full assigned suite green; APP-* scenarios locally walkable — **execution deferred by user decision** to tasks 06/07; all cases are authored and all affected scenarios are `untested`

## Implementation Details

`_spec.md` Part II System Architecture (shell row + trust root), Safety/Security Invariants, and `analysis/electron-reference-blueprint.md` are the build map — the blueprint carries per-subsystem reference paths into `.resources/{synara,hermes,t3code}` and must be read in full. Boot-window strings and overlay roles are contracted in `_dx.md`.

Skills to activate: `typescript-advanced`, `vitest`, `testing-boss`, `eng-consolidate-test-suites`, `no-workarounds`, `context7` (Electron/electron-updater/electron-builder docs), `golang-master` + `eng-code-guidelines` (CLI token client change), `systematic-debugging`.

### Relevant Files

- `desktop/src-tauri/` — the behavior source of truth to port: `src/{shell.rs,windowing.rs,nav.rs,boot_window.rs,control.rs,controller.rs,home.rs,health_monitor.rs}`, `pages/boot.{html,js,css}`, `tauri.conf.json` (identity, scheme)
- `desktop/schema/app-state.schema.json` — v2 contract the shell writes (from task_01)
- `internal/cli/app_control.go`, `internal/cli/app.go` — Go client gaining the `token` field; cold-start deep-link producer
- `internal/cli/app_platform.go` — install detection (AppImage `X-AppImage-Version` key parity)
- `scripts/desktop-dev.sh`, `Makefile:104-115` — dev loop + targets to re-point
- `.github/workflows/ci.yml` (desktop job) — lint/test lanes to re-point
- `web/playwright.config.ts`, `web/package.json` e2e scripts — patterns for the new `_electron` lane

### Dependent Files

- `package.json` (root) — workspace member addition (`bun add`-managed, never hand-edited deps)
- `turbo.json` — desktop lint/test task wiring
- `docs/qa/charters/CH-desktop-*.md` — tauri-driver references become obsolete (flagged; rewritten in task_06/07 cycle)

### Related ADRs

- [ADR-001](adrs/adr-001.md) — the engine decision this task realizes
- [ADR-002](adrs/adr-002.md) — thin shell + bundled bootstrap
- [ADR-005](adrs/adr-005.md) — control methods deleted with the verb
- [ADR-006](adrs/adr-006.md) — overlay non-interactive roles
- [ADR-009](adrs/adr-009.md) — operation consumer discipline

### `.resources/` References (read before building each subsystem)

- `.resources/t3code/apps/desktop/src/main.ts` + `src/window/DesktopWindow.ts` — module layout, window options, reveal, crash recovery
- `.resources/hermes/apps/desktop/electron/{window-reveal.ts,zoom.ts,window-state.ts}` + `main.ts` deep-link paths (`:12485`) — reveal fallback, zoom re-assert, state clamp, 3-path deep-link delivery + queue
- `.resources/synara/apps/desktop/src/{windowState.ts,mediaPermissions.ts,browserUsePipeServer.ts,updateInstallMarker.ts}` — clamp variant, default-deny permissions, socket bounds/backpressure, install marker/watchdog
- `.resources/hermes/.github/workflows/e2e-desktop.yml` — Playwright `_electron` CI lane shape
- `.resources/t3code/apps/desktop/src/electron/ElectronProtocol.ts` — the documented fallback if origin stability ever forces the scheme proxy

### Web/Docs Impact

- `web/`: none — the SPA is untouched by design (zero shell-awareness; engine-parity E2E-011 reuses the existing journey suite unchanged).
- `packages/site`: none in this task — desktop docs pages rewrite happens in task_05 against the shipped shell.
- QA impact: reset to `untested` — all 14 `docs/qa/scenarios/APP-*.md` (shell replaced); charters `CH-desktop-links-instance-{linux,windows}.md` reference `tauri-driver` and are flagged for the QA cycle rewrite. Flag here; the recorded walk happens in task_07.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked; the shell exposes no extension surface and adds none.
- Agent manageability: preserved verbs (`compozy app status/open/retry/diagnose`) byte-compatible over the hardened channel; token is transport-internal (no agent-visible surface change); deterministic errors frozen.
- Config lifecycle: none — the shell reads no `[app]` keys (B-015); `COMPOZY_HOME` honored as today; no new keys.

## Deliverables

- `desktop/` Electron app at parity, packaged locally (unsigned) and bootable on macOS + Linux
- `make test-e2e-desktop` green in CI against the real packaged shell
- Hardened control channel with token auth, CLI client updated lockstep
- Dev loop working (`make desktop-dev`)
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-033–UT-034 — retry no-op integrity + diagnose paths against the live control server
- [x] UT-055 — pre-spawn bundle digest trust check (shell owner)
- [x] UT-061–UT-068 — window-state/zoom/nav/crash/deep-link/consumer policy modules
- [x] UT-078 — socket + token precondition table
- [x] IT-017, IT-021 — retry against live server; control-channel auth boundary
- [x] E2E-001–E2E-018 — authored in the desktop Playwright lane; execution is task_07-owned
- [x] E2E-024–E2E-025 — authored in the desktop Playwright lane; execution is task_07-owned
- [x] E2E-034 — authored in the desktop Playwright lane; execution is task_07-owned

## Success Criteria

- Every assigned test case implemented and passing; `make gate` green
- All 14 `APP-*` scenarios walkable locally against the packaged shell (evidence links)
- E2E-011: the daemon-served Playwright journey suite passes unchanged under the `_electron` fixture
- Every production source file ≤500 lines; boot strings byte-match the `_dx.md` parity reference
