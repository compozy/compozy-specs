---
status: completed
title: Shell tokens, chrome primitives, and overlay portal context
type: frontend
complexity: medium
---

# Task 3: Shell tokens, chrome primitives, and overlay portal context

## Overview

Land the visual foundation the shell builds on: the OS-shell token set in `packages/ui/src/tokens.css` (glass, window shadows, radii waiver, dock metrics, shell motion tier) with `DESIGN.md` regenerated, the authored-prose amendments that reconcile the design-system guardrails with the shell (DESIGN.md §1/§4/§5/§6/§10, web/CLAUDE.md, PRODUCT.md anti-references, COPY.md §6 + glossary vocabulary), the window-chrome primitives with Storybook stories, and the `OverlayContainerContext` seam in the `@agh/ui` Dialog/Sheet/AlertDialog portal wrappers. Runs parallel to tasks 01-02.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST add the shell tokens to `packages/ui/src/tokens.css` per §Design Tokens (`--shell-glass`, `--shell-glass-pop`, `--radius-window: 12px`, `--radius-dock: 22px`, `--shadow-window`, `--shadow-window-unfocused`, dock item metrics, shell motion durations + spring ease) and regenerate `DESIGN.md` via `make codegen`; verify the generator picks up the new token families (extend `scripts/sync-design-md.mjs` category mapping if needed). Never hand-edit generated regions.
2. MUST amend the authored prose that currently contradicts the shell: DESIGN.md §1 (blur carve-out), §4 (layout grammar → desktop model), §5 (two-layer depth: shell chrome glass/shadows vs flat content), §6 (shell motion tier), §10 (anti-pattern references the sanctioned whitelist); web/CLAUDE.md "flat depth/no glass" rule; PRODUCT.md §Anti-references glassmorphism nuance. Activate `writing-agents-md` before touching instruction files.
3. MUST add `space`, `desktop state` (vs `memory` boundary), `window`, `desktop`, `dock`, `menubar` vocabulary to `docs/_memory/glossary.md` and `COPY.md` §6 per the techspec Impact Analysis rows (activate `copywriting` + `documentation-writer`).
4. MUST build the chrome primitives in `web/src/systems/os/components/` (window frame, traffic lights, dock shell + item, menubar shell, wallpaper layer) — domain components per the reuse-before-create rule, consuming only tokens; every primitive ships a Storybook story with the a11y addon.
5. MUST add `OverlayContainerContext` to the `@agh/ui` Dialog/Sheet/AlertDialog portal wrappers with `document.body` fallback — zero call-site changes anywhere in `systems/*` (Modal & Overlay Policy §2); scoped focus containment, no global inert.
6. MUST verify every visual with `eng-ui-screenshot` and produce the Visual Contract evidence bundles below — implementation-only captures are invalid.
</requirements>

## Visual Contract

Reference: `docs/design/opendesign/os/agh-os-v2.html` (+ `os-v2.css`, `os-v2-apps.css`), current working tree. Deltas #1..#11 in `_techspec.md` §Design Tokens are the authorized-difference authority.

| ID | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| --- | --- | --- | --- | --- | --- |
| VC-01 | prototype window chrome — focused window w/ traffic lights + breadcrumb head | Storybook `os-window-frame` story — focused | 1440×900 | normative | Head 48px not 44px (delta #1) |
| VC-02 | prototype window chrome — unfocused (dimmed head, lighter shadow) | Storybook `os-window-frame` story — unfocused | 1440×900 | normative | None |
| VC-03 | prototype dock — resting state with running/minimized indicators | Storybook `os-dock` story — indicators fixture | 1440×900 | normative | Badge caps "9+" (delta #4) |
| VC-04 | prototype menubar — full menus + workspace trigger + bell + ⌘K chip | Storybook `os-menubar` story — populated | 1440×900 | normative | None |
| VC-05 | prototype wallpaper `ember` + dotted grid | Storybook `os-wallpaper` story — ember | 1440×900 | normative | None |

Evidence for each row: `.compozy/tasks/os-shell/evidence/visual/task_03/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 3.1 Token set in `tokens.css` + `make codegen` + generator category verification
- [x] 3.2 DESIGN.md authored-prose amendments (§1, §4, §5, §6, §10)
- [x] 3.3 web/CLAUDE.md + PRODUCT.md rule amendments (`writing-agents-md` active)
- [x] 3.4 Glossary + COPY.md vocabulary entries
- [x] 3.5 Chrome primitives + stories (window frame, traffic lights, dock, menubar, wallpaper)
- [x] 3.6 `OverlayContainerContext` in @agh/ui portal wrappers (+ UT-085, story exercising in-window dialog)
- [x] 3.7 Visual Contract bundles VC-01..05 via `eng-ui-screenshot`

## Implementation Details

Skills: `eng-design` + `ui-craft` + `impeccable` (read matched reference rows in full), `react`, `tailwindcss`, `storybook-stories`, `eng-ui-screenshot`, `writing-agents-md`, `copywriting`, `documentation-writer`. Chrome components are domain-scoped (`systems/os`) — do not shadow `@agh/ui` exports (lint gate `compozy-ui-reuse/no-shadow-ui-primitive`). Glass and window shadows apply to shell chrome only; content inside window bodies stays flat (invariant recorded in DESIGN.md §5 amendment).

### Relevant Files

- `packages/ui/src/tokens.css` — canonical token source
- `scripts/sync-design-md.mjs` — DESIGN.md generator (category mapping)
- `DESIGN.md`, `web/CLAUDE.md`, `PRODUCT.md`, `COPY.md`, `docs/_memory/glossary.md` — prose amendments
- `packages/ui/src/components/` — Dialog/Sheet/AlertDialog portal wrappers (OverlayContainerContext)
- `web/src/systems/os/components/` (new) — chrome primitives + stories

### Dependent Files

- `web/src/systems/os/` (task_04) — consumes primitives + context
- Every `systems/*` dialog call site — unchanged by contract (UT-085 proves the fallback)

### Related ADRs

- [ADR-001](adrs/adr-001.md) — shell replacement + compact reference amendment
- [ADR-002](adrs/adr-002.md) — Amendment 2 (modal policy)
- [ADR-006](adrs/adr-006.md) — identity framing + `space` vocabulary

## Deliverables

- Shell token set live with `DESIGN.md` regenerated and drift-checked (`make codegen-check`)
- All five doc/instruction files amended; vocabulary entries landed
- Chrome primitives + stories; `OverlayContainerContext` wired with body fallback
- Visual Contract bundles VC-01..05 passing
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-085 — dialog mounts in the window's `OverlayContainerContext` container; falls back to `document.body` outside windows

Test-density note: this task is visual-foundation work — pixel truth is owned by the Visual Contract bundles + `make codegen-check` (stronger gates than snapshots, per the static-test prohibition); UT-085 is its single behavioral seam. Chrome interaction behaviors are tested where they integrate (task_04's window lifecycle suite).

## Success Criteria

- Every assigned test case implemented and passing; `bunx turbo run lint typecheck test --filter=./web --filter=./packages/ui` clean
- `make codegen-check` passes; zero hand-edits to generated regions
- All five VC rows PASS with zero unresolved blocking divergence
- QA impact: none yet — primitives are story-only until task_04 mounts the shell (no user-visible route change)
