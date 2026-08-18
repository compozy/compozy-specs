---
schema_version: "compozy.tasks/v2"
workflow: loop-node-lifecycle
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
    - id: task_13
      file: task_13.md
  edges:
    - from: task_01
      to: task_02
    - from: task_02
      to: task_03
    - from: task_03
      to: task_04
    - from: task_03
      to: task_05
    - from: task_05
      to: task_06
    - from: task_04
      to: task_07
    - from: task_06
      to: task_07
    - from: task_07
      to: task_08
    - from: task_07
      to: task_09
    - from: task_07
      to: task_10
    - from: task_07
      to: task_11
    - from: task_08
      to: task_10
    - from: task_08
      to: task_12
    - from: task_09
      to: task_12
    - from: task_10
      to: task_12
    - from: task_11
      to: task_12
    - from: task_12
      to: task_13
---

# Loop Node Lifecycle & Failure Contract Task List

## MVP Boundary

Tasks 01–11 implement the full node lifecycle & failure contract MVP (schema + classification +
grammar, precedence core, effects, target health, liveness/cancel, parked states, surfaces, web
run UI, web editor lifecycle authoring, web hero path, docs). Tasks 12–13 plan and execute QA
over the living `docs/qa/` tree. Post-MVP remains per the TechSpec MVP Boundary: idea 11 (cut),
idea 12 (held), Spec 2/Spec 3 domains (including **start-binding authoring** — editor writes
`start[]`, production sidebar gains the Start lane, docs drop read-only strip language), the
agent-drivable DAG RFC, the node-family state-machine refactor, `on_resume`, and state-writing
effects.

| #  | Title                                                                   | Status  | Complexity | Dependencies              |
| -- | ----------------------------------------------------------------------- | ------- | ---------- | ------------------------- |
| 01 | Schema, classification, DSL grammar and config contracts                | completed | high       | -                         |
| 02 | Precedence core: retry, scheduling, routes and escalation               | completed | critical   | task_01                   |
| 03 | Effects: outbox, relay and delivery identity                            | completed | high       | task_02                   |
| 04 | Target health breaker, quarantine and requeue                           | completed | high       | task_03                   |
| 05 | Liveness, death-resume, cancel ≠ kill and canceled terminal             | pending | critical   | task_03                   |
| 06 | Pause, auto-pause, durable waits and admission dedupe                   | pending | high       | task_05                   |
| 07 | Public surfaces co-ship: contract, routes, CLI, native tools            | pending | high       | task_04, task_06          |
| 08 | Web loops system: lifecycle states, controls and inventories            | pending | medium     | task_07                   |
| 09 | Web loop editor: lifecycle grammar authoring and chrome states          | pending | high       | task_07                   |
| 10 | Web hero path: catalog, run form, and loop detail Visual Contract       | pending | medium     | task_07, task_08          |
| 11 | Site docs, config reference and official skill                          | pending | low        | task_07                   |
| 12 | QA Plan and Session Charters                                            | pending | high       | task_08, task_09, task_10, task_11 |
| 13 | Real-User QA Execution                                                  | pending | critical   | task_12                   |
