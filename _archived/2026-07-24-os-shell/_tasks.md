---
schema_version: "compozy.tasks/v2"
workflow: os-shell
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
    - id: task_08
      file: task_08.md
    - id: task_09
      file: task_09.md
    - id: task_10
      file: task_10.md
    - id: task_11
      file: task_11.md
    - id: task_12
      file: task_12.md
  edges:
    - from: task_01
      to: task_02
    - from: task_02
      to: task_04
    - from: task_03
      to: task_04
    - from: task_04
      to: task_05
    - from: task_04
      to: task_06
    - from: task_05
      to: task_06
    - from: task_04
      to: task_07
    - from: task_06
      to: task_08
    - from: task_07
      to: task_08
    - from: task_08
      to: task_09
    - from: task_05
      to: task_10
    - from: task_09
      to: task_10
    - from: task_10
      to: task_11
    - from: task_11
      to: task_12
---

# AGH OS Shell Task List

## MVP Boundary

Tasks 01-10 implement the complete OS shell v1 — MVP is full parity per `_techspec.md` §MVP Boundary (shell, window manager, all 14 app windows, multi-session, desktop-state service with real-time sync, attention surfaces, palette, unified window head, the ADR-009 window snap layer, spaces, appearance, compact mode, window-scoped modals). Tasks 11-12 prepare and execute QA on the living `docs/qa/` tree. Post-MVP follow-ups (tiling window mode, snap-layout editor / custom zones / multi-zone span / linked seam resize, auto-arrange ("tidy"), dock reordering, popout-to-native-window, extension-facing state API, native `agh__*` desktop tools, first-class attention queue, inline bell approve/deny) remain future TechSpecs.

| # | Title | Status | Complexity | Dependencies |
|---|---|---|---|---|
| 01 | Desktop-state engine (`internal/clientstate`) | completed | high | - |
| 02 | Desktop-state public surface (contract, HTTP/UDS/WS, CLI, wiring) | completed | critical | task_01 |
| 03 | Shell tokens, chrome primitives, and overlay portal context | completed | medium | - |
| 04 | Shell core: window manager, routing coordinator, DesktopShell, palette | completed | critical | task_02, task_03 |
| 05 | Attention surfaces and multi-instance sessions | completed | high | task_04 |
| 06 | App ports wave: ten apps into windows | completed | high | task_04, task_05 |
| 07 | Network app port | completed | high | task_04 |
| 08 | Window head absorbs PageHead | completed | critical | task_06, task_07 |
| 09 | Window snap layer (ADR-009) | pending | high | task_08 |
| 10 | Spaces, appearance, compact mode, and final hard-cut sweep | pending | critical | task_05, task_09 |
| 11 | QA Plan and Session Charters | pending | high | task_10 |
| 12 | Real-User QA Execution | pending | critical | task_11 |
