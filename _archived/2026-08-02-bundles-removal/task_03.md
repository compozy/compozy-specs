---
status: completed
title: "Web: kind collapse, bundle strips, kit inventory panel, confirm affordance (Phase E)"
type: frontend
complexity: medium
---

# Task 3: Web — kind collapse, bundle strips, kit inventory panel, confirm affordance (Phase E)

## Overview

Brings the SPA to the post-cut contract: the marketplace kind system collapses to `{extension, skill, mcp}`, every dedicated bundle file is deleted and every shared file stripped, and the extension detail gains the two new truthful surfaces — the kit inventory panel (`GET /api/extensions/{name}/inventory`) and the shared network-confirm affordance used by both enable and update (409 + digest + retry with `confirm_network_digest`). The Playwright bundle journey is replaced by the extension-kit journey.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST delete the 11 dedicated web files and strip the ~40 shared files exactly per the TechSpec **Delete Targets § Web** inventory (kind unions, kind config, query keys, adapters, hooks, dialogs, barrel exports, OS window activation branch, storybook registry, `tool-labels.ts` bundle icon row, MSW mocks/fixtures).
- MUST keep the shared marketplace shells (extension/skill/MCP flows) working — the kind collapse may not regress install/manage/detail for the surviving kinds.
- MUST add the kit inventory panel to the extension detail consuming `GET /api/extensions/{name}/inventory` (shipped vs live per kind, live badge) and surface `bound_env_keys` + network-confirm state from the extended `ExtensionPayload`.
- MUST implement ONE confirm affordance shared by enable and update: on 409 `extension_network_confirmation_required` it surfaces the returned digest and retries the mutation with `confirm_network_digest` on operator approval (`use-extension-detail-state.ts` → `marketplace-detail-extension-manage.tsx` update path included).
- MUST render enable results from `ExtensionEnableResult` (started automation enumerated) — Truthful UI: no control or metric beyond the shipped routes/payloads; daemon truth wins.
- MUST regenerate `routeTree.gen.ts` after route deletion and refresh MSW fixtures/E2E mocks to the post-cut contract (L-007 co-ship).
- MUST replace the Playwright bundle acquisition journey in `marketplace.spec.ts` with the extension-kit journey and delete the 3 bundle test IDs from `selectors.ts` (+ `scenario-contracts.ts:265` copy); the `bundled-extension-seeder`/SourceBundled fixtures are homonyms — untouched.
- MUST reuse `@compozy/ui` primitives (no shadowing); tokens from `tokens.css`/`DESIGN.md`; signal palette only as information.
- MUST verify the new/changed surfaces with `eng-ui-screenshot` and cite the captures in completion notes.
- Skills to activate: `eng-design`, `ui-craft`, `impeccable`, `react`, `tanstack`, `eng-ui-screenshot`, `storybook-stories` (stories for the new panel), `eng-data-boundaries` (new read models).
</requirements>

## Subtasks

- [x] 3.1 Delete dedicated bundle routes/components/hooks/stories; regenerate `routeTree.gen.ts`
- [x] 3.2 Collapse the marketplace kind system (types, kind config, query keys, adapters, hooks, components, mocks) and strip extensions-system bundle surfaces (barrel, types, query keys/options, API calls, dialogs, hooks, mocks)
- [x] 3.3 Strip OS marketplace window activation branch + detail-location kind branch + storybook registry entries + session tool-label icon row
- [x] 3.4 Kit inventory panel on extension detail (+ story) with shipped/live per kind and `bound_env_keys`/confirm state
- [x] 3.5 Shared confirm affordance for enable + update (digest display, retry with `confirm_network_digest`); enable result rendering
- [x] 3.6 Vitest strips/additions; MSW/E2E mock refresh
- [x] 3.7 Playwright: bundle journey → extension-kit journey; selector cleanup
- [x] 3.8 `eng-ui-screenshot` evidence for panel + confirm affordance

## Implementation Details

The strip inventory with file:line references is in `_techspec.md` **Delete Targets § Web**; the new-surface contracts are in **API Endpoints** and **Web/Docs Impact**. New components follow `web/src/systems/extensions/` conventions (adapters → query-options → hooks → components); domain-prefixed names for composites.

### Relevant Files

