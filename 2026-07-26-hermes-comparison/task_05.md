---
status: pending
title: "Daemon reliability: memory observability, draining, dead-entity registry, retry taxonomy"
type: backend
complexity: high
---

# Task 5: Daemon reliability: memory observability, draining, dead-entity registry, retry taxonomy

## Overview

Ops/reliability slice — memory doctor probe, first-class draining state, dead-entity registry, and
bridgesdk error taxonomy with decorrelated jitter. Makes slow leaks visible, restarts graceful,
dead externals stop hammering while remaining recoverable, and first-party outbound retries stop
synchronizing on overloaded providers.

**Authoritative sources:** this suite has no `_prd.md` / `_user_stories.md` / `_tests.md`.
`_techspec.md` and `adrs/adr-010.md` are authoritative. Concrete test cases are inline below
(exact input/condition/expected).

Merges former tasks 15+16+17+18.

**MANDATORY first subtask:** two-touch determination for the retry surface (ex-18) — if this is
the third touch in the active workstream, STOP and redesign via TechSpec; record the determination
either way before any retry code.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
### Two-touch gate (ex-18) — MUST resolve before any retry code
1. TWO-TOUCH RULE CHECK (mandatory): three retry/config surfaces already exist (`internal/retry`,
   `internal/bridgesdk`, `internal/automation`). If this change constitutes the third touch to the
   retry surface in the active workstream, the executor MUST stop and redesign toward one shared
   retry package via a new TechSpec instead of patching — record the determination either way.

### Memory observability (ex-15)
2. MUST log a structured `[memory]` line (RSS via `runtime/metrics`, heap, goroutines, uptime) at
   a configurable interval (default 5m; `0` disables), plus baseline and shutdown snapshots.
3. MUST register a `runtime.memory` doctor probe surfacing the latest snapshot via CLI/HTTP/UDS
   doctor output.
4. MUST follow goroutine ownership rules (tracked, joined at shutdown — no orphan ticker).

### Draining (ex-16)
5. MUST add a `draining` daemon state that gates new prompt/run admission while letting in-flight
   work finish, driven by `drain`/`undrain` HTTP + UDS + CLI endpoints — never a file marker.
6. MUST make the existing subprocess-group teardown the post-drain step.
7. MUST reflect `draining` in status and doctor output; agents can request/observe a drain
   (agent-manageable, SD-011).
8. IF drain intent is ever persisted across boots, MUST stamp it with a per-boot nonce (staleness
   guard) — otherwise keep it in-memory and document that a restart clears it.

### Dead-entity registry (ex-17)
9. MUST maintain a durable dead-entity set for repeatedly-probed externals (extensions, bridges,
   MCP sidecars): mark on confirmed-permanent failure, auto-clear on success.
10. MUST scope the set per workspace — one workspace's dead sidecar never suppresses another's.
11. MUST expose the set via CLI (structured output) + a doctor probe; native-tool availability
    diagnostics reflect dead status.
12. MUST keep probing semantics recoverable (a low-frequency retry lane still exists — dead ≠
    never-again).

### Retry taxonomy (ex-18) — only if two-touch determination allows proceeding
13. MUST split `overloaded` (529-class) from generic `transient` in `internal/bridgesdk/errors.go`
    with distinct default backoff.
14. MUST switch `retry.Delay` jitter to a decorrelated per-attempt seed.
15. MUST apply ONLY to first-party outbound calls — zero changes to the delegated-agent path
    (ADR-010 §7 reject).
</requirements>

## Subtasks (first: two-touch determination)

- [ ] 5.1 Two-touch determination for the retry surface (documented) — proceed or escalate to a
      consolidation TechSpec; **no retry code before this record exists**.
- [ ] 5.2 Memory sampler goroutine + config key + baseline/shutdown snapshots.
- [ ] 5.3 `runtime.memory` doctor probe + status wiring + config docs.
- [ ] 5.4 Daemon lifecycle `draining` state + admission gate seams.
- [ ] 5.5 `drain`/`undrain` endpoints (api/core shared handlers) + CLI verbs with `-o json`;
      status/doctor reflection + SSE state event; post-drain teardown ordering; docs +
      `skills/agh/`.
- [ ] 5.6 Dead-set store (durable, workspace-keyed) + mark/clear rules per surface.
- [ ] 5.7 Probe-path integration for extensions/bridges/MCP (backoff → dead → low-frequency
      retry); CLI + doctor + availability-diagnostic wiring; docs.
- [ ] 5.8 IF 5.1 allows: taxonomy split + decorrelated jitter + config defaults + docs; ELSE:
      stop and open redesign TechSpec (no patch).

## Implementation Details

See `_techspec.md` §3.5 / ADR-010 §§2–6. Composition-root wiring for sampler and drain. Detached-
lifetime discipline: draining must not cancel in-flight detached work; it only blocks new
admission. Dead-set: registry-adjacent storage; each surface consumes through an interface defined
where consumed. Retry scope is strictly AGH's own outbound calls (bridge providers, catalog
fetches).

### Relevant Files

