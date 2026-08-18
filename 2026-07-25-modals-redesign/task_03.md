---
status: completed
title: Dense dialogs (wizard → Simple/Advanced)
type: frontend
complexity: critical
---

# Task 03: Dense dialogs (wizard → Simple/Advanced)

## Overview

Collapses the remaining dense entity editors onto the shared shell: agent create (4-step wizard → Simple/Advanced), bridge create/edit (3-step wizard → Simple/Advanced + D2 delivery fold), MCP add/edit, and sandbox create/edit. Preserves per-step validation as field-level gates, secret rotation/preserve signals, and ImmutableIdentity locks — the highest regression-risk slice of the wave.

<critical>
- ALWAYS READ `_techspec.md` §4.1, §4.8–4.11, §4.13–4.14, §5.2 T1/T2, §13 delete targets, plus the named HTML references before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST delete agent/bridge wizard steppers (`WIZARD_STEPS`, Back/Continue, Step N of M) and replace with `EntityModeToolbar` Simple/Advanced + field-level validation + footer submit gate.
- MUST cut the agent MCP servers multi-select from production **and** from `docs/design/opendesign/modals/create-agent.html` (T1); category path stays a mono string split/joined at the adapter (T2).
- MUST bind agent create fields to `CreateAgentPayload` only (§5.1); Advanced never hides required name/prompt.
- MUST render bridge create secret slots dynamically from `GET /api/bridges/providers` `secret_slots` via `SecretField`; secrets MUST NOT enter `provider_config`.
- MUST apply ImmutableIdentity for bridge edit platform/extension/scope and MCP/sandbox create-only identity fields; emit MCP `preserve_secrets`/`preserve_env` and bridge secret-binding PUT correctly.
- MUST fold bridge delivery test into edit-bridge Advanced (D2) and delete the competing standalone entry point (`bridge-test-delivery-dialog.tsx`) in the same change.
- MUST migrate MCP editor onto host `md`, accent-strong eyebrow + F1, Simple/Advanced sections per §4.10–4.11 (MCP owns its Dialog shell today — do not reintroduce SettingsEditorDialog unless it simplifies without dual chrome).
- MUST migrate sandbox create/edit body (via upgraded SettingsEditorDialog) with Simple/Advanced, backend RadioCards, Daytona advanced block only when backend=daytona, and secret-env rotation.
- MUST keep agent edit on the settings route (D3); this dialog stays create-only.
- MUST grow no production file past the 500-line cap — extract sections instead of appending into `agent-create-dialog.tsx` / `bridge-create-dialog.tsx`.
</requirements>

## Visual Contract

| ID | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| --- | --- | --- | --- | --- | --- |
| VC-01 | `docs/design/opendesign/modals/create-agent.html` — simple | `agent-create-dialog` story — simple | 1440×900 | normative | MCP multi-select removed (T1); runtime truth wins |
| VC-02 | `docs/design/opendesign/modals/create-agent.html` — advanced | `agent-create-dialog` story — advanced | 1440×900 | normative | Same as VC-01 |
| VC-03 | `docs/design/opendesign/modals/create-agent.html` — simple | same — mobile stack | 360×800 | normative | 44px targets |
| VC-04 | `docs/design/opendesign/modals/create-bridge.html` — simple credentials | `bridge-create-dialog` story — simple | 1440×900 | normative | Provider catalog from daemon fixtures |
| VC-05 | `docs/design/opendesign/modals/create-bridge.html` — advanced | `bridge-create-dialog` story — advanced | 1440×900 | normative | None for section grammar |
| VC-06 | `docs/design/opendesign/modals/edit-bridge.html` — rotate secrets + delivery | `bridge-edit-dialog` story — advanced w/ delivery | 1440×900 | normative | Standalone delivery dialog deleted (D2) |
| VC-07 | `docs/design/opendesign/modals/create-mcp-server.html` — simple local | `mcp-server-editor` / MCP stories — create simple | 1440×900 | normative | Transport RadioCards normative |
| VC-08 | `docs/design/opendesign/modals/edit-mcp-server.html` — advanced remote | MCP stories — edit advanced remote | 1440×900 | normative | ImmutableIdentity for name/scope |
| VC-09 | `docs/design/opendesign/modals/create-sandbox-profile.html` — simple | sandbox create via SettingsEditorDialog | 1440×900 | normative | Inspect sheet stays separate |
| VC-10 | `docs/design/opendesign/modals/edit-sandbox-profile.html` — advanced daytona | sandbox edit — advanced daytona | 1920×1080 | normative | Daytona block only when backend=daytona |
| VC-11 | `docs/design/opendesign/modals/create-agent.html` — dense wide | agent create | 1920×1080 | normative | Layout density only |

Evidence: `.compozy/tasks/modals-redesign/evidence/visual/task_03/<contract-id>/…`.

## Subtasks

