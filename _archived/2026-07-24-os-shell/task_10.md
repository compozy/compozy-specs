---
status: completed
title: Spaces, appearance, compact mode, and final hard-cut sweep
type: frontend
complexity: critical
---

# Task 10: Spaces, appearance, compact mode, and final hard-cut sweep

## Overview

Close the program: spaces (per-workspace desktop arrangements with the ⇧⌘S overview), the appearance layer (wallpapers, dock magnification, reduce-motion), the compact presentation implemented against the prototype's `<960px` reference block, and the terminal quality pass — the complete Delete-Targets sweep, the accessibility pass (shortcut paths, focus, reduced motion, WCAG AA), the performance-envelope scenario, and the single full `make verify` gate.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST implement spaces per §Routing Model rule 8 + US-010: workspace switch persists the outgoing space as one `apply` batch, swaps the store, navigates once; each workspace's arrangement + wallpaper independent; deleted workspaces purge cleanly (daemon side already gated — this task owns the client behavior).
2. MUST implement the Spaces overview (⇧⌘S): per-workspace cards with mini-window thumbnails scaled from live arrangements, click-to-switch, keyboard accessible.
3. MUST implement the appearance layer: three tokenized wallpapers (ember/mesh/carbon) persisted per space, dock magnification toggle, in-product reduce-motion toggle with the system preference always winning (US-015.EC-1), Appearance pane in the settings window.
4. MUST implement compact presentation against the prototype compact block (`os-v2.css:557-628`, `mqMobile`): stacked fullscreen windows (no border/radius/shadow/resize, zoom hidden, enlarged control hit areas), dock as 56px bottom tab bar (scrollable, safe-area insets, no magnification/tooltips), collapsed menubar menus with palette fallback, clamped overlays; crossing the breakpoint mid-gesture ends safely; floating rects survive round-trips.
5. MUST execute the FINAL Delete-Targets sweep: every remaining row in the `_techspec.md` inventory verified deleted/rewritten (grep-clean for the deleted identifiers), plus the spec-artifact sweep; `deslop` + `cy-final-verify` active at completion.
6. MUST run the a11y pass: every window action keyboard-reachable (⌘W/⌘M and the task_09 snap actions, with palette fallback where browsers reserve), visible tokenized focus states across chrome, `role`/`aria-label` on windows/dock/rail/bell, status never color-only, reduced-motion honored globally (incl. the snap overlay).
7. MUST land the performance-envelope scenario (E2E-023) proving the Assumptions budget (12 windows / frame budget / restore <500ms / 3-client convergence) and record the measured numbers in completion notes.
8. MUST finish with ONE full `make verify` (the program's completion gate) — plus `make test-e2e-web` for the Playwright lane.
</requirements>

## Visual Contract

References: `docs/design/opendesign/os/agh-os-v2.html` desktop states + the compact block (`os-v2.css:557-628`) — current working tree. Deltas #1..#11 are the authorized-difference authority.

| ID | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| --- | --- | --- | --- | --- | --- |
| VC-01 | prototype spaces overlay — cards with mini-window thumbnails | ⇧⌘S overview, two seeded workspaces with distinct arrangements | 1440×900 | normative | Real workspaces (delta #4) |
| VC-02 | prototype wallpapers ember/mesh/carbon | Appearance pane switching all three (capture each) | 1440×900 | normative | None |
| VC-03 | prototype compact — stacked window + tab-bar dock | seeded desktop at compact viewport | 390×844 | normative | Rail as overlay sheet (delta #5) |
| VC-04 | prototype compact — palette overlay | ⌘K open at compact viewport | 390×844 | normative | None |
| VC-05 | prototype dock magnification — hover state | dock hover with magnification on | 1440×900 | normative | Disabled under reduced motion (DESIGN motion tier) |

Evidence for each row: `.compozy/tasks/os-shell/evidence/visual/task_10/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [ ] 10.1 Spaces: per-workspace store swap + persist-batch + workspace-switch cause wiring
- [ ] 10.2 Spaces overview overlay (⇧⌘S) with live thumbnails
- [ ] 10.3 Wallpapers + Appearance pane + magnification + reduce-motion precedence
- [ ] 10.4 CompactStack presentation + tab-bar dock + collapsed menubar + overlay clamps
- [ ] 10.5 Cross-workspace deep-link behavior (switch space then open — E2E-016)
- [ ] 10.6 Final Delete-Targets sweep + grep-clean + spec-artifact sweep (`deslop`)
- [ ] 10.7 A11y pass (keyboard paths incl. snap, focus, ARIA, color-independence, reduced motion)
- [ ] 10.8 Performance-envelope scenario + measured numbers
- [ ] 10.9 Full `make verify` + `make test-e2e-web` (single completion gate)

## Implementation Details

Skills: `eng-design` + `ui-craft` + `impeccable`, `react`, `tailwindcss`, `eng-ui-screenshot`, `vitest`, `deslop`, `cy-final-verify`. Compact is a presentation variant of the same logical state — no forked stores, no compact-only state (ADR-001 presentation split). The sweep is the SD-002 hard-cut enforcement point: after this task, no old-shell identifier survives anywhere (code, tests, stories, spec artifacts).

### Relevant Files

- `web/src/systems/os/components/{spaces-overlay,compact-stack,wallpaper-layer,appearance-pane}.tsx` (new)
- `web/src/systems/os/stores/desktop-store.ts` — presentation gating + space swap
- `web/src/systems/workspace/` — active-workspace integration for switch cause
- `_techspec.md` §Delete Targets — the sweep checklist
- `web/e2e/` — compact + spaces + perf specs

### Dependent Files

- All shell components from tasks 03-07 (this task closes over them)
- `docs/qa/scenarios/` — flag-only (QA pair owns authoring)

### Related ADRs

- [ADR-001](adrs/adr-001.md) — responsive single shell + compact reference amendment
- [ADR-006](adrs/adr-006.md)/[ADR-007](adrs/adr-007.md) — identity polish + anti-clutter surfaces closed out here

## Deliverables

- Spaces + overview, appearance layer, and compact mode complete against their references
- Delete-Targets inventory fully executed (grep evidence in completion notes); a11y pass documented
- Performance numbers recorded against the envelope
- Green full `make verify` + Playwright lane
- Visual Contract bundles VC-01..05 passing
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] E2E-009 — workspace-space switch + overview + arrangement restore
- [ ] E2E-011 — compact stack round-trip with preserved floating rects
- [ ] E2E-013 — wallpaper persistence + reduce-motion minimize
- [ ] E2E-016 — cross-workspace deep link switches space and opens focused
- [ ] E2E-020 — compact parity: deep link + badges + rail overlay
- [ ] E2E-021 — system reduced-motion precedence over the in-product toggle
- [ ] E2E-023 — performance envelope (12 windows / drag frames / restore latency / 3-client convergence)

## Success Criteria

- Every assigned test case implemented and passing; **full `make verify` green** (program completion gate)
- Zero old-shell identifiers repo-wide (sweep grep clean); all five VC rows PASS
- Measured perf numbers within the Assumptions envelope, recorded in completion notes
- QA impact: spaces/appearance/compact are new user-visible behavior → task_11 mints their `untested` scenarios; flag recorded here
