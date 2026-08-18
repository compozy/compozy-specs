---
status: completed
title: "qa-execution: isolated platform walks, fix loop, clean teardown"
type: qa-execution
complexity: critical
---

# Task 7: qa-execution: isolated platform walks, fix loop, clean teardown

## Overview

Executes the planned desktop QA cycle as a real user would: walks every journey and charter from task_06 in isolated labs across the three platforms, settles every reset/`untested` scenario to a recorded verdict with evidence, runs the fix loop on defects (production fixes, never weakened tests), and closes the workstream with the full gate and clean teardown evidence. This task owns the entire E2E catalog.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST bootstrap fresh isolated labs via `eng-qa-bootstrap` (unique `COMPOZY_HOME`, non-default ports, provider-home policy per contract) — `eng-worktree-isolation` when parallel; never the default home.
2. MUST walk every E2E case below per its `_tests.md` definition and the task_06 charters, recording verdicts with evidence in the `docs/qa/` tracker; `blocked-verify`/`blocked-decision` are the only acceptable non-pass leftovers, each justified.
3. The **SSE-load release gate (E2E-021)** MUST run on all three webviews with measured connection profiles recorded as release evidence — a failure blocks ship and escalates per TechSpec Known Risks.
4. The **update rehearsal** MUST include: forced mid-install failure (previous app intact + launchable), post-update permission assertion (not `0700`), pending-update-on-quit, roll-forward revert reaching a stale client, and the recovery-state journey (E2E-025).
5. Defects follow the fix loop: forensic reproduction → production fix → regression at the owning layer → re-walk (SD-006; never weaken a test).
6. macOS journeys without WebDriver run as the charter-defined scripted-manual smoke with recorded evidence; Linux/Windows use `tauri-driver` scripting where the charters say so.
7. MUST end with `make gate-full` (workstream close) and **clean teardown on every terminal path**: `eval "$TEARDOWN_COMMAND"` per lab, `teardown.json` `"clean": true` cited, PIDs registered at spawn (L-029). Completing with live labs/daemons/apps is a blocking failure.
8. Reconcile ship-impact: every scenario settled, `_tasks.md` statuses truthful, memory files updated.
</requirements>

## Subtasks

- [x] 7.1 Bootstrap labs (per platform where hardware allows; document platform coverage honestly)
- [x] 7.2 Walk install/first-run + attach + start-installed journeys (E2E-001–005, 020)
- [x] 7.3 Walk connection-state + native-integration journeys (E2E-006–011, 023)
- [x] 7.4 Walk the update system: app update, pending-on-quit, failure fallback, runtime single-experience, managed recommendation, recovery state (E2E-012–015, 022, 025)
- [x] 7.5 Walk channel visibility + agent CLI journeys (E2E-016, 017, 024) + browser dual-door (E2E-018)
- [x] 7.6 Run the SSE-load gate (E2E-021) with measured profiles on the three webviews
- [x] 7.7 Fix loop on defects + re-walks; settle every scenario; strict evidence audit
- [x] 7.8 `make gate-full` + teardown on all labs + ship-impact reconciliation

## Implementation Details

Execution contract: `qa-execution` skill over the task_06 tree; lab lifecycle: `eng-qa-bootstrap` manifest (envelope vars, teardown command); real binaries only (shell build from task_05 artifacts or local release build; real `compozy` daemon). Platform notes per E2E case live in `_tests.md`.

### Relevant Files

- `docs/qa/**` — tracker, journeys, charters, bug registry, dated report
- Task_05 draft-release artifacts — the builds under test
- `<QA_OUTPUT_PATH>/qa/pids/` — process registry per L-029

### Related ADRs

- [ADR-007](adrs/adr-007.md), [ADR-011](adrs/adr-011.md) — the contracts under validation

## Deliverables

- Every E2E case settled with a recorded verdict + evidence; dated QA report; defects fixed and re-walked
- `make gate-full` green at close; `teardown.json` `"clean": true` for every lab
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED — as recorded walks)**

## Tests

- [x] E2E-001, E2E-002, E2E-020 — newcomer install/first-run/reinstall journeys
- [x] E2E-003, E2E-004, E2E-018 — attach, start-installed, dual-door sync
- [x] E2E-005, E2E-006, E2E-011 — skew, crash-recovery, flash-free/bounded launch
- [x] E2E-007, E2E-008, E2E-009, E2E-010, E2E-023 — single instance, deep links (running/cold/malformed), external links, geometry
- [x] E2E-012, E2E-013, E2E-014, E2E-015, E2E-022, E2E-025 — update system incl. failure fallback, single-experience, managed, pending-on-quit, recovery state
- [x] E2E-016, E2E-017, E2E-024 — channel visibility, agent CLI lifecycle, agent-driven update
- [x] E2E-021 — SSE-load release gate (3 webviews, measured)

## Web/Docs Impact

`docs/qa/` verdicts + dated report; any doc corrections found route back as fixes. **QA impact:** this task IS the verification half — every flagged scenario from tasks 01–05 gets walked here.

## Extensibility / Agent Manageability / Config Lifecycle

None — checked: validation-only; defects found in those surfaces are fixed at source under the fix loop.

## References

- `qa-execution`, `eng-qa-bootstrap`, `eng-real-scenario-qa`, `eng-worktree-isolation` skills; `_tests.md` E2E definitions; task_06 charters

## Success Criteria

- Zero `untested`/`fail` scenarios at close (only justified `blocked-*`); strict evidence audit passes with zero blockers
- `make gate-full` green; every lab's `teardown.json` `"clean": true`
- SSE gate evidence recorded; release ship/no-ship recommendation explicit
