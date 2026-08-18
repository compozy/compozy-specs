---
schema_version: "compozy.tasks/v2"
workflow: graph-eng
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
    - from: task_07
      to: task_08
    - from: task_08
      to: task_09
    - from: task_09
      to: task_10
    - from: task_10
      to: task_11
---

# Loop Graph Completion (graph-eng) Task List

## MVP Boundary

Tasks 01–09 implement the entire MVP (spec phases P0–P10; nothing in scope is post-MVP). Tasks 10–11 are the trailing QA pair (planning + execution over the living `docs/qa/` tree). Out of scope is exactly `_spec.md` Part I Non-Goals. **Global precondition: the whole suite executes only after the herdr-parity program is merged to main** (ADR-006).

The backend chain (01→07) is strictly serialized by append-only Goose migration ordering (each migration-owning task needs the previous one's migration applied — L-008/L-021); web (08→09) follows the backend; the QA pair closes.

| #   | Title                                                                 | Status  | Complexity | Dependencies |
| --- | --------------------------------------------------------------------- | ------- | ---------- | ------------ |
| 01  | P0 cleanup: hash_fields deletion, predicate policy, per-gate counters | pending | medium     | -            |
| 02  | P1 router: route node + gate verdict routing                          | pending | high       | task_01      |
| 03  | P2 requests core: ask node, respond plane, ResponderPolicy            | pending | critical   | task_02      |
| 04  | P3 per-lane addressing, review gate, amend overlay                    | pending | critical   | task_03      |
| 05  | P4+P5 strategies, partial, progress namespace, iteration names        | pending | critical   | task_04      |
| 06  | P6 windowed fan-out width                                             | pending | high       | task_05      |
| 07  | P7+P8 time travel: diff, rerun, fork + cross-cutting suites           | pending | critical   | task_06      |
| 08  | P9 web run-page surfaces (S1–S8, S10–S11) + design pass               | pending | critical   | task_07      |
| 09  | P10 bell composition (S9, post-herdr seam)                            | pending | medium     | task_08      |
| 10  | qa-report: QA planning over docs/qa                                   | pending | high       | task_09      |
| 11  | qa-execution: scenario walks + browser e2e                            | pending | critical   | task_10      |
