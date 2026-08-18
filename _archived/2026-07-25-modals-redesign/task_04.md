---
status: completed
title: Provider dialog body grammar
type: frontend
complexity: high
---

# Task 04: Provider dialog body grammar

## Overview

Aligns `provider-detail-dialog` (create/edit/inspect) with the modal standard while keeping the production dialog host (**D1(b)**): F1 header, SettingsFieldRow body grammar, auth-ownership RadioCards, and bound_secret-only credential slots (T3). Updates `MODAL-STANDARD.md` Hosts to retire the unresolved sheet vs dialog fork and fixes stale e2e selectors that still say `provider-inspector-sheet`.

<critical>
- ALWAYS READ `_techspec.md` §4.15–4.16, §5.2 T3, §14 D1, `MODAL-STANDARD.md`, and the provider sheet HTML references before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST keep `provider-detail-dialog.tsx` as the host (dialog + LaneTabs), NOT revert to a 576px sheet (D1(b)).
- MUST apply `EntityDialogHeader` (gear glyph · Settings · Provider · Create/Edit title) and shared footer grammar; add icon well missing today.
- MUST structure body with `SettingsFieldRow` rows, auth-ownership `RadioCard`s (`native_cli` vs `bound_secret`), and runtime/model fields per §4.15 — **RuntimeSelector is forbidden** on this surface.
- MUST gate credential slots and secret writes so they render **only** when `auth_mode = bound_secret`; under `native_cli` show login command guidance only (T3 / provider auth boundary).
- MUST use `ImmutableIdentity` for provider name on edit; credential values are rotate-only via `SecretField`.
- MUST update `MODAL-STANDARD.md` § Hosts to document the dialog host decision (remove pending sheet ambiguity for provider).
- MUST update Playwright/e2e selectors and stories that still reference `provider-inspector-sheet` to the dialog contract.
- MUST NOT invent credential UX that the daemon cannot accept; field map is TechSpec §5.1 Provider row.
</requirements>

## Visual Contract

| ID | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| --- | --- | --- | --- | --- | --- |
| VC-01 | `docs/design/opendesign/modals/create-provider-sheet.html` — create body | `provider-detail-dialog` story — create | 1440×900 | normative body | Host is dialog not sheet (D1(b) / TechSpec §14); LaneTabs inspect chrome is production-authorized |
| VC-02 | `docs/design/opendesign/modals/edit-provider-sheet.html` — edit + rotate | `provider-detail-dialog` story — edit | 1440×900 | normative body | Same host delta as VC-01; ImmutableIdentity for name |
| VC-03 | `docs/design/opendesign/modals/create-provider-sheet.html` — bound_secret slots visible | create — auth_mode=bound_secret | 1440×900 | normative | Slots only when bound_secret |
| VC-04 | `docs/design/opendesign/modals/create-provider-sheet.html` — native_cli no slots | create — auth_mode=native_cli | 1440×900 | normative | Login command guidance only |
| VC-05 | create-provider-sheet — dense | provider create | 1920×1080 | normative | Density/spacing |
| VC-06 | edit-provider-sheet — mobile stack | provider edit | 360×800 | normative | 44px targets; sheet artboard width ignored |

Evidence: `.compozy/tasks/modals-redesign/evidence/visual/task_04/<contract-id>/…`.

## Subtasks

- [x] 4.1 Apply F1/F2 to provider-detail-dialog; keep LaneTabs inspect/edit/create modes
- [x] 4.2 Rebuild create/edit body grammar (SettingsFieldRow, auth RadioCards, no RuntimeSelector)
- [x] 4.3 Implement T3 auth gate + SecretField rotate on edit credentials
- [x] 4.4 ImmutableIdentity for edit name; wire PutSettingsProviderRequest secrets list correctly
- [x] 4.5 Update MODAL-STANDARD Hosts for D1(b); fix e2e/story selectors from sheet → dialog
- [x] 4.6 Add/extend `provider-detail-dialog` tests + stories; capture VC-01…VC-06
- [x] 4.7 Flag QA scenarios (`MS-provider-detail-modal`, providers redesign, etc.)

## Implementation Details

See `_techspec.md` §4.15–4.16, §5.2 T3, §14 D1. Production entry: `web/src/routes/_app/settings/-providers-settings-page.tsx`. Auth boundary authority: `internal/CLAUDE.md` § Provider auth boundary.

