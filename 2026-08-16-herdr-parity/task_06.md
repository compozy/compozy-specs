---
status: completed
title: "Command palette nested views + Sessions view"
type: frontend
complexity: medium
---

# Task 6: Command palette nested views + Sessions view

## Overview

Delivers P6: the generic nested-view mechanism inside the command palette (view stack, breadcrumb, backspace-on-empty pops, dismiss closes all, built-in registry only) and its first registration — the Sessions view with state-filter chips, attention-first ordering, the Show all scope option, and enter-to-land — reachable via ⌘E or the root palette entry. Generalizes the existing single `destinationWindowId` mode seam (ADR-003).

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. The view mechanism MUST be generic (any built-in registered view: search, keyboard nav, enter-to-activate identical), stack-based with breadcrumb (≤3 visible levels, left-truncated), backspace pops only on empty query, dismiss closes the whole stack, reopen starts at root; view results never bleed across levels (Business Rule 32).
2. The view registry MUST be built-in-only in v1 while structurally leaving room for extension registration (ADR-003) — no manifest surface ships.
3. The Sessions view MUST use task_03's exported tone/glyph dictionary and attention model: chips for needs-you/working/finished/idle, attention-first ordering by `attention_changed_at`, Show all option widening to every workspace (US-031.AC-2), enter lands via the shared land-on-session behavior (marking `done` seen on arrival), zero-match chip empty state with one-keystroke clear.
4. ⌘E MUST open the palette pushed into the Sessions view (the action `palette.view.sessions` registered by task_05); a root palette entry pushes it too.
5. The `use-os-command-palette.ts` view-model (343L, near cap) MUST NOT grow — the view stack lives in new sibling hook/component files per `_spec.md` Architectural Boundaries.
6. Live catalog refreshes MUST update view results without stealing keyboard selection (US-029.EC-4, US-030.EC-2); hundreds of sessions stay responsive (scale fixture).
7. MUST implement S12/S13 from the locked visual contract `docs/design/opendesign/herdr-parity/herdr-parity-palette-sessions.html` (`DESIGN-NOTES.md` + the rows in `## Visual Contract`). Activate `eng-design` + `ui-craft` + `impeccable` and `eng-ui-screenshot` Visual Contract Mode before code. Compose from `@compozy/ui` `Command*` primitives; artboard CSS is a contract, never a stylesheet to import. MUST produce the `eng-ui-screenshot` evidence bundle for every Visual Contract row — implementation-only captures are invalid.
</requirements>

## Visual Contract

Reference artboards: `docs/design/opendesign/herdr-parity/` (iterate, never regenerate). All rows viewport 1440×900, fidelity normative. Authorized differences: `DESIGN-NOTES.md` lossiness rules (runtime truth + `COPY.md` own content/copy; `@compozy/ui` `Command*` owns component identity; live palette host chrome owns placement context; selection is a neutral plate, never an accent bar) — record each divergence as an authorized delta in `review.md`.

| ID | Reference artifact + state | Implementation target + state | Viewport | Fidelity |
| --- | --- | --- | --- | --- |
| VC-01 | `herdr-parity-palette-sessions.html` — §01 root with Sessions view entry | Palette root fixture | 1440×900 | normative |
| VC-02 | `herdr-parity-palette-sessions.html` — §01 Sessions view pushed (chips + attention-first rows) | `SessionsPaletteView` unfiltered | 1440×900 | normative |
| VC-03 | `herdr-parity-palette-sessions.html` — §02 three-level breadcrumb (ancestors collapse to "…") | Palette view stack depth-3 | 1440×900 | normative |
| VC-04 | `herdr-parity-palette-sessions.html` — §02 empty view | Palette view empty | 1440×900 | normative |
| VC-05 | `herdr-parity-palette-sessions.html` — §03 zero-match chip (one-keystroke clear) | Sessions view zero-match | 1440×900 | normative |

