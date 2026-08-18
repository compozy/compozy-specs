---
status: pending
title: "Public surfaces co-ship — OpenAPI/CLI/web"
type: frontend
complexity: high
---

# Task 5: Public surfaces co-ship — OpenAPI/CLI/web

## Overview

Co-ship the public participation contract across OpenAPI → generated TS, HTTP/UDS handlers, CLI verbs, and web UX in one change: session/task/loop/automation participation controls, workspace coordination + invitation APIs, usage endpoint, run-detail conversation/bounds/usage, network empty states, settings reframed as availability (never enrolls), and onboarding mention. Design-phase invitation/empty-state composition runs through `eng-design` + `ui-craft` with screenshot evidence.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Contract MUST delete legacy fields (`channel`, `network_channel`, `coordination_channel_id`); unknown legacy fields MUST fail schema validation; responses carry `resolved_network_participation`.
2. New endpoints MUST ship on shared `api/core` BaseHandlers for HTTP+UDS: `GET/PUT .../network-coordination`, `PUT .../network-coordination/invitation`, `GET .../network/usage`; run detail + SSE MUST project Spec, conversation ref, bounds consumption, usage.
3. CLI MUST expose `--network`/`--network-channel-strategy`/`--network-channel`/`--network-bounds` on session/task surfaces; `agh network coordination|usage|status` with `-o json`; error codes identical to HTTP/UDS.
4. Hooks MUST dispatch `network.participation.pre_resolve` (deny/narrow) and `.resolved` (read-only) at owning call sites; extension/bundle live requirement confirmation MUST persist on typed `bundle.activation` resource with CAS (IT-034); `--confirm-network-requirement` CLI parity.
5. Web session create + task Orchestration/fan-out/automation steps MUST serialize `network_participation` exactly; Local default selected.
6. Kanban invitation MUST follow PRD rule 11 (active multi-agent run, scope off, network available); accept writes coordination PUT; dismiss persists daemon-side; copy states future-runs-only.
7. Run detail MUST show participation chip + conversation panel + bounds/usage; network area empty states MUST answer the four orientation questions with one action and no fabricated data; remove dishonest "accept an invite" copy until a real invite flow exists.
8. Settings network page MUST reframe as availability + live defaults/ceilings ("does not opt executions in") + usage link; delete `default_channel` UI.
9. `make codegen` MUST regenerate OpenAPI + TS in the same change; reuse `@agh/ui` Empty/Dialog primitives (no shadowed primitives).
10. Verify UI with `eng-ui-screenshot` before completion; cite captures.
</requirements>

## Subtasks

- [ ] 5.1 Update OpenAPI/contract types + handlers for participation objects and new workspace endpoints; run `make codegen`.
- [ ] 5.2 Ship CLI verbs/flags with structured JSON parity tests against HTTP/UDS.
- [ ] 5.3 Land hook payloads + extension/bundle confirmation flow (IT-033/034/035).
- [ ] 5.4 Web participation controls on session/task/loop/automation create/edit surfaces.
- [ ] 5.5 Invitation component + coordination setting wiring on run detail/kanban.
- [ ] 5.6 Run conversation panel + usage views; network empty states + settings reframe + onboarding mention.
- [ ] 5.7 Vitest units + Playwright E2Es; screenshot verification.

## Implementation Details

See TechSpec "API Endpoints", "CLI", "Web/Docs Impact", Build Order step 8, ADR-002/003/005/009. Design tokens from `packages/ui/src/tokens.css` + `DESIGN.md`; copy from `COPY.md`.

Skills to activate: `eng-design`, `ui-craft`, `impeccable`, `eng-ui-screenshot`, `eng-contract-codegen-coship`, `eng-code-guidelines`, `golang-pro`, `tanstack-query-best-practices`, `tanstack-router-best-practices`, `react`, `vitest`, `testing-boss`, `eng-consolidate-test-suites`, `copywriting`.

### Relevant Files

- `openapi/agh.json` + contract Go types — participation request/response
- `internal/api/core/` — handlers for sessions/tasks/loops/automation/network
- `internal/cli/session.go`, `network.go`, task CLI — flag surfaces
- `internal/hooks/payloads.go` — flat channel fields delete
- `web/src/systems/session/components/session-create-dialog.tsx` + hook
- `web/src/systems/tasks/` — overview/runs/fan-out/editor network fields
- `web/src/systems/network/components/empty-states/` — oriented empties
- `web/src/routes/_app/settings/network.tsx` — default_channel UI delete
- `web/src/routes/_app/tasks.$id.runs.$runId.tsx` — conversation panel host
- `web/src/systems/onboarding/` — light mention without enrollment
- `packages/ui/src/index.ts` — Empty/Dialog reuse inventory

### Dependent Files

- `web/src/generated/agh-openapi.d.ts` — regenerates
- `skills/agh/` — still outdated until task_06
- `docs/qa/state.csv` — new UI rows flagged here or in task_06

### Related ADRs

- [ADR-002](adrs/adr-002.md) — flagship coordinated runs + invitation
- [ADR-003](adrs/adr-003.md) — discoverability disclosed not hidden
- [ADR-005](adrs/adr-005.md) — usage visible at release
- [ADR-009](adrs/adr-009.md) — coordination setting home

## Deliverables

- OpenAPI/TS/CLI/HTTP/UDS participation surfaces
- Web controls, invitation, conversation, usage, empties, settings
- Extension confirmation + hook payload swap
- Screenshot evidence for new UI
- Every assigned test case implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [ ] UT-055, UT-056, UT-057, UT-058, UT-059, UT-060 — web invitation/conversation/empties/onboarding/controls
- [ ] IT-026 — usage endpoint/CLI aggregation + workspace isolation
- [ ] IT-028, IT-029, IT-030, IT-031, IT-032 — API/CLI parity, coordination, availability, bounds
- [ ] IT-033, IT-034, IT-035 — hooks, extension confirmation, bundle default-channel deletion behavior
- [ ] E2E-002 — coordinated-run flagship (runtime)
- [ ] E2E-003 — invitation journey (web/Playwright)
- [ ] E2E-004 — discoverability empties + onboarding (web)
- [ ] E2E-005 — administration disable/enable (runtime + web)
- [ ] E2E-007 — agent manageability structured surfaces (runtime)
- [ ] E2E-009 — loop participation (runtime)

## Success Criteria

- Every assigned test case implemented and passing
- `make codegen` / codegen-check green; no legacy channel fields in public schemas
- Invitation dismiss survives reload via daemon GET; accept affects future runs only
- Screenshot captures cited for invitation, empties, run conversation, settings

### Web/Docs Impact

- `web/`: primary delivery of this task — routes/components/hooks listed above; MSW fixtures updated; Storybook only if existing network stories break.
- `packages/site`: CLI reference regenerates with this change if `make cli-docs` is part of codegen; conceptual MDX rewrites remain task_06.
- QA impact: add `untested` rows for participation controls, invitation, run conversation, usage views, network empties; reset affected session/task/network rows.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: hooks pre_resolve/resolved; extension manifest `network_participation` requirement + activation confirmation CAS; host API diagnostics.
- Agent manageability: full CLI/HTTP/UDS parity for coordination/usage/status/participation; deterministic shared error codes.
- Config lifecycle: settings UI reflects availability + live defaults/ceilings only; no enrollment keys.
