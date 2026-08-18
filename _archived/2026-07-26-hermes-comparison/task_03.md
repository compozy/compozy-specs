---
status: pending
title: "Session and memory: compaction, checkpoints, batch ops, lifecycle affordances"
type: backend
complexity: high
---

# Task 3: Session and memory: compaction, checkpoints, batch ops, lifecycle affordances

## Overview

Pressure-triggered compaction, durable checkpoint summaries with anti-thrash, atomic batch memory
plus prefix-cache audit, and session lifecycle affordances (auto-title, interrupt salvage, file-
mutation verifier). Closes D3 (compaction scaffolding with no production caller) and the memory/
session UX gaps that make long-running work truthful and continuous across sessions.

**Authoritative sources:** this suite has no `_prd.md` / `_user_stories.md` / `_tests.md`.
`_techspec.md` and `adrs/adr-003.md` are authoritative. Concrete test cases are inline below
(exact input/condition/expected).

Merges former tasks 08+09+10+11. **Depends on task_02** (edge `02→03`): resume-replay assembly and
pruner must exist before checkpoint injection and compact→resume proofs.

**CRITICAL subtask order (B-301):** checkpoint summaries (ex-09) BEFORE compaction archive
(ex-08); inject checkpoints into task_02's replay path; then batch memory; then lifecycle.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
### Checkpoint summaries (ex-09) — MUST land before compaction archive
1. MUST add an iteratively *updated* checkpoint-summary memory record produced at AGH-controlled
   boundaries — the compaction boundary (consumed by the pre-compact step — B-301) plus session
   end / dream — injected as a prompt block by the assembler. The record's update API must support
   span-scoped updates so compaction can cover an archived span before flagging it.
2. MUST adopt Hermes' heading schema (Historical Task Snapshot-first structure) in the dream
   consolidation prompt.
3. MUST be append/update-only and revertible via the existing decision WAL; raw transcript and
   Markdown source-of-truth are never overwritten.
4. MUST add dream anti-thrash guards: skip when a run yields sub-threshold improvement + failure
   cooldown; consolidation never destroys candidates on session/provider failure.
5. MUST carry `workspace_id`/scope through store/recall/snapshot paths (no cross-workspace leak).
6. MUST wire checkpoint-summary injection into the resume-replay assembly built by task_02: the
   replay history block and the assembler-injected summary block are the same path, so a resumed
   session receives the summary in place of archived spans (Safety Invariant 8; B-301).

### Compaction (ex-08 / D3)
7. MUST trigger compaction off context-window pressure derived from ACP usage updates, routing
   through the existing `HookContextPreCompact`/`HookContextPostCompact`.
8. MUST port Hermes' guards: per-turn attempt cap, failure cooldown, and non-destructive archiving
   (new `archived` flag on events via an append-only migration — events stay queryable).
9. MUST NEVER rewrite the live subprocess window (ADR-003 §3 rejects) — compaction acts on AGH's
   replay/injection surfaces only.
10. MUST add `[session.compaction]` config keys with explicit zero-value semantics (0 disables).
11. MUST fire `contract.MemoryProvider.OnPreCompress` at the new boundary (ending its inert status).
12. MUST update the durable checkpoint-summary at the pre-compact step to cover the span being
    archived BEFORE setting `archived=1` — ordering guarantee: a crash leaves either raw events or
    a covering summary, never neither (Safety Invariant 8; B-301).

### Batch memory + prefix-cache (ex-10)
13. MUST give `agh__memory_*` an atomic `operations[]` batch with `old_text` substring match and an
    at-capacity consolidate-and-add contract — partial application impossible.
14. MUST keep AGH's byte/token budget unit (no competing char cap).
15. MUST audit every volatile injected prompt segment (memory snapshot, recall augmenter,
    situational augmenters, timestamps) for per-turn churn and fix violations to the invariant:
    byte-identical prefix across turns within a session unless a real memory write refreshes it.
16. MUST update `skills/agh/` memory guidance for the new batch schema (digest co-ship).

### Lifecycle affordances (ex-11)
17. MUST auto-title sessions after the first assistant response via a bounded spawn (memory-
    extractor idiom — never a daemon-side LLM client), writing to `SessionMeta` + emitting a
    session-info event; config-gated.
