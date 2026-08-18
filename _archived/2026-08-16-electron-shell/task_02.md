---
status: completed
title: Web update surface (settings two-track + menubar indicator)
type: frontend
complexity: medium
---

# Task 2: Web update surface (settings two-track + menubar indicator)

## Overview

Delivers ADR-006's web half: the Settings → General Updates section extended to both tracks with apply/cancel actions and live operation progress, plus the discreet OS-menubar update indicator — all rendered from daemon truth with zero shell-awareness, identical in the desktop app and a plain browser. Includes the design pass that produces the two named artboards and the visual-contract evidence.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST implement `_uiux.md` S1 and S2 exactly: every enumerated state, the component plan (`SettingsUpdateTrackRow`, `MenubarUpdateIndicator`), and the signal/state mapping — zero new `@compozy/ui` primitives.
2. MUST render only the daemon projection (`GET /api/settings/update` extended payload incl. `operation.{phase,percent,holder|null,waiting}`): unknown renders as unknown; managed installs and absent app tracks show **no** apply affordance (absent, not disabled) — SD-007.
3. MUST implement apply/cancel via the new endpoints with truthful async semantics: `accepted` starts progress rendering; terminal truth comes from polling GET (existing cadence at rest, 2s while `operation` non-null); `blocked` surfaces the holder without optimistic success.
4. MUST keep the SPA shell-agnostic — no desktop detection, no bridge, no user-agent branching.
5. MUST render the exact phase names from the Part II phase→UI mapping (`download`/`verify`/`install`/`start`/`ready-check`/`ready`) and per-track version/release-link/recommendation/last-error/rollback truth.
6. MUST place the indicator in the OS-menubar composite per OS-shell chrome grammar (DESIGN.md §5 carve-out): hidden unless an update is available, no counts, no pulse, keyboard-activatable, navigates to the Updates section.
7. MUST run the design pass through the `designer` agent (execution mode) producing `docs/design/opendesign/electron-shell/electron-shell-settings-updates.html` and `electron-shell-menubar-update.html` before implementation, and verify every Visual Contract row with the durable `eng-ui-screenshot` bundle.
8. MUST extend the canonical suites (`-general.test.tsx` family, os-menubar suite) rather than creating parallel test files.
9. MUST pass repo-root gates: `make bun-lint`, `bunx turbo run typecheck --filter=./web`, `bunx turbo run test --filter=./web` (never package-local runs as evidence).
</requirements>

## Visual Contract

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | -------- | ---------------------------------- |
| VC-01 | `docs/design/opendesign/electron-shell/electron-shell-settings-updates.html` — checking | `/settings` General → Updates — checking fixture | 1440×900 | normative | None |
| VC-02 | same — up-to-date (both tracks) | `/settings` — both current fixture | 1440×900 | normative | None |
| VC-03 | same — available (runtime only) | `/settings` — runtime-available fixture | 1440×900 | normative | None |
| VC-04 | same — available (both tracks) | `/settings` — both-available fixture | 1440×900 | normative | None |
| VC-05 | same — managed runtime (recommendation, no affordance) | `/settings` — homebrew fixture | 1440×900 | normative | None |
| VC-06 | same — no app installed (single-track) | `/settings` — app:null fixture | 1440×900 | normative | None |
| VC-07 | same — applying (runtime, staged phases + percent) | `/settings` — operation-active fixture | 1440×900 | normative | Live percent value varies; phase names fixed (runtime truth) |
| VC-08 | same — app staged (next-launch copy) | `/settings` — staged fixture | 1440×900 | normative | None |
| VC-09 | same — blocked (holder named) | `/settings` — blocked fixture | 1440×900 | normative | None |
| VC-10 | same — failed + rollback truth (restored version, last error) | `/settings` — rolled-back fixture | 1440×900 | normative | None |
| VC-11 | same — refresh error (last-known + retry) | `/settings` — refresh-failed fixture | 1440×900 | normative | None |
| VC-12 | `docs/design/opendesign/electron-shell/electron-shell-menubar-update.html` — hidden (no update) | OS menubar — idle fixture | 1440×900 | normative | Indicator absent from DOM, not hidden via CSS |
| VC-13 | same — available | OS menubar — update-available fixture | 1440×900 | normative | None |
| VC-14 | same — focus-visible (keyboard) | OS menubar — focused indicator | 1440×900 | normative | None |
| VC-15 | same — suppressed while applying/failed | OS menubar — operation-active fixture | 1440×900 | normative | Indicator absent; progress lives only in S1 |

