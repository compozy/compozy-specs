---
status: completed
title: "QA Plan and Session Charters"
type: qa-report
complexity: high
---

# Task 7: QA Plan and Session Charters

## Overview

Plans the program's verification cycle on the repo's living QA tree: mints/updates journey flowcharts, content-addressed scenario files, and this cycle's session charters covering every public surface tasks 01–06 shipped. Consolidates the per-task QA-impact flags (each implementation task added `untested` scenarios) into walkable coverage.

<critical>ALWAYS READ `_spec.md`, every ADR, and every per-task memory file before planning.</critical>

<requirements>
1. Activate the `qa-report` skill with `qa-docs-path=docs/qa` (bootstrap the tree if absent).
2. Coverage MUST span every public surface from tasks 01–06 — CLI verbs (`session wait/interactions/prompt-cancel/list` flags, `notify`, `spawn --no-notify-creator`, `config set attention.*`/shortcuts), HTTP/UDS routes (presence, interactions, attention-summary, wait, notify, settings/attention, settings/window-manager), web surfaces (bell, toasts, title, Show all, sort, settings pages, cheatsheet, preset, palette views), the seven native tools, the hook event, and `[attention]`/shortcut config keys — expressed as scenario `entry_points` on journey-derived rows, not standalone test cases. Web journeys MUST name the locked board they judge against (`docs/design/opendesign/herdr-parity/` + the Visual Contract rows on tasks 03/05/06).
3. Map regression hot spots from `_spec.md` Part II Safety Invariants (1–16) and ADR-001..006 into the cycle's charter selection: targeted tier (attention truth, wait/wake concurrency, preset/keymap cut) + one adjacent canary journey (existing task-approval bell rows).
4. Dedup against existing `docs/qa/scenarios/` content-addressed ids; reset changed-behavior scenarios to `untested` rather than minting duplicates.
</requirements>

## Subtasks

- [x] 7.1 Inventory the QA-impact flags from tasks 01–06 and the shipped surfaces.
- [x] 7.2 Update `docs/qa/journeys/` flowcharts for the attention, orchestration, and keyboard journeys.
- [x] 7.3 Mint/update content-addressed scenario files in `docs/qa/scenarios/` (dedup-first).
- [x] 7.4 Write this cycle's charters in `docs/qa/charters/` (targeted tier + canary).

## Cycle Selection

This is a targeted cycle: every journey changed by tasks 01–06 has a focused charter, and the
pre-existing task-approval path is the adjacent canary because it shares the rewritten OS attention
row/count model. The cycle contains nine primary sessions and one canary session.

| Charter | Journey | ADRs | Safety invariants |
| --- | --- | --- | --- |
| `CH-herdr-attention-signals` | `J-respond-to-agent-attention` | 001, 002, 005 | 1, 3, 6, 7, 8, 11, 13, 14, 16 |
| `CH-herdr-attention-accessibility` | `J-respond-to-agent-attention` | 001, 002 | 1, 12, 16 |
| `CH-herdr-done-presence` | `J-11` | 001, 005 | 1, 3, 12, 14, 16 |
| `CH-herdr-interaction-recovery` | `J-answer-agent-requests` | 001, 005 | 2, 3, 9, 11, 14, 15 |
| `CH-herdr-session-orchestration` | `J-15` | 001, 002, 005 | 4, 5, 8, 9, 11, 14 |
| `CH-herdr-attention-settings` | `J-administer-runtime-settings` | 002 | — |
| `CH-herdr-keymap-hard-cut` | `J-administer-window-manager` | 004, 006 | 10 |
| `CH-herdr-keyboard-navigation` | `J-operate-desktop-shell` | 003, 004, 006 | 10 |
| `CH-herdr-attention-hook` | `J-agent-marketplace-parity` | 005 | 3, 11, 14 |
| `CH-herdr-task-approval-canary` | `J-operate-home-dashboard` | 002 | — |

The union covers Safety Invariants 1–16 and ADR-001..006. The 27 task-authored scenario flags stay
canonical. Dedup added only `RT-session-native-stop`, because no existing scenario owned the seventh
native tool; reset `ET-042` for the new hook contract; and reset `RT-home-approve-from-dashboard` for
the adjacent task-approval canary. The tracker now has 880 schema-valid scenario files.

Coverage taxonomy:

- Journeys: every changed public surface is reached through its owning flow and true end state.
- Functional: CLI, HTTP, UDS, native tools, hooks, Web, and config round trips compare fresh daemon data.
- Experiential: Cora and Sol cover plain language, keyboard, screen reader, reduced motion, focus, empty, stale, denied, and unavailable states.
- Edge/error/empty: Interrupt, Multi-Tab, Back-Button, and Feature tours cover races, restart, disconnect, zero results, overflow, mute, and rejected writes.
- Cross-cutting: workspace isolation, exact structured parity, locked-board judgment, and the task-approval canary are explicit; mobile is skipped because the changed host is the desktop OS shell.

## Implementation Details

Operates on the committed `docs/qa/` tree only (`scenarios/`, `journeys/`, `charters/`, `bugs/`, `reports/`); `state.csv` is generated output. Inputs: `_spec.md`, `_user_stories.md`, `_dx.md`, `_uiux.md`, `_tests.md`, `adrs/`, `docs/design/opendesign/herdr-parity/` (locked visual contract), and each task's `### Web/Docs Impact → QA impact` line.

### Relevant Files

- `docs/qa/` tree (scenarios/journeys/charters/bugs/reports) — the living contract.
- `.compozy/tasks/herdr-parity/task_01..06.md` — QA-impact flags to consolidate.
- `docs/design/opendesign/herdr-parity/` — seven boards + `DESIGN-NOTES.md`; charter web walks cite the matching Visual Contract rows.

### Related ADRs

- All six (`adrs/adr-001..006`) — charter hot-spot mapping.

## Deliverables

- Journeys, scenario files, and charters committed under `docs/qa/` covering every surface above.
- Charter selection documenting the targeted tier + canary rationale.

## Tests

QA planning task — no `_tests.md` IDs. Its output is the walk plan task_08 executes.

## Success Criteria

- Every task_01–06 QA-impact flag resolved into a scenario file (new or reset), zero dedup collisions.
- Charters name the safety-invariant hot spots and the canary journey.

## Completion Notes

- Reused all 27 implementation-task scenario ids, added only `RT-session-native-stop` for the
  otherwise-unowned seventh native tool, and reset the existing hook and task-approval canary rows.
- Planned 10 durable charters covering 30 unique scenarios: nine targeted primary sessions plus one
  adjacent canary. Task 08 owns all walks, visual bundles, E2E execution, verdicts, and the report.

Compozy Impact Audit:

- Native tools: no runtime change; checked and assigned all seven shipped IDs to scenario entry points and `CH-herdr-session-orchestration`.
- Extensibility and hooks: no runtime change; reset existing `ET-042` to cover the new async-only `session.attention.changed` discovery and real extension observation.
- Workspace data isolation: no runtime data change; charters explicitly cover operator-only aggregation, same-workspace native targets, agent denials, cross-workspace jumps, and cache/event boundaries.
- Official Compozy skill: no change; checked the task memories showing tasks 01–05 already updated the owning runtime, native-tool, configuration, orchestration, and window-management references.
