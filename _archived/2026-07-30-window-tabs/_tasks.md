---
schema_version: "compozy.tasks/v2"
workflow: window-tabs
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
    - from: task_02
      to: task_03
    - from: task_02
      to: task_04
    - from: task_03
      to: task_05
    - from: task_04
      to: task_05
    - from: task_05
      to: task_06
---

# Window Tabs Task List

## MVP Boundary

Tasks 01-04 implement the complete window-tabs v1 scope from `_prd.md`/`_techspec.md`: the daemon-side tab domain (snapshot v3, stack groups, nav stacks, pins, reopen, coalesced activation), the full public surface (contract, hooks, native tools, CLI, parity, codegen), the complete frontend (semantic identity, frame rendering, deck UI, menus, shortcuts, palette, new-tab, D3 strip migration + design-system amendments), and documentation. Tasks 05-06 prepare and execute QA. Post-MVP (explicitly out of scope, PRD Non-Goals): hover peek, live sub-labels, ephemeral preview tabs, a global tab-vs-window preference, manual renaming, cross-workspace tab sharing, small-screen deck behavior, and app-subtree virtualization.

| #  | Title                                                                    | Status  | Complexity | Dependencies       |
| -- | ------------------------------------------------------------------------ | ------- | ---------- | ------------------ |
| 01 | Domain core v3: types, invariants, persistence, config, reducers, coalescer | completed   | critical   | -                  |
| 02 | Public surface: contract, codegen, hooks, native tools, CLI, parity      | completed | high       | task_01            |
| 03 | Frontend: semantic identity, frame rendering, deck UI, strip migration   | completed | critical   | task_02            |
| 04 | Docs: window-management skill + config reference                         | completed | low        | task_02            |
| 05 | QA Plan and Session Charters                                             | completed | high       | task_03, task_04   |
| 06 | Real-User QA Execution                                                   | completed | critical   | task_05            |
