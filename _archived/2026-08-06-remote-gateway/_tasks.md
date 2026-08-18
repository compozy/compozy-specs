---
schema_version: "compozy.tasks/v2"
workflow: remote-gateway
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
  edges:
    - from: task_01
      to: task_02
    - from: task_02
      to: task_03
    - from: task_02
      to: task_05
    - from: task_02
      to: task_04
    - from: task_03
      to: task_04
    - from: task_04
      to: task_06
    - from: task_05
      to: task_06
    - from: task_02
      to: task_07
    - from: task_04
      to: task_07
    - from: task_06
      to: task_07
    - from: task_07
      to: task_08
    - from: task_08
      to: task_09
---

# Remote Gateway Task List

Decomposition of `_techspec.md` (revision 2, post peer-review round 1) into independently implementable slices. Dependency edges above are the execution order: an edge `from → to` means `from` must finish before `to` starts.

## MVP Boundary

Tasks 01–07 implement the complete MVP: exposure foundation, authenticated tier listeners, the connectivity provider seam with its bundled provider, public ingress, remote clients, agent-facing manageability, and the operator surface. Tasks 08–09 plan and execute QA. Nothing in this list is post-MVP; the PRD's Non-Goals (hosted relay, multi-user accounts, native mobile apps, cross-machine Network peering, webhook store-and-forward, additional bundled providers, proof-of-possession credentials) remain out of scope.

## Tasks

| #  | Title                                                     | Status  | Complexity | Dependencies     |
| -- | --------------------------------------------------------- | ------- | ---------- | ---------------- |
| 01 | Gateway foundation: config, schema, and exposure state machine | completed | critical   | -                |
| 02 | Tier listeners and device authentication                  | completed | critical   | task_01          |
| 03 | Connectivity provider seam and bundled Tailscale provider  | completed | high       | task_02          |
| 04 | Public ingress: webhook and bridge reachability            | completed | high       | task_02, task_03 |
| 05 | Remote CLI profiles, operation matrix, and SSH connect     | completed | high       | task_02          |
| 06 | Self-audit, native tool, and cross-surface assurance       | completed | medium     | task_04, task_05 |
| 07 | Gateway operator surface and documentation                 | completed | high       | task_02, task_04, task_06 |
| 08 | QA planning for remote gateway                             | completed | high       | task_07          |
| 09 | QA execution for remote gateway                            | completed   | critical   | task_08          |

## Test Contract Assignment

Every case in `_tests.md` (168 total: 109 UT, 52 IT, 7 E2E) is assigned to exactly one task.

| Task | UT | IT | E2E | Total |
| ---- | -- | -- | --- | ----- |
| 01   | 26 | 3  | 0   | 29    |
| 02   | 25 | 20 | 0   | 45    |
| 03   | 26 | 3  | 0   | 29    |
| 04   | 8  | 12 | 1   | 21    |
| 05   | 11 | 10 | 2   | 23    |
| 06   | 13 | 4  | 0   | 17    |
| 07   | 0  | 0  | 3   | 3     |
| 08   | 0  | 0  | 0   | 0     |
| 09   | 0  | 0  | 1   | 1     |

## Co-ship Discipline

Route registration, OpenAPI/TypeScript regeneration, event registration, config docs, and site reference updates ship **inside the task that creates the surface** — they are not deferred to a later task. Task 06 owns only the suite-level assertions that cannot be verified until every surface exists (complete event family registration, native tool catalog drift, cross-surface secret scanning).
