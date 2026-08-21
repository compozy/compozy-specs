---
status: completed
title: P9 web run-page surfaces (S1–S8, S10–S12) + visual contract
type: frontend
complexity: critical
---

# Task 8: P9 web run-page surfaces (S1–S8, S10–S12) + visual contract

## Overview

Delivers every web surface except the bell: the request-bearing Needs-you card with schema-driven answer forms (S1), the new timeline row families (S2), strategy-aware progress with first-class partial (S3), request wait kinds in the parked panels (S4), amend/rerun node verbs (S5), inspect-sheet lineage + diff/fork entry points (S6), the dedicated run-diff view (S7), the fork dialog (S8), the editor grammar for `ask`/`route`/`strategy`/`bind_as`/`review`/`on_eval_error` with linter-dock codes (S10), the detail DAG glyphs (S11), and the editor chrome & ergonomics addendum — collapsible persisted rails with calm defaults, quick-add, connection-drop picker, node context menu, multi-select, and edge affordances (S12, `_uiux.md` addendum 2026-08-16). Binds to the delivered `graph-eng-*` artboards in `docs/design/opendesign/graph-eng/` (`DESIGN-NOTES.md` + this task's Visual Contract). **Runs through the `designer` agent in execution mode with `eng-design` + `ui-craft` activated** only when a board needs an in-place iteration.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Implementation MUST bind to the delivered artboards in `docs/design/opendesign/graph-eng/` (needs-you-requests, timeline-rows, progress-strategies, node-verbs, inspect-lineage, run-diff, fork-dialog, editor-route-ask, editor-chrome) plus `DESIGN-NOTES.md`. Iterate a board in place only when a `_uiux.md` state is missing — never regenerate the set.
2. Every request form MUST derive from the daemon-persisted schemas (wait-payload machinery base — `lib/loop-node-wait-payload.ts`); decision bars render only the persisted `decisions` set (absent ≠ disabled); no optimistic paints — every mutation reconciles from refreshed truth (existing loops-system rule).
3. Truthful-UI rules from `_uiux.md` MUST hold: previews + fetchable full context (never raw refs), resolved requests never show forms, partial renders from `completion_state` (never client-inferred), aggregate counts at width, neutral absence signals.
4. New domain components land per the `_uiux.md` component plan (`LoopRequestAnswerForm`, `LoopRequestDecisionBar`, `LoopReviewProposedArgs`, `LoopRunDiffView`+`Row`, `LoopForkDialog`, `LoopNodeAmendDialog`, `LoopNodeRerunDialog`, `LoopStrategyProgress`, editor field parts) — composed from `@compozy/ui` (reuse-before-create; zero new primitives expected, justify any exception).
5. Data layer: `requestsByWorkspace` query key + query options; new API adapters for the seven operations; MSW fixtures for every new payload/event; SSE listener additions for the eight kinds.
6. The editor codec MUST stay bijective over the complete new grammar (Graph↔DSL toggle lossless) and the linter dock MUST surface every new code.
7. `make gate` web lanes + Playwright suite green; `eng-ui-screenshot` evidence bundles for every Visual Contract row (implementation-only captures are invalid).
8. S12 chrome MUST follow `_uiux.md` S12 + `analysis/s12-integration.md`: both rails collapsible, default collapsed, state persisted per user in `use-loop-editor-chrome-state.ts` (`compozy:loops:editor-chrome:v1`, flat scope) with the palette-mode ladder (expanded/collapsed/drawer) replacing the CSS-only `lg` gates; editor keyboard bindings are canvas-scoped plain keys ONLY (`a` quick-add, `[`/`]` rails, Escape with `preventDefault`) — never ⌘-chords (⌘K/⌘Z/⌘[ are shell-owned, see the collision table); connection-drop add lands node + edge in ONE draft transition; route handles/edge types stay derived display state, never written into `data.raw` (the daemon drops unknown edge keys); ride-along fixes ship in the same change (`nodeAdded` sets `positionsDirty` when placed; `onNodesDelete` prunes orphan edges).</requirements>

## Visual Contract

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | -------- | ---------------------------------- |
| VC-01 | `docs/design/opendesign/graph-eng/graph-eng-needs-you-requests.html` — pending ask with form | `/loop-runs/$runId` needs-you card, pending-ask fixture | 1440×900 | normative | Demo copy/data → runtime fixtures (SD-007) |
| VC-02 | same — pending review with proposed args + decision bar | run page, pending-review fixture | 1440×900 | normative | None |
| VC-03 | same — validation failure + already-answered states | run page, error + resolved fixtures | 1440×900 | normative | None |
| VC-04 | `graph-eng-timeline-rows.html` — request/route/pruned/amended/forked row families | run page timeline, event fixtures | 1440×900 | normative | Row copy via `COPY.md` |
| VC-05 | `graph-eng-progress-strategies.html` — best_effort partial with coverage | progress panel, partial fixture | 1440×900 | normative | None |
| VC-06 | same — fail_fast triggered + race won + wide aggregate | progress panel fixtures | 1440×900 | normative | None |
| VC-07 | `graph-eng-node-verbs.html` — amend dialog (edit + validation failure) | node row actions, paused-node fixture | 1440×900 | normative | None |
| VC-08 | same — rerun dialog with rerun-set preview | node row actions, settled-node fixture | 1440×900 | normative | None |
| VC-09 | `graph-eng-inspect-lineage.html` — generations with diff/fork entries + lineage block | inspect sheet fixture | 1440×900 | normative | None |
| VC-10 | `graph-eng-run-diff.html` — generation compare grouped by change kind | diff view, generation fixture | 1440×900 | normative | None |
| VC-11 | same — run compare with input block + divergence banner | diff view, run fixture | 1440×900 | normative | None |
| VC-12 | `graph-eng-fork-dialog.html` — picker + prefilled inputs + validation error | fork dialog fixtures | 1440×900 | normative | None |
| VC-13 | `graph-eng-editor-route-ask.html` — palette + route/ask/strategy/review inspectors + lint dock | `/loops/$name/editor` fixtures | 1440×900 | normative | None |
| VC-14 | same — detail DAG glyph column (ask/route/strategy summary) | `/loops/$name` detail fixture | 1440×900 | normative | None |
| VC-15 | `graph-eng-editor-chrome.html` — calm default (both rails collapsed, full-bleed) + palette expanded with search active | `/loops/$name/editor` chrome fixtures | 1440×900 | normative | None |
| VC-16 | same — quick-add dialog, connection-drop picker with live edge, node context menu (editable + read-only sets) | editor interaction fixtures | 1440×900 | normative | None |

Evidence for each row: `.compozy/tasks/graph-eng/evidence/visual/task_08/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

**Reference parity binds visual language only.** The boards stage the locked `release-train` demo story (`docs/design/opendesign/graph-eng/DESIGN-NOTES.md`) — canvas nodes and palette rows are illustrative, never the kind inventory. The palette is **additive**: every existing node kind stays and `ask`/`route` join them (`lib/loop-palette.ts` + the Go DSL own the inventory); never remove, hide, or reorder existing kinds to match a board. Content truth comes from runtime fixtures (SD-007).

## Subtasks

- [x] 8.1 Visual contract: confirm the nine delivered `docs/design/opendesign/graph-eng/graph-eng-*.html` boards cover `_uiux.md` states; iterate in place if a required state is missing — do not regenerate
- [x] 8.2 Data layer: adapters for the seven operations + diff/fork/rerun/amend, `requestsByWorkspace` keys/options, SSE kinds, MSW fixtures
- [x] 8.3 S1/S4: request card + forms + decision bars + parked-panel wait kinds
- [x] 8.4 S2: timeline row families
- [x] 8.5 S3: strategy progress + partial + width aggregates
- [x] 8.6 S5: amend + rerun dialogs and verb affordance rules
- [x] 8.7 S6/S7/S8: inspect lineage, diff view (deep-linkable route), fork dialog
- [x] 8.8 S10/S11: editor palette/inspectors/lint dock + codec round-trip; detail DAG glyphs
- [x] 8.9 S12: chrome store + collapsible rails (palette-mode ladder, toolbar toggles, `[`/`]`), quick-add dialog (`a`/double-click, viewport-center placement), connection-drop picker (atomic node+edge), node context menu, multi-select visuals + bulk delete, custom edge (midpoint ✕, route pills) — including the ride-along fixes (`positionsDirty` on placed add, `onNodesDelete` edge pruning)
- [x] 8.10 Storybook/MSW and Playwright coverage for every assigned E2E contract

Visual bundles, the QA walk, and project-wide gates remain required workstream obligations. The accepted loop plan executes them after task 09 so the final composition cannot invalidate their evidence.

## Implementation Details

Reference `_uiux.md` (surface map + component plan + signal mapping), `_dx.md` (payload shapes the fixtures mirror), `_spec.md` Part II API section.

### Relevant Files

- `web/src/systems/loops/components/run-page/loop-run-needs-you-card.tsx:14-27` — request-bearing extension point
- `web/src/systems/loops/components/run-page/loop-run-parked-panels.tsx` (`WAIT_KIND_SENTENCE`), `loop-run-waits-rail.tsx`
- `web/src/systems/loops/components/run-page/loop-run-story-timeline.tsx:23-32` + `lib/loop-run-story*.ts`
- `web/src/systems/loops/components/run-page/loop-run-page-body.tsx:71-139`, `loop-run-inspect-sheet.tsx:35-45`
- `web/src/systems/loops/components/run-page/loop-node-row-actions.tsx`, `loop-node-control-dialog.tsx:36-45`
- `web/src/systems/loops/lib/loop-node-wait-payload.ts` — schema-driven form base
- `web/src/systems/loops/lib/query-keys.ts`, `query-options.ts`, `adapters/*`, `hooks/use-loop-actions.ts`, `use-loop-node-actions.ts`, `use-loop-stream.ts`, `lib/loop-events.ts`
- `web/src/systems/loops/components/editor/*` (palette, inspector, json-field, lint dock), `lib/codec.ts`, `components/detail/loop-body-dag.tsx:12-15`
- S12 chrome: `loop-editor.tsx:155` (grid + skeleton twin `:238`), `loop-editor-toolbar.tsx`, `loop-editor-palette.tsx:35` + `loop-editor-palette-menu.tsx:43` (mode ladder), `loop-editor-canvas.tsx` (edgeTypes, `screenToFlowPosition`, `onConnectEnd`), `hooks/loop-editor-store-draft.ts:14-18,86-106` (placed add + atomic node+edge), `hooks/use-loop-editor.ts:228-266` (removals + connect)
- S12 precedents: `web/src/systems/session/hooks/use-session-inspector-state.ts` (persisted-visibility store to mirror), `web/src/systems/network/components/shell/network-shell.tsx:86-91` + `channel-toolbar.tsx:136-148` (resize persistence + toggle button contract), `web/src/systems/os/hooks/use-os-shortcuts.ts` + `os/lib/window-manager-command-registry.ts` (shortcut collision authority — never bind ⌘-chords in the editor)
- `web/src/routes/_app/` — diff view route registration
- `packages/ui/src/index.ts` — reuse inventory (check before any new component)

### Dependent Files

- `web/src/generated/compozy-openapi.d.ts` — regenerated types from task_07
- `web/src/systems/loops/mocks/*`, `testing/*` — fixture surface
- `web/e2e/` Playwright suites + fixtures

### Related ADRs

- [ADR-001](adrs/adr-001.md), [ADR-002](adrs/adr-002.md), [ADR-004](adrs/adr-004.md) — surface semantics the UI renders truthfully

### References

- `analysis/02_analysis_graph-frameworks.md` §3.1–§3.3 — the HITL surface catalog (approve/edit/reject/respond decision bars, resume-safety rules) the request forms render
- `analysis/sim.md` mechanism 14 (`.resources/sim/apps/sim/executor/handlers/human-in-the-loop/human-in-the-loop-handler.ts:384-519`) — the pause payload → notification/deep-link fields the run page's request card mirrors
- `docs/design/opendesign/graph-eng/` — delivered visual contracts (`DESIGN-NOTES.md` + the nine boards this task binds); artboard CSS is a contract, never a stylesheet to import
- `docs/design/opendesign/herdr-parity/herdr.css` + `DESIGN.md` + `packages/ui/src/tokens.css` — the visual grammar authorities; `_uiux.md` signal mapping rules
- `analysis/sim-uiux.md` (C1–C18 adopt/adapt/skip calls + parity verdicts) and `analysis/s12-integration.md` (shortcut collision table, draft-store changes, owning test suites) — the S12 authorities
- S12 competitor paths (read before implementing): `.resources/sim/apps/sim/app/workspace/[workspaceId]/w/[workflowId]/workflow.tsx:3397-3486` + `…/components/connection-block-selector/connection-block-selector.tsx` (connection-drop picker), `…/components/panel/components/toolbar/toolbar.tsx:733-778` (palette search + arrow nav), `.resources/sim/apps/sim/stores/panel/store.ts:40-51` (persisted panel state), `.resources/sim/packages/workflow-renderer/src/edge/workflow-edge-view.tsx:262-285` (midpoint edge delete), `.resources/sim/packages/workflow-renderer/src/workflow-block/workflow-block-view.tsx:1108-1190` (per-branch rows + handles)

## Web/Docs Impact

- `web/`: this task. `packages/site`: `visual-editor.mdx` + `running.mdx` screenshots/prose for the new surfaces (co-ship).
- QA impact: UI-bearing → add `untested` scenarios for the request answer flow, diff view, fork dialog, and editor chrome (rail collapse/persistence + quick-add/connection-drop) in `docs/qa/scenarios/`; walk before completion (browser).

## Extensibility / Agent Manageability / Config Lifecycle

- none — checked: frontend-only consumption of daemon contracts; no extension/agent/config surface change.

## Deliverables

- S1–S8 + S10–S12 shipped on the merged design grammar with durable visual evidence; MSW/Playwright suites green
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**
- Every Visual Contract row has a durable passing `eng-ui-screenshot` evidence bundle **(REQUIRED)**

## Tests

- [x] E2E-020, E2E-021, E2E-022 — request forms (ask validation/submit; review proposed+edit; resolved/stale states)
- [x] E2E-023 — strategy progress partial/canceled-distinct/wide aggregates
- [x] E2E-024 — timeline row families
- [x] E2E-025 — diff view (generation + run compare)
- [x] E2E-026 — fork dialog prefill/start/validation
- [x] E2E-027 — amend dialog (no optimistic paint)
- [x] E2E-028 — rerun dialog + affordance absence rules
- [x] E2E-029 — editor round-trip + lint dock codes
- [x] E2E-031, E2E-032, E2E-033, E2E-034 — S12 editor chrome (rails collapse + persistence; quick-add guard/placement/`positionsDirty`; connection-drop atomic add; node menu + multi-select delete with edge pruning + midpoint edge delete)

## Success Criteria

- Every assigned test case implemented and passing; every VC row `PASS` with zero unresolved blocking divergence
- No decision/affordance renders that the daemon payload doesn't authorize (truthful-UI audit against fixtures)
- `make gate` passes (web lanes via repo-root Turbo)
