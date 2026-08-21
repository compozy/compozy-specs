---
status: in_progress
title: "Web Tasks calm default: exclusion, reveal filter & provenance"
type: frontend
complexity: medium
---

# Task 4: Web Tasks calm default: exclusion, reveal filter & provenance

## Overview

Delivers front 1's web half (`_uiux.md` S1-S3): the Tasks list/kanban/dashboard/inbox consume the server-owned calm default, a hidden-by-default ephemeral filter reveals loop records as distinguished plain-words rows linking to their run, and the task-detail page gains the loop provenance block. The id-regex/nesting machinery is deleted outright — provenance is structured fields end to end.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST consume the server default (no `include_loop` param) on list, kanban, dashboard, and inbox reads — zero client-side loop filtering; bucket/group counts become coherent by construction (US-001, US-003).
2. MUST add the reveal filter as a quiet, hidden-by-default control that is OFF on every navigation (ephemeral per context, US-002.AC-3); revealed records render visually distinguished (loop glyph, plain identity "loop_name · run" / "loop_name · round N · step X"), activation lands on `/loop-runs/<id>` — never a machine id as primary text.
3. MUST render the filter-scoped truthful empty state ("no loop records in this workspace") distinct from the generic empty state (US-002.EC-1), and the retention-degrade ("run no longer available") when `loop_name` is absent (US-002.EC-2).
4. MUST add the task-detail loop provenance block (S3): loop name, "Open run" link, round, step, item — plain words; the record is labeled a loop execution record; link back is mandatory (US-015.AC-2).
5. MUST DELETE (no fallback, no compat shim): `LOOP_CELL_TASK_ID`/`LOOP_COORDINATOR_TASK_ID` regexes + `parseLoopTaskId` + loop branches of `taskShortId` (`task-formatters.ts`); loop nesting in `buildTaskListTree` + `subtaskSummaryLabel` (`task-hierarchy.ts`); the `task-subtask-list.tsx` loop path (delete the component if loop-only). Vitest suites for deleted code are deleted with it — regressions move to the provenance renderer's suite.
6. MUST key TanStack Query caches by workspace + the reveal-filter state per existing convention (asserted in web tests); MSW handlers/fixtures updated to the new contract (`loop` object, `include_loop` param).
7. MUST retire `docs/qa/scenarios/TA-web-task-list-loop-subtask-nesting.md` (superseded by the exclusion contract) and flag the replacement scenarios per the QA impact line.
8. MUST reuse `@compozy/ui` primitives (check `packages/ui/src/index.ts` first); domain composites take domain-prefixed names in `web/src/systems/tasks/` (`TaskLoopProvenance`, revealed-row badge variant); pull every color/type/spacing value from tokens (`DESIGN.md`); state is never color-alone (icon + text pairing).
9. MUST run web validation through Turborepo from the repo root (`make bun-test` / `make gate`), never package-local.
</requirements>

## Visual Contract

Reference artboard (landed; binding at execution time): `docs/design/opendesign/loop-legibility/loop-legibility-tasks-list.html`. Semantic contract: `docs/design/opendesign/loop-legibility/DESIGN-NOTES.md`. States below are the `_uiux.md` S1 "States to design" list — one row per staged state. A missing board at execution time is a blocked-precondition.