- `web/src/systems/marketplace/{types.ts,lib/marketplace-kind-config.ts,lib/query-keys.ts,lib/marketplace-installed-items.ts,components/*,hooks/*,adapters/*,mocks/*}` — kind collapse + strips
- `web/src/systems/extensions/{index.ts,types.ts,lib/query-keys.ts,lib/query-options.ts,adapters/extensions-api.ts,components/*,hooks/use-extension-detail-state.ts,hooks/use-extension-actions.ts,mocks/*}` — strips + new panel/affordance
- `web/src/systems/os/apps/marketplace/{marketplace-window.tsx,marketplace-detail-location.tsx}` — activation branch strips
- `web/src/systems/session/lib/tool-labels.ts:71` — bundle icon row
- `web/src/storybook/route-story-registry.ts` — registry entries
- `web/src/routes/_app/marketplace.bundles.tsx`, `marketplace.bundles.activations.$id.tsx` — route deletes
- `web/e2e/__tests__/marketplace.spec.ts`, `web/e2e/fixtures/{selectors.ts,scenario-contracts.ts}` — journey replacement
- `web/src/generated/compozy-openapi.d.ts` — post-cut types (from task_02 regen)

### Dependent Files

- `web/src/routeTree.gen.ts` — regenerated
- Vitest suites listed in `_techspec.md` Delete Targets §tests (web) — strips
- Surviving kind pages/stories — must stay green after the union collapse

### Related ADRs

- [ADR-001](adrs/adr-001.md) — web scope of the cut; [ADR-005](adrs/adr-005.md) — confirm affordance semantics; [ADR-006](adrs/adr-006.md) — inventory read model.

### Web/Docs Impact

- `web/`: this task IS the web impact (deletes/strips/new panel/affordance/mocks/e2e per above).
- `packages/site`: none this task — task_04 owns docs.
- QA impact: user-visible web changes (marketplace without bundles, new inventory panel, confirm dialog) — `ET-web-extension-kit-inventory` (new untested) + web scenario strips are authored in task_04's QA tracker pass.

### Extensibility / Agent Manageability / Config Lifecycle

- none — checked surfaces: no extension point, CLI/HTTP/UDS contract, or `config.toml` key changes in this task (frontend consumes contracts shipped by task_01/02; `COMPOZY_WEB_API_PROXY_TARGET` untouched).

## Deliverables

- Marketplace and extensions systems free of product-bundle code with surviving kinds fully functional
- Kit inventory panel + shared confirm affordance + enable-result rendering, with Storybook story and screenshot evidence
- Playwright extension-kit journey replacing the bundle journey
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-062 — kind system exposes exactly `extension|skill|mcp` (unions, config, MSW)
- [x] UT-063 — inventory panel renders kit items from the route payload
- [x] UT-064 — confirm affordance on `network_confirmation_required` + mutation carries `confirm_network_digest`
- [x] E2E-004 — marketplace 3-kind + extension detail inventory + confirm walk
- [x] E2E-005 — bundle journey replaced by the extension-kit journey

## Success Criteria

- Every assigned test case implemented and passing
- `bunx turbo run lint typecheck test --filter=./web` green; `make test-e2e-web` green
- `eng-ui-screenshot` captures cited for the inventory panel and confirm affordance
- No `bundle` product reference greps in `web/src` outside homonyms (support/tasks/SourceBundled fixtures)

## Completion Notes

- Claude Opus executed the accepted Herdr plan; controller review corrected the final data boundary so the TanStack cache preserves the full `ExtensionInventory` envelope and the global inventory cannot appear on a workspace dev overlay.
- Marketplace kinds are exactly extension, skill, and MCP. Dedicated Bundle routes, components, hooks, stories, selectors, mocks, and shared branches were deleted; the route tree was regenerated.
- Extension detail now shows shipped/live kit resources, bound environment names, network-consent state, exact enable/update retry variables, and automation-start results without exposing secret values.
- Verification: web typecheck passed; 529 Vitest files / 4,207 tests passed; the final Playwright report recorded 132 passed, 0 unexpected, and 0 flaky. Root follow-up suites passed 48/48 and `make bun-lint` passed.
- Visual evidence: `/tmp/eng-ui-screenshot.C4yxlm/shots/kit-inventory.png` and `/tmp/eng-ui-screenshot.C4yxlm/shots/network-confirm.png`.
- Compozy Impact Audit: no native-tool IDs changed in this frontend task; extension kit and lifecycle routes are consumed without adding extension points or config keys; workspace extension lists/logs remain workspace-keyed while the inventory route is explicitly global and hidden for workspace overlays; official `skills/compozy/` updates remain owned by task 04.
