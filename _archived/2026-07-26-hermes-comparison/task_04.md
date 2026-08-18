---
status: pending
title: "Security boundary: shared redaction, SECURITY.md, and daemon lifecycle block"
type: backend
complexity: critical
---

# Task 4: Security boundary: shared redaction, SECURITY.md, and daemon lifecycle block

## Overview

Default-on secret redaction across output/logs/SSE, a truthful `SECURITY.md` trust-model doc, and
a creation-time block on agent-authored daemon lifecycle controls. Closes G2 (cross-surface secret
leak), G6 (undocumented non-isolation of `BackendLocal`), and the SIGTERM-loop class where an agent
can schedule `agh daemon restart` against its own supervisor.

**Authoritative sources:** this suite has no `_prd.md` / `_user_stories.md` / `_tests.md`.
`_techspec.md` and `adrs/adr-005.md`, `adr-001.md`, `adr-010.md` §1 are authoritative. Concrete
test cases are inline below (exact input/condition/expected).

Merges former tasks 12+13+14. Note: `SECURITY.md` is docs work inside this backend-typed slice
(same model routing as the former `docs` type — see `_tasks.md` type→model table). Downstream:
task_07 depends on the lifecycle guard (edge `04→07`).

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
### Shared redaction (ex-12 / G2)
1. MUST create `internal/redact` applied to agent-visible output, logs, and SSE/event payloads;
   default-on; enable flag snapshotted at process start (tamper-resistant).
2. MUST seed with a provider-prefix table adapted from Hermes and preserve the existing
   `claim_token`/vault redaction invariants (`internal/CLAUDE.md` Security Invariants).
3. MUST keep diagnostics redaction consistent (one boundary, no drift between the two).
4. MUST measure and bound hot-path overhead (redaction sits on streaming paths).
5. MUST bound the heuristic engine per Safety Invariants 10-11: free-text/agent-visible content
   fields only; the structured correlation/hash envelope (`claim_token_hash`, session/run ids,
   fingerprints, idempotency keys) is never a candidate on ANY seam. On the logger seam the
   heuristic applies only to the message body; structured slog attribute values go through the
   `RedactLogAttrs` field-aware allow-list so log correlation keys survive intact (B-002).
6. MUST redact content fields BEFORE durable append (N-402): the canonical `runtime.db` ledger AND
   the per-session `events.db` store the redacted form — persist-raw/redact-on-egress is forbidden.
   `events.db` is in the redaction boundary; its history-query egress (and by extension task_02's
   resume replay) must never surface a raw heuristic-class secret.

### SECURITY.md (ex-13 / G6)
7. MUST state which layers are boundaries vs heuristics; document explicitly that `BackendLocal`
   provides no isolation.
8. MUST document the approval model (three modes + durable grants per ADR-001) and the redaction
   boundary truthfully — runtime truth wins over aspiration (COPY.md claim standards).
9. MUST fold in the existing security invariants (claim_token redaction, provider auth boundary
   L-015, symlink hardening) as reader-facing statements, without duplicating `internal/CLAUDE.md`
   normative text.
10. MUST record the declined-for-now items honestly (no command-content classifier; local vault +
    env only) so the doc never overclaims.

### Daemon lifecycle block (ex-14)
11. MUST reject, at creation time, agent-authored schedules/tool calls containing AGH-daemon
    lifecycle commands (`agh daemon stop|restart`, service-manager ops on the agh unit,
    `pkill … agh` class) with a deterministic error naming the blocked class.
12. MUST enforce at the creation seam in `internal/automation` so CLI and the native-tool path
    (`agh__automation_jobs_create`) are both covered by one implementation.
13. MUST use command-shaped regex matching (not prose scanning) to avoid false positives on jobs
    whose *text* merely mentions the daemon.
14. MUST document the blocked class in `skills/agh/` so agents learn the boundary.
15. Operator-initiated (non-agent) creation MAY bypass with an explicit flag — decide and document.
</requirements>

## Subtasks (order: redact package → SECURITY.md → lifecycle guard)

- [ ] 4.1 `internal/redact` package (patterns, snapshot flag, structured-value walker).
- [ ] 4.2 Seams: logger, SSE/event emission, agent-visible tool results; redact-before-durable-
      append on `runtime.db` + `events.db`.
- [ ] 4.3 Diagnostics unification (single source of patterns); benchmark + config docs; leak-path
      tests.
- [ ] 4.4 Draft repo-root `SECURITY.md` documenting shipped redaction + trust model; cross-link
      from `packages/site` + `skills/agh/`; COPY.md claim-standards pass.
- [ ] 4.5 Audit current automation create-path validation (confirm the gap; record findings).
- [ ] 4.6 Lifecycle guard at the creation seam + deterministic error contract; actor-awareness +
      optional operator bypass; docs + skill note.

## Implementation Details

See `_techspec.md` §3.4 / ADR-005 (redaction + SECURITY.md), §3.5 / ADR-010 §1 (lifecycle guard).
Wire redaction at `internal/daemon` seams (SD-008), not by importing redact from every package ad
hoc. High regression risk — hence `critical`. Sequence SECURITY.md after redaction so the doc
describes shipped behavior. One lifecycle-guard implementation covers CLI + native tool; expose
the blocked-class check so extension-created jobs inherit it.

