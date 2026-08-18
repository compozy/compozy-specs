---
status: completed
title: Low-risk entity dialogs
type: frontend
complexity: medium
---

# Task 02: Low-risk entity dialogs

## Overview

Migrates the lower-risk entity editors onto the shared shell: knowledge create/edit, network channel create/edit, vault create body, start session, and add workspace (split shell + DirectoryBrowser). Applies host tokens, F1 headers, ImmutableIdentity where update contracts forbid mutation, and session Simple/Advanced overrides — without the wizard collapses owned by task_03.

<critical>
- ALWAYS READ `_techspec.md`, `MODAL-STANDARD.md`, `STATE-MATRIX.md`, `VISUAL-VALIDATION.md`, and the named HTML references before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST route every surface in this task through `dialogShellClass` with the TechSpec size map (`sm` for session/knowledge/channel/vault; `xl` + split for workspace); MUST eliminate ad-hoc `max-w-*` / `sm:max-w-*` on these editors.
- MUST apply `EntityDialogHeader` + `EntityDialogFooter` with the header copy/glyphs from TechSpec §4.2–4.7 and §4.12.
- MUST implement start-session Simple/Advanced via `EntityModeToolbar` (Simple: agent/workspace/name plus required first-message composer; Advanced: network participation and working path) while keeping the composer-owned RuntimeSelector and catalog status truth visible in both modes (no chrome banner); Send is the only creation action.
- MUST implement add-workspace on the F7 split shell with `DirectoryBrowser` for root selection (plain path input forbidden), preserve the global-default workspace card, and keep onboarding variant on the same content component.
- MUST use `ImmutableIdentity` for knowledge edit name+type and channel edit name+members per update contracts.
- MUST use fanout `RadioCard`s on channel create/edit (coordinator `CommandSelect` only when fanout=`coordinator`).
- MUST use `SecretField` for vault create value (write-only) and require explicit overwrite confirmation copy when ref upsert is possible.
- MUST NOT invent controls without write contracts; field→contract map is TechSpec §5.1.
- SHOULD extend existing component test suites and stories rather than creating standalone regressions.
</requirements>

## Visual Contract

| ID | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| --- | --- | --- | --- | --- | --- |
| VC-01 | `docs/design/opendesign/modals/start-session.html` — simple open | `session-create-dialog` story — simple | 1440×900 | normative | Catalog fixture data may differ; required prompt composer, composer-owned RuntimeSelector, and Send-only creation follow `CreateSessionRequest.prompt` plus runtime admission truth |
| VC-02 | `docs/design/opendesign/modals/start-session.html` — advanced open | `session-create-dialog` story — advanced | 1440×900 | normative | Same as VC-01; composer extends the scroll-owned body while Advanced adds only network participation and working path |
| VC-03 | `docs/design/opendesign/modals/add-workspace.html` — split populated | `workspace-setup` story — dialog variant | 1440×900 | normative | Global-default card is production-authorized addition (TechSpec §4.3) |
| VC-04 | `docs/design/opendesign/modals/create-knowledge.html` — type pick | `knowledge-create-dialog` story | 768×900 | normative | Runtime knowledge types win over artboard labels |
| VC-05 | `docs/design/opendesign/modals/edit-knowledge.html` — immutable identity | `knowledge-edit-dialog` story | 768×900 | normative | None for identity treatment |
| VC-06 | `docs/design/opendesign/modals/create-network-channel.html` — fanout cards | `network-create-channel-dialog` story | 1440×900 | normative | RadioCard selected = glaze/rim, never accent |
| VC-07 | `docs/design/opendesign/modals/edit-network-channel.html` — locked identity | `channel-policy-dialog` story/edit | 1440×900 | normative | Production policy fields that lack write contract stay cut |
| VC-08 | `docs/design/opendesign/modals/create-vault-secret.html` — create | vault create via SettingsEditorDialog story/route | 1440×900 | normative | Inspect/replace remains `vault-secret-sheet` (out of body scope) |
| VC-09 | `docs/design/opendesign/modals/start-session.html` — stacked mobile | `session-create-dialog` — ≤760px | 360×800 | normative | 44px targets; full-width bottom surface; composer-owned Send replaces footer actions per `CreateSessionRequest.prompt` and runtime admission truth |

Evidence: `.compozy/tasks/modals-redesign/evidence/visual/task_02/<contract-id>/…`.

## Subtasks

