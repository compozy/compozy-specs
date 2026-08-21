---
status: pending
title: "Web loop run two-register redesign: DAG, roster & runs re-rank"
type: frontend
complexity: high
---

# Task 5: Web loop run two-register redesign: DAG, roster & runs re-rank

## Overview

Delivers front 4, the spec's largest UI lift (`_uiux.md` S4-S7): the run page becomes one page with two registers — a plain-language default read (briefing strip, unmissable needs-you, steps+rounds progress, durable narrated story, Usage rail) and an operator register one disclosure deeper (live run DAG, complete node roster with attempts, generation history, raw events). The runs roster re-ranks on the server-owned needs-you ordering, and the node inventory aligns vocabulary. Every view binds to task_03's reads — SSE accelerates, reads reconcile.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST rebuild the run-page default read as exactly four elements in order — briefing strip (served verdict: tone + headline + detail + inline unblock), needs-you cards (never collapse, lead the page), steps+rounds progress (fan-out rollup chips, attempts as metadata, round counter hidden on round 1), narrated story (durable timeline pages, meaning-register titles, coalesced chatter) — plus the retained Usage rail (tokens · cost · budget · rounds · duration) and About; everything else demotes behind one "Inspect" disclosure (ADR-002; SD-012). No machine ids or raw enums in the default register (DOM-asserted, E2E-012).
2. MUST render the served briefing verdict — the web never re-derives a different verdict (Safety Invariant 12); terminal runs lead with typed outcome + artifacts (partial labeled, pruned truthful).
3. MUST implement the operator register over task_03's roster read: (a) live run DAG — authored topology, per-node icon+text status chips, fan-out rollup counts, edge liveness toward active nodes, `pending` (calm, reachable) visually and semantically distinct from `not_taken` (durable route evidence, neutral-dim), auto-center on needs-you, node click → panel with attempts/next-retry/timing/error class/cancellation cause + "Open session"/"Open record"/"View child run" (valid post-terminal) + existing node verbs; (b) node roster — every node × round healthy included, columns state/attempt/duration/tokens·cost/session, gantt micro-bar, per-attempt disclosure, round filter, fan-out grouped with rollup; (c) generation history — per-round outcomes, scores only when the loop defines them, per-round usage, Compare/Fork preserved, crash-interrupted rounds truthful.
4. MUST make the story durable: pages from the timeline read (first page = newest window), older history backfills backward on demand, SSE attaches at `after_sequence=head_seq`, de-dupe by seq; DELETE the frame-buffer-as-history reliance (`MAX_STORY_FRAMES` stays a render cap only).
5. MUST re-rank the runs roster (S6) purely from the server-owned extended list read (`attention` + `progress`, pre-pagination ordering) — no client-side page sorting, no N+1 run reads; run id demotes to secondary text; columns Loop · Status/needs-you · Progress · Started · Duration; transport-degraded (connecting/offline) never renders as empty.
6. MUST keep the DAG a read-only observability surface in `web/src/systems/loops/` — reusing only pure layout/geometry utilities from the editor, zero authoring chrome, no editor imports beyond shared pure libs (ADR-003/005; `loop-body-dag.tsx` stays definition-only).
7. MUST meet the accessibility floor: state never color-alone (icon + text per chip, asserted via accessible names), complete keyboard paths for disclosure/DAG node selection/needs-you actions, `prefers-reduced-motion` unmounts the edge pulse (not paused, E2E-019).
8. MUST align S7 node-inventory vocabulary ("step" plain labels) without contract changes; rows keep deep links.
9. MUST reuse `@compozy/ui` primitives (inventory first: `packages/ui/src/index.ts`); new domain components per the `_uiux.md` component plan (`LoopRunBriefing`, `LoopRunNeedsYou`, `LoopRunStepsProgress`, `LoopRunStory`, `LoopRunDag`, `LoopNodeRoster`, `LoopNodePanel`, `LoopGenerationHistory`); tokens only (`DESIGN.md` — flat depth, signal palette as information); copy per `COPY.md` label maps (no enum text leaks, UT-044).
10. MUST update loops MSW handlers/fixtures to the generated contract and run all web validation through repo-root Turbo lanes.
</requirements>

## Visual Contract

