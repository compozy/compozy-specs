---
status: pending
title: QA report — scenario matrix and evidence contract for the Hermes-comparison program
type: test
complexity: medium
---

# Task 12: QA report — scenario matrix and evidence contract for the Hermes-comparison program

## Overview

`make verify` is necessary but not sufficient (SD-005): this program touches approvals, resume,
compaction, drain, cost, suggestions, checkpoints, MCP serve, and the task-orchestration kernel
(loop/lease/scheduler) — surfaces whose drift only real-scenario QA catches. This task plans the
QA cycle in the living `docs/qa/` tree FIRST: scenario rows, session charters, the per-scenario
evidence contract, and the release-verdict criteria that task_13's execution fills in.

**Authoritative sources:** this suite has no `_prd.md` / `_user_stories.md` / `_tests.md`.
`_techspec.md`, `analysis/summary.md`, and ADRs `adrs/adr-001..010.md` are authoritative. The
scenario matrix below is the contract.

Merges former task 34. Depends on all implementation slices (`01..11 → 12`). Update all scenario
refs from former tasks 01–33 → resliced tasks 01–11.

Activate `qa-report` (`qa-docs-path=docs/qa`) before writing any plan artifact.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST plan in the living `docs/qa/` tree per the `qa-report` skill contract
   (`qa-docs-path=docs/qa`): mint one `state.csv` row per scenario below under the existing area
   codes (RT for session/approval/resume/compaction, TA for automation/kernel, ET for MCP/registry,
   MS for redaction/cost as fits), each with expected observable in SD-006 forensic form
   (timestamp, command, observed output), plus session charters grouping the scenarios; verdicts
   start `untested` for task_13 to fill (`pass`/`fail`/`blocked-verify`).
2. MUST map every scenario to the techspec goals it verifies (which of D1-D7, O1-O5, which ADR
   items) so coverage gaps are visible before execution runs. Map ADR-001..010 per scenario.
3. MUST define the machine-readable QA bootstrap block schema (manifest path, lab root, runtime
   home, base URL, verification evidence) that execution persists.
4. MUST state the release-readiness verdict criteria up front (what constitutes blocking), per
   COPY.md claim standards — no "complete" without evidence.
5. MUST update all task-number references from the v1 matrix (tasks 01–33) to the resliced
   program (tasks 01–11) in charters, coverage map, and scenario notes.
</requirements>

## Subtasks

- [ ] 12.1 Mint scenario rows in `docs/qa/state.csv` + write session charters (matrix + evidence
      contract); update refs tasks 01–33 → 01–11.
- [ ] 12.2 Goal/ADR coverage map (gaps visible before execution), covering D1-D7 and O1-O5 and
      ADR-001..010 mapped per scenario.
- [ ] 12.3 Bootstrap-block schema + release-readiness verdict criteria (the report's Final Status
      bar).

## Implementation Details

The `qa-report` skill owns the structure. The plan must be consumable by task_13 (execution fills
verdicts + evidence + the dated report) and by timed continuations without re-deriving lab state.

### Scenario matrix (17 scenarios — each carries expected evidence + goal/ADR mapping)

1. **Approval:** grant `allow_always` → restart daemon → same tool auto-approves; revoke →
   prompts. (D1, ADR-001; task_01)
2. **Clarify:** `RequiresInteraction` extension tool → question in web/CLI → answer → completion.
   (D7, ADR-001; task_01)
3. **Resume:** kill daemon mid-conversation (load-unsupported agent) → resume → context continuity
   + "rebuilt" marker. (D4, ADR-002; task_02)
4. **Compaction:** long-session pressure → compaction fires once with guards; archived events
   queryable; kill daemon post-compaction BEFORE session end → resume reconstructs the archived
   span's facts from the injected checkpoint-summary (no silent loss). (D3, ADR-003, B-301;
   task_03)
5. **Automation:** downtime over a schedule boundary → exactly one catch-up fire; overlap fixture
   skips with reason; lifecycle-command job creation rejected. (D2, ADR-007/010; tasks 01+04+07)
6. **Drain:** drain during active run → run finishes, new admissions refused, undrain restores.
   (ADR-010; task_05)
7. **Cost:** silent-agent session shows estimated cost; subscription provider shows `included`.
   (U1, ADR-006; task_06)
8. **Suggestions:** fresh workspace seeds starter suggestions; accept → job fires; dismiss
   latches. (A1, ADR-007; task_07)