- [x] 2.1 Migrate start session (host sm, F1/F2, Simple/Advanced, catalog status preserved)
- [x] 2.2 Migrate add workspace (xl split, DirectoryBrowser, global-default card, onboarding parity)
- [x] 2.3 Migrate knowledge create + edit (host sm, F1, ImmutableIdentity on edit)
- [x] 2.4 Migrate channel create (host sm, ruled F1, fanout RadioCards)
- [x] 2.5 Migrate channel edit via `channel-policy-dialog` (ImmutableIdentity + editable policy)
- [x] 2.6 Migrate vault create body onto upgraded SettingsEditorDialog + SecretField + overwrite Alert
- [x] 2.7 Extend existing stories/tests; capture Visual Contract evidence for VC-01…VC-09
- [x] 2.8 Flag QA scenarios for session/workspace/knowledge/channel/vault (`untested` / new content-addressed files)

## Implementation Details

See `_techspec.md` §4.2–4.7, §4.12, §5.1 field map, §7–8 a11y/responsive. Edit channel production owner is `channel-policy-dialog.tsx` (no separate edit-channel file). Vault inspect/rotate stays on `vault-secret-sheet.tsx`.

### Relevant Files

- `web/src/systems/session/components/session-create-dialog.tsx`
- `web/src/systems/workspace/components/workspace-setup.tsx`
- `web/src/systems/knowledge/components/knowledge-create-dialog.tsx`
- `web/src/systems/knowledge/components/knowledge-edit-dialog.tsx`
- `web/src/systems/network/components/network-create-channel-dialog.tsx`
- `web/src/systems/network/components/shell/channel-policy-dialog.tsx`
- `web/src/systems/vault/routes/vault-page.tsx` — VaultEditor via SettingsEditorDialog
- `web/src/systems/onboarding/` — DirectoryBrowser
- Matching `components/__tests__/*` and `components/stories/*` / route stories

### Dependent Files

- `web/src/systems/os/components/desktop-shell.tsx` — openers for session/workspace
- Knowledge/network OS locations and toolbars that mount these dialogs
- `docs/qa/scenarios/` — status flags at completion

## Deliverables

- Six surface families on shared shell with correct host tokens and headers
- Split workspace dialog with DirectoryBrowser; no plain root path input
- ImmutableIdentity on knowledge/channel edit locks
- Visual Contract bundles for VC-01…VC-09 **(REQUIRED)**
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] Session simple shows agent/workspace/name plus the required composer; advanced reveals network/path overrides without hiding the composer; RuntimeSelector catalog states still surface
- [x] Workspace dialog uses xl split; DirectoryBrowser selects root; global-default card still renders; ≤980px stacks panes
- [x] Knowledge create submits MemoryCreate fields; knowledge edit cannot mutate name/type (ImmutableIdentity) and PATCH omits them
- [x] Channel create posts fanout_policy + optional coordinator; channel edit ImmutableIdentity for name/members; UpdateNetworkChannelRequest shape only
- [x] Vault create SecretField never prefills from GET; overwrite path shows confirmation Alert
- [x] No migrated dialog in this task retains raw `max-w-*` / `sm:max-w-*` host classes
- [x] Existing suites extended; turbo lint/typecheck/test green for `./web`

### Web/Docs Impact

- `web/`: session, workspace, knowledge, network, vault dialogs + tests/stories listed above; OS openers only if wiring requires.
- `packages/site`: none — checked `packages/site/content/runtime/**`; reason: no CLI/HTTP docs contract change.
- QA impact: reset/add `untested` scenarios for session start, workspace add, knowledge create/edit, channel create/edit, vault write — flag, don't retest.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked bridge/MCP/extension surfaces; reason: form shell only on existing endpoints.
- Agent manageability: none — checked CLI/HTTP/UDS for session/workspace/memory/network/vault; reason: payloads unchanged, UI only.
- Config lifecycle: none — checked `config.toml`; reason: unchanged.

### AGH Impact Audit

- Native tools: no impact — checked tool IDs/descriptors.
- Extensibility and hooks: no impact — checked extensions/hooks/skills/tools/resources/bundles/registries.
- Workspace data isolation: existing `scope`/`workspace_id` propagation retained; prove forms still require workspace_id iff scope=workspace where applicable.
- Official AGH skill: no impact — checked `skills/agh/`.

## Success Criteria

- Every assigned test case implemented and passing
- All listed surfaces on host tokens + F1/F2 with zero ad-hoc widths
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
- QA scenario flags written for touched user-visible flows