Reference artboards (landed; binding at execution time) under `docs/design/opendesign/loop-legibility/`. Semantic contract: `docs/design/opendesign/loop-legibility/DESIGN-NOTES.md`. States are the `_uiux.md` S4/S5/S6 "States to design" lists — one row per staged state. A missing board at execution time is a blocked-precondition. Authorized differences (all rows): runtime truth owns data/metrics (SD-007), `COPY.md` owns copy, `@compozy/ui` owns component identity, host chrome stays (reference parity binds visual language only).

| ID    | Reference artifact + state                                                      | Implementation target + state                                  | Viewport | Fidelity  | Authorized differences + authority |
| ----- | -------------------------------------------------------------------------------- | --------------------------------------------------------------- | -------- | --------- | ---------------------------------- |
| VC-01 | `docs/design/opendesign/loop-legibility/loop-legibility-run-default.html` — running healthy                             | `/loop-runs/$runId` default read, active fixture                 | 1440×900 | normative | Standard (header above)            |
| VC-02 | `docs/design/opendesign/loop-legibility/loop-legibility-run-default.html` — queued/admission-parked with reason         | run page, queued-on-admission fixture                            | 1440×900 | normative | Standard                           |
| VC-03 | `docs/design/opendesign/loop-legibility/loop-legibility-run-default.html` — watching/dormant                            | run page, watching fixture                                       | 1440×900 | normative | Standard                           |
| VC-04 | `docs/design/opendesign/loop-legibility/loop-legibility-run-default.html` — terminal done with artifacts                | run page, done fixture (outcome + artifact links lead)           | 1440×900 | normative | Standard                           |
| VC-05 | `docs/design/opendesign/loop-legibility/loop-legibility-run-default.html` — terminal failed, signal uncollapsed         | run page, failed fixture, all disclosures collapsed              | 1440×900 | normative | Standard                           |
| VC-06 | `docs/design/opendesign/loop-legibility/loop-legibility-run-default.html` — terminal no-op                              | run page, no-op fixture (plain statement, no fake artifacts)     | 1440×900 | normative | Standard                           |
| VC-07 | `docs/design/opendesign/loop-legibility/loop-legibility-run-default.html` — canceled with actor + time                  | run page, canceled fixture                                       | 1440×900 | normative | Standard                           |
| VC-08 | `docs/design/opendesign/loop-legibility/loop-legibility-run-default.html` — partial outputs labeled                     | run page, failed-after-partial fixture                           | 1440×900 | normative | Standard                           |
| VC-09 | `docs/design/opendesign/loop-legibility/loop-legibility-run-default.html` — pruned artifact truthful note               | run page, pruned-blob fixture                                    | 1440×900 | normative | Standard                           |
| VC-10 | `docs/design/opendesign/loop-legibility/loop-legibility-run-default.html` — long-run story paging                       | run page, >500-event fixture, older history loads on demand      | 1440×900 | normative | Standard                           |
| VC-11 | `docs/design/opendesign/loop-legibility/loop-legibility-run-default.html` — fork/time-travel beat                       | run page, forked-run fixture (beat links related run)            | 1440×900 | normative | Standard                           |
| VC-12 | `docs/design/opendesign/loop-legibility/loop-legibility-run-default.html` — budget nearly-exhausted warning             | run page, high-budget-consumption fixture (warning tone)         | 1440×900 | normative | Standard                           |
| VC-13 | `docs/design/opendesign/loop-legibility/loop-legibility-needs-you.html` — single card anatomy (ask/asker/choices/expiry) | run page needs-you area, one open approval                       | 1440×900 | normative | Standard                           |
| VC-14 | `docs/design/opendesign/loop-legibility/loop-legibility-needs-you.html` — multiple needs-you ordered with count         | run page, approval + quarantine + request fixture                | 1440×900 | normative | Standard                           |
| VC-15 | `docs/design/opendesign/loop-legibility/loop-legibility-needs-you.html` — expiry stated plainly                         | run page, near-expiry request fixture                            | 1440×900 | normative | Standard                           |
| VC-16 | `docs/design/opendesign/loop-legibility/loop-legibility-needs-you.html` — resolved elsewhere (who answered)             | run page, request resolved via CLI while open                    | 1440×900 | normative | Standard                           |
| VC-17 | `docs/design/opendesign/loop-legibility/loop-legibility-run-dag.html` — running node + edge liveness                    | operator register DAG, active fixture                            | 1440×900 | normative | Standard                           |
| VC-18 | `docs/design/opendesign/loop-legibility/loop-legibility-run-dag.html` — terminal graph faithful                         | DAG, terminal fixture (final states, no last-live frame)         | 1440×900 | normative | Standard                           |
| VC-19 | `docs/design/opendesign/loop-legibility/loop-legibility-run-dag.html` — failed + quarantined node chips                 | DAG, failure fixture                                             | 1440×900 | normative | Standard                           |
| VC-20 | `docs/design/opendesign/loop-legibility/loop-legibility-run-dag.html` — pending (reachable) ≠ not-taken (route evidence) | DAG, routed fixture showing both states side by side             | 1440×900 | normative | Standard                           |
| VC-21 | `docs/design/opendesign/loop-legibility/loop-legibility-run-dag.html` — wide fan-out stays a rollup chip                | DAG, 100-item fan-out fixture                                    | 1440×900 | normative | Standard                           |
| VC-22 | `docs/design/opendesign/loop-legibility/loop-legibility-run-dag.html` — deep/wide graph navigation                      | DAG, deep-graph fixture (pan/scroll, activity locatable)         | 1440×900 | normative | Standard                           |
| VC-23 | `docs/design/opendesign/loop-legibility/loop-legibility-run-dag.html` — node panel (attempts/links/verbs)               | DAG node click → panel, multi-attempt fixture                    | 1440×900 | normative | Standard                           |
| VC-24 | `docs/design/opendesign/loop-legibility/loop-legibility-run-dag.html` — reduced-motion (pulse unmounted)                | DAG under `prefers-reduced-motion` emulation                     | 1440×900 | normative | Standard                           |
| VC-25 | `docs/design/opendesign/loop-legibility/loop-legibility-run-roster.html` — complete roster, healthy included            | operator register roster, mid-run fixture                        | 1440×900 | normative | Standard                           |
| VC-26 | `docs/design/opendesign/loop-legibility/loop-legibility-run-roster.html` — multi-attempt node + next-retry time         | roster row disclosure, 10-attempt fixture                        | 1440×900 | normative | Standard                           |
| VC-27 | `docs/design/opendesign/loop-legibility/loop-legibility-run-roster.html` — strategy-canceled ≠ operator-canceled        | roster, both cancel dispositions staged                          | 1440×900 | normative | Standard                           |
| VC-28 | `docs/design/opendesign/loop-legibility/loop-legibility-run-roster.html` — zero-action-node run truthful                | roster, terminal-before-round-1 fixture                          | 1440×900 | normative | Standard                           |
| VC-29 | `docs/design/opendesign/loop-legibility/loop-legibility-run-roster.html` — fan-out grouped under node with rollup       | roster, fan-out fixture                                          | 1440×900 | normative | Standard                           |
| VC-30 | `docs/design/opendesign/loop-legibility/loop-legibility-run-roster.html` — generation history (scored + unscored)       | history view, multi-round fixture with and without scoring       | 1440×900 | normative | Standard                           |
| VC-31 | `docs/design/opendesign/loop-legibility/loop-legibility-run-roster.html` — crash-interrupted round true partial         | history view, recovered-run fixture                              | 1440×900 | normative | Standard                           |
| VC-32 | `docs/design/opendesign/loop-legibility/loop-legibility-run-roster.html` — pruned session link degrade                  | node panel, pruned-session fixture                               | 1440×900 | normative | Standard                           |
| VC-33 | `docs/design/opendesign/loop-legibility/loop-legibility-runs-roster.html` — needs-you first and distinct                | `/loop-runs` roster, mixed-state fixture                         | 1440×900 | normative | Standard                           |
| VC-34 | `docs/design/opendesign/loop-legibility/loop-legibility-runs-roster.html` — empty roster explains start                 | `/loop-runs`, fresh workspace                                    | 1440×900 | normative | Standard                           |
| VC-35 | `docs/design/opendesign/loop-legibility/loop-legibility-runs-roster.html` — dozens-active scale                         | `/loop-runs`, 30-run fixture                                     | 1440×900 | normative | Standard                           |
| VC-36 | `docs/design/opendesign/loop-legibility/loop-legibility-runs-roster.html` — transport-degraded ≠ empty                  | `/loop-runs`, connection-lost emulation                          | 1440×900 | normative | Standard                           |

