---
status: completed
title: "P3 — Execution UX: Action Panel, Inline Args, Confirmation, Feedback"
type: frontend
complexity: medium
---

# Task 4: P3 — Execution UX: Action Panel, Inline Args, Confirmation, Feedback

## Overview

Completes the operator execution experience on the task_02 projection: the ⌘K-inside-⌘K action panel with meta-actions on every command row, inline typed arguments in the input bar, the declared destructive confirmation step, and the honest async feedback lifecycle (in-palette pending → toast after close, retry on idempotent-safe failures, single-flight rejection surface). No new public contract — this slice is artboard-bound web behavior over the shipped dispatch seam.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting; read every ADR in `adrs/` and any `memory/task_NN.md` present
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — fix root causes; frozen `_dx.md`/Part II contracts are binding; surface gaps as clarifications, never invent
</critical>

<requirements>
1. MUST implement `PaletteActionPanel` per S7: ⌘K toggles on the selected row; sections, per-action chord badges, primary marked ↩, typing filters, destructive styling; meta-actions (Pin/Unpin, Set alias…, Set shortcut…) on every command row; domain actions on entity rows; disabled rows show meta-actions + verbatim reason only; a vanished row closes the panel with nearest-neighbor selection and no dead fire; action shortcuts dispatch capture-phase with `event.repeat` guarded.
2. MUST implement `PaletteArgsBar` per S8: selecting an argument-bearing command morphs the input into inline typed fields (text/password/dropdown) with placeholders; ⇥ traverses; dropdown popover type-to-filters; ⏎ blocks on missing required with first-empty focus; Esc restores search discarding values; bound-hotkey activation opens directly in argument mode; password fields masked, never echoed or recorded.
3. MUST implement `PaletteConfirmation` per S9: declared confirmations render in-palette with Cancel focused; repeat-guarded against the triggering keystroke; target invalidation between trigger and confirm shows the honest message instead of executing; Esc backs out cleanly.
4. MUST implement the S14 feedback lifecycle on the dispatch seam: sync client ops close-and-show; async invokes render the in-palette pending affordance while open and complete via the surviving toast system after close; failures name command + reason with retry only for `RetrySafe` commands; `already_running` renders the rejection; cross-workspace landings name the switch. No fake progress (SD-007).
5. MUST keep the Esc ladder exact (`_uiux.md` a11y notes): panel → confirmation/args → view stack → close; full keyboard reachability of every action; ARIA combobox pattern preserved.
6. MUST make meta-actions functional against shipped surfaces: Pin/Unpin calls the task_03 pins route; "Set alias…"/"Set shortcut…" deep-link to the settings shortcut table (the mutation flows themselves land in task_05 — deep-link only, no dead controls).
7. MUST block and surface if a cited artboard is missing at execution time.
</requirements>

## Visual Contract

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | -------- | ---------------------------------- |
| VC-01 | `docs/design/opendesign/command-palette/command-palette-action-panel.html` — open panel with sections + filter + primary ↩ | ⌘K on a selected command row | 1440×900 | normative | None |
| VC-02 | `command-palette-action-panel.html` — filtered-empty panel | panel with non-matching filter text | 1440×900 | normative | None |
| VC-03 | `command-palette-action-panel.html` — destructive action emphasis | entity row panel with a destructive domain action | 1440×900 | normative | None |
| VC-04 | `command-palette-action-panel.html` — disabled-row panel (meta + reason only) | panel on a context-unavailable command | 1440×900 | normative | None |
| VC-05 | `command-palette-args-confirmation.html` — args pristine + filled fields | capture-fixture command in argument mode | 1440×900 | normative | None |
| VC-06 | `command-palette-args-confirmation.html` — required-missing block with focused field | ⏎ with empty required | 1440×900 | normative | None |
| VC-07 | `command-palette-args-confirmation.html` — dropdown open with type-to-filter | dropdown argument focused | 1440×900 | normative | None |
| VC-08 | `command-palette-args-confirmation.html` — confirmation standard + destructive emphasis | destructive command confirmation, Cancel focused | 1440×900 | normative | None |
| VC-09 | `command-palette-args-confirmation.html` — invalidated-target state | confirmation whose target changed | 1440×900 | normative | None |
| VC-10 | `command-palette-root-states.html` — in-palette pending affordance | slow async invoke while palette open | 1440×900 | normative | Toast anatomy reuses the existing system (S14 note) |

