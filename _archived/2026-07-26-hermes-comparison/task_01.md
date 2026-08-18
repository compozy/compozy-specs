---
status: pending
title: "Interaction and supervision: approval grants, clarify, automation misfire, health verdicts"
type: backend
complexity: high
---

# Task 1: Interaction and supervision: approval grants, clarify, automation misfire, health verdicts

## Overview

Closes D1/D2/D5/D7 — durable approval grants, the `agh__clarify` question channel, automation
misfire/catch-up/overlap/`RunLimit`, and the health-verdict→action loop. These are independent
defects in the same correction/supervision layer: users get truthful durable approvals, agents can
ask mid-run questions, automation recovers after downtime without silent skips, and unhealthy
subprocess verdicts become observable and escalatable.

**Authoritative sources:** this suite has no `_prd.md` / `_user_stories.md` / `_tests.md`.
`_techspec.md` and `adrs/adr-001.md`, `adr-007.md` §4, `adr-010.md` §4 are authoritative. Concrete
test cases are inline below (exact input/condition/expected).

Merges former tasks 01+02+05+07.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
### Approvals (ex-01 / D1)
1. MUST persist `allow_always`/`reject_always` selections in a durable store keyed by
   `{workspace_id, agent_name?, tool_id, input_digest?}` and consult it BEFORE prompting.
2. MUST keep `deny-all`/`approve-reads`/`approve-all` as the sole policy ceiling: the store only
   widens auto-approval; it never introduces a new prompt class and never re-prompts on a
   remembered grant.
3. MUST expose management via `agh__tool_approvals_{list,revoke}` native tools + CLI/HTTP/UDS
   parity, with the codegen co-ship.
4. MUST NOT add dynamic per-command risk classification (declined in ADR-005 of this program).
5. MUST leave sandbox-level `PermissionDecisionAllowAlways` untouched (distinct surface).

### Clarify (ex-02 / D7)
6. MUST add a native `agh__clarify` tool: `{question, choices[≤4]}` blocks until a user answer is
   returned as the tool result.
7. MUST back it with a pending-request/resolve channel modeled on `internal/acp/permission.go`,
   honoring a config timeout with a deterministic fallback (fail-with-default or fail-open,
   decided in config).
8. MUST route `RequiresInteraction` tools through clarify instead of dead-ending.
9. MUST expose the pending question over SSE + web answer UI + CLI answer path (agent-manageable:
   an agent-facing answer verb too).
10. MUST NOT alter approval frequency or semantics in any way (ADR-001 anti-spam constraint).
11. MUST be reachable by extension tools via the extension host API.

### Automation misfire (ex-05 / D2)
12. MUST wire `MisfireGraceSeconds` so the skip-vs-grace boundary actually functions.
13. MUST add a `run_once_on_catchup` catch-up policy with period-adaptive default grace; the DB
    CHECK on the policy column is migrated append-only to accept the new value.
14. MUST add a self-overlap guard: a fire is skipped (with a recorded reason) while the job's prior
    run is still executing.
15. MUST add a lifetime `RunLimit` (run-N-then-disable/delete) named unambiguously, distinct from
    the rolling `FireLimit` rate cap.
16. MUST keep the SQLite claim-based at-most-once semantics untouched.

### Health verdicts (ex-07 / D5)
17. MUST surface unhealthy verdicts as a doctor probe and as status degradation on the session/
    agent status surfaces.
18. MUST add a config-gated escalation: persistently-unhealthy agents escalate to
    `needs_attention` via the existing scheduler escalator — never a parallel escalation primitive
    (L-005 exclusivity).
19. MUST NOT introduce subprocess auto-restart in this task (restart-loop budget is deferred until
    auto-restart exists — ADR-010 §4).
20. MUST emit the canonical observability event with correlation keys on every escalation.
</requirements>

## Subtasks (order: approvals → clarify → automation → health)

- [ ] 1.1 Durable approval-grant store (schema + append-only migration in the global DB, scoped
      per workspace).
- [ ] 1.2 Bridge consult-before-prompt + persist-on-always in `toolApprovalOutcome` path; delete
      the silent always→once downgrade.