Evidence for each row: `.compozy/tasks/loop-task-legibility/evidence/visual/task_05/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}` via `eng-ui-screenshot` (Visual Contract Mode — implementation-only captures are invalid evidence).

## Subtasks

- [ ] 5.1 Default read composition: `LoopRunBriefing` strip + `LoopRunNeedsYou` cards + `LoopRunStepsProgress` + retained Usage/About rail; demote everything else behind "Inspect"
- [ ] 5.2 `LoopRunStory`: durable timeline paging (newest window + backward backfill) + SSE seam at `head_seq` + meaning-register beat mapper; delete frame-buffer-as-history
- [ ] 5.3 `LoopRunDag`: read-only renderer (pure layout utils only), status chips, edge liveness, pending≠not-taken, rollups, auto-center, reduced-motion unmount
- [ ] 5.4 `LoopNodePanel`: attempts history, next-retry, error class + cancellation cause/actor, session/record/child-run links (post-terminal valid), node verbs
- [ ] 5.5 `LoopNodeRoster`: node×round table, gantt micro-bar, tokens·cost, round filter, fan-out grouping
- [ ] 5.6 `LoopGenerationHistory`: per-round outcomes/scores/usage, Compare/Fork preserved, crash-partial truthful
- [ ] 5.7 Runs roster re-rank from the server-owned read (columns, needs-you grouping, id demotion, degraded-transport state)
- [ ] 5.8 Node inventory vocabulary alignment (S7, light)
- [ ] 5.9 Loops MSW handlers/fixtures updated to the generated contract
- [ ] 5.10 Implement assigned Vitest units (UT-035..038, UT-043..047)
- [ ] 5.11 Implement assigned Playwright journeys (E2E-012..019)
- [ ] 5.12 Capture the Visual Contract evidence bundles (all 36 rows)
- [ ] 5.13 Flag QA scenarios per the QA impact line

