---
schema_version: "compozy.tasks/v2"
workflow: ext-improvs
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
    - from: task_05
      to: task_07
    - from: task_05
      to: task_08
    - from: task_06
      to: task_09
    - from: task_07
      to: task_09
    - from: task_08
      to: task_09
    - from: task_09
      to: task_10
    - from: task_10
      to: task_11
---

# Extension DX Overhaul (ext-improvs) Task List

Requirements source: `_brief.md` (R1–R11; no PRD exists for this program). Design: `_techspec.md` (three peer-review rounds incorporated) + `adrs/adr-001..008.md`. Test contract: `_tests.md` (UT-001..086, IT-001..021, E2E-001..008 — every ID assigned to exactly one task below).

## MVP Boundary

Tasks 01–09 implement the full MVP (TechSpec phases A–G): generated SDK contracts and publishing groundwork, manifest v2 + permissions + config consolidation, the code-first toolchain, the dev lane, open distribution, the contributed-commands surface (fixture-proven — no product command), CLI UX/operability, web surfaces, and the docs/skill/examples program. Tasks 10–11 are the trailing QA pair (planning + execution over the living `docs/qa/` tree). Post-MVP, out of this program: the external bridge SDK program (ADR-006 follow-up), the three remaining AGH-105 Future surfaces, the workflow-archive program (`.compozy/tasks/cmd-archive/_brief.md` — task-domain closure primitive + dev-cycle `archive` command), executable command groups / group-level persistent flags / command depth >2 / top-level fallback dispatch, npm-`create` wrapper, self-hostable registries, protocol-version negotiation, and skills/ClawHub pipeline convergence.

## Tasks

| # | Title | Status | Complexity | Dependencies |
| --- | --- | --- | --- | --- |
| task_01 | Generated SDK contracts + publishing groundwork (Phase A) | pending | high | - |
| task_02 | Manifest v2, permissions, config consolidation + hook source (Phase B) | pending | high | task_01 |
| task_03 | Code-first toolchain: describe mode, build/validate/init (Phase C) | pending | high | task_02 |
| task_04 | Dev lane: links, instance-keyed reload, logs, watch (Phase D) | pending | critical | task_03 |
| task_05 | Distribution: source union, gitsrc, sidecars, publish, search, update (Phase E) | completed | high | task_04 |
| task_06 | Contributed commands surface (ADR-008, fixture-proven) | pending | high | task_05 |
| task_07 | CLI UX & operability: diagnostics, status, doctor, events matrix (Phase F) | pending | medium | task_05 |
| task_08 | Web surfaces: update affordance, install forms, dev badge, logs panel (Phase F) | pending | medium | task_05 |
| task_09 | Docs, official skill, examples, QA scenario flags (Phase G) | pending | medium | task_06, task_07, task_08 |
| task_10 | QA Plan and Session Charters | pending | high | task_09 |
| task_11 | Real-User QA Execution | pending | critical | task_10 |
