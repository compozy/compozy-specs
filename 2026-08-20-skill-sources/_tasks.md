---
schema_version: "compozy.tasks/v2"
workflow: skill-sources
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
    - from: task_03
      to: task_04
    - from: task_04
      to: task_06
    - from: task_05
      to: task_06
    - from: task_04
      to: task_07
    - from: task_05
      to: task_07
    - from: task_06
      to: task_08
    - from: task_07
      to: task_08
    - from: task_08
      to: task_09
---

# Skill Sources Task List

## MVP Boundary

Tasks 01-07 implement the complete skill-sources MVP defined in `_spec.md` (config keys + preset table, discovery on resolved root lists with live apply, `skill_exposures` + expose lifecycle, public surfaces and projections closure, injection suppression + command-catalog projection, web S1-S3, docs + official skill). Tasks 08-09 prepare and execute QA. Post-MVP (deliberately not tasked): additional presets (`codex`, `hermes`, `openclaw`, `cursor`, `opencode`), ACP advertised-command confirmation signal, custom sources as expose targets. Out of scope permanently: Part I Non-Goals (remote installer/sync, agent-definition discovery, extension-registered presets, content mirroring).

Parallelization: after task_02, the chain task_03 → task_04 runs in parallel with task_05; task_06 and task_07 run in parallel once their edges are met.

| #  | Title | Status | Complexity | Dependencies |
| -- | ----- | ------ | ---------- | ------------ |
| 01 | Source config foundation and policy write path | pending | high | - |
| 02 | Discovery engine and live apply with sources read model | pending | critical | task_01 |
| 03 | Exposure records: schema, store, and manager | pending | critical | task_02 |
| 04 | Expose surfaces and public projections closure | pending | high | task_03 |
| 05 | Session injection suppression and command catalog projection | pending | high | task_02 |
| 06 | Web sources settings, picker origin chips, and expose panel | pending | high | task_04, task_05 |
| 07 | Docs, official skill, and instructions updates | pending | medium | task_04, task_05 |
| 08 | QA Plan and Session Charters | pending | high | task_06, task_07 |
| 09 | Real-User QA Execution | pending | critical | task_08 |

Test contract: every ID in `_tests.md` (UT-001..098, IT-001..015, E2E-001..011) is assigned to exactly one task's `## Tests` section — no orphans, no duplicates.
