---
status: completed
title: Frontend: semantic identity, frame rendering, deck UI, strip migration
type: frontend
complexity: critical
---

# Task 3: Frontend: semantic identity, frame rendering, deck UI, strip migration

## Overview

The complete web slice, in internal order: adopt the v3 generated types; replace the `osWindowId` singleton scheme with semantic `(app, instanceKey)` lookup everywhere (routing coordinator, dock with multi-instance cycling, palette, deep links); add the tab command layer and group projection in their own files; refactor stack rendering to frame-based groups; finalize the `packages/ui` context-menu primitive; build the deck UI with gestures, menus, shortcuts, ⌘K tab search, and the new-tab app; and land the global D3 strip migration with the design-system document amendments. Ships against the reference contract `docs/design/opendesign/design-system/window-tabs-variations.html`.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — fix root causes and preserve the accepted contract
</critical>

<requirements>
1. MUST delete `osWindowId` and every derivation call site; window resolution becomes semantic lookup with MRU ordering from `FocusOrder` (ADR-010); dock drops the instance-key skip and aggregates/cycles per app; sessions dedup on `instanceKey === sessionId`; deep links land on the MRU instance.
2. MUST add `runtime/window-manager-tab-commands.ts` (builders/dispatch for group/reorder/activate/pin/reopen/close-scopes/navigate-modes/stack-target open) and `lib/group-projection.ts` (frame/deck projection) — existing runtime/projection files must not grow (500-line cap, TechSpec split plan).
3. MUST refactor to frame-based rendering: one frame per group (tiled pane or floating) hosting the deck at ≥2 members, the active member's head/strip/body, and inactive members kept mounted but hidden inside the frame; delete per-member `display:none` hiding, `disableDragging` for inactive members, and `preferredActiveWindow`.
4. MUST implement the deck per the reference contract: 37px row, traffic lights (close = scope "group"), tab segments (glyph/state-dot + derived leaf label, hover ×, middle-click close, ⌥-click close-others, attention badge, pinned glyph-only form, tooltips with full path), + button, drag region; 96px floor then horizontal scroll; active tab fuses with the head; deck absent at 1 member with controls back in the head.
5. MUST implement gestures with visible, cancelable affordances: in-deck drag reorder (`window.stack.reorder`), drag-out to standalone (`toggle_floating` with pointer rect), drag-merge with insertion indicator committing ONE `window.stack.group{insert_index}`; Escape/pointercancel commit nothing.
6. MUST finalize the `packages/ui` context-menu primitive first (exported from the split barrel, story, test), then ship the tab context menu (close/close others/close right/pin/move to new window/merge all) and launch-surface destination menus ("Open in new window" / "Open as tab in focused window" / "Open new instance" / "Go to tab"), keyboard-reachable.
7. MUST register tab shortcuts (`window.tab.new|next|previous|last|reopen|jump.1..8`) in the TS registry mirroring the Go allow-list, add the palette "Go to tab" group (cross-desktop, attention state, one-action jump) and tab actions, and add the `new-tab` pseudo-app with its stub route and picker empty state.
8. MUST classify navigations at intent points: drill-in links → `push`, breadcrumb/⌘[ → `pop`, everything else → `replace` (route-pop reconciliation stays `replace`).
9. MUST land D3 globally: delete the Topbar `nav` slot (type + zone), publish views as the toolbar's leading pill-group in tasks catalog + marketplace (order: views · filters · spacer · display-mode), and amend `os-shell.html` §02/§07 (S2/S3) + `pagehead-redesign.html` §05 route-nav row.
10. MUST keep the deck accent-discipline (state/attention only; neutral selection) and provide the Visual Contract evidence bundles for every row below.
</requirements>

## Visual Contract

Reference: `docs/design/opendesign/design-system/window-tabs-variations.html` (the chosen contract). Brand/content axes stay lossy per L-032 — runtime truth and `COPY.md` own labels/data; divergences recorded as authorized deltas.

| ID    | Reference artifact + state                                                       | Implementation target + state                                              | Viewport | Fidelity  | Authorized differences + authority                             |
| ----- | -------------------------------------------------------------------------------- | -------------------------------------------------------------------------- | -------- | --------- | -------------------------------------------------------------- |
| VC-01 | `window-tabs-variations.html` — `demo-deck-main`, drilled tab active (breadcrumb head) | OS shell frame: 3-tab group, drilled tab active                            | 1440×900 | normative | Demo copy/data → runtime fixtures (L-032; COPY.md); signal colors use current `DESIGN.md` tokens; actions follow `TaskRunTopbar` runtime truth |
| VC-02 | `window-tabs-variations.html` — `demo-deck-main`, root Tasks tab active (strip: views·filters·Rows\|Cards) | Same group, Tasks tab active with toolbar strip                            | 1440×900 | normative | Filter counts and available actions follow runtime fixtures; signal colors use current `DESIGN.md` tokens |
| VC-03 | `window-tabs-variations.html` — `demo-deck-main`, session document tab active     | Same group, session tab active (document head, composer)                   | 1440×900 | normative | Session identity/status/actions follow runtime fixtures; signal colors use current `DESIGN.md` tokens |
| VC-04 | `window-tabs-variations.html` — `demo-deck-density` (7 tabs, badge, mixed dots)   | 7-tab group with needs-input badge + running dots, compressed segments     | 1440×900 | normative | Signal colors resolve through current `DESIGN.md` tokens; no structural delta |
| VC-05 | `window-tabs-variations.html` — single-tab contract (D1: deck unmounted, controls in head) | Same window reduced to one tab — today's chrome                            | 1440×900 | normative | Available actions follow runtime truth; signal colors resolve through current `DESIGN.md` tokens; no structural delta |
| VC-06 | Reference D5/ADR-006 pinned form (glyph-only, left-collated)                      | Group with 2 pinned + 3 unpinned tabs                                      | 1440×900 | normative | Pinned form follows spec text (no reference demo exists — spec authority: `_techspec.md` + ADR-006) |

Evidence for each row: `.compozy/tasks/window-tabs/evidence/visual/task_03/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}` via `eng-ui-screenshot`.

## Subtasks

- [x] 3.1 Adopt v3 generated types; update stream/schema mirrors and the command-ID union
- [x] 3.2 Replace `osWindowId` with semantic lookup across coordinator/dock/palette/session dedup/deep links; dock multi-instance cycling
- [x] 3.3 Add `window-manager-tab-commands.ts` + runtime controller methods for every tab command
- [x] 3.4 Add `group-projection.ts`; refactor `os-win-layer`/`os-window`/`use-os-window` to frame-based groups
- [x] 3.5 Finalize `packages/ui` context-menu (barrel split, story, test)
- [x] 3.6 Build `os-window-deck` + `os-window-tab` with states, badges, pins, tooltips, overflow
- [x] 3.7 Implement deck gestures (reorder / drag-out / drag-merge with insertion affordance)
- [x] 3.8 Ship tab + launch-surface context menus with destination actions
- [x] 3.9 Register tab shortcuts + palette Go-to-tab group + new-tab app and stub route
- [x] 3.10 Wire navigate-mode classification at link/back intent points
- [x] 3.11 D3 strip migration: Topbar nav slot removal, tasks/marketplace strip views, DS doc amendments
- [x] 3.12 Implement all 64 assigned test cases + capture the 6 Visual Contract bundles

## Implementation Details

Follow TechSpec §Web Design, §Architectural Boundaries (file split), ADR-002/004/005/006/007/010/011. Exact current shapes in the transcript exploration report: `os-types.ts` (:19-191), `routing-coordinator.ts` (:13-341), `window-manager-runtime{,-core}.ts`, `layout-projection.ts:167-172`, `os-window.tsx:73-84`, `use-os-window.ts:110-340`, `use-desktop-dock.ts:44`, `os-dock.tsx:62-95`, `window-manager-command-registry.ts:79-169`, `use-os-shortcuts.ts`, `os-command-palette.tsx`, `app-registry.tsx:24-327`, `sessions-modal.tsx:261-271`, `topbar.tsx` + `use-topbar-slot.ts`, `packages/ui/src/index.ts` (495 lines — split), `context-menu.tsx` (untracked, unexported).

### Relevant Files

- `web/src/systems/os/lib/os-types.ts` — delete `osWindowId`; OsWindow group fields
- `web/src/systems/os/lib/routing-coordinator.ts` — semantic resolution + navigate-mode classification
- `web/src/systems/os/runtime/window-manager-runtime.ts`, `window-manager-runtime-core.ts` — controller methods (no growth); new `window-manager-tab-commands.ts`
- `web/src/systems/os/lib/layout-projection.ts` — extract group logic to new `group-projection.ts`; delete `preferredActiveWindow`
- `web/src/systems/os/components/os-win-layer.tsx`, `os-window.tsx`, `os-window-frame.tsx`; `hooks/use-os-window.ts` — frame-based rendering
- New: `web/src/systems/os/components/os-window-deck.tsx`, `os-window-tab.tsx`
- `web/src/systems/os/hooks/use-desktop-dock.ts`, `components/os-dock.tsx` — per-app aggregation + cycling
- `web/src/systems/os/lib/window-manager-command-registry.ts`, `window-manager-action-dispatch.ts`, `hooks/use-os-shortcuts.ts` — tab actions
- `web/src/systems/os/components/os-command-palette.tsx`, `hooks/use-os-command-palette.ts` — Go-to-tab group + actions
- `web/src/systems/os/lib/app-registry.tsx` + new-tab controller + `web/src/routes` stub — pseudo-app
- `packages/ui/src/components/custom/topbar.tsx`, `components/custom/hooks/use-topbar-slot.ts` — nav slot deletion
- `packages/ui/src/components/context-menu.tsx`, `packages/ui/src/index.ts` (+ new `exports/*.ts`) — primitive finalization + barrel split
- `web/src/systems/os/apps/tasks/tasks-catalog-location.tsx`, `web/src/systems/marketplace/components/marketplace-kind-page.tsx` — strip views
- `docs/design/opendesign/design-system/os-shell.html`, `docs/design/opendesign/os/pagehead-redesign.html` — DS amendments

### Dependent Files

- `web/src/systems/os/**/__tests__/*` — canonical suites (runtime, coordinator, store, projection, dock, palette, shortcuts, components) extended in place
- `web/e2e/__tests__/os-shell.spec.ts` + `web/e2e/fixtures/selectors.ts` — deck journeys + shared helpers (L-007: helpers ship with the contract change)
- `web/src/systems/os/lib/window-manager-types.ts`, `window-manager-stream-schema` / `config-schema` tests — v3 mirrors
- `packages/ui` stories/tests for context-menu and Topbar

### Related ADRs

- [ADR-002](adrs/adr-002.md), [ADR-004](adrs/adr-004.md), [ADR-005](adrs/adr-005.md), [ADR-006](adrs/adr-006.md), [ADR-007](adrs/adr-007.md), [ADR-010](adrs/adr-010.md), [ADR-011](adrs/adr-011.md)

### Web/Docs Impact

- `web/`: everything under "Relevant Files" above; generated types consumed from task_02. Skills per `web/CLAUDE.md` dispatch: `react`, `tailwindcss`, `tanstack`, `state-management`+`xstate-store`, `eng-design`+`ui-craft`+`impeccable`, `eng-ui-screenshot`, `storybook-stories`, `vitest`.
- `packages/site`: none — CLI/config docs owned by task_02 (generated) and task_04 (reference). Checked: no site content depends on web internals.
- QA impact: broad user-visible change. Scenario resets owned by task_05: `ET-web-window-routing-lifecycle`, `ET-window-manager-layout-gestures`, `ET-window-manager-drop-swap`, `ET-window-manager-multi-client`, `ET-web-desktop-shell-lifecycle`, `ET-web-dock-default-window-size`, `RT-desktop-pager-overview` + new tab scenarios — walked in task_06.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: no hooks/tools/manifests/bundles/registries/bridge SDK/MCP changes originate in web; web consumes the task_02 contract only.
- Agent manageability: none direct — the web is the operator surface; agent paths (CLI/HTTP/UDS/tools) land in task_02. Web must render agent-driven changes live (covered by UT-045/IT-006-driven behaviors and E2E-011).
- Config lifecycle: consumes `[window_manager].shortcuts` overrides for the new tab actions via the existing settings surface; no new keys. Checked: `web/src/systems/settings/components/layouts/*` picks up new actions from the registry without schema changes.

## Deliverables

- Deck UI matching the reference contract with all 6 Visual Contract bundles passing
- Multi-instance identity live across dock/palette/deep links with zero `osWindowId` references left (compile-verified)
- Frame-based rendering with state-preserving hidden members and per-pane decks
- D3 strip migration + amended DS documents
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**
- Every Visual Contract row has a durable passing evidence bundle **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-040..UT-045 — semantic lookup, instances, dock cycling, deep links, attention survival
- [x] UT-050..UT-052 — tab shortcuts + rebinding + single-tab degradation
- [x] UT-070..UT-092 — deck mount/controls, head swap, labels/tooltips/overflow, ⌘T/new-tab/picker, context menus, bulk close dispatch, gone-states, go-to-tab depth, no-rename
- [x] UT-093..UT-102 — drag-out/interrupt, drag-merge insertion, merge-all/move menu, state rendering, flapping, editable-guard, palette results, pane decks
- [x] UT-170, UT-171 — context-menu primitive; Topbar nav slot removal + strip order
- [x] UT-130, UT-131 — strip views on tasks/marketplace; absent-views behavior
- [x] UT-178 — group projection at 200-window volume
- [x] E2E-001..E2E-016 — all browser journeys (merge/drag, deck lifecycle, per-tab heads, overflow, keyboard, context-menu open-as-tab, close/reopen after reload, pins, per-tab drill, tear-out, attention routing with agent close, merge-all/split, multi-instance, cross-desktop jump, restart restore, two-client independence)
- [x] E2E-018 — strip migration screenshot-verified

## Success Criteria

- Every assigned test case implemented and passing; `make bun-lint`/`bun-typecheck`/`bun-test` green via turbo; e2e suite green
- All 6 Visual Contract rows `PASS` with zero unresolved blocking divergence
- `rg "osWindowId" web/ packages/` returns zero hits; no production file over 500 lines
- Deck renders only at ≥2 members; accent appears only for state/attention (screenshot evidence)

## Completion Notes

- The 64 assigned contracts are implemented in their canonical web/UI suites; the official
  `make test-e2e-web` lane passed all 131 physical browser cases.
- Visual Contracts VC-01..VC-06 are `PASS` with durable bundles under
  `evidence/visual/task_03/`; no blocking divergence remains.
- `make bun-lint`, `make bun-typecheck`, `make bun-test`, `make source-size`, and the
  automatically escalated full `make gate` passed. `osWindowId` has zero matches in `web/` and
  `packages/`.
- QA impact remains flagged for Tasks 05–06; those tasks own scenario resets, real-user walks, and
  final teardown evidence.

Compozy Impact Audit:

- Native tools: no tool IDs, descriptors, schemas, digests, risk flags, or capability gates changed;
  checked the generated task_02 contract consumed by the web.
- Extensibility and hooks: no extension, hook, capability, bundle, registry, bridge SDK, MCP
  sidecar, or config-lifecycle surface changed in this frontend slice.
- Workspace data isolation: frame, slot, palette, and drag state remain client presentation state;
  workspace/session reads, SSE projection, and cache ownership are unchanged and covered by the
  two-client and cross-workspace browser journeys.
- Official Compozy skill: no direct impact in this slice; Task 04 owns the required public-contract
  documentation.