| ID    | Reference artifact + state                                              | Implementation target + state                                | Viewport | Fidelity  | Authorized differences + authority |
| ----- | ----------------------------------------------------------------------- | ------------------------------------------------------------ | -------- | --------- | ---------------------------------- |
| VC-01 | `docs/design/opendesign/loop-legibility/loop-legibility-tasks-list.html` — default with active loop            | `/tasks` list — active run fixture, work items only           | 1440×900 | normative | Runtime data/copy own content (`COPY.md`, SD-007); `@compozy/ui` owns component identity |
| VC-02 | `docs/design/opendesign/loop-legibility/loop-legibility-tasks-list.html` — revealed with mixed records         | `/tasks` list — reveal filter on, coordinator + cells + work  | 1440×900 | normative | Same as VC-01                      |
| VC-03 | `docs/design/opendesign/loop-legibility/loop-legibility-tasks-list.html` — revealed-empty message              | `/tasks` list — reveal on, zero loop records                  | 1440×900 | normative | Same as VC-01                      |
| VC-04 | `docs/design/opendesign/loop-legibility/loop-legibility-tasks-list.html` — revealed row, run retention-deleted | `/tasks` list — revealed row with degraded provenance         | 1440×900 | normative | Same as VC-01                      |
| VC-05 | `docs/design/opendesign/loop-legibility/loop-legibility-tasks-list.html` — loop-only workspace true empty      | `/tasks` list — workspace with only loop records              | 1440×900 | normative | Same as VC-01                      |
| VC-06 | `docs/design/opendesign/loop-legibility/loop-legibility-tasks-list.html` — kanban default                      | `/tasks` kanban — roots are work items only                   | 1440×900 | normative | Same as VC-01                      |

Evidence for each row: `.compozy/tasks/loop-task-legibility/evidence/visual/task_04/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}` via `eng-ui-screenshot` (Visual Contract Mode — implementation-only captures are invalid evidence). The S3 task-detail provenance block follows VC-02's revealed-row grammar (no dedicated artboard per `_uiux.md`).

## Subtasks

- [x] 4.1 Bind list/kanban/dashboard/inbox reads to the server calm default (drop client loop handling; coherent counts)
- [x] 4.2 Reveal filter control (hidden-by-default, ephemeral per navigation) + query-key wiring
- [x] 4.3 Revealed-row rendering: loop glyph, plain identity, run link; empty + retention-degrade states
- [x] 4.4 Task-detail `TaskLoopProvenance` block (coordinator + cell variants, run-gone degrade)
- [x] 4.5 Delete targets: formatters regex, hierarchy nesting, subtask-list loop path (+ their frozen tests)
- [x] 4.6 MSW handlers/fixtures updated to the generated contract
- [x] 4.7 Implement assigned Vitest units (UT-040..042)
- [ ] 4.8 Implement assigned Playwright journeys (E2E-010, E2E-011, E2E-020) — E2E-010/011 full; E2E-020 covers the link half, its retention degrade has no public seam (see memory)
- [ ] 4.9 Capture the Visual Contract evidence bundles (all six rows)
- [x] 4.10 Retire the superseded nesting scenario + flag replacement scenarios (QA impact)

## Implementation Details

Follow `_spec.md` Part II: Impact Analysis (web tasks row + delete targets), Data Models (`LoopProvenance`), `_uiux.md` S1-S3 (default read contracts, states) and Component plan. Skills to activate: `eng-design` + `ui-craft` + `impeccable` (+ `eng-ui-screenshot` for the Visual Contract evidence); web conventions per `web/CLAUDE.md` (systems boundaries, query keys, SSE); tests per the co-located Vitest suites + the canonical Playwright tasks spec; completion per `deslop` + `cy-final-verify`.

Suite placement (from `_tests.md`): view-model units EXTEND the co-located lib suites in `web/src/systems/tasks`; browser journeys EXTEND the canonical tasks Playwright spec.

### Relevant Files

- `web/src/systems/tasks/lib/task-formatters.ts:73-111` — id-regex identity to DELETE.
- `web/src/systems/tasks/lib/task-hierarchy.ts:17-35` — loop nesting to DELETE.
- `web/src/systems/tasks/components/task-subtask-list.tsx:78-114` — loop summary path to DELETE (component removed if loop-only).
- `web/src/systems/tasks/lib/{task-catalog-filter.ts,tasks-list-filters.ts,task-list-query.ts,query-keys.ts,query-options.ts}` — filter/query plumbing gaining `include_loop` + cache keys.
- `web/src/systems/tasks/components/{tasks-list-surface.tsx,tasks-list-row.tsx,tasks-list-toolbar.tsx,tasks-kanban-board.tsx,task-card.tsx,task-properties-rail.tsx}` — list/kanban/row/detail-rail change sites (`task-properties-rail.tsx:97,270` renders `created_by.ref` as plain text today).
- `web/src/systems/tasks/adapters/tasks-catalog-api.ts` — typed fetch layer binding the generated contract.
- `web/src/systems/tasks/hooks/{use-tasks-page.ts,use-tasks.ts,use-task-detail-page.ts}` — page hooks.
- `web/src/systems/tasks/mocks/{handlers.ts,fixtures.ts,query-responses.ts}` — MSW contract mocks.
- `web/src/routes/_app/tasks.tsx` + `tasks.$id.tsx` — routes.
- `web/src/generated/compozy-openapi.d.ts` — generated types (from task_02).
- `packages/ui/src/index.ts` — primitive inventory (reuse before create).
- `docs/design/opendesign/loop-legibility/loop-legibility-tasks-list.html` — S1 visual contract (VC-01–06).
- `docs/design/opendesign/loop-legibility/DESIGN-NOTES.md` — locked semantic contract (chips, identity, reveal grammar).
- `web/e2e/__tests__/tasks.spec.ts` — canonical Playwright tasks spec to extend.