18. MUST salvage interrupted prompts: on `interrupt` followed by `steer`, compose
    `"<cancelled> + correction: <steer>"` as the effective input (queue semantics preserved).
19. MUST compute failed-write-without-later-success from persisted tool-call events and surface a
    transcript verifier marker (Truthful UI).
20. MUST NOT regress the persisted, generation-fenced input queue.
</requirements>

## Subtasks (CRITICAL order: checkpoints → compaction → batch → lifecycle)

- [ ] 3.1 Checkpoint-summary record type + update flow at session-end/dream boundaries, with a
      span-scoped update API for the compaction boundary (B-301).
- [ ] 3.2 Heading schema in `ConsolidationPrompt`; assembler injection block, including injection
      into task_02's resume-replay assembly (same path).
- [ ] 3.3 Anti-thrash: improvement threshold + failure cooldown in dream runtime; revertibility
      through decision WAL + workspace-scope proof tests.
- [ ] 3.4 Pressure signal from ACP usage updates (threshold config).
- [ ] 3.5 Compaction driver + guards (attempt cap, cooldown) invoking the existing hook pair;
      `archived` event flag migration + queryability preserved.
- [ ] 3.6 Summary-before-archive ordering: pre-compact extraction updates the checkpoint-summary
      for the span, THEN the `archived` flag lands (B-301); `[session.compaction]` config + docs;
      SD-001 supervisor re-evaluation note.
- [ ] 3.7 Batch schema on native memory tools + controller/decision-WAL atomicity; at-capacity
      consolidate-and-retry single-call contract.
- [ ] 3.8 Prefix audit: instrument assembled prefix hashing across turns; fix churn violations;
      skill docs + schema digests co-ship.
- [ ] 3.9 Auto-title spawn + `SessionMeta` write + event + config key.
- [ ] 3.10 Interrupt salvage composition in the busy-input queue.
- [ ] 3.11 Verifier marker computation + transcript/SSE surfacing + web rendering of title +
      verifier marker (screenshot).

## Implementation Details

See `_techspec.md` §3.2 / §3.3 / ADR-003. Migration appends at tail (L-021) for `archived` flag
in `internal/store/sessiondb/`. Extension hook contract already defines compaction hooks
(`internal/extension/contract/sdk.go:595-598`) — they become live. Keep each affordance in its
own file per the file-cap discipline.

### Relevant Files

- `internal/memory/dream.go` / `dream_v2.go` / `consolidation/runtime.go` — guards + boundary
- `internal/memory/assembler.go` — injection
- `internal/memory/decision.go` — revertibility
- `internal/session/manager_hooks.go` — `runContextCompaction` has only test callers today
- `internal/store/sessiondb/` — `archived` flag migration
- `internal/config/` — `[session.compaction]`
- `internal/memory/provider/local/provider.go` — `OnPreCompress` becomes live
- `internal/daemon/native_tools.go` — memory tool schemas
- `internal/memory/controller/` — atomic batch application
- `internal/daemon/prompt_input_composite.go` — prompt block ordering / stability
- `internal/session/manager_busy_input.go` — interrupt salvage
- `internal/daemon/memory_runtime.go` — bounded-spawn idiom for auto-title
- `internal/transcript/` — verifier computation input

### Dependent Files

- `internal/extension/contract/sdk.go` — hook consumers (docs)
- `web/` — session list/timeline title + verifier marker
- `internal/api/contract/` — SessionMeta title field if absent
- `skills/agh/` — memory guidance update
- Task_02 resume-replay assembly — injection target for checkpoint summaries

### Related ADRs

- [ADR-003: Compaction and memory context](adrs/adr-003.md) — pressure compaction, checkpoint
  summaries, anti-thrash, batch ops, prefix-cache stability; rejects live-window rewrite

### Competitor References

- `.resources/hermes/agent/context_compressor.py` — checkpoint-summary + prune/compaction guards
- `.resources/hermes/agent/conversation_loop.py:488` — attempt cap
- `.resources/hermes/hermes_state.py:1824` — failure cooldown
- `.resources/hermes/tools/memory_tool.py` — atomic batch + substring match
- `.resources/hermes/agent/system_prompt.py:448-461` — date-only/byte-stable discipline
- Hermes auto-title/salvage/verifier per `analysis/03_analysis_sessions-lifecycle.md` §4

## Deliverables

