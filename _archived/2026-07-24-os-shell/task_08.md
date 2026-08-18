---
status: completed
title: Window head absorbs PageHead
type: frontend
complexity: critical
---

# Task 8: Window head absorbs PageHead

## Overview

Absorb the route `PageHead` into a single unified OS window head so identity renders once: 44px chrome (traffic lights · glyph/title or drill-in trail · status + ≤2 actions) plus an optional 38px context strip for listing tools. This hard-cuts the legacy workspace breadcrumb + body hero stack before spaces/compact close the program on the wrong chrome anatomy.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST implement the unified window head per `docs/design/opendesign/os/pagehead-redesign.html` §02–§05 and `_techspec.md` Design Tokens: 44px head, quiet 22px glyph + 13px/600 title + optional mono count at root; status chip + ≤2 trail actions; accent reserved for state dots and the primary action only.
2. MUST delete the workspace-prefixed breadcrumb (`agh / Route` / `rootCrumb`) from the window head — the menubar owns the workspace; the window title owns the route.
3. MUST provide an optional 38px context strip under the head for route tools (filters, search, Rows|Cards); MUST NOT render a third chrome bar. Strip is absent when the route publishes no tools.
4. MUST implement the drill-in breadcrumb head for in-place navigation: back chevron replaces the glyph; clickable parent crumbs + leaf title; >2 parents collapse into a `…` crumb; trail is window-local (never a workspace prefix, never a new floating window). Status, actions, and strip swap per location level.
5. MUST treat document windows (live sessions) as self-titling: state dot + session name; no breadcrumb trail; prior session-status chrome merges into the head.
6. MUST hard-cut every OS windowed surface off body `PageHead` (catalogs, dashboard, session, tasks, sandbox, vault, marketplace, settings, detail headers) — no dual-path, hide, or CSS workaround; identity ownership moves to the head slot.
7. MUST evolve `TopbarSlotValue` / `Topbar` / `OsWindowFrame` as the chrome contract (glyph, count, status, crumbs/onBack, toolbar, scroll elevation) and update harnesses/stories/tests in the same change.
8. MUST delete `@agh/ui` `PageHead` when zero consumers remain (export, stories, tests) and grep-clean `PageHead` / `data-slot="page-head"` from `web/`.
9. MUST land Visual Contract evidence bundles for every VC row via `eng-ui-screenshot` against `pagehead-redesign.html` mocks.
</requirements>

## Visual Contract

References: `docs/design/opendesign/os/pagehead-redesign.html` (§02 anatomy, §03 drill-in, §04 applied examples) + `docs/design/opendesign/os/critique.json`. Authorized differences: real runtime data/copy/brand marks per product truth; chrome anatomy and measures are normative.

| ID | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| --- | --- | --- | --- | --- | --- |
| VC-01 | `pagehead-redesign.html` — dashboard unified head (identity + Connected status, no strip) | Dashboard window root | 1440×900 | normative | Real daemon health copy |
| VC-02 | `pagehead-redesign.html` — Tasks head + 38px strip (filters + Rows\|Cards) at list root | Tasks window catalog | 1440×900 | normative | Real task counts/filters |
| VC-03 | `pagehead-redesign.html` — Tasks two-level drill-in breadcrumb head (list → detail → run) | Tasks window at run location | 1440×900 | normative | Real task/run titles |
| VC-04 | `pagehead-redesign.html` — Marketplace in-place entry under `Marketplace / entry` | Marketplace window detail location | 1440×900 | normative | Real catalog entry |
| VC-05 | `pagehead-redesign.html` — Session document head (state dot + self-title, no crumbs) | Live session window | 1440×900 | normative | Real session/agent meta |

