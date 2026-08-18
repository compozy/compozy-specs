---
status: completed
title: "Orchestration DX: generalized wait, session wake bridge, seven native tools"
type: backend
complexity: critical
---

# Task 4: Orchestration DX: generalized wait, session wake bridge, seven native tools

## Overview

Delivers P4: the wait service (badge predicates, epoch pinning, subscribe-then-snapshot, bounded buffers, `resume_id` gapless continuation, caps) with its route/CLI/tool surfaces; the session wake bridge (`notify_creator`, mirroring `task_wake_bridge`) with presence-aware `*bool` opt-out across every spawn surface; and the seven native tools closing the shell gap (`session_wait`, `session_spawn`, `session_stop`, `session_approve`, `session_clarify_answer`, `session_prompt_cancel`, plus task_02's `notify` already landed) — with the `prompt-cancel` CLI verb. This is the safety-primitive-heavy slice (wait registry, wake bridge, cancellation path): treat every numbered Safety Invariant as a test target.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. `WaitForBadge` MUST implement `_spec.md` Core Interfaces exactly: settled-set default, `done ⊨ idle`, epoch pinning (session id + transcript epoch), subscribe-then-snapshot registration, 64-edge buffers with deterministic `overflow`, stop/delete fan-out before cleanup, 10s `resume_id` grace, per-session cap 32, always-bounded timeouts (Safety Invariants 4–5).
2. The wait route MUST be a bounded long-poll (required `timeout_ms` ≤ 1800000; timeout is a 200 outcome; 410 on gone) and is the documented SD-010 exemption (request-coupled by design); CLI `session wait` gains `--until/--timeout/--unbounded` with exit codes 0/65/66-n/a/69/75 per `_dx.md`; `--unbounded` = transparent resume loop.
3. Native `session_wait` MUST emit activity heartbeats while blocked (SD-001; IT-020) and cap `timeout_ms` per the tool contract.
4. The session wake bridge MUST mirror `internal/daemon/task_wake_bridge.go`: typed dispatch at settle/fail/needs-you transitions, `PromptSynthetic` delivery with `context.WithoutCancel` + owned worker group, dedup by `WakeEventID` per cause-episode, suppression reasons recorded (`wake_creator_disabled`, `session_not_live`, `self_wake`, `delivery_failed`), no reaper deadlock (Safety Invariants 6–7); `SpawnWakeEvent.Detail` sanitized + 240-bound (Invariant 13).
5. `notify_creator` MUST be presence-aware on every input (`*bool` wire/tool, flag-changed CLI `--no-notify-creator`), normalized once at the governed spawn boundary (round-2 B-111).
6. The six session tools in this task MUST land in the new sibling files with the full chain (descriptor + schema + meta row + toolset + binding + workspace guard + deterministic errors); self-action denials translate to `ErrorCodeDenied` + `ReasonApprovalSelfDenied`/self-target reasons exactly like `nativeLoopApproveError`; stop is Destructive risk (approval policy applies).
7. Approve/answer MUST return `already-resolved` on races, resolve orphaned interactions through task_01's atomic path (`resolved-after-restart`/`queue-full`), and deny self-action; `prompt-cancel` (verb + tool + existing route) MUST ride the one idempotent cancellation path with exit 0/66 semantics.
8. Route-inventory parity (IT-014) MUST cover the complete program inventory (wait + presence + notify + settings/attention) on both transports.
9. Docs MUST ship in this task: `runtime-operations.md` (wait/prompt-cancel/spawn flag), `native-tools.md` (the six tools), `tasks-and-orchestration.md` (session `notify_creator`), and the site orchestration/CLI reference pages.
</requirements>

## Subtasks

- [x] 4.1 Wait service: registry, predicates, pinning, buffers, resume grace, caps, outcome/fence model.
- [x] 4.2 Wait surfaces: route (both transports + spec + parity), CLI generalization in a new `session_wait.go` CLI file (exit codes incl. new 75), native tool with heartbeats.
- [x] 4.3 Session wake bridge + `SpawnWakeNotifier` domain types + `PromptSyntheticMeta` extension + suppression audit; reaper-race hardening.
- [x] 4.4 `notify_creator` presence-aware plumbing across API/CLI/tool + central default.
- [x] 4.5 Native tools: `session_spawn`, `session_stop`, `session_approve`, `session_clarify_answer`, `session_prompt_cancel`, `session_wait` — full chain in sibling files.
- [x] 4.6 `compozy session prompt-cancel` verb + client method (`client_session_wait.go` sibling for new client calls).
- [x] 4.7 Contracts + codegen + acpmock coverage for wait/wake/tool flows.
- [x] 4.8 Docs: skill references + site pages for every surface above.

## Implementation Details

Reference `_spec.md` Part II (Core Interfaces wait/wake blocks, Key Decisions, Safety Invariants 4–7, 13, 15) and the golden path in `_dx.md`.

### Relevant Files

- `internal/cli/session.go:295-336` — current stop-only wait (poll-then-stream-then-re-read pattern); moves out to `session_wait.go` (file is 338L, must not grow).
- `internal/cli/session_command.go:11-35` — verb registration; `internal/cli/client_session_api.go` (`StreamSessionEvents`) + new `client_session_*.go` siblings.
- `internal/daemon/task_wake_bridge.go` (165L) + `internal/task/wake_dispatch.go` + `internal/task/wake.go:9-46` — the pattern to mirror (interface, ownedWorkerGroup, drain, suppression codes).
- `internal/session/spawn.go:16-62` (`SpawnOpts`, caps, validation chain) + `internal/daemon/spawn_reaper.go:253` — wake/reaper race surface.
- `internal/api/core/interfaces.go:93,110` (`PromptSynthetic`, cancel iface) + `internal/acp/prompt_synthetic_meta.go` + `internal/api/testutil/session_stub.go:389`.
- `internal/tools/builtin_ids.go:68-102` + `builtin/sessions.go` (455L — sibling `sessions_attention.go` from task_02 grows or a `sessions_orchestration.go` sibling) + `internal/daemon/native_loop_tools_support.go:139-151` (`nativeLoopApproveError` — the self-denial translation to mirror) + `internal/tools/reason.go:91` (`ReasonApprovalSelfDenied`).
- `internal/api/httpapi/session_routes.go:34` + `udsapi/session_routes.go:36` — existing prompt/cancel route the verb/tool reuse.
- `internal/session/session_wait.go` / `session_attention.go` from task_01 — the transition stream waits subscribe to.
- `internal/agentidentity/errors.go` — add the timeout exit code (75, EX_TEMPFAIL) + nothing-in-flight (66) mapping via `cliExitCode()`.

### Dependent Files

- `internal/heartbeat/` — activity emission while a native wait blocks (SD-001).
- `internal/testutil/acpmock/` + `internal/testutil/e2e` — E2E-004..008 journeys.
- `skills/compozy/references/{runtime-operations,native-tools,tasks-and-orchestration}.md`; site CLI/API reference pages.
- `web/src/generated/` — wait/spawn payload types (no web UI in this task).

### Related ADRs

- [ADR-001](adrs/adr-001.md) — `done ⊨ idle` in predicates; [ADR-005](adrs/adr-005.md) — fence/registration semantics (round-2 amendments).

### Competitor References

- `.resources/herdr/src/api/wait.rs` — `agent wait --until` settled-set default (`[idle,done,blocked]`), occupant identity pinning, and the 5s `agent_prompt_stalled` effect timeout this task's stall-free design supersedes.

### Web/Docs Impact

- `web/`: `web/src/generated/compozy-openapi.d.ts` only (no UI surface; orchestration is agent/CLI-facing).
- `packages/site`: orchestration reference pages (`wait`, `prompt-cancel`, spawn flag, the six tools); generated CLI docs re-run; `skills/compozy/references/*` listed above.
- QA impact: new scenarios — add content-addressed `untested` files for: `session wait --until/--timeout/--unbounded` journeys, wake-on-settle golden path, native approve/answer incl. post-restart resolution, `prompt-cancel` verb+tool. Flag only — task_08 walks them.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: six new tool IDs join the agent contract (registry + digests + skill docs); no manifest/hook changes beyond task_01's event (checked).
- Agent manageability: the entire task is the manageability plan — wait/spawn/stop/approve/answer/cancel natively + CLI verbs + routes with deterministic errors and structured results.
- Config lifecycle: none — checked surfaces: `internal/config/` (`notify_creator` is deliberately flag-level, no key per spec Assumptions); reason: spawn wake default documented as non-configurable in v1.

## Deliverables

- Wait service + surfaces passing the race/overflow/resume suite; wake bridge with auditable suppression; seven tools live with policy gating.
- Exit-code additions wired through `cliExitCode()`; parity inventory complete.
- acpmock E2E journeys green; docs shipped.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

- [x] UT-012..UT-023 — wait predicates, pinning, timeout math, caps, cancel cleanup
- [x] UT-027..UT-029 — spawn-wake normalize/dedup/suppression
- [x] UT-077 — registration race (subscribe-then-snapshot buffer)
- [x] UT-079 — wake-detail sanitization + bound
- [x] UT-084, UT-085 — resume gapless continuation; overflow + stop ordering
- [x] IT-007, IT-008 — wait route outcomes on both transports
- [x] IT-012, IT-013 — wake delivery/suppression; reaper race; shutdown-spawn rejection
- [x] IT-014 — full route-inventory parity
- [x] IT-017 — seven-tool chain (scoping, denials, shapes)
- [x] IT-019 — presence-aware `notify_creator` across all spawn surfaces
- [x] IT-020 — heartbeats during native wait
- [x] IT-025..IT-028 — CLI wait transport failure; stop/cancel single-path; approve/answer races; prompt-cancel semantics
- [x] IT-031 — `session_prompt_cancel` tool
- [x] E2E-004..E2E-006, E2E-008 — wait transcripts, prompt-cancel, golden-path orchestration, native wait via acpmock

Checkpoint note: E2E-004..006 and E2E-008 are authored in the canonical daemon/acpmock suite and
compile with the integration tag. Their runtime execution remains deferred to task_08 by the
accepted loop directive.

## Success Criteria

- Every assigned test case implemented and passing.
- The `_dx.md` golden path runs verbatim: spawn → wake on needs-input → native answer → settled wait → stopped, zero polling (US-016..US-023 ACs).
- No lost edges under the race suites; every suppression observable; no orphaned watchers after stop/cancel/grace-expiry.
- `make gate` green; `GOOS=windows` cross-build unaffected (no subprocess-signaling changes).
