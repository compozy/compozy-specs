---
status: completed
title: Modal foundation + reference shell
type: frontend
complexity: high
---

# Task 01: Modal foundation + reference shell

## Overview

Delivers the shared modal shell (F1–F7) that every later surface migration depends on: `EntityDialogHeader`, modal footer composition, host size helper, generic Simple/Advanced toolbar, `SecretField`, `ImmutableIdentity`, and the split body variant. Also repairs the two reference surfaces (R1 task header, R2 automation header lift) and upgrades `SettingsEditorDialog` so vault/sandbox inherit ruled chrome before their body migrations.

<critical>
- ALWAYS READ `_techspec.md`, `MODAL-STANDARD.md`, `STATE-MATRIX.md`, `VISUAL-VALIDATION.md`, `CHECKLIST.md`, and `docs/design/opendesign/design-system/patterns.html` § Modals before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST lift the local `EditorHeader` from `automation-editor-dialog.tsx` into `packages/ui` as `EntityDialogHeader` (36px accent icon well + accent-strong Eyebrow + DialogTitle + DialogDescription + optional close) and delete the duplicate local definition in the same change.
- MUST add a modal `EntityDialogFooter` composition on `DialogFooter variant="ruled"` with optional hint slot, Cancel outline, and one primary verb+object action with saving spinner — MUST NOT restyle `EditorFooter`.
- MUST add `dialogShellClass(size)` emitting `--width-modal-{sm,md,lg,xl}` token classes (and height caps) and document the size→surface map from TechSpec §2 F3; provider `sheet` size stays unused pending task_04 D1(b).
- MUST generalize task `ModeToolbar` into a domain-agnostic `EntityModeToolbar` (`simple`/`advanced`, optional trailing ScopeSelector) with PillGroup `role="group"` + `aria-pressed`, one disclosure tier only, and Advanced snap-back of unsupported selections.
- MUST ship shared `SecretField` (states: absent/present/editing/invalid/saving/rotated) by generalizing marketplace `mcp-secret-field.tsx`, delete that marketplace copy, and re-point `mcp-install-dialog.tsx`.
- MUST ship `ImmutableIdentity` readable summary rows; disabled inputs as data display are forbidden.
- MUST ship split-dialog body variant (`xl` host, two scroll owners, collapses ≤980px) for add-workspace (consumed in task_02).
- MUST re-apply `EntityDialogHeader` on `task-editor-surface.tsx` (R1) and remove the in-body description `<p>`.
- MUST upgrade `SettingsEditorDialog` onto F1+F2+F3 so vault/sandbox chrome lifts without per-page header forks; keep `settings-dialogs.stories.tsx` as the shell pin.
- MUST export new composites from `@agh/ui` (`index.ts` / `primitives.ts` as appropriate) with stories + unit tests; MUST NOT shadow existing primitives in `web/`.
- SHOULD leave FormSection ordinal work (D4) out unless an optional ordinal already fits without a structural redesign.
</requirements>

## Visual Contract

