---
schema_version: "compozy.tasks/v2"
workflow: bundles-removal
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
    - from: task_03
      to: task_04
    - from: task_04
      to: task_05
    - from: task_05
      to: task_06
---

# Bundles Removal (bundles-removal) Task List

Requirements source: `_brief.md` (requirement keys R-HC-1..7, R-P0-1..5, R-P1-1..5, R-AM-1..4 — no PRD exists for this program). Design: `_techspec.md` (peer-review round 1 fully incorporated) + `adrs/adr-001..008.md`. Test contract: `_tests.md` (UT-001..070, IT-001..020, E2E-001..006 — every ID assigned to exactly one task below).

## MVP Boundary

Tasks 01–04 implement the full program: task_01 delivers Extension robustness (TechSpec phases A–C — kit completeness on enable, MCP-shaped secrets binding, lifecycle coordinator + network consent + inventory/preview); task_02 executes the hard cut of the Bundle product (phase D); task_03 lands the web surfaces (phase E); task_04 ships docs, the official skill, instruction checklists, and QA scenario flags (phase F). Tasks 05–06 are the trailing QA pair (planning + execution over the living `docs/qa/` tree). Post-MVP, out of this program: workspace-scoped enable, any pack/compose or multi-profile product, per-profile selective resource scoping, Live channel enrollment/memory seeding/task-graph starters, extension-projected declared channels, webhook-event package triggers, bridge SDK expansion, and native tools for secret writes.

## Tasks

| # | Title | Status | Complexity | Dependencies |
| --- | --- | --- | --- | --- |
| task_01 | Extension robustness: kit on enable, secrets binding, lifecycle consent & operability (Phases A–C) | pending | critical | - |
| task_02 | Hard cut: delete the Bundle product across every surface (Phase D) | pending | high | task_01 |
| task_03 | Web: kind collapse, bundle strips, kit inventory panel, confirm affordance (Phase E) | pending | medium | task_02 |
| task_04 | Docs, official skill, instruction checklists, QA scenario flags (Phase F) | pending | medium | task_03 |
| task_05 | QA Plan and Session Charters | pending | high | task_04 |
| task_06 | Real-User QA Execution | pending | critical | task_05 |
