---
schema_version: "compozy.tasks/v2"
workflow: profiles
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
    - from: task_04
      to: task_05
    - from: task_04
      to: task_06
    - from: task_05
      to: task_07
    - from: task_06
      to: task_07
    - from: task_06
      to: task_08
    - from: task_08
      to: task_09
    - from: task_07
      to: task_10
    - from: task_09
      to: task_10
    - from: task_10
      to: task_11
    - from: task_11
      to: task_12
    - from: task_12
      to: task_13
---

# Profiles Task List

Binding inputs: `_spec.md` (Part I + II), `_user_stories.md` (US-001..033), `_dx.md` (frozen surface), `_uiux.md` (S1–S13), `_tests.md` (test contract), `DECISIONS.md` (D1–D27 as amended), `adrs/adr-001..015.md`. Verified anchors: `analysis/11_arch-path-map.md` (wins on drift).

## MVP Boundary

Tasks 01–11 implement the MVP: the complete spec Build Order, phases 0–7 (foundation fixes, schema + stamps, profile domain + lifecycle protocol, lifecycle/selection surfaces, enforcement sweep, web surfaces, layers, extensions/presets, per-profile state, editorial docs). Tasks 12–13 prepare and execute QA. Post-MVP is exactly the recorded future scope in `_spec.md` Non-Goals: work-transfer verb, inbound routing table, per-profile provider homes, capability gating (hiding worktrees), per-profile scheduler fairness, remote profile management. Out of scope: everything else Non-Goals names.

## Tasks

| # | Title | Status | Complexity | Dependencies |
| --- | --- | --- | --- | --- |
| 01 | Phase 0 — Foundation Fixes (Global as Concept + Server-Side Catalog Filter) | pending | critical | - |
| 02 | Profiles Schema and Stamps (Migration 00079 + Memory 00002) | pending | critical | task_01 |
| 03 | Profile Domain and Lifecycle Operation Protocol | pending | critical | task_02 |
| 04 | Lifecycle and Selection Surfaces (CLI + API + Docs) | pending | high | task_03 |
| 05 | Web Switcher, Settings Page, and SymbolPicker | pending | high | task_04 |
| 06 | Read-Scope Enforcement Sweep and Aggregate Reads | pending | critical | task_04 |
| 07 | Web Aggregate UI (Destination Chip, Owner Labels, Usage, Globe) | pending | medium | task_05, task_06 |
| 08 | Resource, Config, and Credential Layers | pending | high | task_06 |
| 09 | Extensions and Notification Presets per Profile | pending | high | task_08 |
| 10 | Per-Profile State and Polish (Desktops + A11y) | pending | medium | task_07, task_09 |
| 11 | Editorial Docs Synthesis | pending | low | task_10 |
| 12 | QA Plan and Session Charters | pending | high | task_11 |
| 13 | Real-User QA Execution | pending | critical | task_12 |

## Test contract totals

195 IDs from `_tests.md` assigned exactly once: 86 UT + 83 IT + 26 E2E (IT-065 withdrawn; IT-039/UT-063..065/UT-067..068/UT-083..084 do not exist). Per-task counts: 01→3 · 02→18 · 03→32 · 04→25 · 05→4 · 06→34 · 07→4 · 08→42 · 09→29 · 10→4 · 11→0.

## Cumulative fixture contract

Two integration suites are born early and extended as later capabilities land their rows (the round-8 peer-review deferred note on Safety Invariant 7):

- **IT-038** (total removal catalog) is owned by task_04. Later tasks extend its fixture without re-owning the ID: task_08 adds config-layer file, MCP sidecar entries, and credential-override rows; task_09 adds `profile_credential_requirements` rows; task_10 adds clientstate desktop partitions. The suite's closing assertion — preview == applied, field for field, nothing profile-owned survives — must stay green after every extension.
- **IT-077** (availability gate) is owned by task_04 (selection, creation trigger, ops list/retry, boot order). Task_08 extends its fixture with the resource-discovery and provider-prestart rejection arms.

## Execution notes

- Parallel windows: after task_04 → 05 ∥ 06; after 06 → 07 (needs 05 too) ∥ 08; 09 follows 08 while 07 closes. Critical chain: 01→02→03→04→06→08→09→10.
- Tasks 05, 07, and 09 carry Visual Contracts against artboards under `docs/design/opendesign/profiles/` that the design pass produces before execution. If the named artboard is absent at execution time, the task stops (blocked-precondition) — it is never downgraded to prose guidance.
- Intermediate tasks flag QA scenarios only (`docs/qa/scenarios/`); the walk runs once, in the QA tail (task_13), per the qa-execution contract.