- [x] 3.1 Flatten agent create wizard → Simple/Advanced; cut T1 from code + design HTML; keep adapter category_path split/join
- [x] 3.2 Flatten bridge create wizard → Simple/Advanced with dynamic SecretField slots
- [x] 3.3 Migrate bridge edit (md host, ImmutableIdentity, SecretField rotate, D2 delivery fold + delete standalone dialog)
- [x] 3.4 Migrate MCP create/edit (md host, F1 accent eyebrow, Simple/Advanced, preserve signals)
- [x] 3.5 Migrate sandbox create/edit body (Simple/Advanced, backend cards, secret-env rotate, ImmutableIdentity on edit name)
- [x] 3.6 Extend existing suites/stories; enforce ≤500 LOC via extractions sections if needed
- [x] 3.7 Capture Visual Contract evidence VC-01…VC-11; flag QA scenarios

## Implementation Details

See `_techspec.md` §4.1, §4.8–4.11, §4.13–4.14, §5.2, §6 states, §13 items 1/6/7. Secret preservation contracts: MCP `settings_collection_payloads.go`, bridge `PutBridgeSecretBindingRequest`. Sandbox inspect sheet must keep launching the editor via `onRequestEdit`.

### Relevant Files

- `web/src/systems/agent/components/agent-create-dialog.tsx` (+ runtime/access step files — delete or collapse)
- `web/src/systems/bridges/components/bridge-create-dialog.tsx`
- `web/src/systems/bridges/components/bridge-edit-dialog.tsx`
- `web/src/systems/bridges/components/bridge-test-delivery-dialog.tsx` — delete target (D2)
- `web/src/systems/settings/components/mcp-server-editor.tsx` (+ mcp-editor-*-section)
- `web/src/systems/settings/components/mcp-secret-binding.tsx` — align with shared SecretField
- `web/src/systems/sandbox/routes/sandbox-page.tsx`
- `docs/design/opendesign/modals/create-agent.html` — T1 removal
- Existing `*bridge*`, `*agent-create*`, `*mcp*`, sandbox tests/stories

### Dependent Files

- Bridge/agent/MCP/sandbox OS locations and marketplace MCP installers
- `internal/api/contract/{agent_observe_payloads,bridges,settings_*}.go` — read-only contract truth
- `docs/qa/scenarios/` — agent create, bridge create/edit, MCP add, sandbox profile flags

## Deliverables

- Agent/bridge wizards removed; Simple/Advanced disclosure live
- T1 cut in code + design HTML; D2 delivery folded; standalone delivery dialog deleted
- MCP + sandbox on shared grammar with secret rotation/preserve
- Visual Contract bundles for VC-01…VC-11 **(REQUIRED)**
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] Agent create: no stepper; simple submits with name+prompt+runtime+permissions; advanced tools/toolsets/deny/skills appear only in advanced; no MCP servers field in UI or request body
- [x] Agent create: category_path mono input maps to `[]string` at adapter boundary
- [x] Bridge create: wizard gone; secret_slots render dynamically; provider_config rejects secret-shaped values (client does not send them)
- [x] Bridge edit: platform/extension/scope ImmutableIdentity; secret rotate; delivery test reachable from Advanced only (standalone dialog gone)
- [x] MCP edit: name/scope ImmutableIdentity; untouched secrets emit preserve flags; remote auth section only for remote transport
- [x] Sandbox: simple shows name+backend; daytona advanced fields gated; edit name ImmutableIdentity; secret-env rotate
- [x] Save-error retains draft; edit submit enabled only when dirty where PATCH HasChanges applies
- [x] Production sources stay ≤500 lines per file after extraction
- [x] Turbo lint/typecheck/test green for `./web`

### Web/Docs Impact

- `web/`: agent, bridges, settings/MCP, sandbox dialogs + deleted delivery dialog + design HTML T1 cut; stories/tests co-ship.
- `packages/site`: none — checked CLI/HTTP docs; reason: no public API shape change (UI-only cut of untruthful control).
- QA impact: reset/add `untested` for agent create, bridge create/edit/delivery, MCP add/edit, sandbox create/edit — flag, don't retest.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: consumes existing `GET /api/bridges/providers` + settings MCP collection; no new extension surface.
- Agent manageability: none for wire — checked bridge/MCP/sandbox/agent HTTP+CLI; reason: payloads unchanged aside from removing a never-accepted MCP field from the UI.
- Config lifecycle: none — checked `config.toml`.

### AGH Impact Audit

- Native tools: no impact — checked tool IDs/descriptors.
- Extensibility and hooks: bridge provider catalog consumption only; checked registries/bundles/hooks unchanged.
- Workspace data isolation: scope+workspace_id form gates retained on agent/bridge/MCP/sandbox.
- Official AGH skill: no impact — checked `skills/agh/` (no new authoring capability for MCP-on-agent).

## Success Criteria

- Every assigned test case implemented and passing
- Wizards deleted; T1/D2 complete; secret/identity contracts proven by tests
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
- No production file over 500 lines introduced or grown past the cap
