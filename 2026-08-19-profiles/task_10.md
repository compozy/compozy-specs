---
status: pending
title: Per-Profile State and Polish (Desktops + A11y)
type: frontend
complexity: medium
---

# Task 10: Per-Profile State and Polish (Desktops + A11y)

## Overview

Closes phase 6: window desktops partition by `(workspace_id, profile_id)` through the existing clientstate aggregate (daemon repository keying + web window-manager store), the cross-surface accessibility sweep over every profile surface (S1–S13), and the remaining browser journeys (two independent clients, desktop restoration, keyboard-only SymbolPicker). Extends the IT-038 delete fixture with clientstate desktop partitions.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

**Skills**: `eng-design`, `ui-craft`, `impeccable`, `better-accessibility`; `golang-master`+`eng-code-guidelines` for the clientstate keying change. Read `web/CLAUDE.md` first.

<requirements>
- MUST partition the window-manager clientstate aggregate by `(workspace_id, profile_id)` — a partitioning of the existing per-workspace snapshot document, not a globaldb schema change (path-map correction 11).
- MUST restore each profile's desktops/windows on switch; a new profile starts clean; archived profiles retain layouts and restore on unarchive; no cross-contamination in either direction (US-026).
- MUST purge a deleted profile's desktop partitions in the delete flow and count them in `RemovalSummary.DesktopPartitions` — extending the IT-038 fixture (cumulative fixture contract in `_tasks.md`).
- MUST run the cross-surface a11y sweep over S1–S13: switcher arrow navigation, picker keyboard search/select, dialog focus traps, text-not-color-alone signals, AA contrast on identity colors — fixing violations, not documenting them.
- MUST keep the S12 behavior artboard-free (no new visual language) — behavior only, per `_uiux.md`.
- MUST NOT leak view state across clients: each client's active view stays ephemeral (ADR-014), proven by the two-context journey.
</requirements>

## Subtasks

- [ ] 10.1 Daemon: window-manager repository/boot keying gains the profile dimension; snapshot load/seed branches per profile; delete-flow purge + removal count.
- [ ] 10.2 Web: window-manager store consumes the partitioned key; switch restores the target profile's desktops; first-entry clean state.
- [ ] 10.3 IT-038 fixture extension (desktop partitions in plan + applied removals, field-for-field).
- [ ] 10.4 A11y sweep across S1–S13 with fixes (keyboard, focus, labels, contrast).
- [ ] 10.5 Remaining Playwright journeys (two clients, desktop restoration, keyboard-only picker).

## Implementation Details

Desktops persist through clientstate (domain `window_manager`, one aggregate per workspace today); the change is the aggregate key, the snapshot seed path, and the purge hook.

### Relevant Files

- `internal/daemon/window_manager_repository.go:17-28` + `window_manager_boot.go:31-40` — clientstate persistence + boot seed.
- `internal/windowmanager/service.go:272-300` — `loadSnapshot`/`initialSnapshot` (`desktop-default` seeding branch point).
- `internal/clientstate/contract.go:1-93` — aggregate contract + purge.
- `web/src/systems/os/stores/window-manager-store.ts:122-160` — client view state consuming the partition.
- `web/src/systems/profiles/` (task_05) — switch action triggering restoration.

### Dependent Files

- `internal/profile` delete finalize — desktop purge step + `RemovalSummary.DesktopPartitions` population.
- Playwright fixtures for multi-context journeys.

### Related ADRs

- [ADR-014](adrs/adr-014.md) — per-client ephemeral active view vs shared remembered choice (E2E-020's contract).
- [ADR-001](adrs/adr-001.md) — archived profiles retain state; delete removes it (enumerated).

## Deliverables

- Desktops isolated, restored, purged-on-delete per profile.
- A11y sweep completed with fixes across every profile surface.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] IT-057 — snapshots keyed `(workspace, profile)`: isolated, restored on switch, archived retained, new profile clean.
- [ ] E2E-020 — two browser contexts hold independent active profiles; only the remembered choice is shared after explicit switch.
- [ ] E2E-024 — arrange in two profiles, switch back and forth → exact restoration; new profile clean.
- [ ] E2E-025 — SymbolPicker keyboard-only journey with a11y assertions (focus trap, labels).

### Web/Docs Impact

- `web/`: `web/src/systems/os/stores/window-manager-store.ts` + profile-switch wiring; a11y fixes across `web/src/systems/profiles/` + touched listing surfaces.
- `packages/site`: none — desktops behavior is documented in the task_04/06 profiles pages' scope statements (checked: no new public contract).
- QA impact: new scenarios — add a content-addressed untested file for per-profile desktop restoration; reset the window-management scenario(s) touching desktop persistence. Walk owned by task_13.
