---
status: completed
title: "Web loops system: lifecycle states, controls and inventories"
type: frontend
complexity: medium
---

# Task 8: Web loops system — lifecycle states, controls and inventories

## Overview

Surfaces the run-time half of the Spec 1 contract truthfully in the web UI: the run page gains
attempt/retry visibility, pause/cancel/kill/requeue node controls bound to daemon truth,
quarantine entry inspection, waiting/attention badges and the `canceled` terminal; the runs area
gains waiting/quarantine/attention inventory views with age sorting and truthful empty states;
the SSE reducer and story pipeline fold the 15 new event kinds. Editor authoring and hero path
Visual Contract land in task_09 and task_10 (ADR-018).

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- R1: MUST fold every new event kind in `lib/loop-events.ts` (+`UNRETAINED_KINDS` choices) and
  build story rows for retry/pause/resume/cancel/kill/quarantine/requeue/wait/attention/
  effect-results/suppression in `lib/loop-run-story-rows.ts`.
- R2: MUST render node lifecycle state from payloads only (SD-007 daemon truth): attempt +
  next-attempt time on retrying nodes, pause/quarantine/attention provenance, wait age, cancel
  state; no control renders for a state the payload does not declare.
- R3: MUST add node control actions (pause/resume variants/cancel/kill/requeue) and run
  cancel/kill to `loop-run-controls.tsx` + row-level menus via the task_07 adapters, with
  deterministic error rendering (idempotent answers as informative, destructive confirms
  showing current state — PRD UI/UX).
- R4: MUST add inventory views (waiting/quarantine/attention) with filters, age sort,
  pagination, and truthful empty states under `components/runs/`; quarantine entry sheet shows
  the classified chain/hints/target/input ref.
- R5: MUST update mocks/fixtures/handlers + Storybook stories for every new state and verify
  with `eng-ui-screenshot` durable captures before completion; signal palette carries state
  semantics (information, never decoration; DESIGN.md grammar).
- R6: Frontend gates run through Turbo from repo root only.
- R7: MUST verify every Visual Contract row below with durable `eng-ui-screenshot` evidence;
  editor/catalog/run-form/detail artboards are out of this task's scope (task_09/task_10).
</requirements>

## Visual Contract

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity  | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | --------- | ---------------------------------- |
| VC-R1 | `docs/design/opendesign/loops/loop-run-detail.html` — live mid-retry (canonical timeline) | `/loop-runs/$runId` — retrying fixture | 1440×900 | normative | Collapse + summary gist per DESIGN-LESSONS L8; copy yields to COPY.md |
| VC-R2 | `loop-run-detail-states.html` — representative parked/attention/canceled states | run page state matrix fixtures | 1440×900 | normative | Enumerate shipped states from daemon truth only (SD-007) |
| VC-R3 | `loop-node-controls.html` — gated verbs + deterministic answers | run-page node/run controls | 1440×900 | normative | None |
| VC-R4 | `loop-quarantine-sheet.html` — two-episode quarantine + requeue | quarantine entry sheet | 1440×900 | normative | None |
| VC-R5 | `loop-inventories.html` — waiting/quarantined/attention/retrying + `canceled` pills | runs-area inventory views | 1440×900 | normative | Aggregation badges only where shell already surfaces attention |

Evidence for each row: `.compozy/tasks/loop-node-lifecycle/evidence/visual/task_08/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [ ] 8.1 Regenerated types consumption + reducer kinds + story rows
- [ ] 8.2 Run-page node lifecycle rendering (attempts, parked states, canceled terminal)
- [ ] 8.3 Node/run control actions + mutation hooks + deterministic error surfaces
- [ ] 8.4 Inventory views + quarantine entry sheet + badges/aggregation
- [ ] 8.5 Mocks/fixtures/stories for every new state
- [ ] 8.6 Vitest suites + Playwright E2E + screenshot evidence bundle

## Implementation Details

Follow TechSpec Web/Docs Impact. Key files from the surfaces exploration: `lib/loop-events.ts`
(309), `lib/loop-run-story-rows.ts` (334), `lib/loop-run-page-view.ts`, `lib/loop-run-progress.ts`
(parked-state exclusion), `components/run-page/loop-run-controls.tsx`,
`loop-run-needs-you-card.tsx`, `loop-run-outcome-card.tsx` (canceled), new inventory components
under `components/runs/`, `mocks/fixtures.ts` + `handlers.ts`, `adapters/loops-runs-api.ts`,
`hooks/use-loop-actions.ts`, `hooks/use-loop-stream.ts`.

### Relevant Files

- `web/src/systems/loops/lib/loop-events.ts` — SSE reducer (kind switch + retained ring)
- `web/src/systems/loops/lib/loop-run-story-rows.ts` — per-kind row builders
- `web/src/systems/loops/components/run-page/loop-run-controls.tsx` — verb buttons
- `web/src/systems/loops/lib/loop-formatters.ts` — status→pill tone (canceled)
- `web/src/systems/loops/mocks/{fixtures,handlers}.ts` — mock daemon incl. linter stand-in
- `web/src/generated/compozy-openapi.d.ts` — regenerated types (task_07)
- `packages/ui/src/index.ts` — primitive inventory (reuse before create)

### Dependent Files

- `web/src/systems/loops/index.ts` — barrel exports for new components
- `web/e2e/__tests__/` — Playwright spec home + shared selectors

### Related ADRs

- [ADR-018: Web Visual Contract expansion](adrs/adr-018.md) — run UI ownership vs editor/hero split
- PRD UI/UX considerations; SD-007 (truthful UI); grill defaults v2 (states to render).

## Web/Docs Impact

This IS the run-page/inventories web impact (editor → task_09; hero path → task_10). Docs
impact: none here (task_11). QA impact below.

## Extensibility / Agent Manageability / Config Lifecycle

None — frontend consumption of daemon truth only (checked: no config keys, no agent surfaces,
no extension points touched).

## QA impact

Flag new `untested` scenario: operator-lifecycle-ui (run page controls + inventories walk);
walked in task_13 with browser evidence.

## Skills

`react`, `tanstack`, `xstate-store`, `app-renderer-systems`, `storybook-stories`, `vitest`,
`eng-design`, `ui-craft`, `impeccable`, `eng-ui-screenshot` (verification), `tailwindcss`.

## References

- Visual Contract artboards for this task only: `loop-run-detail.html`,
  `loop-run-detail-states.html`, `loop-node-controls.html`, `loop-quarantine-sheet.html`,
  `loop-inventories.html` + `DESIGN-BACKLOG.md` §2.1 + `DESIGN-LESSONS.md` (approval recorded;
  editor/hero consumed by task_09/task_10)
- Surfaces exploration §10 (web file map with line counts)
- `web/CLAUDE.md` — web-specific dispatch and gates

## Deliverables

- Truthful lifecycle run UI + inventories with stories, mocks, and durable Visual Contract
  evidence for VC-R1..VC-R5
  (`.compozy/tasks/loop-node-lifecycle/evidence/visual/task_08/…`)
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [ ] WT-001 — reducer folds 15 kinds without dropping retained frames
- [ ] WT-002 — story rows render new states from fixtures
- [ ] WT-003 — progress derivation excludes parked states
- [ ] WT-004 — controls gate on daemon-truth state
- [ ] E2E-015 — run-page lifecycle + inventory browser journey

## Success Criteria

- Every assigned test case implemented and passing (Turbo from repo root)
- Every Visual Contract row has a durable passing evidence bundle
- No control or metric renders that the runtime payload does not model
