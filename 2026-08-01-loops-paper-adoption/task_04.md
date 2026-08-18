---
status: completed
title: "Web loop system: scores, best and provenance"
type: frontend
complexity: medium
---

# Task 4: Web loop system: scores, best and provenance

## Overview

Brings the ratchet and provenance to the operator UI: the `loop-events.ts` reducer migrates to the
new `gate_verdict` payload (score, best), the run detail surfaces score/best/origin truthfully,
and the outcome card points exhausted/stalled runs at the best generation. Truthful UI only —
every rendered value exists in the contract.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST activate skills: `react`, `tanstack`, `eng-design`, `ui-craft`, `impeccable`, `eng-ui-screenshot`, `storybook-stories`, `vitest`, `tailwindcss` (+ `web/CLAUDE.md` dispatch).
- MUST migrate `web/src/systems/loops/lib/loop-events.ts` and types to the new `gate_verdict`/`generation_started` payloads; the legacy `confidence` field is deleted from types, reducer, story rows, mocks, and tests (delete target — no dual read).
- MUST render: score value + best badge per generation, origin chip ("restored from gen N" for `ratchet_restore`), summary best fields on run rows, and best-generation pointer on exhausted/stalled outcome cards — all from generated types, no invented metrics (SD-007).
- MUST reuse `@compozy/ui` primitives (check `packages/ui/src/index.ts` first); tokens only from `tokens.css`/`DESIGN.md`; signal palette is informational.
- MUST add/refresh Storybook stories for the new states and verify with `eng-ui-screenshot` captures cited in completion notes.
- MUST run frontend gates through Turbo from repo root (`make gate` / `make bun-test`), never package-local.
</requirements>

## Subtasks

- [x] 4.1 Regenerate/consume TS types; delete `confidence` from `web/src/systems/loops/types.ts` and all consumers.
- [x] 4.2 Migrate the `loop-events.ts` reducer + `use-loop-stream.ts` handling; update mocks/fixtures.
- [x] 4.3 Run detail: per-generation score/best/origin rendering (`components/detail/`, `components/runs/`, goal-turn timeline row labels).
- [x] 4.4 Outcome card best pointer + run-summary best fields (`loop-run-outcome-card.tsx`, `loop-recent-runs.tsx`, catalog row).
- [x] 4.5 Stories for: scored generation, best badge, restore chip, no-metric run (unchanged), exhausted→best.
- [x] 4.6 Implement UT-042 (vitest) + E2E-006 (Playwright, no `force: true`); capture `eng-ui-screenshot` evidence.

## Implementation Details

TechSpec sections: Web/Docs Impact, API Endpoints (payload shapes). Reducer contract: UT-036's
payload is the single source for UT-042 expectations.

### Relevant Files

- `web/src/systems/loops/lib/loop-events.ts:272` — `gate_verdict` reducer case (migration center)
- `web/src/systems/loops/hooks/use-loop-stream.ts` + `hooks/__tests__/use-loop-stream.test.tsx` — canonical reducer suite
- `web/src/systems/loops/types.ts:31` — `LoopRunGeneration` shape
- `web/src/systems/loops/components/detail/loop-detail.tsx:226-227`, `components/runs/*`, `goal-turn-timeline.tsx:84` — rendering points
- `web/src/systems/loops/components/runs/loop-run-outcome-card.tsx:41`, `loop-run-now-card.tsx:43-55`, `catalog/loop-catalog-row.tsx:76-77`, `loop-recent-runs.tsx:49` — summary/outcome surfaces
- `web/src/systems/loops/lib/loop-run-story-rows.ts` + `mocks/` — fixtures to migrate

### Dependent Files

- `web/src/generated/compozy-openapi.d.ts`, `web/src/generated/loop-enums.ts` — regenerated in task_03
- `web/e2e/__tests__/` — E2E-006 spec home

### Related ADRs

- [ADR-003](adrs/adr-003.md) — what best/score mean (render semantics); [ADR-004](adrs/adr-004.md) — origin vocabulary for chips.

### Web/Docs Impact

- `web/`: this task IS the web impact (paths above).
- `packages/site`: none — docs in task_05; checked: no MDX consumes these components.
- QA impact: UI-visible changes; walked in task_07 via new `LP-ratchet-*` scenarios + reset `LP-loop-run-deep-link` (run page rendering changed).

### Extensibility / Agent Manageability / Config Lifecycle

- none — checked surfaces: no extension points, no CLI/HTTP/UDS changes (frontend consumes task_03 contract), no config keys. Reason: presentational layer over the already-shipped read model.

## Deliverables

- Migrated reducer + truthful score/best/provenance UI with Storybook coverage.
- `eng-ui-screenshot` captures cited for every new visual state (stories list in 4.5).
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-042 — reducer consumes migrated payload; `confidence` structurally gone
- [x] E2E-006 — run detail renders score, best badge, restore chip, exhausted→best link (role/text assertions)

## Success Criteria

- Every assigned test case implemented and passing
- `make gate` green from repo root (Turbo lanes); zero oxlint warnings
- Screenshot evidence cited for each new state; no invented controls/metrics (daemon truth only)
- `rg -i 'confidence' web/src/systems/loops` returns nothing

## Completion Evidence

- Gate: `make gate` escalated to the full monorepo gate and passed at fingerprint
  `c8260cfeffe22e16c40420ecb2e8d18930209b67`; log
  `.cache/gate/logs/full-1785588859.log`.
- UT-042 and presentation suites: the final full gate passed Bun lint, typecheck, tests, and build;
  the focused story regression passed 14/14 tests.
- E2E-006: focused real-daemon Playwright run passed 1/1 in 6.3s with visible-text and role
  assertions; no `force: true`.
- Hard cut: `rg -i 'confidence' web/src/systems/loops` returned no matches.
- React Doctor: changed-scope scan passed at 100/100 with no issues.
- Visual evidence (Storybook iframe, 1440x900, inspected):
  - `.compozy/tasks/loops-paper-adoption/evidence/task_04/scored-best-generation.png`
  - `.compozy/tasks/loops-paper-adoption/evidence/task_04/ratchet-restore.png`
  - `.compozy/tasks/loops-paper-adoption/evidence/task_04/no-metric.png`
  - `.compozy/tasks/loops-paper-adoption/evidence/task_04/exhausted-best-generation.png`
- Process cleanup: owned Storybook PIDs exited cleanly; `make qa-reap` reported
  `TEARDOWN_CLEAN=true` after the Playwright run.
- Full web-lane diagnostic: 97 tests passed and 19 failed. The Task 04 failure exposed duplicate
  generation markers and was fixed; the final focused E2E passes. The remaining cross-surface
  failures are retained as an explicit Task 07 fresh-lab rerun risk.

## Compozy Impact Audit

- Native tools: no Task 04 change — checked `compozy__loop_status` and `compozy__loop_runs`
  descriptors/schemas; this UI consumes the Task 03 contract without changing tool IDs, gates,
  digests, risk flags, or fallbacks.
- Extensibility and hooks: no Task 04 change — checked Loop hook payloads, extension scorers,
  bundles, registries, bridge SDKs, MCP sidecars, and config lifecycle; the reducer only consumes
  the already-shipped `score`, `best_generation`, `origin`, and `parent_generation` fields.
- Workspace data isolation: workspace-scoped read-only presentation — existing workspace-aware
  HTTP/SSE/query-cache paths remain unchanged; no global/session/agent datum or cross-workspace
  cache/event path was added.
- Official Compozy skill: no Task 04 change — checked `skills/compozy/references/loops.md`; Task 03
  already documented the public score/best/provenance contract consumed here.
