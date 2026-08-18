---
schema_version: "compozy.tasks/v2"
workflow: cross-workspace-access
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
      to: task_04
    - from: task_03
      to: task_04
    - from: task_02
      to: task_05
    - from: task_04
      to: task_05
    - from: task_05
      to: task_06
    - from: task_06
      to: task_07
---

# Cross-Workspace Access Task List

Derived from `_techspec.md` (post-ADR-007 mode-anchored design) and the `_tests.md` contract. Decision source is the session `PermissionMode` (`approve-all` allow / `deny-all` hard deny / `approve-reads` ask at the tool seam); no new config keys, tables, CLI verbs, native tools, or Settings surfaces exist in this program.

## MVP Boundary

Tasks 01–05 implement the MVP: phase 1 (policy core, seam enforcement, prompt + session consent, audit) and phase 0 (owner projection + web deep-link confirm), plus docs/skill/QA-flag truth. Tasks 06–07 prepare and execute the QA cycle. Post-MVP / out of scope permanently: durable grants, workspace-pair trust lists, `read`/`full` capability levels (ADR-002/003 superseded by ADR-007); OS-enforced session containment stays a named future security ADR (ADR-005 design record).

`task_03` has no dependencies and is parallelizable from day one alongside `task_01`.

| # | Title | Status | Complexity | Dependencies |
| --- | --- | --- | --- | --- |
| 01 | Workspace-access policy core, mode source, session consent cache, audit substrate | completed | high | - |
| 02 | Seam enforcement wiring, ACP prompt at the tool seam, delete sweep | completed | critical | task_01 |
| 03 | Session owner projection endpoint (phase 0 backend) | completed | low | - |
| 04 | Web deep-link confirm on owner projection (phase 0 web) | completed | medium | task_02, task_03 |
| 05 | Docs, official Compozy skill, glossary, QA scenario flags | completed | low | task_02, task_04 |
| 06 | QA Plan and Session Charters | completed | high | task_05 |
| 07 | Real-User QA Execution | completed | critical | task_06 |