| ID | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| --- | --- | --- | --- | --- | --- |
| VC-01 | `docs/design/opendesign/modals/create-task-redesign.html` — default open header | Task editor surface / `task-editor-modal` story — create mode header | 1440×900 | normative shell | Content/copy follow production `TASK_DESCRIPTION` and runtime truth (`COPY.md`); brand marks from `@agh/ui` inventory |
| VC-02 | `docs/design/opendesign/modals/create-job-redesign.html` — default open header | `automation-editor-dialog` story — create job header on shared F1 | 1440×900 | normative shell | Preview rail and job fields may diverge from historical artboard; header anatomy is normative |
| VC-03 | `docs/design/opendesign/modals/create-vault-secret.html` — header + footer chrome | `settings-dialogs` / vault create via `SettingsEditorDialog` — chrome only | 1440×900 | normative shell | Body field content owned by task_02; this row validates shell tokens/header/footer only |
| VC-04 | `docs/design/opendesign/modals/create-task-redesign.html` — default open | `task-editor-modal` story — create mode, tablet | 768×900 | normative shell | Above the 760px threshold, so the host stays centred by design; artboard geometry vs tokens |
| VC-05 | `docs/design/opendesign/modals/create-task-redesign.html` — default open | `task-editor-modal` story — create mode, bottom surface | 360×800 | normative shell | Artboard never implemented the responsive rule; bottom surface + 44px targets per `MODAL-STANDARD.md` § Hosts |
| VC-06 | `docs/design/opendesign/modals/create-vault-secret.html` — chrome | `settings-dialogs` VaultCreate — chrome only, tablet | 768×900 | normative shell | Body owned by task_02; above the 760px threshold |
| VC-07 | `docs/design/opendesign/modals/create-vault-secret.html` — chrome | `settings-dialogs` VaultCreate — bottom surface | 360×800 | normative shell | Body owned by task_02; bottom surface + 44px targets |
| VC-08 | `docs/design/opendesign/modals/add-workspace.html` — split populated | `entity-dialog-body` Split story — two panes | 1440×900 | normative shell | task_01 ships F7 structure only; pane content and the global-default card land in task_02 |
| VC-09 | `docs/design/opendesign/modals/add-workspace.html` — split populated | `entity-dialog-body` Split story — stacked | 768×900 | normative shell | Collapse threshold 980px per TechSpec/task text vs `patterns.html` "≤960"; pane content owned by task_02 |

