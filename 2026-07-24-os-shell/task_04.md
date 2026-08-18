---
status: completed
title: "Shell core: window manager, routing coordinator, DesktopShell, palette"
type: frontend
complexity: critical
---

# Task 4: Shell core: window manager, routing coordinator, DesktopShell, palette

## Overview

Replace the AppShell with the desktop: the WM store (windows/focus/z/presentation/hydration), the app registry, the routing coordinator (URL↔WM with causes), the `OsStateClient` (snapshot fence, pending mutations, debounced applies, degraded mode), `DesktopShell` mounted in `_app` (menubar + dock + win-layer + onboarding gate), route sync-controllers for every path, window lifecycle (drag/resize via react-rnd controlled, zoom/minimize/restore/close, re-clamp), window-scoped modal integration, and the global ⌘K palette with the RuntimeSelector ⌘J rebind. First runnable shell ships with the dashboard and settings windows; this task is the program's riskiest integration surface.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST implement the `OsDesktopStore`, `OsWindow`, and `OsAppDefinition` contracts exactly as pasted in `_techspec.md` §Core Interfaces — store side-effect-free (no navigation in actions), single-instance guard in `openOrFocus` (invariant 14), z compaction on hydrate (invariant 12).
2. MUST implement the routing coordinator per §Routing Model rules 1-8 with explicit causes; route-originated causes never write history; user causes write exactly one entry; hydration precedes URL intent; keyboard activation focuses like pointer (invariant 13; UT-080/081 are the acceptance tests).
3. MUST implement `OsStateClient` per §Safety Invariants 3/6/10/15/16: as_of_seq fence, `req`-tracked pending mutations with seq-settled LWW, 250ms trailing debounce + gesture-end/beforeunload flush, apply-batches for invariant-spanning actions, snapshot-authority reconnect with degraded-keys replay, degraded mode with backoff.
4. MUST convert every route leaf to a sync-controller (`createOsRouteSync`) preserving path/validateSearch/beforeLoad/loader; unfocused window mounts warm caches via the kept `-*-preload.ts` modules.
5. MUST implement window lifecycle in floating mode with `react-rnd` controlled (bounds clamp with menubar/dock gutters; re-clamp on viewport resize — UT-065), zoom/minimize/restore/close with the minimize=unmount posture and the open-modal-dialog exemption (invariant 18, UT-086), soft-cap guidance at 12 (UT-064), and the StrictMode spike assertion recorded in the task notes (ADR-003 risk).
6. MUST mount `DesktopShell` in `_app` — menubar (workspace trigger, Session/View/Help, bell placeholder wired to task_05's aggregator seam, settings cog, logo), dock (groups per registry, `dock-new`), win-layer, wallpaper, desk hint, onboarding/workspace gate rewrite — deleting `AppShell`/`TopbarShell`/`use-app-layout`/sidebar-store rows from the Delete Targets inventory in this same change; per-window head reuses `@agh/ui` `Topbar` (48px) with a per-window `TopbarSlotProvider`.
7. MUST ship the global ⌘K palette (apps/sessions/actions via cmdk) owning the shortcut, delete `command-k-registry.ts` + `use-command-k-ownership.ts`, and rebind the RuntimeSelector to ⌘J with its visible Kbd hint updated (ADR-005).
8. MUST verify the Vite dev proxy forwards WS upgrades (`ws: true`) and consume the generated frame types from task_02 — no hand-authored wire types.
9. MUST keep every production file under 500 lines — the store, coordinator, client, registry, and each chrome composite are separate files by design.
</requirements>

## Visual Contract

Reference: `docs/design/opendesign/os/agh-os-v2.html` (current working tree, ≥960px states). Deltas #1..#11 in `_techspec.md` are the authorized-difference authority.

| ID | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| --- | --- | --- | --- | --- | --- |
| VC-01 | prototype desktop — empty state with ⌘K hint | `/` with no windows | 1440×900 | normative | First-run empty (delta #2) |
| VC-02 | prototype desktop — two overlapping windows, one focused | `/` with dashboard + settings windows | 1440×900 | normative | Head 48px (delta #1); real data (delta #4) |
| VC-03 | prototype palette overlay — open with results | ⌘K open over the desktop | 1440×900 | normative | Real apps/sessions/actions sources |
| VC-04 | prototype minimize genie → dock indicator | minimize animation end state + dock indicator | 1440×900 | normative | Reduced-motion path instant (DESIGN motion tier) |

Evidence for each row: `.compozy/tasks/os-shell/evidence/visual/task_04/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 4.1 `systems/os` skeleton per web systems architecture (lib/hooks/components/apps barrels)
- [x] 4.2 Desktop store + app registry (14 apps, paths, default rects, dock groups, matchInstance, preloads)
- [x] 4.3 Routing coordinator + route sync-controller factory; convert all route leaves; focus/close URL semantics
- [x] 4.4 `OsStateClient` with injectable socket factory; hydration/degraded/replay state machine
- [x] 4.5 Window chrome composite (rnd wrapper, head=Topbar+controls, per-window TopbarSlotProvider, error boundary rewrite) + lifecycle actions + modal exemption
- [x] 4.6 DesktopShell in `_app` (menubar/dock/win-layer/wallpaper/gate) + Delete-Targets rows for the old shell
- [x] 4.7 Palette (⌘K takeover) + RuntimeSelector ⌘J rebind + registry deletion + shortcut set (⌘N/⇧⌘S/⌘W/⌘M/Esc)
- [x] 4.8 react-rnd StrictMode spike assertion + dev-proxy WS verification
- [x] 4.9 Full assigned suite (28 UT + 9 E2E) + Visual Contract bundles

## Implementation Details

Skills: `react`, `tanstack`, `zustand`, `tailwindcss`, `eng-data-boundaries`, `eng-design` + `ui-craft` + `impeccable`, `vitest`, `eng-ui-screenshot`, `vercel-composition-patterns`. New dependency: `bun add react-rnd` (web only). The Delete Targets inventory in `_techspec.md` names exactly which files this task deletes (shell chrome group) — delete in the same change, no dual paths. Session windows are registered but their controller arrives in task_05; dashboard + settings prove the pattern here.

### Relevant Files

- `web/src/systems/os/` (new) — stores/desktop-store.ts, lib/{app-registry,routing-coordinator,os-state-client}.ts, components/{desktop-shell,os-window,dock,menubar,command-palette}.tsx, apps/{dashboard,settings}/
- `web/src/routes/__root.tsx`, `web/src/routes/_app.tsx`, every route leaf — sync-controller conversion
- `web/src/routes/_app/-app-shell.tsx`, `web/src/components/topbar-shell.tsx`, `web/src/hooks/routes/{use-topbar-shell-model,use-app-layout}.ts`, `web/src/stores/sidebar-store.ts`, `web/src/systems/runtime/components/app-sidebar.tsx`, `.../runtime-selector/command-k-registry.ts` — DELETE rows
- `web/src/systems/runtime/components/runtime-selector/` — ⌘J rebind
- `web/vite.config.*` — WS proxy verification
- `web/src/generated/agh-openapi.d.ts` — frame/DTO types (from task_02)

### Dependent Files

- `web/e2e/` — new shell fixtures/selectors (L-007: helpers ship with the contract change)
- Existing suites asserting `app-shell`/`app-grid` testids — updated here (Delete Targets)
- `systems/runtime` RuntimeSelector hosts (scoped ⌘J), `systems/workspace` stores (menubar trigger)

### Related ADRs

- [ADR-002](adrs/adr-002.md) — window↔route model + coordinator + modal amendments
- [ADR-003](adrs/adr-003.md) — react-rnd controlled + StrictMode spike + fallback
- [ADR-005](adrs/adr-005.md) — ⌘K/⌘J ownership cut
- [ADR-004](adrs/adr-004.md)/[ADR-008](adrs/adr-008.md) — client sync semantics consumed here

## Deliverables

- Runnable desktop shell replacing the old chrome (its Delete-Targets rows executed) with dashboard + settings windows, full window lifecycle, URL↔WM semantics, live+degraded sync, palette
- react-rnd StrictMode spike result recorded (pass, or fallback triggered per ADR-003)
- Visual Contract bundles VC-01..04 passing
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-040, UT-041, UT-043–UT-049 — store lifecycle, focus/z, zoom, close-successor, applyRemote seq-guard
- [x] UT-050, UT-051 — registry instance matching + path ownership
- [x] UT-052–UT-056 — OsStateClient fence/echo/debounce/reconnect/degraded
- [x] UT-057 — sync-controller URL→store reporting
- [x] UT-059, UT-060 — palette sources/filter/enter; ⌘K global + ⌘J scoped to a real RuntimeSelector form/dialog host
- [x] UT-061 — presentation derivation + rect-op gating at the breakpoint
- [x] UT-063–UT-065 — hydration drops invalid entries; soft-cap guidance; viewport re-clamp
- [x] UT-074, UT-075 — pending-mutation rebase; degraded-keys replay
- [x] UT-080, UT-081 — coordinator causes/history discipline; keyboard activation parity
- [x] UT-086 — minimize exemption with open modal dialog
- [x] E2E-001 — boot: empty desktop + hint
- [x] E2E-004 — minimize→dock indicator→restore
- [x] E2E-008 — palette open-app/jump-session; ⌘K from a session composer; ⌘J in the New session RuntimeSelector host
- [x] E2E-010, E2E-018 — two-context live convergence; simultaneous same-window edit settles
- [x] E2E-012, E2E-019 — degraded posture + recovery replay
- [x] E2E-014 — `agh desktop-state set` moves a window live (CLI→web)
- [x] E2E-017 — overlay unwinding one layer at a time
- [x] E2E-022 — menubar journey (menus, trigger, logo, cog)

## Success Criteria

- Every assigned test case implemented and passing; `bunx turbo run lint typecheck test --filter=./web` clean
- Old shell files deleted per inventory — no dual chrome, no dangling imports (typecheck is the gate)
- URL semantics hold: deep link, back/forward focus history, single history write per user action (E2E-… + UT-080 evidence)
- All four VC rows PASS with zero unresolved blocking divergence
- QA impact: shell replaces all route chrome → task_10 resets `ET-web-route-chrome-topbar` (+ any chrome-asserting scenarios) to `untested` and mints desktop/window/palette scenarios (flag recorded here)
