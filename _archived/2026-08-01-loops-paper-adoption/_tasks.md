---
schema_version: "compozy.tasks/v2"
workflow: loops-paper-adoption
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
    - from: task_03
      to: task_05
    - from: task_04
      to: task_06
    - from: task_05
      to: task_06
    - from: task_06
      to: task_07
---

# Loops Paper Adoption Task List

## MVP Boundary

Tasks 01-05 implement the full TechSpec scope — cross-generation repair context, re-attempt
semantics fix, metric-gated ratchet, lineage-lite provenance, surface co-ship, web, and docs.
Tasks 06-07 prepare and execute QA on the living `docs/qa/` tree. Post-MVP (explicitly out of
scope): multi-metric ratchets, lineage Level A/B (ADR-004), memory graph plane
(`.compozy/tasks/_toplan/memory-graph-plane/_briefing.md`), and the loop eval harness.

## Tasks

| #  | Title                                                    | Status  | Complexity | Dependencies     |
| -- | -------------------------------------------------------- | ------- | ---------- | ---------------- |
| 01 | Persistence, grammar and gate-plane foundation           | completed | high       | -                |
| 02 | Succession semantics, namespace history and fenced state | completed | critical   | task_01          |
| 03 | Public surfaces, codegen co-ship and runtime E2E         | completed | high       | task_02          |
| 04 | Web loop system: scores, best and provenance             | completed | medium     | task_03          |
| 05 | Site docs, generated references and official skill       | completed | low        | task_03          |
| 06 | QA Plan and Session Charters                              | completed | high       | task_04, task_05 |
| 07 | Real-User QA Execution                                    | completed | critical   | task_06          |

Test contract: all 73 IDs from `_tests.md` are assigned exactly once — task_01 (25 UT + 5 IT),
task_02 (12 UT + 11 IT), task_03 (4 UT + 8 IT + 6 E2E), task_04 (1 UT + 1 E2E); tasks 05-07 gate
on build/link/codegen checks and the QA contract rather than test IDs.