- [ ] 1.3 `agh__tool_approvals_{list,revoke}` native toolset + HTTP/UDS routes + CLI verbs with
      `-o json` + config lifecycle + web remembered-grants list/revoke view.
- [ ] 1.4 Clarify pending-request/resolve channel (session-scoped, workspace-isolated) +
      `TurnSource`-correct blocking semantics.
- [ ] 1.5 `agh__clarify` descriptor/toolset + `RequiresInteraction` routing through clarify.
- [ ] 1.6 SSE question event + web answer UI + CLI/HTTP/UDS answer surfaces + config timeout key +
      extension host API exposure + `skills/agh/` docs.
- [ ] 1.7 Wire `MisfireGraceSeconds` + add `run_once_on_catchup` (+ append-only CHECK migration).
- [ ] 1.8 Self-overlap guard with recorded skip reason + lifetime `RunLimit` field +
      disable/delete-on-exhaustion + surfaces (CLI/HTTP/UDS + `agh__automation_jobs_*` schema).
- [ ] 1.9 Health-verdict consumer wiring at the composition root (SD-008).
- [ ] 1.10 Doctor probe + status surface degradation + config-gated `needs_attention` escalation
      through the scheduler escalator.

## Implementation Details

See `_techspec.md` §3.1 (approvals/clarify/offload), §3.7 (automation), §3.5 (health). Migrations
are append-only at the registry tail (L-021). Contract changes co-ship OpenAPI + TS types + E2E
mocks (L-007). Delete targets: silent always→once collapsing branches; dead unwired
`MisfireGraceSeconds` state (Delete Targets table, `_techspec.md` §4).

### Relevant Files

- `internal/daemon/tool_approval_bridge.go` — consult/persist wiring (silent downgrade today)
- `internal/tools/policy.go` — grant-widening under the three-mode ceiling
- `internal/store/globaldb/` — grant table + automation CHECK/RunLimit migrations
- `internal/acp/permission.go` — pending/resolve pattern for clarify
- `internal/automation/schedule.go`, `internal/automation/model/types.go` — misfire/overlap/RunLimit
- `internal/subprocess/health.go` — verdict source (no consumer today)
- `internal/scheduler/starvation.go` — escalator entry
- `internal/doctor/` — health + memory probe registration
- `internal/api/core/`, `internal/cli/`, `internal/sse/` — management surfaces

### Dependent Files

- `internal/api/contract/` + generated TS — payload additions
- `internal/extension/` host API — clarify exposure
- `web/` — approval grants view, clarify answer UI, automation job form, status surfaces
- `skills/agh/` — approvals/clarify/automation/health docs
- E2E runtime harness mock agent — load/health scenarios

### Related ADRs

- [ADR-001: Durable approval grants and clarify channel](adrs/adr-001.md) — grant store + clarify
  design; three-mode ceiling; anti-spam constraint
- [ADR-007: Automation suggestions and schedule semantics](adrs/adr-007.md) §4 — catch-up policy,
  overlap guard, `RunLimit` vs `FireLimit`
- [ADR-010: Daemon reliability primitives](adrs/adr-010.md) §4 — health-verdict→action loop;
  no auto-restart in this slice

### Competitor References

- `.resources/hermes/tools/approval.py` — persisted allowlist semantics (adapt keying)
- `.resources/hermes/tools/clarify_tool.py`, `.resources/hermes/tools/clarify_gateway.py`
- `.resources/hermes/cron/jobs.py`, `.resources/hermes/cron/scheduler.py` — catch-up/grace/overlap
- `.resources/hermes/gateway/restart_loop_guard.py` — classification→action linkage (concept)

## Deliverables

- Durable grants honored end-to-end (prompt → store → auto-decision → revoke); silent downgrade gone
- Working clarify: agent call → user answer → tool result; `RequiresInteraction` tools serviceable
- Functional misfire grace + catch-up policy + overlap guard + lifetime `RunLimit`
- Unhealthy verdicts visible (doctor + status) and actionable (gated escalation)
- Native toolsets + CLI/HTTP/UDS + web surfaces + `skills/agh/` updates
- Every test case in `## Tests` implemented and passing **(REQUIRED)**

