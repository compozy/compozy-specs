---
schema_version: "compozy.tasks/v2"
workflow: agent-comms
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
    - from: task_01
      to: task_03
    - from: task_03
      to: task_04
    - from: task_02
      to: task_05
    - from: task_04
      to: task_05
    - from: task_05
      to: task_06
    - from: task_05
      to: task_07
    - from: task_06
      to: task_08
    - from: task_07
      to: task_08
    - from: task_08
      to: task_09
---

# Agent Comms Task List

## MVP Boundary

Tasks 01-07 implement the complete MVP of `_spec.md` — the typed call primitive, the unified contract regime across all five legacy pipelines, the lineage mailbox, park/revive lifecycle, subagents on the AGENT.md registry, the full surface chain (native tools, CLI, HTTP/UDS, hooks, Host API, Network publish bridge, `[calls]` config), the Agents-app web extension, and the docs/skill area. Tasks 08-09 prepare and execute QA. Post-MVP (recorded Non-Goals / deferred evolutions — not in any task): admission ceilings for call wakes (ADR-011), AGENT.md default output schemas, typed mailbox payloads, cross-workspace/cross-host reach, broadcast/groups, auto-routing, a unified loops+calls activity tree (ADR-012), and Network-content promotion.

## Tasks

| # | Title | Status | Complexity | Dependencies |
| --- | --- | --- | --- | --- |
| 01 | Foundations: `internal/contracts`, `[calls]` config, store layering | pending | high | - |
| 02 | Legacy pipeline adoption: one contract regime | pending | high | task_01 |
| 03 | Calls domain core: schema, admission/activation, settlement | pending | critical | task_01 |
| 04 | Mailbox, delivery, and lifecycle: park/revive, loop-breakers | pending | high | task_03 |
| 05 | Surfaces: native tools, CLI, HTTP/UDS, roster, hooks, publish | pending | high | task_02, task_04 |
| 06 | Web: agent-comms system in the Agents app | pending | high | task_05 |
| 07 | Docs and official skill: agent-comms area | pending | low | task_05 |
| 08 | QA Plan and Session Charters | pending | high | task_06, task_07 |
| 09 | Real-User QA Execution | pending | critical | task_08 |

## Notes

- **Test contract accounting**: `_tests.md` defines 162 assignable unit cases (UT-001..UT-163 with UT-153 formally withdrawn), 72 integration cases (IT-001..IT-072), and 30 E2E cases (E2E-001..E2E-030). Every ID is assigned to exactly one task's `## Tests` section; UT-153 is assigned nowhere by design.
- **Execution waves**: task_01 → {task_02 ∥ task_03} → task_04 → task_05 → {task_06 ∥ task_07} → task_08 → task_09. task_02 and task_03 touch disjoint files (legacy pipelines vs the new calls domain) and may run in parallel worktrees; both overlap only inside `internal/task` on different files.
- **Visual prerequisite**: the six `agent-comms-*.html` artboards named in `_uiux.md` do not exist yet under `docs/design/opendesign/agent-comms/`. The operator's OpenDesign pass must land them before task_06 executes (blocking prerequisite recorded in task_06).
- **Schema authorship**: migration `00078` (task_03) carries every schema change of this feature in one migration — fragment 73, the `72_task_runs.sql` CHECK rebuild + run snapshot columns, `sessions` park/drain columns, and `tasks.expect_digest`. No other task writes migrations.
