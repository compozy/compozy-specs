---
status: pending
title: "Web: agent-comms system in the Agents app"
type: frontend
complexity: high
---

# Task 6: Web: agent-comms system in the Agents app

## Overview

Extend the existing Agents app with the collaboration surface — the product's main differentiator: the Activity location (delegation trees, live), call detail with schema-aware typed results, the inbox with delivery outcomes, roster enrichment with the Call compose flow, the session inspector Calls tab + timeline wake rows, and the attention/dock `calls` badge. Everything renders daemon truth only: no unread marks, no Revive button, counts from summary projections, cost through `describeCost()`.

**Blocking prerequisite**: the six `agent-comms-*.html` artboards named in `_uiux.md` MUST exist under `docs/design/opendesign/agent-comms/` before this task executes (operator OpenDesign pass — same flow as loop-legibility). Do not start implementation, and do not downgrade the Visual Contract to prose, while they are absent.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Activate `eng-design` + `ui-craft` + `impeccable` + `eng-ui-screenshot` (this task names visual references — Visual Contract Mode applies: rendered reference/implementation bundles, zero unresolved structural mismatches; implementation-only captures are invalid evidence). Follow `web/CLAUDE.md` dispatch for the web stack skills.
2. This feature EXTENDS the existing Agents app (`app-catalog.ts:70`) — new window-location kinds for `/agents/activity`, `/agents/inbox`, `/agents/calls/$callId`; it registers NO new app and tears down NO host chrome.
3. Reuse before create: compose from `@compozy/ui` exports (`Tree/TreeItem/TreeItemLabel` — first real consumer, budget its story/test/a11y pass in `packages/ui` with no API change expected; `JsonViewer`, `MetadataList`, `Timeline`, `Marker`, `Pill`, `StatusDot`, …). Domain components carry the `Agent` prefix; no new `@compozy/ui` primitives in this change.
4. New system `web/src/systems/agent-comms/` follows the house system shape (`adapters/`, `components/` with colocated `__tests__/` and `stories/`, `hooks/`, `lib/` holding `query-keys.ts`/`query-options.ts` + projections, `mocks/` for MSW handlers/fixtures — the directory is `mocks/`, not `msw/`); roster changes extend the existing `web/src/systems/agent/` system and the app's location files.
5. One projection module (`lib/agent-comms-tree.ts`, modeled on `session-hierarchy.ts`) owns tree building + cycle guard; components stay render-only; query keys follow the `networkKeys` factory pattern; types come from `web/src/generated/compozy-openapi.d.ts` (task_05's codegen) — no hand-mirrored DTOs.
6. Truthful UI is non-negotiable: nine call states rendered from the `_dx.md` vocabulary (never invented synonyms); `extracted` renders as extracted; no mark-read control; no Revive button (revival IS call-again/message); no budget-exhausted attention state (ADR-011); counts from daemon summary projections; cost lines use `describeCost()` verbatim; controls map 1:1 to real operations (cancel in-flight, call again, message, stop subtree) with affordances absent — not disabled — when the operation does not exist.
7. Attention: widen `OsAttentionBadges` and the app-catalog badge union with `calls`; needs-you causes exactly invalid-result / completed-without-result / child-blocked-on-decision; coalesced per tree; auto-resolve clears without any dismissal mechanics.
8. Signal palette per `_uiux.md` mapping (tokens only, flat depth); identity by glyph, color for state only; keyboard: full tree traversal (↑↓←→ Enter); live data gates on `useWindowLiveDataEnabled`; SSE resync to daemon truth with stale-action feedback.
9. Playwright suites extend `web/e2e/__tests__/` with the daemon fixture (`openAppWindow(page, "Agents", "agents")`), selectors added to `web/e2e/fixtures/selectors.ts`, no `force: true`.
</requirements>

## Visual Contract

Reference artboards: `docs/design/opendesign/agent-comms/` (operator OD pass — prerequisite above). Standing authorized differences on every row, per the reference-parity rule: real runtime data/copy replaces artboard placeholder content (authority: daemon truth + `COPY.md`); component identity comes from `@compozy/ui` (authority: reuse-before-create); host chrome (dock, window frame) stays live-app (authority: host surface). Rows list only differences beyond these.

| ID | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| --- | --- | --- | --- | --- | --- |
| VC-01 | `agent-comms-activity-tree.html` — default, 2-3 live trees | `/agents/activity` — seeded multi-tree fixture | 1440×900 | normative | None |
| VC-02 | `agent-comms-activity-tree.html` — state-row spectrum (queued/running/completed/invalid-result/completed-without-result/failed/canceled/timeout/expired) | `/agents/activity` — fixture covering all nine states | 1440×900 | normative | None |
| VC-03 | `agent-comms-activity-tree.html` — parked vs running vs gone rows | `/agents/activity` — parked/gone fixture | 1440×900 | normative | None |
| VC-04 | `agent-comms-activity-tree.html` — parent-drained subtree | `/agents/activity` — post-drain fixture | 1440×900 | normative | None |
| VC-05 | `agent-comms-activity-tree.html` — deep/wide tree, collapsed with urgency escalation | `/agents/activity` — 150-call fixture collapsed | 1440×900 | normative | Row virtualization thresholds (authority: runtime perf) |
| VC-06 | `agent-comms-activity-tree.html` — empty state (teaching) | `/agents/activity` — fresh workspace | 1440×900 | normative | None |
| VC-07 | `agent-comms-activity-tree.html` — SSE-stale + stale-action feedback | `/agents/activity` — dropped-SSE fixture | 1440×900 | normative | None |
| VC-08 | `agent-comms-call-detail.html` — completed, verdict `returned` | `/agents/calls/$callId` — completed fixture | 1440×900 | normative | None |
| VC-09 | `agent-comms-call-detail.html` — `extracted` and `repaired` provenance | `/agents/calls/$callId` — extraction/repair fixtures | 1440×900 | normative | None |
| VC-10 | `agent-comms-call-detail.html` — invalid-result, both attempts' errors | `/agents/calls/$callId` — invalid-result fixture | 1440×900 | normative | None |
| VC-11 | `agent-comms-call-detail.html` — completed-without-result | `/agents/calls/$callId` — silent-finish fixture | 1440×900 | normative | None |
| VC-12 | `agent-comms-call-detail.html` — canceled + superseded evidence; timeout with deadline header | `/agents/calls/$callId` — canceled/timeout fixtures | 1440×900 | normative | None |
| VC-13 | `agent-comms-call-detail.html` — running (cancel available) + over-budget preview + untrusted-text frames | `/agents/calls/$callId` — running/overflow fixtures | 1440×900 | normative | None |
| VC-14 | `agent-comms-inbox.html` — mixed inbox with delivery outcomes | `/agents/inbox` — mixed fixture | 1440×900 | normative | None |
| VC-15 | `agent-comms-inbox.html` — queued→delivered transition + brake receipts (rate-limited/dedup) | `/agents/inbox` — brakes fixture | 1440×900 | normative | None |
| VC-16 | `agent-comms-inbox.html` — blocked-target compose error; empty inbox; cap pressure | `/agents/inbox` — blocked/empty/cap fixtures | 1440×900 | normative | None |
| VC-17 | `agent-comms-roster.html` — catalog with descriptions, scope badges, shadowed + empty-description rows | `/agents` — enriched catalog fixture | 1440×900 | normative | None |
| VC-18 | `agent-comms-roster.html` — zero-definitions empty state; large roster bounded | `/agents` — empty/50-definition fixtures | 1440×900 | normative | None |
| VC-19 | `agent-comms-roster.html` — Call compose flow (editing, contract invalid inline, submitting, accepted) | `/agents/$name` — compose flow states | 1440×900 | normative | None |
| VC-20 | `agent-comms-session-calls-panel.html` — Calls tab mixed states + wake timeline row with preview | session window — inspector Calls tab fixture | 1440×900 | normative | None |
| VC-21 | `agent-comms-session-calls-panel.html` — hundreds of calls (pagination, truthful counts); pruned counterpart | session window — paginated/pruned fixtures | 1440×900 | normative | None |
| VC-22 | `agent-comms-attention.html` — bell with call rows among session/task rows; coalesced tree row; auto-resolve clearing; dock badge strip | OS shell — bell + dock with `calls` badge fixtures | 1440×900 | normative | None |

Evidence for each row: `.compozy/tasks/agent-comms/evidence/visual/task_06/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}` (or `<QA_OUTPUT_PATH>/qa/visual-contract/task_06/...` for isolated QA).

## Subtasks

- [ ] 6.1 Verify the artboard prerequisite; render every referenced artboard state via `eng-ui-screenshot` before writing components
- [ ] 6.2 Scaffold `web/src/systems/agent-comms/` (adapters, query keys/options in `lib/`, MSW `mocks/`, fixtures) over the generated OpenAPI types
- [ ] 6.3 Build `lib/agent-comms-tree.ts` projection (+ call-detail view-model, empty-state resolution) with unit suites
- [ ] 6.4 Add the Activity location: window-location kinds, tree rendering (`AgentCallTreeRow`), collapse/urgency, keyboard traversal, live gating, stop-subtree control
- [ ] 6.5 Add the call-detail location (`AgentCallDetail`, `AgentCallResultView`): header/contract/timeline/result/cost + real controls per state
- [ ] 6.6 Add the inbox location (`AgentInboxRow`, `AgentComposeMessage`) with delivery outcomes and typed compose failures
- [ ] 6.7 Enrich catalog/detail in `web/src/systems/agent/`: descriptions, scope badges, instance counts, recent calls, `AgentCallCompose` flow
- [ ] 6.8 Add the session inspector Calls tab (`AgentCallsInspectorPanel`) + timeline `AgentCallMarkerRow` wake rows
- [ ] 6.9 Widen attention model + app-catalog badge union with `calls`; needs-you causes + coalescing + auto-resolve
- [ ] 6.10 `packages/ui` story/test/a11y pass for `Tree` (first consumer); Storybook stories for the new domain components
- [ ] 6.11 Playwright journeys E2E-015..022 + E2E-030; capture the Visual Contract evidence bundles; close on web lanes green

## Implementation Details

All anchors are in `_spec.md` File References › Web and `_uiux.md` (surface map + component plan are the authority for locations, components, and state mappings). MSW fixtures + Storybook stories accompany every component per the system conventions; adapters speak both transports through `web/src/lib/api-client.ts`.

### Relevant Files

- `web/src/systems/os/lib/app-catalog.ts:33,70` — badge union + the existing Agents app descriptor
- `web/src/systems/os/apps/agents/agents-window.tsx:10` + `agent-window-location.ts` + `agents-catalog-location.tsx` + `agent-detail-location.tsx` + `use-agents-catalog.ts` / `use-agent-detail.ts` — the window controller + locations being extended
- `web/src/systems/session/lib/session-hierarchy.ts:50` + `session-list-thread.tsx:33` — cycle-guarded tree projection + renderer precedent
- `web/src/systems/session/components/session-inspector.tsx:29-40` + `session-inspector-sections.tsx` + `session-timeline-render.tsx:275-293` + `runtime-activity-notice.tsx:29` + `tool-call-card.tsx:14` — inspector tabs, timeline markers, wake causes, tool-row grammar
- `web/src/systems/os/lib/attention-model.ts:1-17,32` + `session-badge.ts:37,166` + `dock-badges.ts:4` + `attention-bell.tsx:44-55` — attention model, badge grammar, bell sections
- `web/src/systems/network/` — the system-shape template (adapters/components/hooks/lib/mocks, query keys under `lib/`)
- `web/src/lib/api-client.ts:13` + `api-contract.ts:57` + `cost-provenance.ts:94` + `ticketed-event-source.ts` — transports, generated-contract import, `describeCost()`, SSE
- `web/src/generated/compozy-openapi.d.ts` — generated types (from task_05)
- `web/e2e/fixtures/os-navigation.ts:16` + `web/e2e/fixtures/test.ts:15` + `web/e2e/fixtures/selectors.ts` + `web/e2e/__tests__/agents.spec.ts` — Playwright fixture + selector conventions
- `packages/ui/src/index.ts` — the primitive inventory (Tree/TreeItem/TreeItemLabel, JsonViewer, MetadataList, Timeline, Marker, Pill, StatusDot, Eyebrow, Metric, DataSurface*, ListingRow*, KindChip, MonoId, OwnerAvatar, Time, CopyIconButton, CommandSelect, ActionResultBanner)

### Dependent Files

- `web/src/systems/os/lib/__tests__/app-registry.test.ts` — app path-ownership suite extends with the new locations
- `packages/ui/src/**` — `Tree` story/test/a11y pass (no API change expected)
- `web/e2e/fixtures/selectors.ts` — new selectors for the journeys

### Related ADRs

- [ADR-001: Typed call result lives in a first-class durable call record](adrs/adr-001.md) — the record every surface renders
- [ADR-003: Finished children are parked and revivable; TTL is an idle ceiling](adrs/adr-003.md) — no Revive button; revival is call/message
- [ADR-011: Accounting-only call activations in v1 (no admission ceilings)](adrs/adr-011.md) — no budget-exhausted attention state exists

### Web/Docs Impact

- `web/`: new `web/src/systems/agent-comms/` (adapters, hooks, components + stories + tests, lib projections, mocks); modified `web/src/systems/agent/` (catalog/detail enrichment + compose), `web/src/systems/os/` (window locations, attention model, app-catalog badge union, dock badges), `web/src/systems/session/` (inspector tab, timeline rows); `web/e2e/__tests__/` + fixtures/selectors.
- `packages/site`: none — checked surfaces: docs pages ship in task_07; no site code consumes web systems.
- QA impact: new user-visible UI — add content-addressed `untested` scenarios for: Activity tree navigation + drill-in, call detail per terminal state, inbox compose + delivery outcomes, roster compose flow, session Calls tab + wake row, attention badge/bell for call causes. Flag only; task_09 walks them via Playwright/browser.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: hooks, Host API, tools/resources, registries, bridge SDKs; reason: web consumes public APIs only.
- Agent manageability: none new — the app renders the same operations agents/operators already reach via task_05's CLI/HTTP/tools (parity, not new capability).
- Config lifecycle: none — checked: no `config.toml` keys read or introduced by web code.

## Deliverables

- The three new Agents-app locations + catalog/detail enrichment + inspector Calls tab + attention/dock `calls` badge, all daemon-truthful
- `web/src/systems/agent-comms/` complete with projections, adapters, MSW mocks, stories, and colocated tests
- Every Visual Contract row with a durable passing `eng-ui-screenshot` evidence bundle **(REQUIRED)**
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-125, UT-126, UT-127, UT-128, UT-129 — tree projection + cycle guard, query-key factory, call-detail view-model, empty-state resolution, 200-node virtualization
- [ ] UT-130, UT-131, UT-132 — inspector tab registration exhaustiveness, panel pagination counts, pruned-counterpart degradation
- [ ] UT-133, UT-134, UT-135, UT-136 — attention badge derivation, auto-resolve clearing, per-tree coalescing, `describeCost()` verbatim
- [ ] E2E-015, E2E-016, E2E-017, E2E-018 — Activity tree live, call detail, inbox, SSE resync + stale-action
- [ ] E2E-019, E2E-020, E2E-021, E2E-022 — session Calls panel + wake row, attention badge/bell clearing, empty states, 150-call scale + keyboard
- [ ] E2E-030 — roster journey (descriptions, scope badges, counts, compose → call appears in Activity)

## Success Criteria

- Every assigned test case implemented and passing; `make test-e2e-web` green
- Every Visual Contract row `PASS` with zero unresolved blocking divergence; evidence bundles durable at the declared paths
- Zero `compozy-ui-reuse/no-shadow-ui-primitive` violations; `make bun-lint` + `bun-typecheck` clean through repo-root Turbo
- Truthful-UI review holds: no control without a runtime operation, no state the daemon does not model
