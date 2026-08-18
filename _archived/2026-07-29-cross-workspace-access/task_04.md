---
status: completed
title: Web deep-link confirm on owner projection (phase 0 web)
type: frontend
complexity: medium
---

# Task 4: Web deep-link confirm on owner projection (phase 0 web)

## Overview

Stop dead-ending operator deep links to foreign-workspace sessions: when the session deep-link loaders miss in the active workspace, resolve the owner via the task_03 projection and show a routed confirmation dialog ("this session belongs to workspace B — switch?"). Confirm activates the owning workspace and re-enters the route; cancel keeps today's not-found rendering. Also lands the combined regression journey (E2E-011) that pins the program's named behavior deltas.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST implement the structural route-ownership correction defined by
  `corrections/task-04-route-ownership/_techspec.md`, its `_tests.md`, and ADR-008 before another
  production patch. The parent `/agents` and `/agents/$name` routes are search-neutral layouts; their
  new index leaves own fleet/detail search/load/sync respectively; the session leaf exclusively owns
  `workspaceSwitch`.
- MUST fetch only the owner projection pre-confirm, cached under a dedicated `session-owner` query key that is never merged into session catalogs or detail caches — no foreign payload client-side before confirmation (ADR-004, UT-062/E2E-008 network-log assertion).
- MUST implement the confirmation as a routed dialog (deep-link-replayable, E2E-addressable), not a toast; cancel keeps today's not-found state; projection 404 keeps today's behavior with no dialog. No routed-dialog pattern exists in `web/` today — build it from the `@compozy/ui` `ConfirmDialog` primitive plus a search-param validator modeled on `task-detail-search.ts` (the established deep-link-replayable state pattern).
- MUST account for `workspace-switch-barrier` semantics: `setActiveWorkspaceId` already triggers `coordinator.beginWorkspaceSwitch()` via the store subscription — reuse that path on confirm, never duplicate the barrier call.
- MUST delete the unconditional foreign-session not-found branch in both loaders in the same change (TechSpec delete target).
- MUST switch the active workspace through the existing `active-workspace-store` mechanism on confirm, then navigate — never switch silently (ADR-004 rejected auto-switch).
- MUST use the generated OpenAPI type for the projection payload (no hand-mirrored DTO) and follow the `web/src/systems/<domain>/` conventions (`app-renderer-systems`) for the query-key/service module.
- MUST verify the UI change with `eng-ui-screenshot` and cite the capture in completion notes (no named visual reference exists — no Visual Contract table; the capture evidences the shipped dialog).
- MUST reuse `@compozy/ui` primitives for the dialog (check `packages/ui/src/index.ts` first; `compozy-ui-reuse/no-shadow-ui-primitive` is a blocked lint error); copy follows `COPY.md`.
- Frontend gates run through Turborepo from the repo root: `bunx turbo run lint typecheck test --filter=./web`.
- Skills: `react`, `tanstack`, `xstate-store`, `eng-design`, `ui-craft`, `impeccable`, `eng-ui-screenshot`, `vitest`.
</requirements>

## Subtasks

- [x] 4.1 Create the `session-owner` query module (dedicated query key + service over the generated projection type) under the owning system per `app-renderer-systems` conventions.
- [x] 4.2 Rework `-agent-session-route-loader.ts` and `-session-permalink-route.ts`: active-workspace miss → owner projection → dialog state carrying only `{sessionId, workspaceId, workspaceName}`; 404 → today's not-found.
- [x] 4.3 Build the routed confirmation on `ConfirmDialog` + a `validateSearch` param (pattern: `task-detail-search.ts`) naming the target workspace; wire confirm → `setActiveWorkspaceId` (barrier handles OS-shell hydration) → route re-entry; cancel → not-found rendering.
- [x] 4.4 Delete the unconditional foreign-session not-found branches; sweep both consuming routes (`agents.$name.sessions.$id.tsx`, `session.$id.tsx`).
- [x] 4.5 Implement the Vitest cases inside the canonical `-agent-session-deeplink.router.integration.test.tsx` suite (update its Invariant header to the confirm-first contract) plus store/search-validator units.
- [x] 4.6 Implement the Playwright journeys (both permalink forms, confirm switch, cancel, nonexistent session, no-foreign-payload network assertion).
- [x] 4.7 Implement E2E-011 by rewriting the existing cross-workspace deep-link spec in `web/e2e/__tests__/os-shell.spec.ts` (old refusal contract → confirm-first contract) + the native foreign-workspace matrix with default mode, asserting each named invariant-1 delta explicitly.
- [x] 4.8 Capture `eng-ui-screenshot` evidence of the dialog states (default + cancel path) and cite in completion notes.

