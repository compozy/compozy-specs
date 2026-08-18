---
status: pending
title: "Context and replay: transcript pruner, event-log resume, tool-result offload"
type: backend
complexity: high
---

# Task 2: Context and replay: transcript pruner, event-log resume, tool-result offload

## Overview

D3-adjacent context path — a no-LLM transcript pruner, resume via event-log replay (not bare
`session/new`), and overflow tool-result offload to artifacts. AGH already persists the data to
fix silent context loss on resume and unrecoverable truncation past `MaxResultBytes`; this slice
makes those paths production-real without mutating the raw event store.

**Authoritative sources:** this suite has no `_prd.md` / `_user_stories.md` / `_tests.md`.
`_techspec.md` and `adrs/adr-002.md`, `adr-003.md` §4 are authoritative. Concrete test cases are
inline below (exact input/condition/expected).

Merges former tasks 03+04+06. Downstream: task_03 depends on this slice (edge `02→03`).

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
### Transcript pruner (ex-03)
1. MUST implement pruning in `internal/transcript` as a deterministic, LLM-free pass: 1-line
   summaries for old tool results, dedup of repeated results, oversized-argument truncation.
2. MUST NOT drop or reorder any conversational (user/assistant) message.
3. MUST be consumable by memory extraction (`internal/memory/extractor/`) and by resume replay.
4. MUST preserve the raw persisted events untouched (prune is a projection, never a mutation).

### Event-log resume (ex-04 / D4)
5. MUST replay the persisted `events.db` transcript (assembled by `internal/transcript`, pruned)
   into the fresh subprocess when `session/load` is unsupported or fails with resource-missing.
6. MUST emit a transcript marker ("context rebuilt from log") so UI/CLI truthfully show
   degraded-resume (SD-007).
7. MUST keep replay session-scoped — no cross-workspace or cross-session content.
8. MUST re-evaluate against the SD-001 activity supervisor (touches `manager_*.go` +
   `acp/client.go`) and record the outcome in the completion notes.
9. MUST classify stale ACP session ids as fresh-start fallback, never propagate.

### Tool-result offload (ex-06 / D6)
10. MUST write the full content to an `ArtifactRef` on `MaxResultBytes` overflow and return a
    preview + page-back pointer instead of dropping.
11. MUST provide an agent-usable read-back path for the artifact (existing read surface or a
    documented tool path).
12. MUST co-ship the result-payload contract change (OpenAPI + TS + E2E mocks).
13. MUST scope artifacts per workspace and clean up under a retention policy.
</requirements>

## Subtasks (order: pruner → replay → offload)

- [ ] 2.1 Pruner pass in `internal/transcript` (own file per the one-responsibility rule) with
      documented thresholds.
- [ ] 2.2 Consumption seams: extractor opt-in + replay opt-in (flag/param, not behavior change for
      existing callers).
- [ ] 2.3 Replay assembly path: pruned transcript projection → prime-prompt/history block.
- [ ] 2.4 Resume-path wiring: try load → on unsupported/resource-missing → replay → mark;
      classify stale ACP session ids as fresh-start.
- [ ] 2.5 Transcript marker + SSE/UI/CLI surfacing of degraded-resume; SD-001 supervisor
      re-evaluation note + heartbeat behavior during replay.
- [ ] 2.6 Overflow branch: artifact write + preview + page-back pointer; delete drop-on-overflow.
- [ ] 2.7 Read-back path + retention (count/size/age config keys) + contract/codegen co-ship +
      web result viewer handling of artifact-backed results.

## Implementation Details

See `_techspec.md` §3.2 (resume), §3.3 (pruner), §3.1 (offload) / ADR-002 / ADR-003 §4. Replay
content must pass through the pruner. Watch Manager goroutine/WaitGroup invariants
(`internal/CLAUDE.md` Concurrency). Keep files under the 500-line cap — pruner and offload logic
each in their own file. Delete target: drop-on-overflow branch (`_techspec.md` §4).

### Relevant Files

- `internal/transcript/` — new pruner file + assembly integration
- `internal/session/manager_lifecycle.go` — load-failure falls to `session/new` today (no replay)
- `internal/acp/client.go` — `session/load` negotiation / `IsLoadSessionResourceMissing`
- `internal/tools/result_limit.go` — truncation drops content; no offload branch
- artifact store under `$AGH_HOME` (workspace-keyed)