### Relevant Files

- `web/src/systems/settings/components/provider-detail-dialog.tsx`
- `web/src/systems/settings/components/provider-edit-form*.tsx` (discover exact siblings at execution)
- `web/src/systems/settings/components/stories/provider-detail-dialog.stories.tsx`
- `web/src/routes/_app/settings/-providers-settings-page.tsx`
- `.compozy/tasks/modals-redesign/MODAL-STANDARD.md` — Hosts update
- `docs/design/opendesign/modals/create-provider-sheet.html`, `edit-provider-sheet.html`
- E2E specs under `web/e2e/` referencing provider inspector/sheet (discover at execution)

### Dependent Files

- Settings providers MSW fixtures / handlers
- `docs/qa/scenarios/MS-provider-detail-modal.md`, `MS-web-settings-providers-redesign.md`

## Deliverables

- Provider dialog on F1 + SettingsFieldRow grammar with T3 auth gate
- MODAL-STANDARD Hosts updated for D1(b)
- Stale sheet selectors removed
- Visual Contract bundles for VC-01…VC-06 **(REQUIRED)**
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] Create/edit render EntityDialogHeader with icon well; RuntimeSelector is not mounted
      — `provider-detail-dialog.test.tsx` "Should render the entity header with an icon well and no runtime selector on create"
- [x] `auth_mode=native_cli` hides credential SecretFields and shows login command guidance
      — same suite, "Should hide credential controls under native_cli…" + "Should reveal credential slots only after auth ownership becomes bound_secret"; VC-04 proves the absence visually
- [x] `auth_mode=bound_secret` shows credential slots; edit path is rotate-only (no GET plaintext)
      — same suite, "Should keep a stored credential rotate-only on edit" + "Should offer a write input only for a vault-backed credential ref"; hook suite "Should seed an edit draft with empty secret values for every stored slot"
- [x] Edit name is ImmutableIdentity; PUT payload matches PutSettingsProviderRequest field set
      — same suite, "Should render the provider name as immutable identity on edit"; hook suite "submits a full replacement on save…", "submits vault-backed provider secrets without reading them back", "Should drop credential slots from the request when auth ownership is not bound_secret"
- [x] LaneTabs inspect mode still opens without regressing create/edit entry points
      — same suite, "Should open the edit form from the Configure tab without losing inspect" + "Should open the edit form from the inspect footer action"
- [x] E2E selectors targeting `provider-inspector-sheet` are updated or removed
      — verified absent from `web/src` and `web/e2e` (the sheet component was already replaced by `provider-detail-dialog`); `settingsProvidersTestIds` gained `editorModeSimple`/`editorModeAdvanced` and `settings.spec.ts` now opens Advanced before writing the default model
- [x] Turbo lint/typecheck/test green for `./web`; provider story pins auth gate
      — `bunx turbo run lint typecheck test --filter=./web --filter=./packages/ui --force`: 7/7 tasks, agh-web 488 files / 3857 tests, @agh/ui 122 files / 609 tests, 0 lint warnings; `CreateBoundSecret` / `CreateNativeCli` stories pin the gate

### Web/Docs Impact

- `web/`: provider-detail-dialog + edit forms + stories/tests + providers settings page + e2e selectors.
- `packages/site`: none — checked provider/settings docs pages; reason: no CLI/config key change (UI host decision only).
- QA impact: reset `MS-provider-detail-modal` and `MS-web-settings-providers-redesign` (and any sheet-named twins) to `untested`; add new content-addressed scenarios for auth-gate create if missing.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked provider settings collection consumption only.
- Agent manageability: none — checked provider CLI/HTTP; reason: payload unchanged, UI gating only.
- Config lifecycle: none — checked `config.toml` provider keys; reason: unchanged.

### AGH Impact Audit

- Native tools: no impact — checked tool IDs/descriptors.
- Extensibility and hooks: no impact — checked extensions/hooks/skills.
- Workspace data isolation: provider settings remain on existing scope rules; no new cross-workspace read.
- Official AGH skill: no impact — checked `skills/agh/`.

## Success Criteria

- Every assigned test case implemented and passing
- D1(b) + T3 proven; MODAL-STANDARD Hosts updated
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
- QA scenario flags written for provider flows