### Relevant Files

- `internal/redact/` — new package
- `internal/logger/`, `internal/sse/`, `internal/observe/` — seams
- `internal/store/sessiondb/` — `events.db` append + history-query egress (N-402)
- `internal/diagnostics/item.go` — unification (only redaction boundary found today)
- `SECURITY.md` (new, repo root)
- `packages/site/` — trust-model docs page
- `internal/automation/` — creation validation seam
- `internal/agentidentity/` — actor classification (consumed)
- `internal/sandbox/local/provider.go` — host subprocess, no isolation (doc evidence)

### Dependent Files

- `internal/daemon/` — composition wiring
- `internal/daemon/native_tools.go` — create-tool error surfacing
- `skills/agh/` — redaction/trust/lifecycle blocked-class notes
- `web/` — no change expected for redaction (payloads arrive redacted)
- Task_07 suggestion accept path — must pass the lifecycle guard

### Related ADRs

- [ADR-005: Shared redaction and trust-model documentation](adrs/adr-005.md) — `internal/redact`
  boundary; SECURITY.md structure; declined classifier
- [ADR-001: Durable approval grants and clarify channel](adrs/adr-001.md) — approval model
  documented in SECURITY.md
- [ADR-010: Daemon reliability primitives](adrs/adr-010.md) §1 — creation-time lifecycle guard

### Competitor References

- `.resources/hermes/agent/redact.py:71-113` — provider-prefix table + import-time flag snapshot
- `.resources/hermes/SECURITY.md:32-135` — boundary-vs-heuristic structure
- `.resources/hermes/cron/lifecycle_guard.py:48-66` — command-shaped regex model

## Deliverables

- One redaction boundary covering output/logs/SSE with docs; leak-path scan green
- Repo-root `SECURITY.md` + site page; every capability claim traces to code or an ADR
- Creation-time lifecycle guard covering CLI + native tool paths
- Every test case in `## Tests` implemented and passing **(REQUIRED)**

## Tests

No `_tests.md` for this suite — concrete inline cases below. Coverage target ≥80% on touched
packages (runtime paths); all Go tests under `-race`. Docs paths: site build + link check.

### Shared redaction

- Unit (new `internal/redact/*_test.go` + logger/SSE suite extensions):
  - [ ] Each seeded provider prefix (table-driven) → redacted in plain text and inside JSON values
  - [ ] `agh_claim_*` tokens redacted (existing invariant composes, not duplicated)
  - [ ] Flag disabled at boot → snapshot honored; flipping config at runtime does NOT re-enable
        (tamper resistance)
  - [ ] Non-secret content byte-identical through the boundary (no false-positive corruption on a
        code-heavy fixture)
  - [ ] Log-seam correlation-key survival: a log record carrying `claim_token_hash` (64-hex),
        `session_id`/`run_id` (UUID hex) attributes passes with those attribute values intact —
        the entropy heuristic never rewrites them (B-002)
  - [ ] Event/SSE envelope correlation/hash fields survive `RedactJSON` unchanged (B-003)
- Integration:
  - [ ] Session emitting a fixture secret → SSE stream, event store, and log sink all contain the
        redacted form and never the raw value
  - [ ] Persist-before-append ordering: durable rows (`runtime.db` AND `events.db`) hold the
        redacted form; a history query never returns the raw secret (N-402)
- E2E (`make test-e2e-runtime`):
  - [ ] Harness greps captured logs/streams for planted secrets → zero hits

### SECURITY.md

- Unit / Integration / E2E: N/A — prose artifact; link integrity owned by site build/link-check
  (stronger existing gate). Docs-only portions exempt from coverage floor per verify exception.
- [ ] Site build + link check pass; `BackendLocal` non-isolation statement present and unambiguous
- [ ] Claim-standards review: no overclaims vs declined-items list (COPY.md)

### Daemon lifecycle block

- Unit (`internal/automation` creation/validation suite — extend):
  - [ ] `agh daemon restart` payload (agent actor) → rejected with the deterministic reason
  - [ ] `pkill -f agh` and service-manager variants (table-driven) → rejected
  - [ ] Prose mention ("check whether agh daemon restart happened") in a non-command field →
        accepted (false-positive guard)
  - [ ] Operator actor with bypass flag (if adopted) → accepted + audit event
- Integration:
  - [ ] `agh__automation_jobs_create` with a lifecycle command → tool error round-trip; nothing
        persisted
- E2E: N/A — creation-time validation covered by unit/integration; W7 QA exercises operator flow

## Success Criteria

- Every assigned test case implemented and passing
- Coverage ≥80% on touched packages (runtime paths)
- Leak-path scan green on all three surfaces (output/logs/SSE)
- Streaming-path overhead within the benchmark bound recorded in completion notes
- `SECURITY.md` exists, renders on the site, and every capability claim traces to code or an ADR
- The SIGTERM-loop class is structurally impossible via agent-created schedules
- Audit findings from create-path audit recorded in completion notes
