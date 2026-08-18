---
status: completed
title: QA Plan and Session Charters
type: qa-report
complexity: high
---

# Task 5: QA Plan and Session Charters

## Overview

Plans the window-tabs QA cycle on the living `docs/qa/` tree: resets every scenario invalidated by the feature, mints the new tab scenarios as content-addressed files, updates journey flowcharts, and writes the cycle's session charters. Planning only — execution and verdicts belong to task_06.

<critical>ALWAYS READ _techspec.md, every ADR, and every per-task memory file before planning.</critical>

<requirements>
1. MUST activate the `qa-report` skill with `qa-docs-path=docs/qa` (tree exists — do not re-bootstrap).
2. MUST reset to `untested` the scenarios named across task_01..04 QA-impact lines: `ET-window-manager-layout-recovery`, `RT-home-usage-window-persistence`, `MS-configure-window-manager`, `ET-window-manager-public-parity`, `ET-window-manager-hooks-resources`, `MS-layout-profile-cli-roundtrip`, `ET-web-window-routing-lifecycle`, `ET-window-manager-layout-gestures`, `ET-window-manager-drop-swap`, `ET-window-manager-multi-client`, `ET-web-desktop-shell-lifecycle`, `ET-web-dock-default-window-size`, `RT-desktop-pager-overview`.
3. MUST mint new content-addressed `untested` scenario files covering: deck lifecycle + grouping (drag/menu/⌘T), close scopes + multi-level reopen (incl. reload survival), pins, multi-instance apps + dock cycling, ⌘K tab search, per-tab navigation stacks, agent tab management via CLI/native tools (parity + deterministic errors), v3 discard on stale snapshots, and the D3 strip relocation.
4. MUST express coverage as journey-derived `entry_points` rows spanning every public surface touched (CLI verbs, HTTP/UDS, web routes, config keys, hook events, doc pages) — not standalone test cases.
5. MUST map regression hot spots from TechSpec Safety Invariants 4/9/13/15 and ADR-009/012/013 into the charter selection (targeted tier + one adjacent canary journey, e.g., desktops/overview).
6. MUST dedup against the existing registry (content-addressed ids; no shared-counter coordination).
</requirements>

## Subtasks

- [x] 5.1 Audit task_01..04 completion notes + QA-impact lines; consolidate the reset list
- [x] 5.2 Reset affected scenario files' `qa_status` with feature references
- [x] 5.3 Mint the new tab scenario files (content-addressed, journey-derived entry points)
- [x] 5.4 Update `docs/qa/journeys/` flowcharts for the tabbed shell
- [x] 5.5 Write session charters (personas: keyboard-heavy operator, multi-agent supervisor, agent-via-CLI) with the invariant hot-spot map
- [x] 5.6 Verify tree consistency (`state.csv` regenerates; no dangling references)

## Implementation Details

Scenario inventory and current assertions are listed in the transcript exploration report §16. Follow the `qa-report` skill contract for file shapes; keep `docs/qa/` as the single source (no per-round trees).

### Relevant Files

- `docs/qa/scenarios/*.md` — resets + new files
- `docs/qa/journeys/`, `docs/qa/charters/` — cycle planning artifacts
- `docs/qa/bugs/` — registry consulted for dedup

### Dependent Files

- `docs/qa/reports/` — task_06 writes the dated report against this plan

### Related ADRs

- [ADR-004](adrs/adr-004.md), [ADR-006](adrs/adr-006.md), [ADR-009](adrs/adr-009.md), [ADR-012](adrs/adr-012.md), [ADR-013](adrs/adr-013.md) — behaviors the charters must stress

## Deliverables

- Updated `docs/qa/scenarios/` (resets + new content-addressed files), journeys, and charters for the cycle
- Charter selection with invariant hot-spot mapping and one canary journey
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED — none assigned: planning task; the qa-execution walk owns verification)**

## Tests

No `_tests.md` IDs assigned — this task plans scenario coverage; automated-suite IDs are owned by tasks 01-03 and the walk by task_06.

## Success Criteria

- Every public surface touched by tasks 01-04 appears as an entry point in at least one scenario
- All resets and new scenarios committed with valid content-addressed ids; zero dangling references
- Charters name personas, scope, invariant hot spots, and evidence expectations for task_06

## Completion Notes

- Reset all 13 named scenarios and added explicit `2026-07-31` window-tabs impact notes.
- Added nine content-addressed scenarios derived from `J-organize-tabbed-work` and
  `J-agent-manage-window-tabs`; their entry points span Web routes, CLI, HTTP, UDS, native tools,
  config keys, hooks, streams, and official docs.
- Added four targeted charters for Bruno, Théo, Ada, and the Home canary through Cora. The plan maps
  TechSpec invariants 4/9/13/15 plus ADR-009/012/013 and retains the desktop pager as the adjacent
  canary.
- `materialize_state.py docs/qa` parsed the complete tree and regenerated 711 rows with no schema or
  reference error.

Compozy Impact Audit:

- Native tools: planning covers the five tab tools, descriptors, capability-gated discovery,
  deterministic errors, and CLI/API fallback parity; no runtime mutation in this task.
- Extensibility and hooks: planning covers all five tab hook events, rejection silence, resources,
  bundles, config lifecycle, and the official skill; no extension code changed.
- Workspace data isolation: charters explicitly compare shared workspace topology with per-client
  presentation and foreign-workspace rejection across CLI/HTTP/UDS/Web/stream paths.
- Official Compozy skill: no new edit; the Task 04 skill reference is an Ada charter entry point and
  execution oracle.
