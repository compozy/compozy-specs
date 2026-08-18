---
schema_version: "compozy.tasks/v2"
workflow: desktop-app
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
    - id: task_07
      file: task_07.md
  edges:
    - from: task_01
      to: task_02
    - from: task_02
      to: task_03
    - from: task_03
      to: task_04
    - from: task_04
      to: task_05
    - from: task_05
      to: task_06
    - from: task_06
      to: task_07
---

# CompozyOS Desktop App Task List

## MVP Boundary

Tasks 01–05 implement the MVP: the complete PRD scope (thin-shell app, attach-first resolution with guided provisioning, full update system, agent-operable surfaces, release pipeline) plus the absorbed Compozy→CompozyOS product-language hard cut (ADR-012). Tasks 06–07 are the mandatory QA tail (planning + execution) over the living `docs/qa/` tree. Post-MVP items (bundled `frontendDist` escalation, tray mode, stable-channel activation, command-identifier renames, multi-window) are explicitly out of scope per the TechSpec MVP Boundary.

| #  | Title | Status | Complexity | Dependencies |
| --- | ----- | ------ | ---------- | ------------ |
| task_01 | Tauri shell: windows, native integration, runtime resolution | completed | high | - |
| task_02 | Go surfaces: `compozy app` verbs, config lifecycle, detection, quiesce contract | completed | high | task_01 |
| task_03 | Update system: provisioning, runtime apply, app auto-update, control socket | completed | critical | task_02 |
| task_04 | Product language + docs: brand hard cut, site pages, official skill | completed | medium | task_03 |
| task_05 | Release pipeline: build matrix, signing, feeds, gates, custody runbook | completed | critical | task_04 |
| task_06 | qa-report: plan the desktop QA cycle in docs/qa | completed | high | task_05 |
| task_07 | qa-execution: isolated platform walks, fix loop, clean teardown | completed | critical | task_06 |