Evidence for each row: `.compozy/tasks/herdr-parity/evidence/visual/task_06/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 6.1 View-stack mechanism (new sibling hook + `PaletteViewStack` glue): push/pop/dismiss/breadcrumb/backspace semantics over the cmdk primitives.
- [x] 6.2 Built-in view registry (typed, list-shaped views; extension door structurally open, not exposed).
- [x] 6.3 Sessions view: chips, ordering, Show all, enter-to-land, empty/scale states.
- [x] 6.4 Entry points: ⌘E dispatch + root palette entry; palette reopen-at-root behavior.
- [x] 6.5 Selection-preservation on live refresh; bounded scrolling list at scale.
- [x] 6.6 Stories + workspace-aware MSW fixtures; the existing shortcuts page wording remains truthful.
- [x] 6.7 Visual-contract capture bundles for VC-01..VC-05 — completed in task_08; all 5 bundles validate with zero blocking divergences.

## Implementation Details

Reference `_spec.md` Part II (System Architecture web row, Key Decisions), `_uiux.md` S12/S13 (states from US-029/US-030 ACs/ECs), and `docs/design/opendesign/herdr-parity/DESIGN-NOTES.md` (locked visual facts).

### Relevant Files

- `web/src/systems/os/hooks/use-os-command-palette.ts` (343L — the `destinationWindowId` mode seam at `paletteIntent`; `jumpToSession` at :81; `paletteSessions` at :44) — consume, don't grow.
- `web/src/systems/os/components/os-command-palette.tsx:14-65` + `os-command-palette-results.tsx` + `-shell-actions.tsx:26` (existing sessions-modal row) + `packages/ui/src/components/command.tsx` (cmdk `Command*`, vim bindings, `shouldFilter`).
- `web/src/systems/os/lib/routing-coordinator.ts:178,214` — land-on-session behavior (shared with task_03's bell/toast jumps).
- `web/src/lib/status-tone.ts` (task_03's dictionary home) + `web/src/systems/os/lib/attention-model.ts` — filters/ordering inputs.
- `web/src/systems/os/lib/window-manager-command-registry.ts` — `palette.view.sessions` action (registered in task_05).
- `docs/design/opendesign/herdr-parity/DESIGN-NOTES.md` + `herdr-parity-palette-sessions.html` — locked visual contract for S12/S13.

### Dependent Files

- `web/src/systems/os/components/os-shortcuts-dialog.tsx` — ⌘E row derives from the registry (task_05's derivation).
- `web/e2e/` — E2E-016/019 journeys; Storybook palette stories.

### Related ADRs

- [ADR-003: Command palette nested views](adrs/adr-003.md) — mechanism scope, built-in-only registry, naming.

### Competitor References

- `.resources/herdr/src/ui/navigator.rs` + `src/app/input/modal.rs` — the searchable navigator with state filters `b/w/i/d/a` and backspace-clears-filter semantics the Sessions view mirrors in palette form.

### Web/Docs Impact

- `web/`: palette component/hook siblings under `web/src/systems/os/`, stories, MSW fixtures.
- `packages/site`: ⌘E mention on the shortcuts page (task_05 owns the page; this task keeps it truthful).
- QA impact: new scenarios — add content-addressed `untested` files for: nested navigation semantics, Sessions view filter→land journey (incl. Show all + closed-window restore). Flag only — task_08 walks them.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: view registry deliberately not extension-facing in v1 (ADR-003 non-goal; structural door left open — documented in the registry's typing, no manifest surface).
- Agent manageability: none — checked: palette is operator-only UI; every underlying capability (list/filter/land) already has CLI/HTTP parity from tasks 01/04.
- Config lifecycle: none — checked `internal/config/`: no keys; ⌘E binding lives in the task_05 keymap.

## Deliverables

- Generic view stack + Sessions view live behind ⌘E and the root entry, keyboard-first end to end.
- Stories/fixtures updated; palette root behavior unchanged for existing flows (destination mode intact).
- Every Visual Contract row has a durable passing evidence bundle **(REQUIRED)**.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

- [x] UT-059..UT-061 — stack semantics, no-bleed isolation, chips/empty states
- [x] UT-076 — selection preservation on live refresh
- [x] E2E-016 — passed in task_08's full Web E2E run
- [x] E2E-019 — passed in task_08's full Web E2E run

## Success Criteria

- Every assigned test case implemented and passing.
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence.
- Filter → pick → land in two keystrokes from anywhere (US-030); mechanism proven generic by the registry shape (US-029).
- `make gate` green (web lanes via Turbo from repo root).
