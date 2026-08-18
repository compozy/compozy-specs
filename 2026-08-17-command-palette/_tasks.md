---
schema_version: "compozy.tasks/v2"
workflow: command-palette
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
  edges:
    - from: task_01
      to: task_02
    - from: task_02
      to: task_03
    - from: task_02
      to: task_04
    - from: task_03
      to: task_05
    - from: task_04
      to: task_06
    - from: task_05
      to: task_07
    - from: task_06
      to: task_07
    - from: task_07
      to: task_08
    - from: task_04
      to: task_09
    - from: task_05
      to: task_09
    - from: task_08
      to: task_10
    - from: task_09
      to: task_10
    - from: task_10
      to: task_11
    - from: task_11
      to: task_12
---

# Command Palette Task List

Spec: [`_spec.md`](_spec.md) · Stories: [`_user_stories.md`](_user_stories.md) · Surface: [`_dx.md`](_dx.md) + [`_uiux.md`](_uiux.md) · Test contract: [`_tests.md`](_tests.md) · ADRs: [`adrs/`](adrs/)

## Artboard Prerequisite (P0 — operator-produced, outside the graph)

The ten visual references below are produced by the operator **before execution** at `docs/design/opendesign/command-palette/`. Every UI-bearing task cites its artboards in `## Visual Contract`; a task whose artboard is missing at execution time **blocks and surfaces the gap** — it never improvises the reference.

`command-palette-root.html` · `command-palette-root-states.html` · `command-palette-view-shell.html` · `command-palette-view-bands.html` · `command-palette-domain-list.html` · `command-palette-form-grid.html` · `command-palette-action-panel.html` · `command-palette-args-confirmation.html` · `command-palette-settings.html` · `command-palette-settings-palette.html`

## MVP Boundary

Tasks 01–05 implement the MVP (Build Order P1–P4): the daemon-canonical registry + async approval substrate with its full agent surface, the web absorption/projection, personalization + entity/settings search, execution UX, and the keyboard slice. Tasks 06–10 are post-MVP inside this spec (P5–P8: view system, extension contribution, view programs, desktop global hotkeys, NL fallback + close). Tasks 11–12 prepare and execute QA. Out of scope: the Part I Non-Goals (composer `/` menu, deeplinks, background commands, chord sequences, Go view programs, embedded JS engine).

## Tasks

| #  | Title                                                                     | Status  | Complexity | Dependencies     |
| -- | ------------------------------------------------------------------------- | ------- | ---------- | ---------------- |
| 01 | P1 Daemon — Unified Registry, Approval Substrate, Agent Surface           | pending | critical   | -                |
| 02 | P1 Web — Registry Projection, Dispatch Seam, Core Absorption              | pending | high       | task_01          |
| 03 | P2 — Personalization, Single TS Scorer, Entity + Settings Search          | pending | high       | task_02          |
| 04 | P3 — Execution UX: Action Panel, Inline Args, Confirmation, Feedback      | pending | medium     | task_02          |
| 05 | P4 — Keyboard: Open-ID Keymap, Aliases, Config, Settings Surfaces         | pending | high       | task_03          |
| 06 | P5 — View System: Vocabulary, Patch Engine, Four Kinds, Domain Views      | pending | high       | task_04          |
| 07 | P6 — Extension Contribution: Manifest Family, Projection, Declarative Tier | pending | high       | task_05, task_06 |
| 08 | P6b — View Programs: view.provider Runtime, TS SDK, React Kit             | pending | critical   | task_07          |
| 09 | P7 — Desktop Global Hotkeys: Preload Bridge, globalShortcut, Reconciler   | pending | high       | task_04, task_05 |
| 10 | P8 — NL Fallback, Event Matrix, Integration Close                         | pending | medium     | task_08, task_09 |
| 11 | QA Plan and Session Charters                                              | pending | high       | task_10          |
| 12 | Real-User QA Execution                                                    | pending | critical   | task_11          |

## Test Contract Notes

- Every `UT-`/`IT-`/`E2E-` ID from `_tests.md` is assigned to exactly one task's `## Tests` section: 136 UT + 31 IT + 33 E2E.
- **IT-002 and IT-004 are withdrawn** in `_tests.md` (web owns entity search and view assembly); their invariants live in UT-110/UT-112 + E2E-003 (task_03) and UT-133/UT-134 + E2E-009 (task_06). They are intentionally unassigned.
- IT-020 (migrations `00069` + `00070` + approval races) lives in task_03 — the task that completes both migrations' coverage; `00069` itself lands in task_01.
- E2E-016 and E2E-025 live in task_07: their frozen transcripts exercise the `ext.notes` fixture extension, which first exists in P6. The P4 slice (task_05) closes on its settings/keymap suites.
- IT-033 (canonical event matrix) lives in task_10 — it asserts event families contributed across all slices.
