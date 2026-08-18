---
status: completed
title: "Web surfaces: update affordance, install forms, dev badge, logs panel (Phase F)"
type: frontend
complexity: medium
---

# Task 8: Web surfaces — update affordance, install forms, dev badge, logs panel (Phase F)

## Overview

Close the web parity gaps against what the API already exposes (R2/R3/R4, truthful UI): render the built-but-unrendered update affordance and cross-scope update counts, add install forms for the source union (local path, github, git), show source/verification badges, and give dev-linked extensions their badge, failure counters, origin path, and a live logs panel. Every control maps to a route shipped by tasks 04–05 — nothing the daemon doesn't model.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST render the Update action in extension detail (wire `use-extension-detail-state.ts` into `marketplace-detail-extension-manage.tsx`) and count updates across scopes, not only `installed` (`use-marketplace-kind-page.ts:200,256`) — the landing count stops being structurally zero.
- MUST add install forms for `local_path` + `github`/`git` refs submitting the union request (`use-marketplace-action-controller.tsx` — the contract already carried `path`; the shape is now the union), surfacing the consent dialog and per-source validation errors.
- MUST show source + verification badges on cards (tier and `digest_matched` as an integrity fact — never rendered as publisher verification, invariant 5) and update the empty-state `cliHint` to real verbs.
- MUST show in `web/src/systems/extensions`: the `dev (overrides published)` badge, failure counters + backoff + origin path in detail, a logs panel consuming `GET /api/extensions/{name}/logs` with SSE follow via `addEventListener("extension_log", …)` (L-017 — never the unnamed `message` event), and provenance dialog source kinds.
- MUST reuse `@compozy/ui` primitives (no shadowing — lint-blocked) and pull every visual value from tokens; domain composites live under `web/src/systems/{marketplace,extensions}`.
- MUST co-ship Playwright coverage with fixtures/mocks matching the new runtime contract (L-007) and validate through the repo-root turbo lane.
</requirements>

## Subtasks

- [x] 8.1 Update affordance + cross-scope counts (marketplace detail + kind page)
- [x] 8.2 Union install forms (local path, github, git) + consent dialog + error surfacing
- [x] 8.3 Source/verification badges + `cliHint` correction
- [x] 8.4 Extensions system: dev badge, failure counters/backoff/origin in detail, provenance source kinds
- [x] 8.5 Logs panel with SSE follow (named event, reconnect-safe)
- [x] 8.6 Playwright specs + fixtures (`marketplace.spec.ts` extension; new `extensions.spec.ts`); visual recapture explicitly waived by the operator on 2026-07-29
- [x] 8.7 Implement every assigned test case via the turbo lane

## Implementation Details

TechSpec: Web/Docs Impact (`web/` bullets), API Endpoints (payload fields), Safety Invariant 5 (integrity vs trust display). Brief §Current State (exact gap file:line evidence). Skills: `eng-design` + `ui-craft` + `impeccable` (+ `react`, `tanstack`, `tailwindcss`, `shadcn`); verification via `eng-ui-screenshot`. Follow `web/CLAUDE.md`.

### Relevant Files

- `web/src/systems/marketplace/hooks/{use-extension-detail-state.ts,use-marketplace-kind-page.ts,use-marketplace-action-controller.tsx}` — the three evidenced gaps
- `web/src/systems/marketplace/components/marketplace-detail-extension-manage.tsx` — update affordance render target
- `web/src/systems/extensions/` — detail/status components gaining badge/counters/logs panel
- `web/src/generated/compozy-openapi.d.ts` — regenerated types from tasks 04–05 (consume, never hand-edit)
- `web/src/routes/_app/settings/-extensions-policy-section.tsx` + `web/src/systems/settings/hooks/use-settings-extensions-page.ts` — the settings policy section consumes the ADR-007 key rename (`extensions.trust.allow_unverified` replaces `extensions.marketplace.*`) — must reflect the new keys truthfully
- `web/e2e/__tests__/marketplace.spec.ts` (extends) + new `web/e2e/__tests__/extensions.spec.ts`; `web/e2e/fixtures/marketplace-server.ts` — mock updates for the union contract

### Dependent Files

- `packages/ui/src/index.ts` — primitive inventory (reuse before create; any new generic primitive lands there with story + test)

### Related ADRs

- [ADR-005: GitHub/git-first distribution](adrs/adr-005.md) — source union + badge semantics
- [ADR-002: First-class dev lane](adrs/adr-002.md) — dev badge/overrides label truthfulness

## Web/Docs Impact

This IS the web task: `web/src/systems/{marketplace,extensions}` as enumerated above. Docs screenshots/site pages owned by task_09.

**QA impact**: reset web marketplace/extensions ET scenarios (update visibility, install dialogs, extension detail) to `untested`; add content-addressed `untested` scenarios for the logs panel and local-path/github install forms.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: web-only consumption of daemon contracts; no manifest/hook/tool change.
- Agent Manageability: none new — agents use CLI/HTTP/UDS (tasks 04–07); web adds no agent-exclusive control (truthful-UI parity only).
- Config Lifecycle: none — checked surfaces: no config keys; web reads daemon payloads only.

## Deliverables

- Update affordance + counts live; union install forms; badges; dev/observability surfaces; logs panel streaming
- Playwright coverage green in the daemon-served lane; visual recapture explicitly waived by the operator on 2026-07-29
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] E2E-005 — kind page non-zero update count on landing scope; detail Update action; apply transitions the card
- [x] E2E-006 — dev badge + overrides label; logs panel streams; local-path install through the union contract with consent dialog

## Success Criteria

- Every assigned test case implemented and passing (`bunx turbo run lint typecheck test --filter=./web` + `make test-e2e-web` lane green)
- No control renders for anything the daemon doesn't model; integrity facts never read as publisher trust
- Visual recapture waived by the operator on 2026-07-29; functional browser coverage remains required and passing
