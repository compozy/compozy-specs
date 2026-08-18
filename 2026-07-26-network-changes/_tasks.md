---
schema_version: "compozy.tasks/v2"
workflow: network-changes
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
---

# Opt-In Agent Network Participation Task List

> Source: `_techspec.md` (approved + peer-reviewed) · `_prd.md` · `_tests.md` · ADR-001..011. Program slug: `network-changes`.

## Task Sizing Directive

Tasks are deliberately **large and cohesive**: each backend task closes a whole Build Order phase (or forced merge of phases) end-to-end because execution runs on long-running, high-context models (`.compozy/config.toml` per-type routing: backend → codex; frontend/docs → claude opus). Dependency edges above are the only sequencing contract — the graph is linear because ADR-004 one-complete-release and destructive schema/API hard-cuts forbid public hybrid windows.

## MVP Boundary

MVP boundary: tasks 01–06 implement the entire single complete cut (ADR-004) — participation contract + schema, owner wiring, autonomy decoupling + wake substrate, delivery/live admission/projection, public OpenAPI/CLI/web co-ship, and docs/skill/delete-target sweep. Tasks 07–08 prepare and execute QA on the living `docs/qa/` tree. Post-MVP (explicit non-goals, PRD + ADR-004/005/007): mailbox mode, offline address layer, configurable spend caps, cross-installation federation, embedded broker restoration.

## Tasks

| # | Title | Status | Complexity | Dependencies |
|---|---|---|---|---|
| 01 | Contract, resolver, schema hard-cut & config lifecycle | pending | high | - |
| 02 | Owner wiring — resolve-and-persist across session/task/loop/automation | pending | high | task_01 |
| 03 | Autonomy decoupling & network_wake substrate | pending | critical | task_02 |
| 04 | Delivery restructure, live admission & projection gating | pending | critical | task_03 |
| 05 | Public surfaces co-ship — OpenAPI/CLI/web | pending | high | task_04 |
| 06 | Docs, official skill, glossary & delete-target sweep | pending | medium | task_05 |
| 07 | QA Plan and Session Charters | pending | high | task_06 |
| 08 | Real-User QA Execution | pending | critical | task_07 |

Parallelism: none for implementation — the hard-cut forces a linear chain. Worktree + unique `AGH_HOME`/ports/tmux isolation is mandatory for any concurrent QA (see task_07/08 bodies).
