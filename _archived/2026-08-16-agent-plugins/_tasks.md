---
schema_version: "compozy.tasks/v2"
workflow: agent-plugins
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
    - from: task_02
      to: task_03
    - from: task_02
      to: task_04
    - from: task_03
      to: task_04
    - from: task_04
      to: task_05
    - from: task_04
      to: task_06
    - from: task_05
      to: task_06
    - from: task_06
      to: task_07
    - from: task_07
      to: task_08
    - from: task_08
      to: task_09
---

# Agent Plugins Ingestion Task List

## MVP Boundary

Tasks 01–06 implement the MVP: the conformance foundations (`mcppolicy` + `agentplugin`), the ingestion core (detection, synthesis, persistence, lifecycle coordinator), the MCP wire + secrets vertical, the operator/agent surfaces, the web + catalog surfaces, and the docs/official-skill/QA-flag close. Tasks 07–08 are the QA pair (plan, then real-user execution with e2e). Task 09 closes the Phase 2 compatible-clients handoff after the evidence narrowed the supported provider-delivery claim to Claude Code and Hermes; the user owns the external PR after the docs deploy. Out of scope (spec Non-Goals): Phase 3 export/dual-target, Claude Code plugin adapter, OpenClaw gateway-side MCP bridge, `sse` transport support, dedicated marketplace section, exposing MCP header fields on operator config surfaces.

| # | Title | Status | Complexity | Dependencies |
|---|-------|--------|------------|--------------|
| 01 | Conformance foundations: mcppolicy + agentplugin | completed | high | - |
| 02 | Ingestion core: detection, synthesis, persistence, lifecycle | completed | critical | task_01 |
| 03 | MCP wire + secrets: headers, executor, runtime health, remote-header binding | completed | high | task_01, task_02 |
| 04 | Operator/agent surfaces: CLI output, contracts, error codes, parity | completed | high | task_02, task_03 |
| 05 | Web + catalog: format badges, skipped components, feed hard cut | completed | medium | task_04 |
| 06 | Docs, official skill, and QA scenario flags | completed | low | task_04, task_05 |
| 07 | QA Plan and Session Charters | completed | high | task_06 |
| 08 | Real-User QA Execution | completed | critical | task_07 |
| 09 | Phase 2: compatible-clients listing handoff | completed | low | task_08 |