## Tests

No `_tests.md` for this suite — concrete inline cases below. Coverage target ≥80% on touched
packages; all Go tests under `-race`.

### Approvals

- Unit (`internal/daemon` tool-approval bridge suite — extend):
  - [ ] `allow_always` selection → grant persisted; identical follow-up call auto-approves with no
        prompt round-trip
  - [ ] `reject_always` → follow-up auto-denies with `ReasonApprovalRequired`-class deterministic
        error
  - [ ] Grant in workspace A never matches an identical call in workspace B (isolation)
  - [ ] Revoked grant → next call prompts again; `allow_once` still never persists
  - [ ] Mode ceiling: `deny-all` still denies even with a stored allow grant (store widens, never
        overrides mode semantics per ADR-001)
- Integration (`make test-integration`):
  - [ ] Full ACP round-trip: prompt → always → restart daemon → grant survives and auto-approves
  - [ ] `agh__tool_approvals_list/revoke` + CLI `-o json` + HTTP/UDS parity return identical data
- E2E (`make test-e2e-web`):
  - [ ] Web grants view lists a stored grant and revoke updates the daemon state

### Clarify

- Unit (tools + daemon channel suites — extend):
  - [ ] Clarify with 4 choices → answer `"2"` returns choice payload as tool result
  - [ ] >4 choices rejected with deterministic validation error
  - [ ] Timeout elapses → configured fallback outcome returned, request cleared
  - [ ] Second concurrent clarify on one session queues/deterministically rejects (no interleaved
        corruption)
  - [ ] Workspace isolation: pending question invisible to other workspaces' list endpoints
- Integration:
  - [ ] `RequiresInteraction` extension tool → clarify prompt → answer → tool completes
  - [ ] Answer via CLI and via HTTP produce identical resolution
- E2E (`make test-e2e-web`):
  - [ ] Running session shows question card via SSE; answering unblocks the turn

### Automation misfire

- Unit (`internal/automation` schedule suite — extend):
  - [ ] Downtime shorter than grace → job fires exactly once on catch-up under
        `run_once_on_catchup`
  - [ ] Downtime beyond grace with `skip_missed` → skip recorded with reason (grace-aware)
  - [ ] Prior run still executing at next fire → fire skipped, reason recorded, next cycle normal
  - [ ] `RunLimit=3` → job auto-disables after third run; `FireLimit` untouched by lifetime logic
  - [ ] CHECK migration: fresh DB + upgrade/reopen + recorded-prefix (L-021 trio)
- Integration:
  - [ ] Restart-with-downtime across the real scheduler loop fires exactly once (no double-fire
        under claim CAS)
- E2E: N/A — job form fields covered by web unit tests and W7 QA scenario

### Health verdicts

- Unit (`internal/subprocess` + doctor suites — extend):
  - [ ] Unhealthy verdict → doctor probe reports degraded with the verdict reason
  - [ ] Persistent unhealthy beyond threshold + gate enabled → exactly one escalation event
        (idempotent under repeated ticks)
  - [ ] Gate disabled (`0`) → surfacing only, no escalation
- Integration:
  - [ ] Kill the mock agent process mid-session → status shows degradation; escalation lands in
        `needs_attention` with correlation keys
- E2E (`make test-e2e-runtime`):
  - [ ] Dead subprocess session appears as needs-attention in CLI status output

## Success Criteria

- Every assigned test case implemented and passing
- Coverage ≥80% on touched packages
- Re-prompt count for a granted tool is exactly zero (D1 closed)
- Approval prompt frequency measurably unchanged by clarify (no new prompt class)
- D2 reproduction (downtime → silent skip) now fires once with an audit trail
- D5 reproduction (verdict with no consumer) now observable end-to-end
- Migration trio green for grant store + automation CHECK/`RunLimit`
- Web screenshots via `eng-ui-screenshot` cited for grants view and clarify answer UI
- Observability coverage-matrix includes the new escalation event