### Dependent Files

- `internal/memory/extractor/` — consumes pruned projection
- `web/` — degraded-resume marker; result viewer artifact preview/pointer
- `internal/api/contract/` + TS — result payload gains artifact ref
- E2E runtime harness mock agent — must model load-unsupported (L-007 co-ship)
- Task 03 checkpoint-summary injection — consumes this resume-replay assembly path

### Related ADRs

- [ADR-002: Event-log resume replay](adrs/adr-002.md) — fallback replay when `session/load`
  unsupported/missing; degraded-resume marker
- [ADR-003: Compaction and memory context](adrs/adr-003.md) §4 — no-LLM pruner as projection;
  prefix-cache / batch memory context for downstream task_03

### Competitor References

- `.resources/hermes/agent/context_compressor.py:1179` — prune semantics
- Hermes always-replay posture (`analysis/03_analysis_sessions-lifecycle.md` §3) — concept
- Hermes overflow spill-to-disk + preview (`analysis/02_analysis_tools-toolsets.md` §4)

## Deliverables

- Deterministic pruner with documented thresholds; raw event store byte-identical before/after
- Resume never silently drops context when AGH holds a transcript; degraded-resume marker visible
- No tool result silently lost past the byte cap; drop-on-overflow deleted
- Every test case in `## Tests` implemented and passing **(REQUIRED)**

## Tests

No `_tests.md` for this suite — concrete inline cases below. Coverage target ≥80% on touched
packages; all Go tests under `-race`.

### Pruner

- Unit (`internal/transcript/*_test.go` — extend assembly suite):
  - [ ] Transcript with 10 tool results → old ones summarized to 1 line, latest N preserved
        verbatim per threshold
  - [ ] Duplicate identical tool results → deduped to one + count marker
  - [ ] Oversized tool argument (> threshold) → truncated with explicit ellipsis marker
  - [ ] Zero user/assistant messages dropped or reordered (property-style assertion over a
        generated transcript)
  - [ ] Pruned projection is stable/deterministic across two runs on identical input
- Integration:
  - [ ] Extraction over a pruned projection produces the same memory candidates as unpruned for a
        fixture where prune only touches tool noise (no signal loss)
- E2E: N/A — internal projection; exercised through resume lane below

### Resume replay

- Unit (`internal/session/manager_*_test.go` lifecycle suite — extend):
  - [ ] Agent advertises no `session/load` → replay path chosen, marker event appended
  - [ ] `session/load` returns resource-missing → replay path chosen (not bare `session/new`)
  - [ ] `session/load` succeeds → NO replay (fallback only)
  - [ ] Replay content contains only this session's events (cross-session fixture proves isolation)
- Integration (mock ACP agent):
  - [ ] Stop daemon mid-conversation → restart → resume with load-unsupported agent → next prompt
        answers with knowledge of pre-restart turns
  - [ ] Mock agent asserts received replay block equals pruned transcript fixture
- E2E (`make test-e2e-runtime`):
  - [ ] Full resume on load-unsupported harness agent shows "context rebuilt from log" marker and
        a coherent continued conversation

### Tool-result offload

- Unit (`internal/tools` result-limit suite — extend):
  - [ ] Result exactly at cap → passes through unmodified, no artifact
  - [ ] Result 10× cap → artifact written, preview returned, pointer resolves to full content
  - [ ] Artifact write failure → deterministic error surfaced, truncated preview still returned
        (fail-open on the money path, never silent loss of the error)
  - [ ] Workspace A cannot read workspace B's artifact ref
- Integration:
  - [ ] `agh__logs` oversized fixture: call → overflow → page-back read returns full content
        byte-identical
- E2E (`make test-e2e-web`):
  - [ ] Web result viewer renders preview + opens full artifact content

## Success Criteria

- Every assigned test case implemented and passing
- Coverage ≥80% on touched packages
- Measured token reduction on a long-session fixture (report % in completion notes)
- Raw event store byte-identical before/after prune runs
- Zero silent-context-loss resumes for transcript-holding sessions (D4 closed)
- D6 reproduction (oversized result) fully recoverable; retention enforced at cap in a fixture
- SD-001 re-evaluation recorded in completion notes
