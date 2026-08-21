---
schema_version: "compozy.tasks/v2"
workflow: loop-task-legibility
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
    - from: task_02
      to: task_04
    - from: task_03
      to: task_05
    - from: task_04
      to: task_06
    - from: task_05
      to: task_06
    - from: task_06
      to: task_07
---

# Loop & Task Legibility Task List

## MVP Boundary

Tasks 01-05 implement the full MVP — all four fronts of `_spec.md`: (1) terminal settlement invariant + reconciliation sweep, (2) catalog loop classification with 4-surface parity, (3) loop run read layer (roster/briefing/timeline) + CLI verbs, (4) web two-register presentation (Tasks calm default + run-page redesign incl. live DAG). Tasks 06-07 prepare and execute QA. Post-MVP (explicitly deferred, not in this graph): cross-run analytics, dashboard/inbox redesign, run-health-as-indexed-attribute rollups, timeline `GROUP BY` aggregations. Out of scope permanently: separate loop executor, DSL/editor changes, retry-semantics changes.

Execution notes: tasks 04 and 05 are parallel waves (disjoint systems `web/src/systems/tasks` vs `web/src/systems/loops`). The six visual-contract artboards are landed at `docs/design/opendesign/loop-legibility/` (`loop-legibility-tasks-list.html`, `loop-legibility-run-default.html`, `loop-legibility-needs-you.html`, `loop-legibility-run-dag.html`, `loop-legibility-run-roster.html`, `loop-legibility-runs-roster.html`) with companions `DESIGN-NOTES.md`, `loop-legibility.css`, and `index.html` — task bodies cite the full paths as binding visual contracts. A cited board missing at execution time blocks; it is never improvised.

| #  | Title                                                              | Status  | Complexity | Dependencies     |
| -- | ------------------------------------------------------------------ | ------- | ---------- | ---------------- |
| 01 | Terminal settlement invariant, reconciliation sweep & loops config | pending | critical   | -                |
| 02 | Catalog loop classification with 4-surface parity                  | pending | high       | task_01          |
| 03 | Loop run read layer: roster, briefing, timeline & CLI verbs        | pending | high       | task_02          |
| 04 | Web Tasks calm default: exclusion, reveal filter & provenance      | pending | medium     | task_02          |
| 05 | Web loop run two-register redesign: DAG, roster & runs re-rank     | pending | high       | task_03          |
| 06 | QA Plan and Session Charters                                       | pending | high       | task_04, task_05 |
| 07 | Real-User QA Execution                                             | pending | critical   | task_06          |