## Implementation Details

Follow `_spec.md` Part II: System Architecture (web two-register row), Impact Analysis (loops row + story delete target), Technical Considerations (DAG renderer, story hard cut), `_uiux.md` S4-S7 + Component plan + Signal & state mapping. Skills to activate: `eng-design` + `ui-craft` + `impeccable` + `eng-ui-screenshot` (visual references are named — Visual Contract Mode required); dimension deep-dives as scoped (`better-accessibility` for keyboard/reduced-motion paths); web conventions per `web/CLAUDE.md`; completion per `deslop` + `cy-final-verify`. Signal palette final call happens in the design pass — implement from the artboards + `DESIGN.md` tokens, never invent tones.

Suite placement (from `_tests.md`): view-model units EXTEND the co-located lib suites in `web/src/systems/loops`; browser journeys EXTEND the canonical loop-run Playwright spec (one canonical spec for the redesigned page).

### Relevant Files

- `web/src/systems/loops/components/run-page/loop-run-page-body.tsx` — today's 9+ section cockpit; the composition root this task re-registers.
- `web/src/systems/loops/components/run-page/{loop-run-now-card.tsx,loop-run-needs-you-card.tsx,loop-run-outcome-card.tsx,loop-run-story-timeline.tsx,loop-run-progress-panel.tsx,loop-run-parked-panels.tsx,loop-run-usage-rail.tsx,loop-run-about-rail.tsx,loop-run-waits-rail.tsx,loop-run-inspect-sections.tsx}` — existing surfaces to re-register/demote (`loop-run-now-card.tsx:258-268` task-run link is hero-only/live-only today).
- `web/src/systems/loops/lib/{loop-events.ts,loop-node-lifecycle.ts,loop-run-progress.ts,loop-run-story.ts,loop-generation-presentation.ts,loop-node-now-view.ts,loop-formatters.ts}` — view-model layer (`loop-events.ts:139-152` frame buffer = delete target; `loop-node-lifecycle.ts:195` exception-only rows; `loop-run-progress.ts:153-201` widest-fan-out-only progress).
- `web/src/systems/loops/os/apps/loops/use-loop-run-page.ts:99-148` — page hook binding SSE + reads.
- `web/src/systems/loops/hooks/{use-loop-stream.ts,use-loops.ts,use-loop-actions.ts,use-loop-node-inventory.ts,use-loop-requests.ts}` — SSE + query hooks.
- `web/src/systems/loops/adapters/{loops-runs-api.ts,loop-nodes-api.ts,loop-requests-api.ts}` — fetch layer binding task_03's reads.
- `web/src/systems/loops/components/runs/{loop-runs-view.tsx,loop-runs-table.tsx,loop-run-row.tsx,loop-runs-filters.tsx,loop-runs-kpis.tsx,loop-node-inventory-view.tsx}` — S6 roster + S7 inventory.
- `web/src/systems/loops/components/detail/loop-body-dag.tsx:16-46` — static definition DAG (stays definition-only; run DAG is new).
- `web/src/systems/loops/mocks/{handlers.ts,fixtures.ts,fixture-run-details.ts}` — MSW contract mocks.
- `web/src/routes/_app/{loop-runs.tsx,loop-runs.$runId.tsx}` — routes.
- `web/src/generated/compozy-openapi.d.ts` — generated types (from task_03).
- `packages/ui/src/index.ts` — primitive inventory (reuse before create).
- `web/e2e/__tests__/loops.spec.ts` — canonical Playwright loops spec to extend.
- `docs/qa/scenarios/LP-web-detail-inventory-contract.md` — S7's frozen contract (must stay green).
- `docs/design/opendesign/loop-legibility/DESIGN-NOTES.md` + `docs/design/opendesign/loop-legibility/loop-legibility.css` — this set's locked semantic contract and domain CSS.
- `docs/design/opendesign/loop-legibility/loop-legibility-run-default.html` + `docs/design/opendesign/loop-legibility/loop-legibility-needs-you.html` — S4 visual contracts (VC-01–16).
- `docs/design/opendesign/loop-legibility/loop-legibility-run-dag.html` + `docs/design/opendesign/loop-legibility/loop-legibility-run-roster.html` — S5 visual contracts (VC-17–32).
- `docs/design/opendesign/loop-legibility/loop-legibility-runs-roster.html` — S6 visual contract (VC-33–36).
- `docs/design/opendesign/graph-eng/DESIGN-NOTES.md` + `docs/design/opendesign/graph-eng/graph-eng.css` — incumbent locked request grammar on the run page; the loop-legibility artboards resolve needs-you tones inside this lock (per `_uiux.md` Incumbent grammar).
- `web/CLAUDE.md` — systems/query/SSE conventions binding this work.

