---
status: completed
title: "Web hero path: catalog, run form, and loop detail Visual Contract"
type: frontend
complexity: medium
---

# Task 10: Web hero path — catalog, run form, and loop detail

## Overview

Brings the redesigned arrive-and-use hero path to Visual Contract parity: loops catalog (roster
and filters including `canceled`), run form (this form = `http`, required-input gating, no
`stop`), and loop detail (contract, reliability tiles, DAG, recent runs). Reuses status
pills/formatters from task_08 so lifecycle terminals stay one truth across run page and hero
surfaces (US-029 / ADR-018).

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- R1: MUST render the catalog with built-in/custom groups, search, status filter including
  `canceled`, Rows|Cards layout, truthful empty + clear-filters, and primary Start run
  (US-029 AC-1/EC-1/EC-4).
- R2: MUST render the run form as arrive-and-use: inputs open, limits folded-but-informative,
  Dry run + Start run, Start gated on required inputs, no `stop` control; Ways to start
  identifies this form as `http` (US-029 AC-2/AC-3/EC-2).
- R3: MUST render loop detail with Contract, reliability tiles (plain-language Spec 1 posture),
  steps DAG, and recent runs including `canceled` pill when present (US-029 AC-4).
- R4: MUST keep one-story continuity for the same loop entity across catalog → run form →
  detail (shared identity + status chrome; reuse `loop-formatters` / `LoopStatusPill` from
  task_08) (US-029 AC-5).
- R5: MUST NOT invent Start-binding authoring on detail/run-form beyond read-truth of declared
  `start[]` kinds; write path is Spec 3 (ADR-018).
- R6: MUST verify every Visual Contract row with durable `eng-ui-screenshot` evidence; frontend
  gates via Turbo from repo root only; reuse `@compozy/ui` primitives (no shadow).
</requirements>

## Visual Contract

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity  | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | --------- | ---------------------------------- |
| VC-H1 | `docs/design/opendesign/loops/loops-catalog.html` — populated listing | `/loops` — fixture catalog | 1440×900 | normative | Configure/history links may still point at `_done` surfaces not redesigned this pass |
| VC-H2 | `loop-run-form.html` — arrive-and-use compose | `/loops/$name/run` — required + optional inputs | 1440×900 | normative | No `stop`; this form = `http`; Start-lane authoring deferred (ADR-018) |
| VC-H3 | `loop-detail.html` — contract + reliability + DAG + recent runs | `/loops/$name` — detail fixture incl. `canceled` run | 1440×900 | normative | Reliability tiles are plain-language Spec 1 posture, not new metrics |
| VC-H4 | catalog ↔ form ↔ detail continuity (`software-delivery` canon) | same entity across three routes | 1440×900 | normative | Prototype copy/brand marks yield to COPY.md + runtime truth |

Evidence for each row: `.compozy/tasks/loop-node-lifecycle/evidence/visual/task_10/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 10.1 Catalog roster/filters/`canceled`/empty states + Start run CTA
- [x] 10.2 Run form arrive-and-use layout, required-input gate, `http` truth, no `stop`
- [x] 10.3 Loop detail Contract + reliability tiles + DAG + recent runs pills
- [x] 10.4 Shared formatter/pill reuse from task_08; continuity fixtures
- [x] 10.5 Mocks/stories for catalog/run-form/detail states
- [x] 10.6 Vitest WT-009..010 + Visual Contract evidence bundles
- [x] 10.7 Flag content-addressed QA scenario `catalog-runform-walk` as `untested`

## Implementation Details

Follow TechSpec Web/Docs Impact (hero path bullet) and ADR-018. Depends on task_08 for
`canceled`/lifecycle status chrome. Prefer composition over boolean prop proliferation on
existing catalog/run-form/detail components.

### Relevant Files

- `web/src/systems/loops/components/catalog/loop-catalog.tsx` — catalog shell
- `web/src/systems/loops/components/catalog/loop-catalog-filters.tsx` — status filter
- `web/src/systems/loops/components/catalog/loop-catalog-row.tsx` — row + pills
- `web/src/systems/loops/components/run-form/loop-run-form.tsx` — arrive-and-use form
- `web/src/systems/loops/hooks/use-loop-run-form.ts` — submit/gate
- `web/src/systems/loops/components/detail/loop-detail.tsx` — detail shell
- `web/src/systems/loops/components/detail/loop-contract-panel.tsx` — contract
- `web/src/systems/loops/components/detail/loop-recent-runs.tsx` — recent runs
- `web/src/systems/loops/components/detail/loop-body-dag.tsx` — steps DAG
- `web/src/systems/loops/lib/loop-formatters.ts` — status→tone (from task_08)
- `web/src/systems/loops/components/loop-status-pill.tsx` — shared pill
- `web/src/routes/_app/loops.tsx` — catalog route
- `web/src/routes/_app/loops.$name.run.tsx` — run form route
- `web/src/routes/_app/loops.$name.tsx` — detail route
- `docs/design/opendesign/loops/loops-catalog.html` — VC final
- `docs/design/opendesign/loops/loop-run-form.html` — VC final
- `docs/design/opendesign/loops/loop-detail.html` — VC final

### Dependent Files

- `web/src/systems/loops/index.ts` — barrel exports
- `web/src/systems/loops/mocks/{fixtures,handlers}.ts` — hero fixtures
- `packages/ui/src/index.ts` — primitive inventory

### Related ADRs

- [ADR-018: Web Visual Contract expansion](adrs/adr-018.md) — hero path in Spec 1 MVP
- [ADR-008: Cancel ≠ kill](adrs/adr-008.md) — `canceled` terminal chrome
- SD-007 (truthful UI); DESIGN-BACKLOG §2.3

## Web/Docs Impact

This IS the hero-path web impact. Docs impact: none here (task_11). QA impact below.

## Extensibility / Agent Manageability / Config Lifecycle

None — frontend consumption of existing list/start/detail APIs (checked: no new config keys,
no new native tools; start verbs remain task_07 surfaces).

## QA impact

Flag new `untested` content-addressed scenario: catalog-runform-walk (catalog → run form →
detail continuity + `canceled` roster); walked in task_13 with browser evidence. No new
Playwright visual suite — parity is VC evidence + QA walk.

## Skills

`react`, `tanstack`, `xstate-store`, `app-renderer-systems`, `storybook-stories`, `vitest`,
`eng-design`, `ui-craft`, `impeccable`, `eng-ui-screenshot` (verification), `tailwindcss`.

## References

- TechSpec Web/Docs Impact — hero path bullet + Visual Contract authority
- `docs/design/opendesign/loops/DESIGN-BACKLOG.md` §2.3
- `docs/design/opendesign/loops/DESIGN-LESSONS.md`
- `web/CLAUDE.md` — web-specific dispatch and gates

## Deliverables

- Hero path surfaces at Visual Contract parity (VC-H1..VC-H4 evidence)
  (`.compozy/tasks/loop-node-lifecycle/evidence/visual/task_10/…`)
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**
- QA scenario `catalog-runform-walk` flagged `untested`

## Tests

- [x] WT-009 — catalog roster/filters/`canceled`/empty + clear-filters
- [x] WT-010 — run form required-input gate, `http` truth, no `stop`

## Success Criteria

- Every assigned test case implemented and passing (Turbo from repo root)
- Every Visual Contract row has a durable passing evidence bundle
- Shared status chrome matches task_08 formatters; no invented `stop` or metrics
- `catalog-runform-walk` scenario flagged for task_13
