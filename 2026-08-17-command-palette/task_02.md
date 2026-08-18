---
status: pending
title: "P1 Web — Registry Projection, Dispatch Seam, Core Absorption"
type: frontend
complexity: high
---

# Task 2: P1 Web — Registry Projection, Dispatch Seam, Core Absorption

## Overview

Rewrites the web palette from four hand-written JSX groups + a 343-line god hook into one registry-driven projection: SWR hydration keyed by `catalog_revision` with an IndexedDB **structural-only** cache, a client context evaluator over daemon-declared predicates, and a single dispatch seam (client op vs daemon invoke). Menubar, cheatsheet, and the settings shortcut table become projections of the same client registry, and every legacy delete target falls in this change — no shims, no dual paths.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting; read every ADR in `adrs/` and any `memory/task_NN.md` present
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — fix root causes; frozen `_dx.md`/Part II contracts are binding; surface gaps as clarifications, never invent
</critical>

<requirements>
1. MUST render every surface (palette root, destination mode, menubar items, cheatsheet, settings shortcut table listing) from ONE client-side registry projection — no surface-local command arrays survive (`_uiux.md` Component plan rule).
2. MUST hydrate stale-while-revalidate: last-known catalog renders instantly from the IndexedDB record `cmd_palette.catalog` keyed `(canonical workspace id, contract version)` holding STRUCTURAL state only (descriptors/sources/bindings/aliases — never resolved availability, never rank signals); availability re-evaluates against the current client's context on hydration; corrupt/version-mismatched records drop with a full refetch (BR-19, Key Decisions).
3. MUST implement the context evaluator over the closed context-key set v1, evaluating daemon-declared predicates against the client's volatile snapshot; with no daemon context it never defaults to allow-all (US-008.EC-3); availability flaps debounce (US-037.EC-2).
4. MUST implement one dispatch seam: `client_op` → window coordinators/overlay actions; `tool` → POST invoke; `navigate` → app route; `url` → sanctioned opener; results feed the shared feedback path; usage reporting is fire-and-forget POST `/usage` (recording behavior itself lands with task_03's store).
5. MUST derive menubar items (six menus) from the registry via `MenubarCommandItem` — label, chord badge, availability + verbatim reason, same dispatch; grouping/order stays hand-curated (BR-17); items never vanish mid-session (US-035.EC-1).
6. MUST re-source the settings shortcut table to list the whole registry with source filters (the derivation only — alias cell and mutation flows land in task_05) and extend the cheatsheet derivation to registry ids; zero chord literals in TS (BR-6).
7. MUST enforce session-landing single-path (BR-20): entity/session rows route through the shared attention-jump semantics; the root `jumpToSession` direct `coordinator.userOpen` divergence is deleted.
8. MUST execute every P1 delete target in this same change: `os-command-palette-{results,window-actions,shell-actions,views}.tsx`, `use-os-command-palette.ts`, `window-manager-command-registry.ts`, `window-manager-action-dispatch.ts` (keyboard path re-routes through the seam), the `PALETTE_VIEW_FRAMES` closed map / `PaletteViewId` union (registry-driven view resolution — Sessions view keeps working), and the third keymap copy `web/src/storybook/window-manager-shortcut-fixtures.ts`.
9. MUST land the `packages/ui` Command custom-filter tests/stories (`shouldFilter={false}` + external ordering + selection survival) BEFORE the projection swap (Known Risks / arch risk 9).
10. MUST consume `GET /api/cmd-palette/stream` only through `createStreamEventSource` with a NAMED event listener (L-017); one event → one refetch converging on the revision (UT-104).
11. MUST update the canonical suites in place (`use-os-command-palette.test.tsx` lineage, `os-interaction-hooks.test.tsx`, palette stories, `os-shell.spec.ts`) and promote the inline palette selectors into `web/e2e/fixtures/os-navigation.ts` — never duplicate suites.
12. MUST block and surface if a cited artboard is missing at execution time — never improvise the visual reference.
</requirements>

## Visual Contract

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | -------- | ---------------------------------- |
| VC-01 | `docs/design/opendesign/command-palette/command-palette-root.html` — first-run curated rest state | ⌘K overlay on `/` — fresh workspace fixture, empty query | 1440×900 | normative | Pins/Recents groups absent (land in task_03 — build order P2); fixture data replaces mock copy (SD-007) |
| VC-02 | `command-palette-root.html` — query with mixed groups + chord badges | ⌘K overlay — seeded catalog, 2-char query | 1440×900 | normative | Ghost tail absent (task_03); entity sections absent (task_03); fallback row absent (task_10) |
| VC-03 | `command-palette-root-states.html` — daemon-unavailable degradation | ⌘K overlay with daemon stopped — disabled rows + exempt commands live | 1440×900 | normative | None |
| VC-04 | `command-palette-root-states.html` — disabled-with-reason rows | ⌘K overlay — context-unavailable commands with verbatim reason hints | 1440×900 | normative | None |
| VC-05 | `command-palette-root-states.html` — overflow note at scale | ⌘K overlay — 60+ commands seeded, capped group with exact "showing N of M" | 1440×900 | normative | None |
| VC-06 | `command-palette-root-states.html` — destination mode + zero-eligible empty | new-tab destination palette, populated + zero-eligible variants | 1440×900 | normative | None |

Evidence for each row: `.compozy/tasks/command-palette/evidence/visual/task_02/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [ ] 2.1 `packages/ui` Command custom-filter tests + stories (`shouldFilter={false}`, external order, selection survival) — lands first
- [ ] 2.2 cmd-palette client in `web/src/systems/os/`: adapter (typed error class, AbortSignal), query keys/options, generated-type consumption via `apiClient`
- [ ] 2.3 IndexedDB structural cache: wrapper module + `fake-indexeddb` test double + corrupt/version-mismatch drop semantics
- [ ] 2.4 Hydration hook (SWR, revision-keyed) + SSE consumer (named listener via `createStreamEventSource`) + MSW fixtures/handlers
- [ ] 2.5 Context evaluator (closed key set, snapshot-driven, debounced flaps, never allow-all)
- [ ] 2.6 Dispatch seam (client_op/tool/navigate/url) + stale-target honesty + usage POST wiring
- [ ] 2.7 `PaletteResults` projection + section assembly replacing the four group components; destination mode re-sourced as a registry query
- [ ] 2.8 Registry-driven view resolution replacing `PALETTE_VIEW_FRAMES`/`PaletteViewId` (Sessions view intact on the new resolution)
- [ ] 2.9 Menubar projection (`MenubarCommandItem` across the six menus) + cheatsheet derivation + settings shortcut-table re-source (listing only)
- [ ] 2.10 Keyboard path re-route: `use-os-shortcuts` dispatches registry ids through the seam; delete the if-chain + BR-20 landing divergence
- [ ] 2.11 Execute the delete list + update canonical suites/stories + promote palette selectors into `web/e2e/fixtures/os-navigation.ts`
- [ ] 2.12 Visual Contract evidence bundles for VC-01..06

## Implementation Details

Reference `_spec.md` Part II: System Architecture (web projection row), Key Decisions (catalog caching, attached clients, entity search stays client-side), Impact Analysis delete targets, Safety Invariants 3, 8, 17.

### Skills

`eng-design` · `ui-craft` · `impeccable` · `eng-ui-screenshot` · `vitest` · `eng-consolidate-test-suites` · `testing-boss` (+ `web/CLAUDE.md` dispatch)

### Relevant Files

- `web/src/systems/os/components/os-command-palette.tsx` — root/view branch + single mount; the rework starts here
- `web/src/systems/os/hooks/use-os-command-palette.ts` (343 L) — god view-model; its 27-field model + 21 action closures are the absorption checklist
- `web/src/systems/os/components/os-command-palette-{results,window-actions,shell-actions,views}.tsx` — the four delete targets; their rows enumerate the core inventory
- `web/src/systems/os/lib/window-manager-command-registry.ts` + `window-manager-action-dispatch.ts` + `hooks/use-os-window-commands.ts` — TS mirror, if-chain, and the shared window model (the two divergent dispatch paths converge on the seam)
- `web/src/systems/os/lib/palette-view-registry.ts` + `palette-view-stack.ts` + `components/os-palette-view-{shell,stack}.tsx` — shipped stack mechanics kept; closed view map replaced
- `web/src/systems/os/hooks/use-os-shortcuts.ts` + `lib/window-manager-shortcuts.ts` + `hooks/use-desktop-shell-body.ts:159-219` — keyboard listener, chord grammar/labels, palette-open handlers
- `web/src/systems/os/hooks/use-window-manager-stream.ts` + `lib/window-manager-stream-schema.ts` — WS consumer whose `client_command` branch task_01 co-shipped; this task consumes acks/results
- `web/src/systems/os/stores/window-manager-store.ts:181-205` + `-store-types.ts` + `hooks/use-desktop-overlays.ts` — palette events/intent/overlay slot (kept)
- `web/src/systems/os/lib/app-catalog.ts` — `OS_APP_DESCRIPTORS`; navigate targets the projection maps to
- `web/src/lib/status-tone.ts` + `compareAttentionFirst` + `useAttentionJump` + routing-coordinator land-on-session — BR-20 shared authorities
- `web/src/systems/os/components/menubar/{go,session,compozy,window,workspace,help}-menu.tsx` + `desktop-menubar.tsx` + `os-menubar.tsx` — the six menus + surviving chrome
- `web/src/systems/os/components/os-shortcuts-dialog.tsx` + `web/src/systems/settings/components/layouts/window-manager-shortcut-table.tsx` — cheatsheet + table to re-source
- `web/src/systems/extensions/adapters/extensions-api.ts` + `lib/query-{keys,options}.ts` + `hooks/use-extensions.ts` — adapter/hook shape for the new client
- `web/src/lib/ticketed-event-source.ts` + `web/src/systems/workspace/hooks/use-worktree-catalog-stream.ts` — SSE facade + best-exemplar consumer recipe
- `packages/ui/src/components/command.tsx` + `__tests__/command.test.tsx` + `stories/command.stories.tsx` — cmdk wrappers gaining the custom-filter coverage
- `web/src/systems/os/hooks/__tests__/use-os-command-palette.test.tsx` (748 L) + `__tests__/os-interaction-hooks.test.tsx` (1574 L) + `lib/__tests__/*` — canonical suites to update in place
- `web/e2e/__tests__/os-shell.spec.ts` + `web/e2e/fixtures/os-navigation.ts` — palette E2E + the fixture helper that gains the promoted selectors

### Dependent Files

- `web/src/systems/os/index.ts` — barrel exports for the new modules
- `web/src/storybook/window-manager-shortcut-fixtures.ts` — third keymap copy, deleted
- `web/src/systems/os/components/stories/os-command-palette.stories.tsx` (+ `_shell-fixture.tsx`) — stories re-pointed at the projection (incl. AtScale coverage for VC-05)
- `web/src/generated/compozy-openapi.d.ts` — consumed via `apiClient` (regenerated in task_01)
- `web/package.json` — `fake-indexeddb` dev dependency via `bun add -d`

### Competitor References

- `.resources/supercmd/src/renderer/src/hooks/useLauncherCommandModel.ts` — pure derivation pipeline + empty-query grouping (the projection's shape; empty-query personalization itself is task_03).

### Related ADRs

- [ADR-001](adrs/adr-001.md) — registry drives every rendering surface; the absorption this task renders
- [ADR-005](adrs/adr-005.md) — `cmd_palette` identifier in client module naming
- [ADR-006](adrs/adr-006.md) — web evaluates only volatile context; one dispatch seam; SWR + revision + SSE skew mitigation

### Web/Docs Impact

- `web/`: this task IS the web impact — `web/src/systems/os/` palette tree rewrite, menubar/cheatsheet/settings-table derivation, `packages/ui` command tests/stories, MSW fixtures, stories, `web/e2e` selector promotion (paths above).
- `packages/site`: none — checked: no doc page documents the palette's internal rendering; user-facing palette docs land with their owning backend slices (tasks 05/07/09).
- QA impact: flag only (walks in task_12): add content-addressed `untested` scenario **registry-driven root**; reset `docs/qa/scenarios/ET-window-tab-palette-search.md` to `untested` (tab rows re-sourced); `ET-palette-nested-views.md` NOT reset (stack semantics untouched this task — changes in task_06).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: no manifest, hook, registry, or SDK change (web projection only; extension contributions render via task_07).
- Agent manageability: none new — checked: agent surface shipped in task_01; this task adds no CLI/HTTP/UDS route.
- Config lifecycle: none — checked: no `config.toml` key added/removed; keymap validation delete happens in task_05 with the open-id space.

## Deliverables

- One registry projection rendering root, destination mode, menubar, cheatsheet, and the settings table listing — with all delete targets removed in the same change
- IndexedDB structural cache + hydration + SSE invalidation live (instant open, honest staleness)
- Dispatch seam + context evaluator + keyboard re-route on registry ids
- `packages/ui` custom-filter tests/stories landed before the swap
- Updated canonical web suites/stories + promoted E2E selectors
- Visual Contract evidence bundles for VC-01..06 **(REQUIRED)**
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [ ] UT-095, UT-096, UT-097, UT-098, UT-099, UT-100, UT-101, UT-102, UT-103, UT-104 — hydration/SWR, daemon-loss + exempt, hidden-vs-disabled, verbatim reasons, globe write path, context evaluator, no-allow-all, chord badges, flap debounce, named SSE listener
- [ ] UT-105, UT-106, UT-107, UT-108 — dispatch seam routing, stale-target honesty, destination∧stack exclusion, zero-eligible empty
- [ ] UT-145, UT-146, UT-147 — menubar projection, disabled-with-reason stability, extension-item removal (fixture-level catalog data)
- [ ] UT-152, UT-153, UT-154 — `packages/ui` Command custom-filter path
- [ ] E2E-015 — destination flow + zero-eligible variant
- [ ] E2E-018 — daemon stop/restart degradation + exempt cheatsheet + re-enable
- [ ] E2E-019 — menubar chord/label/availability parity + live rebind reflection

## Success Criteria

- Every assigned test case implemented and passing
- All eight delete targets removed; `rg` finds no references to the deleted modules; no shim or re-export remains
- Palette opens instantly against a cold daemon (stale catalog + disabled-with-reason) — walked in E2E-018
- Menubar/cheatsheet/settings-table render identical id/label/chord for the same command (US-001.AC-4 pinned by the suites)
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
- `make gate` green (web lanes + affected packages/ui lane)