### Dependent Files

- `web/src/systems/loops/components/__tests__/{loop-run-page.test.tsx,loop-runs-view.test.tsx}` + `hooks/__tests__/use-loop-stream.test.tsx` — suites extended.
- `web/src/storybook/__tests__/web-storybook-msw-contract.test.ts` — MSW-vs-OpenAPI guard must stay green.
- `docs/qa/bugs/BUG-20260719-autonomous-progress-unobservable.md` — the P1 this task closes (verify + link in the QA phase).

### Competitor References

- `.resources/smithers/apps/cli/src/monitor-ui/monitorModel.ts:506,685,285` — verdict cascade, event tiers, attempt sentences (briefing strip + story register).
- `.resources/smithers/packages/ui-core/src/runs/{runProgress,runEta,runHealth}.ts` — truthful-metrics types (`ratio: null`) for progress/usage rendering.
- `.resources/smithers/packages/gateway-ui/src/runNodeStatus.ts` — latest-execution-wins status merging (current-state truth on chips).
- `.resources/mastra/packages/playground/src/domains/workflows/workflow/use-workflow-graph-runtime.tsx` — edge=data-flowed truth model (DAG edge lighting).
- `.resources/mastra/packages/playground/src/domains/workflows/workflow/workflow-suspended-steps.tsx` — HITL card register (needs-you anatomy).
- `.resources/mastra/packages/playground/src/domains/workflows/context/workflow-run-provider.tsx` — stream + conditional snapshot poll reconciliation (SSE accelerates, reads reconcile).
- `.resources/sim/apps/sim/app/workspace/[workspaceId]/logs/components/log-details/components/trace-view/trace-view.tsx` — roster gantt micro-bar, jump-to-error, leaf-most-failure selection.
- `.resources/sim/packages/workflow-renderer/src/edge/workflow-edge-view.tsx` — edge liveness + reduced-motion unmount.