9. **Checkpoints:** native mutation → checkpoint → restore reverts; delegated edit NOT
   checkpointed AND a delegated edit made after the snapshot survives restore (diff-scoped blast
   radius, overlapping paths previewed). (ADR-004, N-301; task_08)
10. **Redaction:** planted secret absent from logs/SSE/output captures. (G2, ADR-005; task_04)
11. **MCP serve:** third-party MCP client lists sessions + operates a task; workspace isolation
    held. (ADR-008; task_10)
12. **Registry:** offline search serves seed/cache; dead MCP sidecar goes low-frequency +
    recovers. (ADR-009/010; tasks 09+05)
13. **Crash-loop bound:** worker claims a run then is killed pre-heartbeat repeatedly →
    lease-expiry reclaims terminate after `max_attempts` with a forensic reason, not forever.
    (O1, §3.10; task_11)
14. **Loop breaker:** two-node loop, one node always fails / one always succeeds → loop `Stalled`
    at the node failure streak, NOT run to the iteration cap; unbounded watch loop hits its
    backstop. (O2, §3.10; task_11)
15. **Double-claim:** two concurrent `ClaimRun` on one queued run → exactly one owner, the other a
    typed already-claimed error (`-race`). (O3, §3.10; task_11)
16. **Wedged node:** action-node run heartbeats but makes no progress past its window →
    terminalizes `node_timeout`, lease freed, loop advances; a progressing long run is untouched.
    (O4, §3.10; task_11)
17. **Back-pressure:** saturate a workspace's active-run cap via a wide fan-out → new admissions
    defer (recorded) instead of piling up/over-spawning; deferred work drains; workspace B
    unaffected. (O5, §3.10; task_11)

### Relevant Files

- `docs/qa/state.csv` + `docs/qa/charters/` — the plan (new rows/charters)
- `docs/qa/journeys/` — journey flowcharts if the skill requires updates

### Dependent Files

- task_13 execution — fills this artifact

### Related ADRs

- [ADR-001](adrs/adr-001.md) — scenarios 1–2 (approvals, clarify)
- [ADR-002](adrs/adr-002.md) — scenario 3 (resume/replay)
- [ADR-003](adrs/adr-003.md) — scenario 4 (compaction/checkpoints)
- [ADR-004](adrs/adr-004.md) — scenario 9 (shadow-git)
- [ADR-005](adrs/adr-005.md) — scenario 10 (redaction)
- [ADR-006](adrs/adr-006.md) — scenario 7 (cost)
- [ADR-007](adrs/adr-007.md) — scenarios 5, 8 (automation, suggestions)
- [ADR-008](adrs/adr-008.md) — scenario 11 (MCP serve)
- [ADR-009](adrs/adr-009.md) — scenario 12 (registry/MCP catalog)
- [ADR-010](adrs/adr-010.md) — scenarios 5–6, 12 (lifecycle, drain, dead-entity)
- TechSpec §3.10 — scenarios 13–17 (O1–O5)

### Competitor References

- Not applicable — QA authoring task

## Deliverables

- 17 scenario rows + charters in `docs/qa/`, evidence contract, coverage map, verdict criteria
- All task refs updated 01–33 → 01–11
- ADR-001..010 mapped per scenario; D1-D7 + O1-O5 coverage visible

## Tests

No `_tests.md` for this suite — concrete inline cases below (review gate, not code tests).

- Unit: N/A — QA authoring artifact; no code changes
- Integration: N/A — same rationale
- E2E: N/A — execution happens in task_13
- Review gate:
  - [ ] Every scenario names expected evidence AND its goal/ADR mapping — a scenario without both
        is a defect of this task
  - [ ] 17/17 scenarios exist as `state.csv` rows (status `untested`)
  - [ ] Coverage map shows every D1-D7 and O1-O5 defect and every accepted ADR (001–010) verified
        by ≥1 scenario
  - [ ] All scenario notes reference resliced tasks 01–11 (no stale 01–33 refs)
  - [ ] Verdict criteria and bootstrap-block schema defined and machine-readable

## Success Criteria

- 17/17 scenarios exist as `state.csv` rows (status `untested`), each carrying expected evidence +
  goal/ADR mapping; charters group them into runnable sessions
- Coverage map shows every D1-D7 and O1-O5 defect and every accepted ADR verified by ≥1 scenario
- Verdict criteria and bootstrap-block schema defined and machine-readable
- Task-number refs updated to the resliced program (01–11)
