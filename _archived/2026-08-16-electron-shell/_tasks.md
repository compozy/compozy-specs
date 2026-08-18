---
schema_version: "compozy.tasks/v2"
workflow: electron-shell
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
    - from: task_01
      to: task_03
    - from: task_03
      to: task_04
    - from: task_04
      to: task_05
    - from: task_02
      to: task_06
    - from: task_05
      to: task_06
    - from: task_06
      to: task_07
---

# Electron Desktop Shell Migration Task List

Spec: [_spec.md](_spec.md) (finalized after 3 incorporated peer-review rounds — see `qa/peer-review-*-round{1,2,3}.md`). Companions: [_user_stories.md](_user_stories.md), [_dx.md](_dx.md), [_uiux.md](_uiux.md), [_tests.md](_tests.md), ADRs 001–009 under [adrs/](adrs/).

## MVP Boundary

Tasks 01–05 implement the migration MVP — Part II phases P1–P4: the Update Operation mechanism with its single command and daemon surfaces (01), the web update surface (02), the Electron shell at parity (03), the release pipeline with the channel authority (04), and the cutover with the docs truth pass (05). Tasks 06–07 prepare and execute QA over the living `docs/qa/` tree, including the Part I release gate (all `APP-*` scenarios recorded `pass` on macOS + Linux and one real beta N→N+1 auto-update cycle). Post-MVP, out of scope per Part I Non-Goals: the Windows target, tray/notifications/global shortcuts/auto-launch, any migration bridge, Safari-parity work, web UI changes beyond the update surface, and rebranding.

Parallelism: after task_01, tasks 02 (web/) and 03 (desktop/) touch disjoint files and run as parallel waves.

Test-contract audit: 82 active `UT-` ids (UT-073 is withdrawn in `_tests.md`), 24 `IT-` ids, and 34 `E2E-` ids are each assigned to exactly one task below.

| #  | Title | Status | Complexity | Dependencies |
| -- | ----- | ------ | ---------- | ------------ |
| 01 | Update Operation, single command, and daemon surfaces | pending | critical | - |
| 02 | Web update surface (settings two-track + menubar indicator) | pending | medium | task_01 |
| 03 | Electron shell at parity | pending | critical | task_01 |
| 04 | Release pipeline and channel authority | pending | high | task_03 |
| 05 | Cutover and docs truth pass | pending | high | task_04 |
| 06 | QA Plan and Session Charters | pending | high | task_02, task_05 |
| 07 | Real-User QA Execution | pending | critical | task_06 |
