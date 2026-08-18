---
status: completed
title: Attention surfaces and multi-instance sessions
type: frontend
complexity: high
---

# Task 5: Attention surfaces and multi-instance sessions

## Overview

Deliver the shell's attention story and its headline capability: truthful dock badges (waiting sessions from the catalog; `awaiting_approval_tasks` from the tasks dashboard), the menubar bell as an aggregator with focus actions, the sessions rail (recent/all, filter, collapsible groups, `toggleRail`, compact overlay variant), and multi-instance session windows — N live transcripts side by side, the thing the old shell could never do.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST bind every attention affordance to the named runtime projections in `_techspec.md` §Attention Surfaces (invariant 17): sessions badge = waiting count over the canonical session-catalog cache; tasks badge = `awaiting_approval_tasks` from the dashboard snapshot; zero renders nothing; counts cap at "9+"; a stale source hides its badge. The nav-counts store keeps its existing 8 keys untouched for its current consumers.
2. MUST implement the bell as an aggregator (waiting sessions + awaiting-approval tasks) whose rows focus the owning window — no inline approve/deny (delta #3), truthful disconnect state, live removal when items resolve at their source.
3. MUST implement the sessions rail per the prototype anatomy (recent 6 + grouped-all sliding panes, filter by title/agent, persisted group collapse, status vocabulary mapped 1:1 to catalog statuses) with `toggleRail` as the single owner of `railOpen` (dock sessions item + palette both call it); compact renders the rail as an overlay sheet whose dismissal restores prior focus.
4. MUST implement multi-instance session windows (`session:<sessionId>` via registry `matchInstance`): independent streaming transcripts, per-window composer, single-window-per-session idempotence, minimize=unmount with instant cache-backed restore, and the session app controller absorbing `use-session-page-controls`/`-session-page.tsx` (their Delete-Targets rows execute here).
5. MUST rehost the existing session views (`SessionPage` internals, inspector, composer) unchanged — coupling stays in the controller; terminal-state truth remains owned by the existing session-view suites.
6. SHOULD publish per-window topbar slots for session actions via the per-window `TopbarSlotProvider` (rewriting `use-session-topbar-slot.tsx` per its REWRITE row).
</requirements>

## Visual Contract

Reference: `docs/design/opendesign/os/agh-os-v2.html` — sessions rail + bell + dock badge states (current working tree).

| ID | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| --- | --- | --- | --- | --- | --- |
| VC-01 | prototype sessions rail — recent view with mixed statuses | rail open, seeded sessions across all five statuses | 1440×900 | normative | Real catalog data (delta #4) |
| VC-02 | prototype sessions rail — grouped "all" view, one group collapsed | rail full view, grouped fixture | 1440×900 | normative | None |
| VC-03 | prototype bell popover — populated approvals list | bell open with waiting session + awaiting task rows | 1440×900 | normative | Focus-action rows, no inline approve/deny (delta #3) |
| VC-04 | prototype dock badges on sessions+tasks | dock with live badge fixture | 1440×900 | normative | Projection-backed counts (delta #4) |

Evidence for each row: `.compozy/tasks/os-shell/evidence/visual/task_05/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 5.1 Badge adapter over session-catalog cache + tasks dashboard snapshot (stale semantics included)
- [x] 5.2 Bell aggregator model + popover (focus actions, disconnect state, live updates)
- [x] 5.3 Sessions rail component set (panes, filter, groups, statuses) + `toggleRail` wiring (dock + palette)
- [x] 5.4 Compact rail overlay variant
- [x] 5.5 Session app controller: multi-instance windows, transcript/composer rehost, topbar slots, glue deletion
- [x] 5.6 Assigned suite + Visual Contract bundles

## Implementation Details

Skills: `react`, `tanstack`, `eng-data-boundaries`, `ui-craft`, `impeccable`, `vitest`, `eng-ui-screenshot`. The projections exist today — session catalog streams via `use-session-catalog-streams` invalidation, and `awaiting_approval_tasks` ships in the dashboard payload (`internal/api/contract/tasks.go:430`) already fetched by the nav-counts reconciliation path; this task adds adapters, never invented counts (SD-007).

### Relevant Files

- `web/src/systems/os/apps/session/` (new) — controller, window composition
- `web/src/systems/os/components/{sessions-rail,bell,dock-badges}.tsx` (new)
- `web/src/systems/session/` — rehosted views, catalog cache, `use-session-topbar-slot.tsx` (REWRITE row)
- `web/src/systems/runtime/hooks/nav-counts-store.ts` + `nav-counts-fetchers.ts` — read-only source reference (untouched keys)
- `web/src/hooks/routes/{use-session-detail-page.ts}` + `web/src/routes/_app/-session-page.tsx` — DELETE rows executed here

### Dependent Files

- `web/src/systems/os/lib/app-registry.ts` — session `matchInstance` + rail-toggle dock entry
- `web/e2e/` fixtures — seeded multi-session helpers (L-007)

### Related ADRs

- [ADR-007](adrs/adr-007.md) — v1 attention posture (this task IS its implementation)
- [ADR-002](adrs/adr-002.md) — multi-instance session windows

## Deliverables

- Truthful badges + bell aggregator + sessions rail (floating and compact variants)
- Multi-instance session windows with independent live streams; session glue rows deleted
- Visual Contract bundles VC-01..04 passing
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-042 — two session instanceKeys → two windows
- [x] UT-062, UT-066 — badge adapter mapping; zero-hidden + "9+" cap + stale-hides
- [x] UT-067, UT-068 — rail filter; persisted group collapse
- [x] UT-069, UT-083 — bell aggregator rows/live removal/zero state; focus actions + disconnect state
- [x] UT-082 — projection sources (catalog waiting count; `awaiting_approval_tasks`)
- [x] UT-084 — `toggleRail` persistence + compact overlay variant
- [x] E2E-006 — two streaming session windows, independent reply, minimize-while-streaming restore
- [x] E2E-015 — bell journey incl. CLI-resolved item truthfulness

## Success Criteria

- Every assigned test case implemented and passing; turbo lint/typecheck/test clean
- No attention affordance renders without its named projection (code-reviewable against §Attention Surfaces)
- All four VC rows PASS with zero unresolved blocking divergence
- QA impact: new attention + multi-session behavior → task_10 mints `untested` scenarios (rail, bell, badges, multi-session observation); flag recorded here
