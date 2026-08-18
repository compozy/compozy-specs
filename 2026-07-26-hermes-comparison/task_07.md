---
status: pending
title: Consent-first automation suggestion store with starter catalog
type: backend
complexity: high
---

# Task 7: Consent-first automation suggestion store with starter catalog

## Overview

Consent-first suggestion store with catalog seeding; plain Job payload only (loop-template
deferred). AGH never *offers* automations today — without templates or suggestions, users and
agents must hand-author a self-contained prompt plus correct cron (the dominant product gap of the
automation slice). Per ADR-007, blueprints were skipped (loop templates are the canonical
parameterized-template system); the suggestion layer alone closes the offering gap.

**Authoritative sources:** this suite has no `_prd.md` / `_user_stories.md` / `_tests.md`.
`_techspec.md` and `adrs/adr-007.md` are authoritative. Concrete test cases are inline below
(exact input/condition/expected).

Merges former task 22. **Depends on task_01** (Job model: `RunLimit`, catch-up fields, etc.) and
**task_04** (lifecycle guard) — edges `01→07` and `04→07`.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST implement a consent-first Suggestion store: sources `catalog|usage|integration` (NO
   `blueprint` source — ADR-007), dedup-latched dismissals (a dismissed suggestion never
   reappears), pending cap (default 5, configurable), accept → create.
2. Suggestion payload MUST be a prefilled plain `Job` spec (prompt + schedule — no slot
   machinery), and only that. The loop-template payload variant is NOT built in this task — it
   lands with the Loops program as a hard cut (peer-review round 1 N-001; ADR-007: no inert
   feature-gated scaffolding without a production caller).
3. MUST ship `agh__automation_suggestions_{list,accept,dismiss}` native tools + CLI/HTTP/UDS
   parity + a web card (SD-011 triad).
4. MUST seed a 3–5-entry starter catalog per new workspace; suggestions and catalog are
   workspace-scoped.
5. Accepted suggestions creating jobs MUST pass the task_04 lifecycle guard and task_01 model
   fields (`RunLimit`, catch-up policy, etc. available in the prefill shape).
</requirements>

## Subtasks

- [ ] 7.1 Suggestion model + tables (append-only migration) + dedup/cap semantics.
- [ ] 7.2 Sources: starter catalog seeding + usage/integration emitters (minimal viable set).
- [ ] 7.3 Native toolset + CLI/HTTP/UDS + codegen co-ship.
- [ ] 7.4 Web suggestions card (accept/dismiss) — screenshot.
- [ ] 7.5 Docs (`skills/agh/`) for the suggestion flow (loop-template variant explicitly out of
      scope — deferred to the Loops program, N-001).
- [ ] 7.6 Accept path integration: prefill Job shape includes task_01 fields; creation passes
      through task_04 lifecycle guard (blocked payloads never reach the store).

## Implementation Details

See `_techspec.md` §3.7 / ADR-007. New tables append at migration tail. Workspace isolation:
`Scope`+`WorkspaceID` follow the existing automation-row pattern. Accept creates a job via the
same creation seam that enforces the lifecycle guard (task_04) — do not bypass it.

### Relevant Files

- `internal/automation/` — suggestion model/store/service
- `internal/store/globaldb/` — tables + migration
- `internal/daemon/native_tools.go` — toolset
- task_01 Job model fields (`RunLimit`, catch-up policy) — prefill shape consumers
- task_04 lifecycle guard — creation-seam validation (must not be bypassed on accept)

### Dependent Files

- `internal/api/contract/` + TS — payloads
- `web/` automation views — suggestions card
- `skills/agh/` — suggestion flow docs
- `.compozy/tasks/loops/_techspec.md` — loop-template variant deferred here (N-001)

### Related ADRs

- [ADR-007: Automation suggestions and schedule semantics](adrs/adr-007.md) — consent-first store,
  no blueprint source, plain Job payload only; loop-template deferred

### Competitor References

- `.resources/hermes/cron/suggestions.py` — consent-first store, dedup latch, cap
- `.resources/hermes/cron/suggestion_catalog.py` — starter catalog shape

## Deliverables

- Working suggest→accept→job flow with consent + dedup guarantees
- Starter catalog seeds 3–5 suggestions per new workspace
- Plain Job payload only; loop-template variant explicitly out of scope
- Every test case in `## Tests` implemented and passing **(REQUIRED)**

## Tests

No `_tests.md` for this suite — concrete inline cases below. Coverage target ≥80% on touched
packages; all Go tests under `-race`.

- Unit (`internal/automation/*_test.go` — extend):
  - [ ] Dismiss → identical suggestion re-emission is latched (never reappears)
  - [ ] Pending cap 5 → sixth suggestion rejected/queued per documented rule
  - [ ] Accept job-variant → job created matching prefill; passes task_04 lifecycle guard
        (blocked payloads never reach the store)
  - [ ] Prefill Job shape carries task_01 fields (`RunLimit`, catch-up policy) when present in
        the catalog entry
  - [ ] Workspace isolation on list/accept/dismiss
- Integration (`make test-integration`):
  - [ ] Fresh workspace → starter catalog seeds 3–5 suggestions; accept one → automation job
        exists and fires under the scheduler
  - [ ] Native tools + CLI + HTTP parity on the full flow
  - [ ] Accept of a lifecycle-command prefill → rejected by task_04 guard; no job persisted
- E2E (`make test-e2e-web`):
  - [ ] Suggestions card renders, accept creates the job, dismiss latches

## Success Criteria

- Every assigned test case implemented and passing
- Coverage ≥80% on touched packages
- A new workspace offers automations without any hand-authoring (the A1 gap closes)
- Accept path cannot bypass the task_04 lifecycle guard
- Web screenshot via `eng-ui-screenshot` cited for the suggestions card
- Loop-template payload variant absent (hard cut; no inert scaffolding)