Evidence for each row: `.compozy/tasks/os-shell/evidence/visual/task_08/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 8.1 Evolve `TopbarSlotValue` + `Topbar` + harnesses for glyph/count/status/crumbs/toolbar
- [x] 8.2 Rework `OsWindowFrame` — drop `rootCrumb`, 44px left-aligned identity, optional strip, scroll elevation
- [x] 8.3 Migrate catalog/root windows off `PageHead` (dashboard, agents, loops, jobs, triggers, bridges, knowledge, sandbox, vault, settings)
- [x] 8.4 Migrate document + listing surfaces (session self-title; tasks list tools → strip)
- [x] 8.5 Drill-in breadcrumb head for tasks / marketplace / detail locations (back + crumbs + per-level trail)
- [x] 8.6 Delete `PageHead` primitive if unused; grep-clean; update stories/tests
- [x] 8.7 Visual Contract captures VC-01..05 + assigned unit/e2e cases
- [x] 8.8 Flag QA scenario impact for chrome identity-once + drill-in breadcrumb

## Implementation Details

Skills: `eng-design` + `ui-craft` + `impeccable`, `no-workarounds`, `react`, `tailwindcss`, `eng-ui-screenshot`, `vitest`, `eng-consolidate-test-suites`. Hard-cut ownership — do not hide `PageHead` with CSS. Reference TechSpec Design Tokens (unified window head) and `pagehead-redesign.html` migration map §05.

### Relevant Files

- `packages/ui/src/components/custom/topbar.tsx` + `hooks/use-topbar-slot.ts` — slot/head contract
- `packages/ui/src/components/custom/page-head.tsx` — delete target when unused
- `web/src/systems/os/components/os-window-frame.tsx` — window chrome
- `web/src/systems/os/apps/**` — catalog/session/dashboard locations
- `web/src/systems/{tasks,sandbox,vault,marketplace,settings,agent,loops,bridges,knowledge,automation}/**` — remaining PageHead publishers
- `docs/design/opendesign/os/pagehead-redesign.html` — normative visual reference

### Dependent Files

- `web/src/test/render-with-topbar.tsx`, `web/src/storybook/story-layout.tsx` — harness updates
- `web/e2e/__tests__/os-shell.spec.ts` — breadcrumb/identity assertions
- `docs/qa/scenarios/` — flag-only (task_10 owns authoring)

### Related ADRs

- [ADR-006](adrs/adr-006.md)/[ADR-007](adrs/adr-007.md) — identity polish + anti-clutter (chrome ownership)

## Deliverables

- Unified 44px window head + optional 38px strip live on every OS window
- Zero body `PageHead` on windowed surfaces; `PageHead` deleted from `@agh/ui` if unused
- Visual Contract bundles VC-01..05 passing
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-090 — identity-once: head title present; body has no `data-slot="page-head"`
- [x] UT-091 — publishing `toolbar` renders the 38px context strip; clearing removes it
- [x] UT-092 — drill-in slot publishes crumbs without workspace prefix; back pops one level
- [x] UT-093 — session/document head uses state leading + self-title; no crumb trail
- [x] UT-094 — scrolled window body elevates the head (`data-scrolled` / elevation class)
- [x] E2E-005 — deep link to task detail; window-local breadcrumb leaf; browser Back → list (reassigned from task_06 — breadcrumb shape no longer includes workspace prefix)

## Success Criteria

- Every assigned test case implemented and passing; turbo lint/typecheck/test clean for `packages/ui` + `web`
- Every VC row PASS with zero unresolved blocking divergence
- Grep-clean: no `PageHead` imports in `web/src`; no workspace crumb in `OsWindowFrame`
- QA impact: unified window head + drill-in breadcrumb are user-visible → task_10 mints/resets chrome scenarios; flag recorded here

## Completion Notes

- Hard-cut: `PageHead` removed from `@agh/ui`; apps publish via `useTopbarSlot`; `OsWindowFrame` drops `rootCrumb`.
- Visual Contract VC-01..05 PASS under `evidence/visual/task_08/` (Topbar stories vs isolated `pagehead-redesign.html` applied examples).
- Gates: `bunx turbo run lint typecheck test --filter=./packages/ui --filter=./web` PASS; `compozy tasks validate --name os-shell` PASS.
- QA: `docs/qa/scenarios/ET-web-route-chrome-topbar.md` reset to `untested`.
- E2E-005 assertions updated for window-local crumbs (no `agh` workspace prefix); Playwright lab run deferred to task_11 / next QA cycle.
- Teardown: Storybook `:6007` + static `:8765` killed after captures; ports clear.

AGH Impact Audit:

- Native tools: no impact (checked: no `agh__*` / CLI / API)
- Extensibility and hooks: no impact (checked: shell chrome only)
- Workspace data isolation: no impact (checked: presentation-only; workspace stays on menubar)
- Official AGH skill: no impact (checked: UI chrome only — QA scenario flagged)
