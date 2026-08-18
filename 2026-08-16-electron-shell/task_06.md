---
status: completed
title: QA Plan and Session Charters
type: qa-report
complexity: high
---

# Task 6: QA Plan and Session Charters

## Overview

Plans the migration's QA cycle over the living `docs/qa/` tree: updates journeys, mints/updates scenario files for every surface the program touched, and writes this cycle's session charters — including the macOS + Linux matrix the release gate demands and the browser-e2e coverage the UI-bearing surface requires.

<critical>ALWAYS READ _spec.md, every ADR, and every per-task memory file before planning.</critical>

<requirements>
1. MUST activate the `qa-report` skill with `qa-docs-path=docs/qa` (the tree exists — never bootstrap a per-round tree).
2. MUST express coverage as scenario `entry_points` on journey-derived rows — never standalone test cases — for every public surface tasks 01–05 touched: `compozy update` (+`--cancel`), the deleted verb's absence, `compozy app status/open/retry/diagnose` on the new shell, `GET/POST /api/settings/update{,/apply,/cancel}` (HTTP+UDS), the web Updates section + menubar indicator (browser and app), deep links, install/first-run offline, quit contract, zoom/geometry, update cycles (app + runtime, staged, blocked, rollback), publish/repair operator surface, `[app]` config keys, and the docs entry points (installation, desktop-app, runbook).
3. MUST reconcile the program's flag set: every `APP-*` and `REL-*` reset plus the new scenarios added by tasks 01–05 appear with `qa_status: untested` and correct content-addressed ids; dedup same-behavior adds.
4. MUST update the desktop journeys (`J-desktop-first-run`, `J-desktop-attach-daily`, `J-desktop-update-moment`, `J-desktop-link-driven`, `J-desktop-agent-headless`) where behavior changed (offline first run, single update command, web update surface, overlay non-interactive) — flows preserved, mechanics updated.
5. MUST write this cycle's charters in `docs/qa/charters/`: the macOS + Linux shell matrix (replacing the `tauri-driver` charters with Playwright-`_electron`-era equivalents), the update-moment rehearsal charter (incl. the real beta N→N+1 gate procedure), the browser-parity charter (same UI in Chrome vs app), and the headless-agent charter — targeted tier plus one adjacent canary journey per the template.
6. MUST map regression hot spots from `_spec.md` Part II Safety Invariants 1–18 and ADRs 002/009 (single-flight, recovery, staged/installer-handoff, token auth, CSP) into the charter selection.
</requirements>

## Subtasks

- [x] 6.1 Inventory every public surface from tasks 01–05 and reconcile the scenario flag set (resets + adds, content-addressed, deduped)
- [x] 6.2 Update the five desktop journeys to the shipped mechanics
- [x] 6.3 Write the cycle charters (shell matrix macOS+Linux, update rehearsal incl. N→N+1 gate, browser parity, headless agent) with hot-spot mapping
- [x] 6.4 Verify `state.csv` regenerates cleanly from the tree (generated view only)
- [x] 6.5 Hand the cycle package to task_07 with explicit walk order and gate criteria

## Implementation Details

Operate exclusively on `docs/qa/{scenarios,journeys,charters,bugs,reports}` — the living contract. The release gate (Part I): all 14 `APP-*` recorded `pass` on both OSes + one real beta N→N+1 auto-update cycle; the charters must make that walkable and evidence-able.

### Relevant Files

- `docs/qa/scenarios/APP-*.md`, `REL-*.md` + the new scenario files from tasks 01–05
- `docs/qa/journeys/J-desktop-*.md`
- `docs/qa/charters/CH-desktop-*.md` (incl. the obsolete `tauri-driver` ones to replace)
- `_spec.md` Safety Invariants + `qa/peer-review-summary-round{1,2,3}.md` — hot-spot sources

### Related ADRs

- [ADR-009](adrs/adr-009.md), [ADR-006](adrs/adr-006.md), [ADR-003](adrs/adr-003.md) — the behaviors this cycle pressure-tests

## Deliverables

- Updated journeys + complete scenario flag set + this cycle's charters in `docs/qa/`
- Walk order + gate criteria for task_07

## Tests

No `_tests.md` ids — planning task. The walk evidence contract lives in task_07.

## Success Criteria

- Every touched surface has a journey-derived scenario row with entry points
- Charters cover both OSes, the N→N+1 gate, browser parity, and the hot-spot map
- No per-round `qa/` tree, no standalone test-case files, no duplicated scenario ids
