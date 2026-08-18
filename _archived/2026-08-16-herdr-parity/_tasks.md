---
schema_version: "compozy.tasks/v2"
workflow: herdr-parity
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
    - from: task_01
      to: task_03
    - from: task_01
      to: task_04
    - from: task_01
      to: task_06
    - from: task_02
      to: task_03
    - from: task_02
      to: task_04
    - from: task_03
      to: task_06
    - from: task_05
      to: task_06
    - from: task_04
      to: task_07
    - from: task_06
      to: task_07
    - from: task_07
      to: task_08
---

# Herdr Parity Task List

Program: session attention · orchestration DX · shortcuts v2 (+ palette views). Spec set: `_spec.md` (two peer-review rounds incorporated), `_user_stories.md` (US-001..031), `_dx.md` + `_uiux.md` (frozen surfaces), `_tests.md` (UT-001..086, IT-001..033, E2E-001..020 — every ID assigned to exactly one task), `adrs/adr-001..006`. Visual contract (locked): `docs/design/opendesign/herdr-parity/` — seven boards + `DESIGN-NOTES.md`; tasks 03/05/06 carry the Visual Contract rows.

## MVP Boundary

Tasks 01–03 implement the MVP (the session-attention seed: daemon truth core, notification channel, web attention surfaces). Tasks 04–06 complete the program (orchestration DX, shortcuts v2, palette views). Tasks 07–08 prepare and execute QA. Task 05 has no dependencies and may run in parallel with 01–04.

## Tasks

| #  | Title | Status | Complexity | Dependencies |
| -- | ----- | ------ | ---------- | ------------ |
| 01 | Attention truth core: canonical records, badges, presence leases, discovery + CLI | completed | critical | - |
| 02 | Attention config, settings section, and the notify service + transport | completed | high | task_01 |
| 03 | Web attention surfaces: unified tones, bell sections, title count, Show all, notifier | completed | high | task_01, task_02 |
| 04 | Orchestration DX: generalized wait, session wake bridge, seven native tools | completed | critical | task_01, task_02 |
| 05 | Shortcuts v2: grammar, daemon-owned keymap, new actions, migrations, preset | completed | high | - |
| 06 | Command palette nested views + Sessions view | completed | medium | task_01, task_03, task_05 |
| 07 | QA Plan and Session Charters | completed | high | task_04, task_06 |
| 08 | Real-User QA Execution | completed | critical | task_07 |
