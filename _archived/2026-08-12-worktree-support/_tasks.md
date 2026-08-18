---
schema_version: "compozy.tasks/v2"
workflow: worktree-support
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
  edges:
    - from: task_01
      to: task_02
    - from: task_01
      to: task_03
    - from: task_03
      to: task_04
    - from: task_02
      to: task_05
    - from: task_04
      to: task_05
    - from: task_02
      to: task_06
    - from: task_05
      to: task_07
    - from: task_06
      to: task_07
    - from: task_07
      to: task_08
    - from: task_08
      to: task_09
    - from: task_09
      to: task_10
---

# Native Worktree Support Task List

## MVP Boundary

Tasks 01-08 implement the complete v1 scope of `_prd.md`/`_techspec.md` — schema and worktree domain core, public surfaces, session binding, task/loop environment policies, assisted exit with the `forge.provider` surface and bundled GitHub extension, web UI, and docs. Tasks 09-10 prepare and execute QA over the living `docs/qa/` tree. Post-MVP (not in this program): automation jobs/triggers worktree mode, archive/restore, bulk cleanup, GitHub device-flow OAuth, disk-size measurement, a worktree directive on the loop `fan-out` node itself, non-GitHub forge extensions, transcript-carrying session fork.

| # | Title | Status | Complexity | Dependencies |
| --- | --- | --- | --- | --- |
| 01 | Schema and worktree domain core | completed | critical | - |
| 02 | Worktree public surface: API, CLI, native tools, streams | completed | high | task_01 |
| 03 | Session binding, containment, spawn inheritance, fork | completed | high | task_01 |
| 04 | Environment policies: task worktree policy + loop environment | completed | high | task_03 |
| 05 | Assisted exit, forge.provider surface, bundled GitHub extension | pending | high | task_02, task_04 |
| 06 | Web: navigation, lifecycle dialogs, worktree data layer | pending | high | task_02 |
| 07 | Web: environment surfaces and assisted exit | pending | high | task_05, task_06 |
| 08 | Docs, config examples, official Compozy skill | pending | medium | task_07 |
| 09 | QA Plan and Session Charters | pending | high | task_08 |
| 10 | Real-User QA Execution | pending | critical | task_09 |

Execution waves: `task_01` → {`task_02`, `task_03`} → {`task_04`, `task_06`} → `task_05` → `task_07` → `task_08` → `task_09` → `task_10`. Intermediate tasks close on `make gate` + task-scoped suites; `make gate-full` runs once at the program's close (QA tail).