- `internal/doctor/` — memory + dead-entity probes
- `internal/daemon/` — sampler + lifecycle state owner
- `internal/config/` — interval / drain keys
- `internal/api/core/` — drain handlers + status
- `internal/cli/` — drain/undrain + doctor verbs
- admission seams in `internal/session` / `internal/task` entry paths
- probe paths in `internal/extension/`, `internal/bridges/`, `internal/mcp/`
- `internal/bridgesdk/errors.go` — taxonomy
- `internal/retry/` — jitter primitive

### Dependent Files

- `internal/api/contract/` + TS — draining state field; doctor payload
- `web/` status header — draining indicator (truthful UI)
- `internal/tools/` availability diagnostics — dead-backed tools report unavailable-with-reason
- bridge provider call sites — classification consumers
- `skills/agh/` — drain verb documentation

### Related ADRs

- [ADR-010: Daemon reliability primitives](adrs/adr-010.md) — memory monitor (§2), drain (§3),
  dead-entity registry (§5), retry taxonomy/jitter (§6), delegated-agent path reject (§7)

### Competitor References

- `.resources/hermes/gateway/memory_monitor.py` — cadence + baseline/shutdown snapshots
- `.resources/hermes/gateway/drain_control.py` — finish in-flight, refuse new, epoch guard
- `.resources/hermes/gateway/dead_targets.py` — mark/auto-clear-on-success
- `.resources/hermes/agent/chat_completion_helpers.py` — taxonomy granularity (reference only)

## Deliverables

- Periodic structured memory telemetry + doctor probe
- Graceful drain lifecycle end-to-end
- No infinite hammering of confirmed-dead externals; recovery preserved
- Finer error classes + decorrelated jitter on first-party calls **or** a documented escalation to
  a consolidation TechSpec per the two-touch rule
- Every test case in `## Tests` implemented and passing **(REQUIRED)** for shipped paths

## Tests

No `_tests.md` for this suite — concrete inline cases below. Coverage target ≥80% on touched
packages; all Go tests under `-race`.

### Memory observability

- Unit (`internal/doctor/*_test.go` — extend):
  - [ ] Probe returns populated snapshot fields after one sampler tick
  - [ ] Interval `0` → sampler never starts; probe reports disabled deterministically
  - [ ] Shutdown joins the sampler goroutine (no leak under `-race` + goroutine count assertion)
- Integration:
  - [ ] `agh doctor -o json` (HTTP + UDS) includes `runtime.memory` with identical data
- E2E: N/A — operator observability covered by integration

### Draining

- Unit (daemon lifecycle suite + admission-gate tests in owning entry packages — extend):
  - [ ] Drain → new prompt admission rejected with deterministic reason; in-flight prompt
        completes untouched
  - [ ] Undrain → admission restored
  - [ ] Drain is idempotent (second drain call = no-op, same status)
  - [ ] Draining state visible in status payload + doctor
- Integration:
  - [ ] Drain via UDS + HTTP produce identical state; CLI `-o json` matches
  - [ ] Drain → in-flight run finishes → teardown proceeds → shutdown clean under `-race`
- E2E (`make test-e2e-runtime`):
  - [ ] Drain during active session → session completes, new session refused, undrain restores

### Dead-entity registry

- Unit (owning-surface probe suites + new dead-set store tests):
  - [ ] N consecutive confirmed-permanent failures → marked dead; probe cadence drops to the
        low-frequency lane
  - [ ] Success on a dead entity → auto-cleared, normal cadence restored
  - [ ] Workspace A dead mark invisible to workspace B's probes and listings
  - [ ] Transient failure class (timeout) never marks dead (classification boundary)
- Integration:
  - [ ] Dead MCP sidecar fixture → tool availability shows unavailable-with-reason; revive →
        available again without daemon restart
- E2E: N/A — doctor/CLI covered above; W7 QA includes a dead-sidecar scenario

### Retry taxonomy (conditional on 5.1 proceed)

- Unit (`internal/bridgesdk` + `internal/retry` suites — extend):
  - [ ] 529 response → `overloaded` class with its distinct backoff profile
  - [ ] 500 → `server_error`; connection reset → `transient` (table-driven)
  - [ ] Decorrelated jitter: successive delays are not multiplicatively correlated (statistical
        assertion over N samples with fixed seed injection)
- Integration:
  - [ ] Bridge provider fixture returning 529 then success → one retry with overloaded backoff,
        then success
- E2E: N/A — internal retry mechanics
- If 5.1 escalates to redesign: unit/integration above are N/A; deliverable is the determination
  record + redesign TechSpec stub/link instead

## Success Criteria

- Every assigned test case implemented and passing (for shipped paths)
- Coverage ≥80% on touched packages
- Two-touch determination recorded in completion notes **before** any retry diff
- Structured `[memory]` lines present on the configured cadence; baseline + shutdown snapshots
  visible in a full daemon lifecycle test
- Restart-during-active-runs is no longer abrupt (drain path)
- Admission primitives' authority unchanged (gate refuses; primitives untouched)
- Probe-log noise for a dead sidecar drops to the low-frequency lane (measured in notes)
- Auto-recovery of dead entities proven without restart
- Delegated-agent path provably untouched (no diff under `internal/acp`/session prompt paths) if
  retry code ships