### Related ADRs

- [ADR-002: One loop run page, two registers via progressive disclosure](adrs/adr-002.md) — the page contract.
- [ADR-003: Run-bound live DAG enters scope](adrs/adr-003.md) — the DAG surface decision.
- [ADR-005: Run reads are computed projections](adrs/adr-005.md) — one source, several projections; SSE accelerates.

## Deliverables

- Run page passing the 30-second briefing test in the default register, operator depth one disclosure deeper (DAG/roster/history), durable story
- Runs roster re-ranked from the server-owned read; node inventory vocabulary aligned
- Loops MSW fixtures on the generated contract; frame-buffer-as-history deleted
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**
- Every Visual Contract row has a durable passing `eng-ui-screenshot` evidence bundle **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-035, UT-036, UT-037, UT-038 — steps-progress view-model (control-only rounds, round-1 hiding, not-taken exclusion + neutral glyph, all-parked reason)
- [ ] UT-043, UT-044, UT-045, UT-046, UT-047 — pruned-artifact note; story beat mapper (meaning titles, no enum leaks, ×N coalescing); runs-roster grouping + empty state; DAG chip icon+text per state; node panel links incl. pruned degrade and not-taken no-links
- [ ] E2E-012 — briefing/needs-you/progress/usage visible without disclosure; zero `loop.`/`looprun-` ids in the default read (DOM assertion)
- [ ] E2E-013 — approval flow in place; two needs-you ordered with count; CLI-resolved request clears live
- [ ] E2E-014 — failed leads uncollapsed; done leads with outcome + artifacts; canceled shows actor
- [ ] E2E-015 — >500-event run: reload + scroll to run start pages full history, no missing tail
- [ ] E2E-016 — operator register DAG states (pulse, succeeded, failed, not-taken neutral, rollup chip); node panel links work on a terminal run
- [ ] E2E-017 — roster healthy+parked with attempts and tokens/cost; generation history outcomes + usage; requeue from panel updates live
- [ ] E2E-018 — runs roster: needs-you first and distinct, plain outcomes, empty state
- [ ] E2E-019 — reduced-motion: edge pulse unmounted; every chip carries accessible text+icon

## Success Criteria

- Every assigned test case implemented and passing
- Briefing test: status, needs-you, progress, spend, and outcome readable in the default register with all disclosures collapsed (E2E-012/014 green)
- DAG, roster, steps, and story render from the same reads with no derived-state divergence; reload never loses story history
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
- `make gate` green on the task's diff (repo-root Turbo lanes)

### Web/Docs Impact

- `web/`: this task — `web/src/systems/loops/**` (run-page + runs components, lib view-models, hooks, adapters, mocks) + routes above; `web/src/systems/tasks` untouched (task_04's claim).
- `packages/site`: none — checked surfaces: app UI change only; CLI/API docs shipped with task_03's codegen; no hand-authored site page renders the run page UI.
- QA impact: user-visible UI redesign → reset `docs/qa/scenarios/LP-web-detail-inventory-contract.md` to `untested` (S7 vocabulary touch) and any existing run-page scenario ids touched by the redesign (e.g. `LP-web-attention-loop-rows.md`, `LP-loop-run-deep-link.md`); add content-addressed `untested` scenarios for "run-page default read briefing test" and "operator register DAG/roster"; flag only — the walk runs in the loop's QA phase (also re-verify `BUG-20260719-autonomous-progress-unobservable` as fixed).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: presentation over task_03's reads; no extension points render or consume the web run page.
- Agent manageability: none in this task — agents consume task_03's structured surfaces (verb parity already guaranteed there).
- Config lifecycle: none — no `config.toml` keys (checked `internal/config`; register disclosure is in-page state, deliberately not a persisted setting per ADR-002).