Evidence for each row: `.compozy/tasks/command-palette/evidence/visual/task_04/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 4.1 `PaletteActionPanel` + `PaletteActionPanelItem`: registration model per row kind, filter, sections, chord badges, primary marking
- [x] 4.2 Meta-actions on every command row (pins route wiring; settings deep-links) + entity domain actions with destructive styling
- [x] 4.3 Panel guards: vanished-row auto-close + nearest-neighbor, disabled-row meta-only, capture-phase shortcut dispatch + repeat guard
- [x] 4.4 `PaletteArgsBar`: field morphing, traversal, dropdown popover, validation messages, Esc restore, hotkey-entry argument mode
- [x] 4.5 Password argument handling: masking, exclusion from echo/history, clear-on-exit
- [x] 4.6 `PaletteConfirmation`: declared copy render, Cancel focus, repeat guard, target-invalidation message
- [x] 4.7 Feedback lifecycle on the seam: pending affordance, completion/failure toasts (surviving unmount), retry gating by `RetrySafe`, `already_running`, cross-workspace notice
- [x] 4.8 Esc-ladder + a11y integration across the three new surfaces; stories for every VC state
- [ ] 4.9 Visual Contract evidence bundles VC-01..10 — deferred to task_12 by the accepted tail-only QA policy

## Implementation Details

Reference `_spec.md` Part II: System Architecture (story→component map US-014..017), `_uiux.md` S7/S8/S9/S14 states, Safety Invariants 9 (UI confirmation never substitutes policy).

### Skills

`eng-design` · `ui-craft` · `impeccable` · `eng-ui-screenshot` · `vitest` · `eng-consolidate-test-suites` · `better-accessibility` (+ `web/CLAUDE.md` dispatch)

### Relevant Files

- `web/src/systems/os/` projection/dispatch/seam modules from task_02 — the substrate every surface plugs into
- `packages/ui/src/components/command.tsx` — cmdk wrappers (panel is a nested Command instance)
- `packages/ui/src/components/sonner.tsx` + `web/src/lib/user-feedback.ts` + `web/src/systems/os/components/attention-toast.tsx` — the feedback system that survives unmount (US-017 rides it)
- `web/src/systems/os/components/os-palette-view-shell.tsx` + `os-palette-footer.tsx` — chrome/footer conventions the new surfaces match
- `web/src/systems/os/hooks/use-desktop-overlays.ts` — overlay/Esc ownership the ladder integrates with
- `web/src/systems/os/hooks/__tests__/os-interaction-hooks.test.tsx` — capture-phase/keyboard suite lineage
- `web/e2e/fixtures/os-navigation.ts` — palette selectors promoted in task_02; panel/args selectors join them

### Dependent Files

- `web/src/systems/os/index.ts` — barrel exports
- `web/src/systems/os/components/stories/` — panel/args/confirmation/pending stories (VC states)
- `web/src/systems/os/` MSW fixtures — slow/failing invoke scenarios for feedback tests

### Competitor References

- `.resources/supercmd/src/renderer/src/raycast-api/action-runtime-registry.tsx` — action registry anatomy (per-row action registration)
- `.resources/supercmd/src/renderer/src/raycast-api/action-runtime-overlay.tsx` — ⌘K panel anatomy

### Related ADRs

- [ADR-001](adrs/adr-001.md) — meta-actions on every command row (bindable/pinnable/aliasable by construction)
- [ADR-004](adrs/adr-004.md) — action-panel parity contract that view rows (task_06) and grid tiles inherit

### Web/Docs Impact

- `web/`: new `PaletteActionPanel`/`PaletteArgsBar`/`PaletteConfirmation` composites + feedback wiring under `web/src/systems/os/` (paths above), stories, MSW scenarios.
- `packages/site`: none — checked: no public contract change; operator docs for execution UX are covered by the palette page updates in later slices.
- QA impact: flag only (walks in task_12): add content-addressed `untested` scenarios **action panel** and **inline args + confirmation** (per the `_tests.md` QA section list).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: no manifest/hook/SDK change; extension row actions render through this panel when task_07 lands (contract already fixed by the vocabulary).
- Agent manageability: none new — checked: agent confirmation parity is the approval flow shipped in task_01 (US-016.EC-3); no new route/verb here.
- Config lifecycle: none — checked: no `config.toml` key touched.

## Deliverables

- Action panel, inline args, confirmation, and feedback lifecycle live on the projection with the exact Esc ladder
- Meta-actions wired (pins functional; alias/shortcut deep-links) — no dead controls
- Stories covering every VC state; canonical suites extended in place
- Visual Contract evidence bundles VC-01..10 **(REQUIRED)**
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-120, UT-121, UT-122 — args morphing/traversal/execute, blocking + Esc restore + paste validation, hotkey-entry argument mode
- [x] UT-123, UT-124 — confirmation render/Cancel focus/Esc, repeat guard + target invalidation
- [x] UT-125, UT-126, UT-127, UT-128, UT-129 — panel open/filter/close, meta-actions + domain actions, vanished-row close, disabled-row meta-only, capture-phase + repeat guard
- [x] UT-159, UT-160 — feedback: sync close-and-show, async pending→toast, cross-workspace naming; failure toast + retry gating
- [ ] E2E-014 — deferred to task_12 by the accepted tail-only QA policy

## Success Criteria

- Every assigned test case implemented and passing
- Esc ladder verified end-to-end (panel → confirmation/args → stack → close) in the interaction suite
- No action ever fires against a vanished/invalidated target (UT-124/UT-127 pin it)
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
- `make gate` green (web lanes)