Evidence for each row: `.compozy/tasks/electron-shell/evidence/visual/task_02/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 2.1 Design pass (designer agent, execution mode): produce the two artboards from `_uiux.md` states + DESIGN.md grammar
- [x] 2.2 Extend settings system: adapter methods + query/mutation hooks for apply/cancel + operation-aware polling cadence
- [x] 2.3 `SettingsUpdateTrackRow` + two-track `GeneralUpdateSection` rework (all VC-01..VC-11 states)
- [x] 2.4 `MenubarUpdateIndicator` in the os-menubar composite (VC-12..VC-15) + navigation to the section
- [x] 2.5 Update `settingsUpdateView` presentation logic for the extended payload (nullable operation/holder/app)
- [x] 2.6 Extend canonical Vitest suites (section states, indicator visibility, keyboard) — UT-040..UT-052
- [x] 2.7 Browser E2E journeys (E2E-019..E2E-022) in the daemon-served Playwright lane
- [x] 2.8 Visual-contract verification: `eng-ui-screenshot` bundles for VC-01..VC-15 completed in the QA/PR tail
- [x] 2.9 Storybook stories for the new composites (story + test per web conventions)

## Implementation Details

Data source and states: `_spec.md` Part II API Endpoints + phase→UI mapping; `_uiux.md` is the surface inventory. Mutations follow the systems architecture (adapter → query-options → hooks → components; canonical keys; envelope snapshots).

Skills to activate: `react`, `tailwindcss`, `tanstack`, `shadcn`, `state-management`, `eng-design`, `ui-craft`, `impeccable`, `eng-ui-screenshot`, `storybook-stories`, `eng-consolidate-test-suites`, `vitest`, `testing-boss`.

### Relevant Files

- `web/src/routes/_app/settings/-general-update-section.tsx` — today's single-track section (S1 home)
- `web/src/systems/settings/lib/update-presentation.ts` — `settingsUpdateView` state logic to extend
- `web/src/systems/settings/` (adapters, `lib/query-options.ts`, hooks, `index.ts`) — apply/cancel mutations + polling cadence
- `web/src/systems/os/components/menubar/os-menubar.tsx` — S2 host composite
- `web/src/systems/os/components/os-hydration-status.tsx` — nearest existing truthful-glyph pattern for the indicator
- `web/src/routes/_app/-settings-preload.ts` — settings query preload wiring
- `web/src/generated/compozy-openapi.d.ts` — regenerated types from task_01
- `packages/ui/src/index.ts` — primitive inventory (reuse before create; `Pill`, `Button`, `Spinner`)

### Dependent Files

- `web/src/routes/_app/settings/__tests__/-general.test.tsx` — canonical suite to extend
- os-menubar test suite under `web/src/systems/os/` — indicator coverage
- `web/e2e/` daemon-served Playwright suite — E2E-019..022 journeys

### `.resources/` References

- `.resources/t3code/apps/web/src/components/sidebar/SidebarUpdatePill.tsx` + `apps/web/src/state/desktopUpdate.ts` — update-state UI + async state-stream pattern (theirs is desktop-bridge-fed; ours is daemon-query-fed — the state taxonomy and pill/action ergonomics are the useful reference)
- `.resources/synara/apps/web/src/components/Sidebar.tsx` (update section, ~lines 5291-5623) — check/download/install action cluster + manual-fallback ergonomics

### Related ADRs

- [ADR-006](adrs/adr-006.md) — update offers in the web UI; daemon truth; menubar indicator; overlay reduction
- [ADR-009](adrs/adr-009.md) — the operation lifecycle this UI renders

### Web/Docs Impact

- `web/`: the files above (settings system + os menubar + stories + suites) — this task is the web change.
- `packages/site`: none in this task — checked `content/docs/`: no settings-UI doc page exists for updates today; the desktop/docs truth pass (task_05) covers user-facing docs.
- QA impact: add content-addressed `untested` scenarios — web update section two-track behavior and menubar indicator lifecycle (browser-first entry points) — for the task_07 walk.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked; pure web rendering of daemon truth, no extension surface.
- Agent manageability: none new — the UI consumes the agent-visible endpoints task_01 ships; no UI-only capability is introduced (SD-007 keeps parity).
- Config lifecycle: none — no config keys touched; polling cadence is code-defined per spec Assumptions.

## Deliverables

- Two artboards under `docs/design/opendesign/electron-shell/` matching `_uiux.md`
- Two-track Updates section + menubar indicator live in the SPA, browser and app identical
- Every Visual Contract row with a durable passing `eng-ui-screenshot` evidence bundle **(REQUIRED)**
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-040–UT-048 — S1 section states (two-track, managed, staged, blocked, rollback, refresh-error, keyboard)
- [x] UT-049–UT-052 — S2 indicator visibility/suppression/navigation/focus
- [x] E2E-019–E2E-022 — browser journeys (states, apply from browser, indicator lifecycle, keyboard-only) — authored; execution deferred with the QA tail

## Success Criteria

- Every assigned test case implemented and passing
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
- `make bun-lint` + turbo typecheck/test (repo root) green
- Apply affordances provably absent (not disabled) for managed/absent tracks (DOM assertions)