- Durable, revertible, workspace-scoped checkpoint summaries injected across sessions and into
  resume replay
- Compaction fires under pressure with guards; scaffolding is no longer inert; summary-before-
  archive ordering holds
- Atomic batch memory ops; churn-free prompt prefix
- Titled sessions, salvageable interrupts, truthful write markers
- Every test case in `## Tests` implemented and passing **(REQUIRED)**

## Tests

No `_tests.md` for this suite — concrete inline cases below. Coverage target ≥80% on touched
packages; all Go tests under `-race`.

### Checkpoint summaries

- Unit (`internal/memory` dream/consolidation suites — extend):
  - [ ] Two consecutive session ends → one summary record updated in place (not duplicated)
  - [ ] Consolidation run with < threshold improvement → next run skipped until cooldown expiry
  - [ ] Provider failure mid-consolidation → candidates intact (non-destructive assertion)
  - [ ] Summary revert via decision WAL restores the prior summary content
  - [ ] Workspace A summary never surfaces in workspace B's assembled prompt
- Integration:
  - [ ] New session's assembled prompt contains the checkpoint block reflecting the prior
        session's fixture facts
  - [ ] Resumed session's replay assembly contains the summary block (task_02 path — the
        compact→resume loss assertion itself is owned by compaction integration below)

### Compaction

- Unit (`internal/session/manager_hooks_test.go` — extend into production-path coverage):
  - [ ] Usage above threshold → driver fires once; below → never fires
  - [ ] Second attempt within the per-turn cap window → suppressed
  - [ ] Failed compaction → cooldown honored; no event destroyed (archived flag only)
  - [ ] `0` threshold config → feature fully disabled, no hook dispatch
  - [ ] `OnPreCompress` invoked exactly once per compaction with the extraction payload
- Integration:
  - [ ] Long-transcript fixture crosses threshold → archived events excluded from replay
        projection but returned by history queries
  - [ ] Compact→resume (B-301): compaction archives a span, daemon killed BEFORE a clean session
        end, resume with a load-unsupported agent → archived span's facts reconstructed from the
        injected checkpoint-summary (no silent loss), and raw archived rows are not replayed (no
        re-inflation)
  - [ ] Summary-before-archive ordering: crash injected between summary update and archive flag →
        resume never observes an archived span without a covering summary
  - [ ] Migration trio: fresh DB / upgrade-reopen / recorded prefix
- E2E (`make test-e2e-runtime`):
  - [ ] Long-session harness scenario completes without provider context-length failure

### Batch memory + prefix-cache

- Unit (memory controller/decision + native-tool schema suites — extend):
  - [ ] Batch of add+replace+remove applies atomically; injected mid-batch failure → zero applied
  - [ ] `old_text` substring with 0 or 2+ matches → deterministic rejection naming the ambiguity
  - [ ] At-capacity add with consolidation op in the same call → succeeds in one round-trip
  - [ ] Assembled prefix hash identical across 3 turns with no memory write; differs exactly once
        after a write
- Integration:
  - [ ] Agent-driven batch via `agh__memory_*` round-trip persists and is recalled next session
- E2E: N/A — agent-facing tool semantics; W7 QA covers the operator flow

### Lifecycle affordances

- Unit (`internal/session` busy-input + lifecycle suites — extend):
  - [ ] First assistant response → exactly one title spawn; second response → none
  - [ ] interrupt→steer → composed salvage input enqueued once, generation fencing intact
  - [ ] interrupt→plain new prompt (not steer) → no salvage composition
  - [ ] Failed write event with no later success → marker present; with later success → absent
- Integration:
  - [ ] Title lands in `SessionMeta` and session list payloads across HTTP/UDS
- E2E (`make test-e2e-web`):
  - [ ] Session list shows generated title; timeline shows verifier marker on the failure fixture

## Success Criteria

- Every assigned test case implemented and passing
- Coverage ≥80% on touched packages
- "No scaffolding without a production caller" satisfied (D3 closed)
- Cross-session continuity demonstrated (facts survive session rotation via checkpoint)
- Zero destructive consolidation paths under failure injection
- Prefix-stability invariant holds on the long-session fixture
- Input-queue invariants unchanged (existing queue suite still green)
- SD-001 re-evaluation recorded in completion notes
- Web screenshot via `eng-ui-screenshot` cited for title + verifier marker