### Dependent Files

- `web/src/systems/tasks/components/__tests__/{tasks-list-surface.test.tsx,tasks-list-row.test.tsx}` + `lib/__tests__/tasks-list-filters.test.ts` + `hooks/__tests__/use-tasks-page.test.tsx` — suites extended; suites of deleted libs removed.
- `docs/qa/scenarios/TA-web-task-list-loop-subtask-nesting.md` — retired (superseded).
- `web/src/storybook/__tests__/web-storybook-msw-contract.test.ts` — MSW-vs-OpenAPI guard must stay green.

### Related ADRs

- [ADR-001: Loop execution records leave the Tasks surfaces by default](adrs/adr-001.md) — the surface contract.
- [ADR-002: One loop run page, two registers](adrs/adr-002.md) — revealed records link into the run page (the observability home).
- [ADR-004: Classification rides existing provenance columns](adrs/adr-004.md) — structured provenance replaces the id-regex.

## Deliverables

- Tasks surfaces on the server calm default with coherent counts; reveal filter + distinguished rows + truthful empty/degrade states
- Task-detail provenance block with mandatory run link
- Delete targets removed (regex/nesting/subtask loop path) with their frozen tests
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**
- Every Visual Contract row has a durable passing `eng-ui-screenshot` evidence bundle **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-040, UT-041, UT-042 — revealed-row plain identity + href; filter-scoped empty message; run-gone degrade
- [x] E2E-010 — active loop: list/kanban zero loop rows, coherent counts, dashboard excludes cells, loop-only true empty
- [x] E2E-011 — reveal flow: distinguished rows, navigation resets filter, cell click lands on `/loop-runs/<id>`, reveal-empty message
- [ ] E2E-020 — task detail of a revealed cell: provenance block + "Open run" authored; deleted-run degrade blocked — no public loop-run delete/retention verb exists

## Success Criteria

- Every assigned test case implemented and passing
- Zero loop rows in default Tasks during an active run (DOM-asserted); reveal is explicit per context and never persists
- `grep` finds no `parseLoopTaskId`/loop-id-regex references anywhere in `web/`
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
- `make gate` green on the task's diff (repo-root Turbo lanes)

### Web/Docs Impact

- `web/`: this task — `web/src/systems/tasks/**` (components, lib, hooks, adapters, mocks) + routes above; no other system touched (loops system untouched — task_05's claim).
- `packages/site`: none — checked surfaces: app UI change only; docs pages describing task listings were updated by task_02's generated CLI/API references; no hand-authored site page renders the web Tasks UI.
- QA impact: user-visible UI change → retire `TA-web-task-list-loop-subtask-nesting` and add content-addressed `untested` scenarios for "Tasks calm default + reveal filter (web)" and "task-detail loop provenance block"; flag only — the walk runs in the loop's QA phase.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: no extension points read the web Tasks UI; the underlying contract change shipped in task_02.
- Agent manageability: none in this task — agents use the task_02 surfaces; web-only presentation here.
- Config lifecycle: none — no `config.toml` keys; the reveal filter is ephemeral UI state by design (US-002.AC-3), deliberately not a persisted setting.
