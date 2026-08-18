---
status: completed
title: "App ports wave: ten apps into windows"
type: frontend
complexity: high
---

# Task 6: App ports wave: ten apps into windows

## Overview

Port the ten listing/detail apps into windows — tasks, agents, loops (+loop-runs), jobs, triggers, marketplace, bridges, knowledge, sandbox, vault. Each port replaces the app's route glue with a window controller (WM-location-driven, shared zod schemas, per-window topbar slots), converts its route-backed modal flows (`/tasks/new`, `/tasks/$id/edit`) into in-window locations with deepening breadcrumbs, verifies window-scoped dialogs across the app's existing dialog call sites, and deletes its Delete-Targets rows in the same change. The `systems/*` views themselves are rehosted unchanged.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST implement one window controller per app under `systems/os/apps/<app>/`, reading `location` from the WM store, parsing search with the app's shared zod schemas (route `validateSearch` untouched), publishing topbar slots per window, and calling the kept `-*-preload.ts` on unfocused mounts.
2. MUST convert `/tasks/new` and `/tasks/$id/edit` to in-window locations (breadcrumb `agh / Tasks / New` etc.) — deep-linkable, no desktop-blocking modal (Modal & Overlay Policy §3).
3. MUST keep `<Link>`-driven cross-app navigation working through the coordinator (in-window links act on the focused window; cross-app links open/focus the target app).
4. MUST delete each app's Delete-Targets rows (route glue files + per-page hooks named in `_techspec.md` §Delete Targets) in the same change as its port — no dual paths; typecheck is the dangling-import gate.
5. MUST verify each app's existing dialogs/sheets scope to their window via the `OverlayContainerContext` (no call-site edits expected; fix any component that portals explicitly to `document.body`).
6. MUST rehost `loop-runs` under the loops app (registry `paths: ["/loops", "/loop-runs"]`) and keep detail routes (`$name`, `$id`, runs, configure/editor/run) as internal locations.
7. SHOULD update existing route/component suites in place where they assert old glue (eng-consolidate-test-suites — canonical owners keep their invariants; no standalone duplicates).
</requirements>

## Subtasks

- [x] 6.1 Tasks app (list/kanban modes, new/edit as in-window locations, detail + runs)
- [x] 6.2 Agents app (fleet, detail, settings; session paths delegate to the session app via `matchInstance`)
- [x] 6.3 Loops app (catalog, detail, configure/editor/run + loop-runs rehost)
- [x] 6.4 Jobs + Triggers apps (automation pair)
- [x] 6.5 Marketplace app (kinds, entry detail, bundles/activations)
- [x] 6.6 Bridges + Knowledge apps
- [x] 6.7 Sandbox + Vault apps (incl. `-sandbox-dialogs.tsx` REWRITE row into `systems/sandbox`)
- [x] 6.8 Per-app glue deletion sweep + suite updates + assigned E2E journeys

## Implementation Details

Skills: `react`, `tanstack`, `vercel-composition-patterns`, `eng-consolidate-test-suites`, `vitest`. The ports are mechanical by design — the per-app pattern is proven by task_04's dashboard/settings controllers; follow it. Each subtask is one app's complete vertical (controller + conversions + deletions + suite updates) so the wave can be executed incrementally with the shell always green.

### Relevant Files

- `web/src/systems/os/apps/{tasks,agents,loops,jobs,triggers,marketplace,bridges,knowledge,sandbox,vault}/` (new controllers)
- `web/src/routes/_app/-{tasks-route,tasks-new-route,tasks-edit-route,agent-detail-page,sandbox-page,sandbox-dialogs,vault-page}.tsx` + settings-shell group — DELETE/REWRITE rows
- `web/src/hooks/routes/use-{home,tasks…,agents-fleet,agent-detail,automation*,bridges,bridge-detail,knowledge,loops-catalog,loop-detail,loop-run,loop-runs,sandbox,settings*}-page.ts` — DELETE rows per inventory
- `web/src/systems/{tasks,agent,loops,automation,marketplace,bridges,knowledge,sandbox,vault}/` — rehosted views (unchanged)

### Dependent Files

- `web/src/systems/os/lib/app-registry.ts` — per-app entries (paths, rects, dock groups, preloads)
- Existing Vitest/Playwright suites asserting old glue — updated in place

### Related ADRs

- [ADR-002](adrs/adr-002.md) — window = app subtree; in-window modal-route conversion (Amendment 2)

## Deliverables

- Ten apps fully window-hosted with internal navigation and per-app glue deleted
- `/tasks/new` + `/tasks/$id/edit` as shareable in-window locations
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-058 — tasks controller search parsing falls back safely on invalid mode
- [x] E2E-002, E2E-003 — tasks window open/drag/reload persistence; resize + zoom restore
- [x] E2E-007 — tasks↔agents focus history via browser back/forward
- [x] E2E-024 — window-scoped confirm in tasks while a session window stays interactive

E2E-005 reassigned to task_08 when the window-head breadcrumb contract hard-cut (no workspace prefix).

Test-density note: per-app view behaviors keep their canonical owners (existing route/component suites, updated in place); this task's new invariants are the controller seam (UT-058 pattern), the URL journeys, and the modal scoping — each covered above. Adding per-app duplicate suites would violate the consolidation rule.

## Success Criteria

- Every assigned test case implemented and passing; turbo lint/typecheck/test clean with zero dangling imports after deletions
- Every app reachable via dock, palette, and deep link with correct breadcrumbs
- QA impact: all ten app surfaces re-chromed → task_10 resets their chrome-dependent scenarios to `untested` (e.g. `ET-web-tasks-mode-url`) and mints window-navigation scenarios; flag recorded here
