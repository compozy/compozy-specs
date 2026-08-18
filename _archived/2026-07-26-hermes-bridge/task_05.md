---
status: pending
title: Web setup orchestrator with checklist, manifest, and verify
type: frontend
complexity: medium
---

# Task 5: Web setup orchestrator with checklist, manifest, and verify

## Overview
Turns the bridges web UI into a setup orchestrator — Slack manifest step, verify/send-test actions, state-aware checklist, progress config fields — truthful to daemon capabilities only. The bridges web UI is a capable manager but offers zero setup acceleration today; this slice surfaces the task_04 manifest/verify APIs and task_01 progress types so users stop bouncing between the UI and external docs.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST add a Slack manifest step to the create wizard's provider stage (fetch from the task_04 manifest endpoint; copy button + "open api.slack.com" deep link) — only for providers that expose one.
- MUST render task_04 verify results inline on the detail panel's secret-binding cards ("token valid" / failure remediation) and add a "Verify" action; expose "Send test" beside the existing dry-run test-delivery, labeled to distinguish them.
- MUST add a per-platform setup checklist section to the detail panel that derives state from the bridge itself (provider installed → secrets bound → webhook path configured/registered → verified → enabled → healthy) with deep links to the platform dashboards; for Telegram the checklist offers the webhook-registration action via the task_04 HTTP route (no CLI needed).
- MUST add `delivery_defaults.progress` fields (mode/grouping/typing/reactions) to create/edit dialogs using the regenerated types from task_01.
- MUST render only what the daemon supports — no invented controls (SD-007); checklist items map 1:1 to real API-observable state.
- MUST pull all styling from tokens (`packages/ui/src/tokens.css` via DESIGN.md grammar).
</requirements>

## Subtasks
- [ ] 5.1 Manifest step in create wizard (adapter + hook + component) — present only when the provider exposes a manifest endpoint
- [ ] 5.2 Verify action + inline results on secret cards; send-test action wired beside dry-run test-delivery with distinguishing labels
- [ ] 5.3 State-aware setup checklist section in `bridge-detail-panel` (incl. Telegram webhook-register action via HTTP)
- [ ] 5.4 Progress config fields (mode/grouping/typing/reactions) in create/edit dialogs using regenerated types
- [ ] 5.5 Screenshot verification of the new surfaces via `eng-ui-screenshot`; cite captures in completion notes

## Implementation Details
Depends on task_01 (progress fields / regenerated types) and task_04 (manifest/verify/send-test/webhook-register APIs). Reference `_techspec.md` §8 (Web/Docs Impact). Uses regenerated TS types from `make codegen` artifacts — consume, do not hand-edit. Activate `eng-design` + `ui-craft` for the checklist/verify visual language; verify with `eng-ui-screenshot` before completion. Cross-refs updated: old task_08/10 → task_04; progress types from task_01.

### Relevant Files
- `web/src/systems/bridges/adapters/bridges-api.ts` — manifest/verify/send-test/webhook-register calls
- `web/src/systems/bridges/components/bridge-create-dialog.tsx` — manifest step + progress fields
- `web/src/systems/bridges/components/bridge-detail-panel.tsx` — checklist, verify, send-test
- `web/src/systems/bridges/hooks/` — new queries/mutations for manifest/verify/send-test
- `web/src/systems/bridges/components/` — new checklist/manifest components as needed

### Dependent Files
- `web/src/systems/bridges/mocks/` — MSW handlers for the new endpoints
- `web/src/generated/` — regenerated types (consumed, not edited)
- `web/src/systems/bridges/` component/hook test suites — vitest cases below

### Competitor References
- `.resources/hermes/website/docs/user-guide/messaging/slack.md:19-118` — step taxonomy the checklist encodes (translated into a live, state-aware UI checklist rather than prose)

## Deliverables
- Create wizard with manifest hand-off; detail panel with verify, send-test, and live checklist; progress config fields
- Screenshot captures cited in the completion notes
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests
Cases assigned from `_tests.md` (old task_11 ownership). Read each case's full definition there before writing tests. Logic/behavior only — no snapshot/render-shape filler. Visual parity is `eng-ui-screenshot`'s job (subtask 5.5).

### Web Unit / Component (`bunx turbo run test --filter=./web`, vitest)
- [ ] Should render the manifest step for a Slack provider selection and copy the fetched JSON; the step is ABSENT for providers without a manifest endpoint
- [ ] Should render per-check status on the matching secret card on verify-mutation success and the remediation text on failure
- [ ] Should derive every checklist item from mocked bridge state (unbound secret → unchecked with CTA; enabled+healthy → all checked) — items map 1:1 to daemon-observable facts, no invented controls (SD-007)
- [ ] Should serialize the progress fields (mode/grouping/typing/reactions) into `delivery_defaults.progress` on both create and edit
- [ ] Should produce the correct POST body from the full create-wizard MSW flow with manifest step + progress fields

### E2E — Web (`make test-e2e-web`, Playwright; mocked daemon contract)
- [ ] Bridges create flow — Should open the create wizard, select a provider WITH a manifest endpoint (Slack), surface the manifest step (copy + deep link), complete creation, and show the state-aware setup checklist on the detail panel
- [ ] Verify flow — Should run the Verify action from the detail panel and render per-check status inline on the matching secret-binding cards, including a failure remediation
- [ ] Send-test vs dry-run — Should expose "Send test" beside the existing dry-run test-delivery with distinguishing labels, and reflect the send-test result (SD-007 truthful-UI)

Test coverage target: >=80% for touched packages. All tests must pass under the repo gates.

## Success Criteria
- Every assigned test case implemented and passing
- `bunx turbo run lint typecheck test --filter=./web` green
- Screenshot captured via `eng-ui-screenshot` and cited for wizard + detail panel
- Every checklist item corresponds to a daemon-observable fact (SD-007 audit in PR/completion notes)
