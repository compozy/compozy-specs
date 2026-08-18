---
status: completed
title: Network app port
type: frontend
complexity: high
---

# Task 7: Network app port

## Overview

Port the network domain — the app most deeply coupled to the router (`$workspaceId` in every path, nested channel/threads/directs routes, the route-driven thread overlay, its own `ResizablePanelGroup` shell) — into a single network window with internal navigation. This is deliberately isolated so its complexity never blocks the shell or the other ports.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST host the full network subtree (`/network`, `/$workspaceId/$channel/{activity,directs,threads}` + thread/direct details) as internal locations of one network window, breadcrumb deepening accordingly; the `$workspaceId` segment always follows the active workspace (Routing Model rule 8).
2. MUST rewrite the network-specific route hooks (`use-network-route-shell.ts`, `use-network-channel-threads-route.ts` and peers using `useParams`/`useChildMatches strict:false`) as window-controller logic reading WM location — the one domain where coupling lives inside `systems/`, so the controller boundary moves into `systems/network` deliberately and is documented in the barrel.
3. MUST keep the thread overlay as an in-window overlay (route-driven location), the channel rail + `RightRail` + resizable layout intact, and every network dialog window-scoped via the inherited portal context.
4. MUST preserve the network shell's persisted split layouts (`useDefaultLayout`/`LayoutStorage`) — they are per-window content state, not WM state; no migration.
5. MUST update the network suites in place (canonical owners) for the controller seam; no standalone duplicates (eng-consolidate-test-suites).
6. MUST delete the network glue rows from the Delete Targets inventory in the same change.
</requirements>

## Subtasks

- [x] 7.1 Network app registry entry + window controller shell (location parsing for the nested tree)
- [x] 7.2 Route-hook rewrite inside `systems/network` (params/child-match reads → WM location)
- [x] 7.3 Thread/direct overlays + composer flows verified in-window; dialogs scoped
- [x] 7.4 Workspace-switch behavior (window content follows the active workspace's channels)
- [x] 7.5 Suite updates in place + glue deletion + `make verify`-lane checks

## Implementation Details

Skills: `react`, `tanstack`, `eng-data-boundaries`, `eng-consolidate-test-suites`. Reference the explore-web finding: network is the only system with router hooks inside `systems/` — the port moves that coupling to the controller seam rather than pretending it doesn't exist. The network shell (`network-shell.tsx`) renders unchanged inside the window body.

### Relevant Files

- `web/src/systems/os/apps/network/` (new controller)
- `web/src/systems/network/components/shell/network-shell.tsx` + `thread-overlay/` — rehosted
- `web/src/systems/network/hooks/use-network-route-shell.ts`, `use-network-channel-threads-route.ts` + sibling route hooks — rewritten to WM-location inputs
- `web/src/routes/_app/network*` leaves — sync-controller conversion (pattern from task_04)

### Dependent Files

- `web/src/systems/os/lib/app-registry.ts` — network entry
- Existing network Vitest/Playwright suites — updated in place

### Related ADRs

- [ADR-002](adrs/adr-002.md) — window = app subtree (network is its stress test)

## Deliverables

- Network fully window-hosted with nested navigation, overlays, and split layouts intact
- Network route-hook coupling rewritten at the controller seam; glue rows deleted
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

No new contract IDs are assigned to this task by design: the network domain's behaviors keep their canonical owners — the existing network route/hook/component suites — which this task updates **in place** for the controller seam (eng-consolidate-test-suites; peer-review B-010 density note). The invariant inventory for the port: nested-location parsing, thread-overlay open/close via location, workspace-follow behavior, and dialog scoping — each must land as an updated assertion in the owning existing suite, enumerated in the completion notes. Cross-shell journeys that traverse network windows are exercised by task_09's E2E-016 and the QA pair.

## Success Criteria

- Every updated network suite passing; turbo lint/typecheck/test clean; no dangling imports after deletion
- Thread deep links open the network window at the exact thread; browser back steps the overlay closed
- Completion notes enumerate the in-place suite updates per invariant (the no-new-IDs rationale made concrete)
- QA impact: network chrome re-hosted → task_10 resets network navigation scenarios to `untested`; flag recorded here

## Completion Notes

- The Network registry now loads one WM-driven controller for `/network` and every nested channel, tab, thread, and direct location; route leaves are sync-controllers only.
- `use-network-route-shell.ts` and its peers consume canonical window-location inputs. No `useParams` or `useChildMatches` reads remain under `systems/network`.
- Canonical suite updates by invariant:
  - Nested-location parsing and workspace follow: `use-network-route-shell.test.tsx`, including stale-workspace rejection, root deepening, and background-window focus protection.
  - Thread-overlay close behavior: `thread-overlay.test.tsx` and `thread-overlay-header.test.tsx`, now asserting controller-owned location transitions.
  - Window-scoped dialogs: `network-create-channel-dialog.test.tsx`, plus the shared `Dialog` and `Sheet` owner-container suites.
- The existing `NetworkShell`, `RightRail`, and `LayoutStorage` composition remains the content owner; no layout-state migration was introduced.
- Verification: React Doctor 100/100; Web/UI lint, typecheck, and tests pass (412 Web files / 3,286 tests); final `make verify` passes.
- Visual evidence: `.compozy/tasks/os-shell/evidence/visual/task_07/` contains refreshed 1100×700 default and right-rail captures; the owned Storybook process was torn down and port 6006 is closed.
- QA tracker impact is intentionally deferred to Task 09, which owns scenario creation/reset for the re-hosted Network chrome.
