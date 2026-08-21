---
status: pending
title: QA Plan and Session Charters
type: qa-report
complexity: high
---

# Task 8: QA Plan and Session Charters

## Overview

Plans the real-user QA cycle for skill sources over the living `docs/qa/` tree: journey updates, scenario coverage for every public surface tasks 01-07 touched, and session charters for the execution pass. Planning only — execution happens in task_09.

<critical>ALWAYS READ `_spec.md`, every ADR under `adrs/`, and every per-task memory file before planning.</critical>

<requirements>
1. Activate the `qa-report` skill with `qa-docs-path=docs/qa` (bootstrap the tree if absent).
2. Coverage: every public surface touched by tasks 01-07 — CLI verbs (`skill sources|expose|unexpose|create --expose|info|where|list`, `config set/get/unset` for the two keys), HTTP/UDS routes (settings skills GET/PATCH both scopes, expose/unexpose, skill detail/list), web routes (Settings > Skills sources section, marketplace skill detail exposures, session composer picker), native tools (`compozy__skill_list/_search/_view`, `compozy__config_*` for the keys), extension Host API skill listing, suppression diagnostics, observe events, and the `config.toml` keys — expressed as scenario `entry_points` on journey-derived rows, not standalone test cases.
3. Reconcile the scenarios flagged `untested` by tasks 01-06 (config keys; live toggling; sources diagnostics; expose lifecycle; origin attribution; picker + suppression; settings UI; expose panel) — dedup same-behavior files by content-addressed id, never a shared counter.
4. Map regression hot spots from `_spec.md` Part II Safety Invariants and ADRs into the cycle's charter selection (targeted tier + one adjacent canary journey — skill invocation and marketplace install/remove are the natural canaries).
5. Output: journey flowcharts updated in `docs/qa/journeys/`, scenario files minted/updated in `docs/qa/scenarios/`, session charters in `docs/qa/charters/` for this cycle.
</requirements>

## Subtasks

- [ ] 8.1 Read the shipped surface set (tasks 01-07 completion notes + `_dx.md`/`_uiux.md`) and inventory entry points
- [ ] 8.2 Update/mint journey flowcharts covering source configuration, absorption, session usage, expose, diagnostics
- [ ] 8.3 Mint/update content-addressed scenario files; reconcile and dedup the task-flagged `untested` scenarios
- [ ] 8.4 Select charters: targeted tier for this feature + one adjacent canary journey
- [ ] 8.5 Record the cycle's charter set in `docs/qa/charters/`

## Implementation Details

Operates exclusively on the committed `docs/qa/` tree (`state.csv` is generated output only). No per-round `qa/` trees, no standalone test-case documents.

### Relevant Files

- `docs/qa/scenarios/` + `docs/qa/journeys/` + `docs/qa/charters/` + `docs/qa/bugs/` — the living QA tree
- `.compozy/tasks/skill-sources/{_spec.md,_dx.md,_uiux.md,_user_stories.md}` — behavior authority for scenario derivation

### Dependent Files

- `docs/qa/reports/` — task_09 writes the dated run report against this plan

### Related ADRs

- All 16 (`adrs/adr-001..016`) — charter hot-spot mapping draws on the full decision set; Safety Invariants 1-12 are the regression map.

## Deliverables

- Updated journeys, content-addressed scenarios (every touched surface has an entry point), and this cycle's charters in `docs/qa/`

## Tests

No `_tests.md` IDs — planning task. The plan's completeness gate: every public surface named in requirement 2 appears as an `entry_points` value in at least one scenario file.

## Success Criteria

- Every task-flagged scenario reconciled (no orphan `untested` files without a charter path)
- Charters name persona, scope, and evidence expectations for the execution pass
- No scenario duplicates an existing same-behavior file (content-addressed dedup applied)
