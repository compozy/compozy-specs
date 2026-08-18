---
status: completed
title: Window snap layer (ADR-009)
type: frontend
complexity: high
---

# Task 9: Window snap layer (ADR-009)

## Overview

Give floating windows FancyZones-class snap: dragging a window near a desktop edge/corner previews a zone overlay and release snaps it to that half or quarter; snapped geometry persists as viewport-proportional fractions (`snap: {fx,fy,fw,fh}` on the `win:*` doc) and renders derived from each client's viewport, so halves stay halves across resizes and clients. Palette + keyboard reach every snap action; agents snap by writing fractions through the existing desktop-state surface. Zero daemon change — payloads are opaque (ADR-008); this task is web codec + store + gesture + overlay + docs.

<critical>
- ALWAYS READ the PRD, the TechSpec (§Window Snap Layer, §Safety Invariants 19), ADR-009, `analysis/02_analysis_hermes-window-arrangement.md`, and the catalogs (`_user_stories.md` US-021, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST add `snap: OsSnapZone | null` to `OsWindow` + the `win:*` codec (whole-doc writes; validation salvages invalid fractions to `null` without dropping the window) and the store action `snapWindow(id, zone)` with invariant-19 semantics: `snap`/`maximized` mutually exclusive, `prevRect` = pre-snap rect, any `commitRect` clears `snap`.
2. MUST implement derived geometry: while `snap` is set, the rendered rect = desktop work area (menubar/dock gutters respected) × fractions on the same non-Rnd absolute path maximized windows use, clamped to window minimums (280×180); viewport resize re-derives and NEVER persists; `rect` keeps the snapping client's px from commit time for thumbnails/readers.
3. MUST implement the drag gesture: pure zone resolver (20px sensitivity radius, rects snapshotted at drag start, one published hint per frame) targeting left/right halves + four corner quarters (top edge unbound — zoom owns full); drag-away from a snapped state detaches and restores `prevRect` dimensions under the pointer.
4. MUST implement the drop overlay to the normative motion contract (TechSpec §Window Snap Layer): 80ms linear fade-in, 150ms ease-out target morph transitioning only insets/background/border/opacity, backdrop blur on the active target only, dashed tokenized accent outline, Escape/outside-release cancel, full collapse under reduced motion (system preference and in-product toggle).
5. MUST wire palette commands for every zone + restore and the ⌃⌥ chords (←/→ halves, U/I/J/K quarters, ↓ restore), listed in the Help menu; palette is the guaranteed keyboard path where the browser reserves a chord; compact presentation: snap actions no-op and no affordance renders (UT-061 gating).
6. MUST update agent-facing docs in the same change: `skills/agh` payload key conventions gain the `snap` field, and the `packages/site` desktop-state page documents fraction semantics + the derived-rendering rule (agents write fractions, clients derive px).
7. MUST ship a Storybook story for the snap overlay states (idle/eligible/active/reduced-motion) and cite `eng-ui-screenshot` captures + designer review as parity evidence — the constants are the contract; no rendered Hermes reference bundle (ADR-009).
</requirements>

## Visual Evidence (constants-as-contract — ADR-009)

No Visual Contract reference bundle: the parity contract is the adopted constant set (20px radius, 4px threshold, 80ms/150ms motion, active-only blur), with provenance in `analysis/02_analysis_hermes-window-arrangement.md`. Evidence per completion: Storybook story states captured via `eng-ui-screenshot` at 1440×900 (drag-over-left-half, drag-over-quarter, reduced-motion) + a `review.md` from the designer pass, stored at `.compozy/tasks/os-shell/evidence/visual/task_09/`.

## Subtasks

- [x] 9.1 Types + codec: `OsSnapZone`, `snap` on `OsWindow` + `win:*` payload, validation salvage (UT-099)
- [x] 9.2 Store: `snapWindow`, exclusivity with `maximized`, `prevRect`, `commitRect` clears snap (UT-095–UT-097)
- [x] 9.3 Derived geometry selector + work-area math + minimum clamp + no-commit-on-resize (UT-098)
- [x] 9.4 Pure zone resolver + drag integration (snapshot rects, single hint) (UT-100)
- [x] 9.5 Drop overlay component to the motion contract + cancel paths + reduced motion
- [x] 9.6 Palette commands + ⌃⌥ chords + Help menu rows + compact gating (UT-101)
- [x] 9.7 Docs: `skills/agh` + site payload conventions (`snap` fractions, derived rendering)
- [x] 9.8 Storybook story + `eng-ui-screenshot` captures + designer review evidence
- [x] 9.9 E2E journeys (E2E-025..027) + scoped web lane green (`bunx turbo run lint typecheck test --filter=./web`)

## Implementation Details

Skills: `eng-design` + `ui-craft` + `impeccable`, `react`, `zustand`, `tailwindcss`, `storybook-stories`, `eng-ui-screenshot`, `vitest`. The zone resolver and derived-geometry selector are pure TS (testable without DOM), mirroring the Hermes separation of pure math from render (see analysis). Snapped windows bypass react-rnd exactly like maximized ones (`os-window.tsx` absolute path). Persistence rides the existing binder — `snapWindow` is one whole-doc write; no debounce changes.

Competitor references (explicit paths, per the `.resources/` rule):

- `.resources/hermes/apps/desktop/src/components/pane-shell/tree/zones-engine.ts` — sensitivity-radius capture + highlight state (the resolver pattern)
- `.resources/hermes/apps/desktop/src/components/pane-shell/tree/renderer/drag-session.ts` — snapshot-at-drag-start, rAF coalescing, Esc abort, commit-on-release
- `.resources/hermes/apps/desktop/src/components/pane-shell/tree/renderer/tree-group.tsx` — `ZoneDropOverlay` treatment (dashed sheet, active-only blur, longhand-inset morph)
- Digest: `analysis/02_analysis_hermes-window-arrangement.md` (what transfers vs tiling-only)

### Relevant Files

- `web/src/systems/os/lib/os-types.ts` — `OsSnapZone` + `OsWindow.snap`
- `web/src/systems/os/lib/os-state-payloads.ts` — codec + salvage validation
- `web/src/systems/os/stores/desktop-store.ts` — `snapWindow` + invariant 19
- `web/src/systems/os/components/os-window.tsx` + `hooks/use-os-window.ts` — derived path + drag integration
- `web/src/systems/os/lib/` — new pure modules: zone resolver + derived-geometry selector (one responsibility per file)
- `web/src/systems/os/components/` — new drop-overlay component (+ story)
- `web/src/systems/os/components/os-command-palette.tsx`, `hooks/use-os-shortcuts.ts`, menubar Help — commands + chords
- `skills/agh/`, `packages/site` desktop-state docs — payload conventions

### Dependent Files

- `web/src/systems/os/components/os-window-frame.tsx` — task_08's unified head is the drag surface (do not fork chrome)
- `web/src/systems/os/lib/desktop-persistence.ts` — whole-doc write path (no schema of its own)

### Related ADRs

- [ADR-009](adrs/adr-009.md) — this task's contract (zones, fractions, derived rendering, exclusivity, follow-ups)
- [ADR-003](adrs/adr-003.md) — store as single arrangement authority; rnd bypass precedent
- [ADR-004](adrs/adr-004.md)/[ADR-008](adrs/adr-008.md) — whole-doc `win:*` writes over opaque payloads
- [ADR-007](adrs/adr-007.md) — the amended deferral (tidy/queue remain follow-ups)

## Deliverables

- Snap layer complete: gesture + overlay + derived geometry + palette/keyboard + codec, to the constants contract
- `skills/agh` + site payload-convention docs updated for `snap` (agent-writable fractions)
- Storybook story + `eng-ui-screenshot` captures + designer review at `evidence/visual/task_09/`
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-095 — snapWindow sets snap + prevRect, clears maximized; derived selector math
- [x] UT-096 — snap/maximized exclusivity + exact prevRect restores
- [x] UT-097 — commitRect clears snap; drag-away restore semantics
- [x] UT-098 — viewport resize: derived reflow with zero persistence writes; minimum clamp
- [x] UT-099 — codec round-trip + invalid-snap salvage to null
- [x] UT-100 — pure zone resolver: edges/corners/center, single hint
- [x] UT-101 — palette/chords dispatch + compact no-op gating
- [x] E2E-025 — drag-snap → reload restore → viewport reflow
- [x] E2E-026 — cross-client fractional convergence + CLI agent snap
- [x] E2E-027 — keyboard snap + drag-away restore + reduced motion

## Success Criteria

- Every assigned test case implemented and passing; scoped web lane green (full `make verify` remains task_10's program gate)
- Zero Go/wire/CLI diffs (opaque-payload guarantee held — checked surfaces: `internal/api/contract`, OpenAPI, CLI verbs)
- Overlay motion matches the constants contract; captures + designer review recorded
- QA impact: window snap is new user-visible behavior → flag recorded here; task_11 mints the `untested` snap scenario (flag, don't retest)

## Completion Notes (2026-07-20)

Resolved interpretations (Authority & Contract Precedence, recorded per cy-execute-task):

1. "Sub-minimum zones" (UT-099/invariant 19) pinned as `OS_SNAP_MIN_FRACTION = 0.1` per axis — no literal existed in the corpus; the floor is documented in `skills/agh` + the site page as the codec contract.
2. `snapWindow(id, zone | null)` — `null` is the restore path (returns `prevRect` exactly); palette restore renders only for a snapped focused window (zoom keeps its own menu row).
3. Work area unified: the maximized path's insets became `OS_WORK_AREA_INSETS {top:8,right:10,bottom:78,left:10}` — one authority for zoom and snap derivation.
4. Invariant 19 "viewport resize never commits" applied to maximized windows too: `clampToViewport` now skips both derived states (behavior change vs prior clamp-everything).
5. A `win:*` doc claiming `snap` + `maximized` keeps `maximized`, salvages `snap` to null (deterministic decode).
6. Drag-away restore dims fall back to the registry `defaultRect` when an agent-written snap carries `prevRect: null`.
7. `reduceMotion`/`dockMagnify` round-trip through store↔codec in this task (requirement 4's in-product toggle path; encode previously reset both on every desktop write). Toggle UI remains task_10.
8. AC-1 "releasing outside every zone changes nothing" read as "no snap occurs" — plain drag commit semantics stay (E2E-002 precedence).

Evidence: UT-095..101 green in canonical suites; E2E-025..027 green and stable across 3 consecutive trio runs; scoped lane `bunx turbo run lint typecheck test --filter=./web` green; captures + `review.md` at `evidence/visual/task_09/` (1440×900).

AGH Impact Audit:

- Native tools: no impact — no `agh__*` IDs/toolsets/descriptors/schemas touched; snap rides the existing `desktop-state` surface (checked: `skills/agh` tool list, CLI verb family unchanged).
- Extensibility and hooks: no new surfaces; agent manageability extended via the client-owned `win:*` schema (`snap` fractions) documented in `skills/agh/references/desktop-state.md` + site; no hook events, bundles, registries, or bridge SDK changes (checked: extension/hook registries untouched).
- Workspace data isolation: `snap` lives inside existing workspace-scoped `win:*` docs; no new datum class, no cross-workspace read/cache/SSE path change (checked: desktop-state store/stream paths unchanged — zero Go diffs).
- Official AGH skill: updated — `skills/agh/references/desktop-state.md` gains the snap fraction conventions, salvage rules, exclusivity, and derived-rendering contract.

QA tracker impact: new user-visible behavior (drag-snap + overlay + chords + palette commands + agent snap). Flag recorded here per this task's success criteria; task_11 mints the content-addressed `untested` snap scenario (flag, don't retest).