Evidence for each row: `.compozy/tasks/modals-redesign/evidence/visual/task_01/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

VC-04…VC-09 were added by the controller's designer-agent execution audit. Non-contract state coverage (header with/without description and close, footer hint/default/saving/disabled, SecretField absent/present/editing/invalid/saving/rotated/bound, toolbar simple/advanced/no-trailing, focus-visible on close/mode/footer, reduced motion) ships as Storybook stories plus unit tests rather than parity bundles — those states have no artboard counterpart to diff against.

## Subtasks

- [x] 1.1 Create `EntityDialogHeader` in `packages/ui`, export it, add story + unit test
- [x] 1.2 Create modal `EntityDialogFooter` composition (hint + Cancel + primary), story + test
- [x] 1.3 Add `dialogShellClass` helper and wire reference hosts (task md, automation xl)
- [x] 1.4 Extract generic `EntityModeToolbar`; keep task form consuming it without behavior regression
- [x] 1.5 Ship `SecretField` + delete marketplace `mcp-secret-field.tsx`; re-point install dialog
- [x] 1.6 Ship `ImmutableIdentity` composite with story + test
- [x] 1.7 Ship split body variant (structure only; workspace migrate in task_02)
- [x] 1.8 Refactor automation `EditorHeader` onto F1 (R2)
- [x] 1.9 Restore task editor header (R1) and remove in-body description paragraph
- [x] 1.10 Upgrade `SettingsEditorDialog` + `settings-dialogs.stories.tsx` pin
- [x] 1.11 Run `bunx turbo run lint typecheck test --filter=./packages/ui --filter=./web` for touched packages

## Implementation Details

See `_techspec.md` §2 Foundation (F1–F7), §3 Shell contract, §4.17 reference repairs, §13 delete targets 2–3 and 5. Canonical lift source: `web/src/systems/automation/components/automation-editor-dialog.tsx` local `EditorHeader`. Token geometry is non-negotiable (`MODAL-STANDARD.md` § Hosts).

### Relevant Files

- `packages/ui/src/components/dialog.tsx` — `DialogHeader`/`DialogFooter` ruled variants
- `packages/ui/src/components/custom/editor-footer.tsx` — page footer; do not restyle
- `packages/ui/src/components/custom/confirm-dialog.tsx` — out-of-scope neutral well reference
- `packages/ui/src/index.ts`, `packages/ui/src/primitives.ts` — export surface
- `web/src/systems/automation/components/automation-editor-dialog.tsx` — R2 lift source
- `web/src/systems/tasks/components/task-editor-surface.tsx` — R1 header regression
- `web/src/systems/tasks/components/task-form/mode-toolbar.tsx` — F4 source
- `web/src/systems/marketplace/components/mcp-secret-field.tsx` — F5 source (delete)
- `web/src/systems/settings/components/settings-editor-dialog.tsx` — shared shell for vault/sandbox
- `web/src/systems/settings/components/stories/settings-dialogs.stories.tsx` — shell pin
- `docs/design/opendesign/modals/modal-system.css` — static anatomy reference

### Dependent Files

- `web/src/systems/marketplace/components/mcp-install-dialog.tsx` — SecretField consumer after delete
- All §4 surfaces (tasks 02–04) — consume F1–F7
- `web/src/systems/tasks/components/__tests__/task-editor-modal.test.tsx` — header assertions
- `web/src/systems/automation/components/__tests__/automation-editor-dialog.test.tsx` — header refactor
- `web/src/systems/settings/components/__tests__/settings-editor-dialog.test.tsx` — shell upgrade

## Deliverables

- Shared F1–F7 primitives exported from `@agh/ui` / web shared helpers with stories + tests
- R1/R2 reference headers on the shared primitive; no local `EditorHeader` duplicate
- `SettingsEditorDialog` on F1+F2+F3
- Marketplace secret field deleted; install dialog re-pointed
- Every Visual Contract row has a durable passing evidence bundle **(REQUIRED)**
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

No `_tests.md` — inline cases:

- [x] `EntityDialogHeader` renders icon well + accent eyebrow + title + optional description/close; ConfirmDialog remains unaffected
- [x] `EntityDialogFooter` exposes hint slot and exactly one primary action; saving shows Spinner and disables duplicate submit
- [x] `dialogShellClass("sm"|"md"|"lg"|"xl")` emits modal width token classes (no raw `max-w-*`)
- [x] `EntityModeToolbar` toggles simple/advanced via `aria-pressed` PillGroup; Advanced never hides required fields; leaving Advanced snaps unsupported advanced-only values
- [x] `SecretField` create path is write-only; edit path cycles present → editing → present on cancel without exposing plaintext from GET
- [x] `ImmutableIdentity` is not an input/`disabled` field
- [x] Task editor surface shows F1 header and no in-body description paragraph
- [x] Automation editor uses shared header (no local EditorHeader definition)
- [x] `SettingsEditorDialog` stories/tests pin ruled header + footer hint slot + host token

### Web/Docs Impact

- `web/`: `packages/ui` composites; `settings-editor-dialog.tsx`; `automation-editor-dialog.tsx`; `task-editor-surface.tsx`; `mode-toolbar.tsx`; marketplace secret field delete + `mcp-install-dialog.tsx`; stories/tests listed above.
- `packages/site`: none — checked surfaces: `packages/site/content/runtime/**`; reason: no CLI/HTTP/config public contract change.
- QA impact: reset `qa_status` to `untested` on scenarios covering task/automation editor chrome if present; flag shell-related UI changes — do not retest in this task.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked extensions, hooks, skills/capabilities, tools/resources, bundles, registries, bridge SDKs, MCP sidecars; reason: UI primitive extraction only.
- Agent manageability: none — checked CLI verbs, HTTP/UDS routes, structured outputs; reason: no wire change.
- Config lifecycle: none — checked `config.toml` keys/defaults; reason: unchanged.

### AGH Impact Audit

- Native tools: no impact — checked `internal/tools/**`, descriptors, capability gates.
- Extensibility and hooks: no impact — checked extensions/hooks/skills/tools/resources/bundles/registries/bridge SDKs/MCP sidecars.
- Workspace data isolation: no impact — no new scoped datum; shell only.
- Official AGH skill: no impact — checked `skills/agh/`; no public semantic change.

## Success Criteria

- Every assigned test case implemented and passing
- F1–F7 land with stories/tests; R1/R2 and SettingsEditorDialog consume them
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
- `bunx turbo run lint typecheck test --filter=./packages/ui --filter=./web` green for touched filters
