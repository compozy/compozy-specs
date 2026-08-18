---
schema_version: "compozy.tasks/v2"
workflow: hermes-bridge
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
  edges:
    - from: task_01
      to: task_03
    - from: task_01
      to: task_05
    - from: task_01
      to: task_06
    - from: task_04
      to: task_05
    - from: task_02
      to: task_07
    - from: task_03
      to: task_07
    - from: task_04
      to: task_07
    - from: task_06
      to: task_07
    - from: task_01
      to: task_09
    - from: task_02
      to: task_09
    - from: task_03
      to: task_09
    - from: task_04
      to: task_09
    - from: task_05
      to: task_09
    - from: task_06
      to: task_09
    - from: task_07
      to: task_09
    - from: task_08
      to: task_09
    - from: task_09
      to: task_10
---

# Bridge Usability Parity (Hermes Research) Task List

Research-driven task suite from `analysis/summary.md` + `_techspec.md`. Two P0 goals: live
tool-call progress in channels and one-paste/wizard setup; plus delivery robustness, contract
completeness, docs parity, and structural hygiene.

Resliced (2026-07-09) per `cy-create-tasks` sizing: 18 micro-tasks → 8 robust vertical slices +
QA pair. Prior task bodies live in
`.compozy/tasks/_archived/20260709-215833-hermes-bridge-tasks-v1/`.

## Suggested execution order (waves — advisory, not edges)

- **W1 — Progress + text core:** task_01 (progress contract + labels + accumulator),
  task_02 (chunking + markdown dialects).
- **W2 — Provider rendering + setup:** task_03 (6-provider progress render), task_04
  (CLI setup: manifest/wizards/verify), task_05 (web orchestrator).
- **W3 — Robustness + hygiene:** task_06 (inbound edits + durable ledger), task_07
  (bridgesdk hoist + conformance), task_08 (docs parity).
- **QA tail (SD-005):** task_09 (qa-report) → task_10 (qa-execution).

## Task summary

| Task | Title | Type | Complexity | Merges (v1) |
| --- | --- | --- | --- | --- |
| task_01 | Progress core: event family, labels/redaction, ProgressAccumulator | backend | critical | 02+03+04 |
| task_02 | Outbound text pipeline: shared chunking + markdown dialects | backend | high | 01+07 |
| task_03 | Progress rendering across six channel providers | backend | high | 05+06 |
| task_04 | Setup CLI: Slack manifest, wizards, verify/doctor/send-test | backend | high | 08+09+10 |
| task_05 | Web setup orchestrator (checklist/manifest/verify) | frontend | medium | 11 |
| task_06 | Delivery robustness: inbound edits + durable ledger | backend | high | 12+13 |
| task_07 | bridgesdk scaffolding hoist + conformance/drift suite | refactor | critical | 16+14 |
| task_08 | Docs parity (8/8) + truthfulness fixes | docs | medium | 15 |
| task_09 | QA cycle planning (qa-report) | test | medium | 17 |
| task_10 | QA cycle execution (qa-execution) | test | medium | 18 |

## Scope notes

- Deferred (techspec §1 non-goals): outbound media upload, GChat per-user OAuth, pairing-code
  issuer. Rejected: Telegram QR broker onboarding, Discord voice, startup auto-manifest.
- Open Decisions live in `_techspec.md §6` — notably the per-platform progress defaults
  (recommendation: `new`+`accumulate` on Slack/Telegram/Discord).