## Implementation Details

Follow TechSpec §Web: Operator Deep-Link Confirm and ADR-004 (Implementation Notes name the exact modules). The loaders sit post-`xstate-refac` (#268) — respect the current loader/store shapes rather than the pre-refactor ones.

### Relevant Files

- `web/src/routes/_app/-agent-session-route-loader.ts` (miss branch = `workspaceId: null` at L22-31) and `-session-permalink-route.ts` (`resolveSessionPermalink` L52-69; foreign == missing today) — the two loaders to rework.
- `web/src/routes/_app/agents.$name.sessions.$id.tsx` (toast + redirect at L16-23) and `session.$id.tsx` (toast-only inert fallback L22-29) — consuming routes whose not-found handling changes.
- `web/src/systems/workspace/stores/active-workspace-store.ts` (`setActiveWorkspaceId`) + `web/src/systems/os/lib/workspace-switch-barrier.ts` (store subscription → `beginWorkspaceSwitch`) — the switch mechanism; reuse, don't duplicate.
- `packages/ui/src/components/custom/confirm-dialog.tsx` (`ConfirmDialog`, exported via `packages/ui/src/index.ts`) — the dialog primitive.
- `web/src/systems/tasks/lib/task-detail-search.ts` (+ its `__tests__`) — the search-param validator pattern for deep-link-replayable state.
- `web/src/systems/session/lib/query-keys.ts` — all session keys are workspace-prefixed today; the `session-owner` key is the first workspace-agnostic one (dedicated module, deliberate placement per ADR-004).
- `web/src/systems/session/adapters/session-api-errors.ts` + sibling one-concern adapters — adapter conventions for the new `session-owner` fetch.
- `web/src/routes/_app/__tests__/-agent-session-deeplink.router.integration.test.tsx` — the canonical deep-link suite (Suite/Invariant header); UT-047/048/049/062 extend it, not a new standalone suite.
- `web/e2e/__tests__/os-shell.spec.ts` (existing cross-workspace deep-link spec at ~L1486-1526 asserts the OLD refusal contract — rewritten by E2E-011) + `web/e2e/fixtures/workspace.ts`/`selectors.ts` helpers (L-007: extend helpers in the same PR).

### Dependent Files

- `web/src/systems/session/types.ts` + `index.ts` — `SessionOwnerResponse` via the `OperationResponse` pattern; barrel exports.
- `web/src/systems/session/mocks/handlers.ts` — MSW handler for the owner endpoint; `web/src/systems/session/routes/session-permalink.stories.tsx` — foreign-workspace story next to the NotFound template.
- `web/e2e/fixtures/selectors.ts` — confirm-dialog test ids.
- `docs/qa/scenarios/ET-web-session-deep-link-isolation.md` — expected text rewritten by task_05 to the confirm-first contract this task ships.

### Related ADRs

- [ADR-004](adrs/adr-004.md) — confirm-before-switch decision, minimal-projection contract, affected modules.
- [ADR-007](adrs/adr-007.md) — agent path is policy-governed; this flow is operator UX only.
- [ADR-008](adrs/adr-008.md) — corrective parent-layout/index-leaf route ownership after the two-touch trigger.

### Web/Docs Impact

- `web/`: the modules above (two loaders, two routes, one new query module, one routed dialog, store wiring) — this task IS the web impact.
- `packages/site`: none in this task — narrative docs land in task_05.
- QA impact: user-visible change (deep-link behavior). Task_05 resets `ET-web-session-deep-link-isolation` (rewritten expectations) and adds the deep-link-confirm scenario; this task's completion notes flag the behavior for it.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: no extension/hook/tool/bundle/bridge/MCP change; pure SPA routing UX.
- Agent manageability: none — operator UX only; agents never reach web routing (ADR-004); daemon surfaces unchanged.
- Config lifecycle: none — checked surfaces: no keys, no defaults.

## Deliverables

- Deep-link confirm flow live on both permalink forms with cache-isolated owner resolution.
- Deleted unconditional not-found branches; store-mediated workspace switch on confirm.
- `eng-ui-screenshot` capture cited in completion notes.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-047 — loader miss calls the owner projection (not the catalog) and returns minimal dialog state.
- [x] UT-048 — projection 404 → today's not-found, no dialog.
- [x] UT-049 — confirm switches the store then navigates; cancel leaves it untouched.
- [x] UT-062 — owner response cached only under `session-owner`; no foreign catalog/detail entries pre-confirm.
- [x] E2E-008 — both permalink forms show the confirm dialog naming the owner; confirm switches and opens; network log shows only the projection pre-confirm.
- [x] E2E-009 — cancel keeps workspace + arrangement with not-found; nonexistent session → not-found, no dialog.
- [x] E2E-011 — regression journey: rewritten web isolation contract + native foreign-workspace matrix with default mode; each invariant-1 delta asserted explicitly.
- [x] Corrective IT-001–IT-005 — implement the companion route-ownership contract in
  `corrections/task-04-route-ownership/_tests.md` without adding a standalone suite.

## Success Criteria

- Every assigned test case implemented and passing
- `bunx turbo run lint typecheck test --filter=./web` green; `make test-e2e-web` green.
- Zero foreign session payload in any client cache pre-confirm (test-asserted, not claimed).
- Screenshot evidence cited; dialog uses `@compozy/ui` primitives with token-derived styling only.

## Completion Notes

- Implemented the confirm-first flow for canonical and short session permalinks. Before confirmation,
  the SPA reads only `getSessionOwner` after the active-workspace detail miss and stores the three-field
  projection under the workspace-agnostic `session-owner` query key.
- Applied ADR-008's corrective hard split: `/agents` and `/agents/$name` are Outlet-only layouts;
  their index leaves own fleet/detail search, preloads, and OS route sync. The session leaf alone owns
  `workspaceSwitch`, and `OsRouteHold` prevents authoritative focus reconciliation from dismissing the
  routed decision.
- Preserved accepted route authority through asynchronous `openOrFocus` completion. Restored focus
  frames can no longer replace a newer route intent and trigger a reverse reconciliation; duplicate
  same-route reports remain single-flight.
- Test placement: invariant = the foreign-session decision is projection-only before consent and the
  session leaf retains its routed state through every ancestor; owning layer = TanStack Router
  integration; canonical suite = `-agent-session-deeplink.router.integration.test.tsx`. The existing
  suite was extended. Invariant = an accepted route intent remains authoritative until its command
  completes; owning layer = URL-to-window-manager coordinator; canonical suite =
  `routing-coordinator.test.ts`. Both canonical suites own their regressions; no duplicate standalone
  test was added.
- UI evidence: `/tmp/eng-ui-screenshot.3jTNS0/evidence/task04/foreign-confirm-final.png` and
  `/tmp/eng-ui-screenshot.3jTNS0/evidence/task04/foreign-cancel-final.png`. No named visual reference
  exists, so Visual Contract Mode is not applicable.
- QA tracker impact: user-visible deep-link behavior changed. Task 05 resets
  `ET-web-session-deep-link-isolation` and adds the content-addressed confirm scenario as `untested`.

Compozy Impact Audit:

- Native tools: no impact — checked `compozy__*` tool IDs, toolsets, descriptors, schemas, digests,
  risk flags, diagnostics, and capability gates; this task changes only SPA routing over task 03's
  existing HTTP projection.
- Extensibility and hooks: no impact — checked extensions, hooks, skills/capabilities,
  tools/resources, bundles, registries, bridge SDKs, MCP sidecars, and config lifecycle; no extension
  surface or configuration key changed.
- Workspace data isolation: changed at the Web projection boundary — the owner projection is global
  by session ID, while session detail/transcript remain workspace-scoped. Canonical router tests and
  E2E network assertions prove no target-workspace detail, catalog, transcript, window, or cache entry
  is read before consent; confirmed navigation switches through the existing active-workspace store.
- Official Compozy skill: no task-local impact — checked `skills/compozy/`; Task 05 owns the already
  implemented public-behavior documentation update.

VERIFICATION REPORT
-------------------
Claim: Task 04 Web deep-link confirmation is ready for checkpoint.
Command: `bunx turbo run lint typecheck test build --filter=./web`; `make test-e2e-web`
Executed: 2026-07-29, after the routing-authority production correction
Exit code: 0 for Turbo; 0 for the official Web E2E lane
Output summary: Turbo 7/7 successful; 516 test files and 4,060 tests passed; typecheck/lint clean;
production Vite build completed. The official Web E2E lane passed the complete unchanged 113-case
catalog, including the foreign-session confirmation and existing session-hardening journeys.
Warnings: Vite emitted the repository's existing large-chunk advisory; no lint warning.
Errors: none.
Contract parity: PASS — compared task 04, `_user_stories.md`, `_tests.md`, ADR-004, ADR-007,
ADR-008, and `corrections/task-04-route-ownership/{_techspec.md,_tests.md}`.
Visual contract: n/a — no named visual reference found; default/cancel implementation captures cited above.
Verdict: PASS.
