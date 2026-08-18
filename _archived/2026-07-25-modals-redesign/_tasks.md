---
schema_version: "compozy.tasks/v2"
workflow: modals-redesign
graph:
  nodes:
    - id: task_01
      file: task_01.md
    - id: task_02
      file: task_02.md
    - id: task_03
      file: task_03.md
    - id: task_04
      file: task_04.md
    - id: task_05
      file: task_05.md
    - id: task_06
      file: task_06.md
  edges:
    - from: task_01
      to: task_02
    - from: task_01
      to: task_03
    - from: task_01
      to: task_04
    - from: task_02
      to: task_05
    - from: task_03
      to: task_05
    - from: task_04
      to: task_05
    - from: task_05
      to: task_06
---

# Modal Redesign Task List

Land the 16 designed entity editors from `docs/design/opendesign/modals/` into production `web/` using the shared modal shell in `_techspec.md` (copied from `_spec.md`). Authority on conflict: runtime contract > tokens/`@agh/ui` > TechSpec > static HTML. Open decisions are locked per recommendations: **D1(b)** keep provider dialog, **D2** fold bridge delivery test into edit-bridge, **D5** cut MCP-on-agent.

## MVP Boundary

- **MVP — tasks 01–04.** Foundation primitives + reference repair + SettingsEditorDialog upgrade; low-risk entity dialogs; dense wizard→Simple/Advanced migrations; provider body grammar.
- **QA — tasks 05–06.** Plan and execute against the living `docs/qa/` tree with Playwright/visual-contract evidence.
- **Out of scope this wave:** agent edit route symmetry (D3), NumberedSection ordinal standardization as a separate redesign (D4 may land a minimal optional ordinal only if FormSection already supports it without a third patch), Task/Job/Trigger product expansions beyond reference header repair.

## Tasks

| # | Title | Status | Complexity | Dependencies |
| --- | --- | --- | --- | --- |
| 01 | Modal foundation + reference shell | completed | high | - |
| 02 | Low-risk entity dialogs | completed | medium | task_01 |
| 03 | Dense dialogs (wizard → Simple/Advanced) | completed | critical | task_01 |
| 04 | Provider dialog body grammar | completed | high | task_01 |
| 05 | QA Plan and Session Charters | pending | high | task_02, task_03, task_04 |
| 06 | Real-User QA Execution | pending | critical | task_05 |

## Test Contract Assignment

No `_tests.md` exists for this workflow. Each implementation task carries concrete inline cases in its `## Tests` section. No orphan or duplicate IDs.

## AGH Impact Audit

- **Native tools:** no impact — UI shell/form migration only; checked `internal/tools/**`, descriptors, capability gates.
- **Extensibility and hooks:** consumption of existing bridge provider catalog + settings MCP collection only; no registry/bundle/hook/config-lifecycle change. Checked extensions, hooks, skills/capabilities, tools/resources, bundles, registries, bridge SDKs, MCP sidecars.
- **Workspace data isolation:** scoped surfaces continue to send `scope` + `workspace_id` on existing contracts; no new cross-workspace read path.
- **Official AGH skill:** no impact — no public tool ID, CLI path, hook event, or capability semantic change; checked `skills/agh/`.
