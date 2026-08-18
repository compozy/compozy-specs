---
schema_version: "compozy.tasks/v2"
workflow: hermes-comparison
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
    - from: task_02
      to: task_03
    - from: task_01
      to: task_07
    - from: task_04
      to: task_07
    - from: task_01
      to: task_12
    - from: task_02
      to: task_12
    - from: task_03
      to: task_12
    - from: task_04
      to: task_12
    - from: task_05
      to: task_12
    - from: task_06
      to: task_12
    - from: task_07
      to: task_12
    - from: task_08
      to: task_12
    - from: task_09
      to: task_12
    - from: task_10
      to: task_12
    - from: task_11
      to: task_12
    - from: task_12
      to: task_13
---

# Hermes-Referenced AGH Improvements — Task List

Program derived from the `hermes-comparison` research (9 slice analyses + `analysis/summary.md`),
governed by `_techspec.md` and ADRs `adrs/adr-001..010.md`. Excludes channels/bridges/
communications (dedicated research later) and everything already covered by the archived
task-blocks spec.

Resliced (2026-07-09) per `cy-create-tasks` sizing: 35 micro-tasks → 11 robust vertical slices +
QA pair. Prior task bodies live in
`.compozy/tasks/_archived/20260709-215833-hermes-comparison-tasks-v1/`.

## Suggested execution order (waves — advisory, not edges)

- **W1 Correctness:** task_01 (interaction & supervision), task_02 (context & replay).
- **W2 Session & memory:** task_03 (compaction, checkpoints, batch memory, lifecycle).
- **W3 Security & reliability:** task_04 (redaction + SECURITY.md + lifecycle block),
  task_05 (daemon reliability).
- **W4 Product layers:** task_06 (cost & usage), task_07 (suggestion store),
  task_08 (shadow-git checkpoints).
- **W5 Extensibility:** task_09 (registry/MCP/skills machinery), task_10 (AGH as MCP server).
- **W6 Orchestration kernel (O1–O5):** task_11 (lease/loop/claim/timeout/wake-cap) — independent
  of W1–W5; internal subtask order absorbs former 29↔32 and 31↔33 merge-time couplings.
- **W7 QA pair (SD-005):** task_12 (report) → task_13 (execution).

## Task summary

| Task | Title | Type | Complexity | Merges (v1) |
| --- | --- | --- | --- | --- |
| task_01 | Interaction & supervision: approvals, clarify, automation, health | backend | high | 01+02+05+07 |
| task_02 | Context & replay: pruner, transcript replay, tool-result offload | backend | high | 03+04+06 |
| task_03 | Session & memory: compaction, checkpoints, batch ops, lifecycle | backend | high | 08+09+10+11 |
| task_04 | Security boundary: redaction, SECURITY.md, daemon lifecycle block | backend | critical | 12+13+14 |
| task_05 | Daemon reliability: memory probe, draining, dead-entity, retry taxonomy | backend | high | 15+16+17+18 |
| task_06 | Cost & usage: estimator, provenance, account-usage (L-015 gate) | backend | medium | 19+20+21 |
| task_07 | Consent-first suggestion store + starter catalog | backend | high | 22 |
| task_08 | Shadow-git workspace checkpoints | backend | high | 23 |
| task_09 | Extensibility: registry cache, MCP catalog, parse snapshot, when.* | backend | high | 24+27+28+25 |
| task_10 | Serve AGH as an MCP server | backend | high | 26 |
| task_11 | Orchestration kernel O1–O5 | backend | critical | 29+30+31+32+33 |
| task_12 | QA report — scenario matrix and evidence contract | test | medium | 34 |
| task_13 | QA execution — run matrix and close release verdict | test | high | 35 |

## Notes

- True cross-group edges only: `02→03` (replay/pruner before session/memory — former 04→09→08),
  `01→07` + `04→07` (suggestion needs final Job model + lifecycle guard), all work → `12→13`.
- Former internal edges (03→04, 19→20→21, 24→27, 12→13 docs) are subtask order inside the merged
  slices.
- Task_05 carries a mandatory two-touch determination (former task_18) before any retry-surface
  code change.
- Task_06's account-usage half is verification-first: implementation conditional on the L-015
  reachability spike (techspec §6.1).
- Task_07 ships the plain-`Job` suggestion payload only; the loop-template variant is deferred to
  the Loops program as a hard cut (ADR-007; peer-review N-001).
- Task_11 absorbs former 31↔33 and 29↔32 coordination hazards into one critical slice — subtasks
  MUST land the shared guarded-claim helper and attempt-budget consumption in one coherent change.

## Type → execution model (compozy tasks run routing)

`type:` encodes the **execution domain**, which selects the model via
`[[tasks.run.task_runtime_rules]]` in `.compozy/config.toml` when running `compozy tasks run
hermes-comparison`:

| `type:` | Model (per current config.toml) |
| --- | --- |
| `backend` | `codex` / `gpt-5.5` / `xhigh` |
| `docs`, `test` | no rule → falls back to `[defaults]` (`codex` / `gpt-5.5` / `xhigh`) |
| `frontend` | `claude` / `opus` / `xhigh` |

This suite is backend-heavy: all runtime/autonomy/memory/cost work is `backend`; SECURITY.md is
authored inside task_04 (`backend` — same model fallback as former `docs` type); `12/13` are the
`test` QA tail. There are no frontend tasks. Domain drives the model, not change-kind — do **not**
reclassify these back to `bugfix`/`refactor`. Mixed web+Go tasks take the majority surface (all
mixed tasks here are Go-dominant → `backend`). Note: `[[tasks.run.task_runtime_rules]]` applies
only to `compozy tasks run`, not `compozy exec`.
