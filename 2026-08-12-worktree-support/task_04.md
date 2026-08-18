---
status: completed
title: "Environment policies: task worktree policy + loop environment"
type: backend
complexity: high
---

# Task 4: Environment policies: task worktree policy + loop environment

## Overview

Delivers both execution-environment policy consumers. Task side: `WorktreePolicy` across all 11 sandbox-precedent layers plus the dedicated patch primitive, the enqueue-transaction snapshot on `task_runs`, the lease-fenced bridge application with `MaterializeForRun`, the reuse fingerprint, and fan-out per-run isolation. Loop side: `EnvironmentSpec` on `run-agent`/`goal` with the `params.cwd` hard cut, bind-time resolution sharing the same semantics, the loop-level default, and linter reason codes.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST add `WorktreePolicy{Mode, WorktreeRef}` to `ExecutionProfile` mirroring `SandboxPolicy` at every layer: type + enum + normalizer (empty ⇒ inherit), config-backed validation (`ref ⇔ non-empty`, same-workspace attached resolution via an injected validator — `task` never imports `worktree`), whole-profile PUT unchanged, native-tool projection, config default `task.orchestration.profile.default_worktree_mode`.
2. MUST add the patch primitive `task.Service.SetWorktreePolicy` (RMW under the active-run lock) surfaced as `PATCH /tasks/:id/execution-profile/worktree` (HTTP+UDS), `compozy task profile set-worktree`, and `compozy__task_worktree_policy_set` — other blocks byte-preserved (US-014 AC-3, B-010).
3. MUST resolve the policy once in the **enqueue transaction** and write `task_runs.resolved_worktree_mode/_ref`; the session bridge consumes only the claimed run's snapshot — the live profile is never re-read post-claim (B-002).
4. MUST apply the snapshot in `applySessionWorktreePolicy` beside the sandbox policy: `none`/empty ⇒ root; `ref` ⇒ re-resolve (fail `worktree_ref_invalid`, never silent root); `per_run` ⇒ `MaterializeForRun` inside the active lease (heartbeats continue; all binding/terminal writes via `task.Service` under the claim token; stale claimant cannot bind; failure fails the run with `per_run_materialization_failed` and unwinds — no orphan; the worktree domain never sees the claim token).
5. MUST include worktree mode+resolved ref in `taskRoleProfileFingerprint`; `per_run` never reuses sessions (BR-23 structural).
6. MUST wire fan-out per-run isolation (`compozy__task_fanout_runs.worktree_per_run`, CLI/API equivalents) with per-run failure attribution, and coordinator awareness (`PromptOverlay` names each worker's worktree).
7. MUST replace loop `params.cwd` with `Environment{mode: root|worktree|per_run|directory}` on `RunAgentParams` **and** `GoalParams` — hard cut: `cwd` fails validation with a lint reason naming the one-line migration; web FieldSpec deletion is task 07; fixtures/docs mentions die here.
8. MUST resolve node environments at action bind with node-over-loop-default precedence, template-rendered `directory` contained by the workspace root, `per_run` materializing per execution instance (per fan-out branch), and `run-loop` forwarding the parent run's environment; add the loop-level `worktree` default to `LoopConfig` (configure + per-run override) and the loop tool/config surfaces (`compozy__loop_create`/`_configure`/`_run` inputs + digests).
9. MUST keep claim eligibility untouched — no claim-filter SQL changes; `ClaimNextRun` remains the only claim authority.
</requirements>

## Subtasks

- [x] 4.1 Profile block: type/normalize/validate + columns already shipped in 01 — Go-side wiring, defaults, validation options, config key
- [x] 4.2 Patch primitive + PATCH route + CLI verb + native tool + codegen co-ship
- [x] 4.3 Enqueue snapshot write + bridge consumption + `applySessionWorktreePolicy`
- [x] 4.4 `MaterializeForRun` integration under lease fencing + rollback-on-failure + run/session binding via `task.Service`
- [x] 4.5 Fingerprint extension + per_run no-reuse
- [x] 4.6 Fan-out isolation across surfaces + coordinator awareness
- [x] 4.7 Loop DSL: EnvironmentSpec, `cwd` hard cut + linter reason, goal parity
- [x] 4.8 Bind-time resolution + loop-level default + loop tool/config surfaces
- [x] 4.9 Contract/codegen artifacts for every changed tool schema and route

## Implementation Details

Follow `_techspec.md` §Key flows (Task policy application, Loop resolution), ADR-009 (amended), invariants 7, 8, 17. The 11-layer sandbox map with exact file:line lives in `analysis/06_analysis_worktree-entry-points.md` §3.

### Relevant Files

- `internal/task/profile.go:52-141,163-253` + `manager_profile.go:17-167` + `profile_policy_validation.go` — the block + PUT service + validation gates
- `internal/store/globaldb/global_db_task_claim_select.go:285-340` — claim filters (explicitly untouched; regression-pin)
- `internal/daemon/session_policy_gate.go:42-91` + `task_session_bridge.go:121-188` + `task_role_sessions.go:46-76,237-301` — bridge, application point, fingerprint
- `internal/task` enqueue path + `72_task_runs.sql` snapshot columns (shipped in 01)
- `internal/loop/dsl/node_params.go:3-25` + `types_nodes.go` + `graph.go:29-60` + `action_runagent.go:103-169` + `action_types.go:66-101,236` + `service_types.go:136-150` — DSL + bind seam + LoopConfig
- `internal/loop/linter_references.go`, `linter_actions.go`, `runtime_validation.go`, `compiler.go` — lint reason home
- `internal/config/task_orchestration.go` + `tool_surface_task.go:12-14` — config default
- `internal/tools/builtin/task_mutation_schemas.go:259-340` + `builtin_ids.go:234-238,250,306-336` — tool schemas + ids
- `internal/cli/task_commands_crud.go:208-281` + `internal/cli/loop.go:220-299` — CLI homes

### Dependent Files

- `internal/api/{contract,core,httpapi,udsapi}` — PATCH route + loop/task payload changes + OpenAPI/TS
- `internal/daemon/native_profile_tools.go:28-32` — typed-block projection
- `internal/tools/builtin/testdata/native-tool-catalog.json` — digests for 5 changed tools + 1 new
- `web/src/systems/{tasks,loops}` generated types — consumed by task 07

### Related ADRs

- [ADR-009](adrs/adr-009.md) — policy block, snapshot, lease fencing, patch primitive
- [ADR-003](adrs/adr-003.md) — surface set, per-run mode, coordinator never pinned
- [ADR-011](adrs/adr-011.md) — per-run branch composition

### Competitor References

- `.resources/synara/apps/server/src/agentGateway/creationCoordinator.ts:600-647,1000-1058` (deterministic per-run placement + uninterruptible provisioning envelope)
- `.resources/synara/apps/web/src/components/ChatView.tsx:7978-7999` (lazy first-send materialization — the interactive model rejected for queue runs, ADR-009 Alt 3)

### Web/Docs Impact

- `web/`: generated types for profile (worktree block + PATCH), loop node schema, fan-out flag — consumed by task 07 (S10-S13); no component work here.
- `packages/site`: generated CLI/API refs for `task profile set-worktree`, loop tools; config key `default_worktree_mode` documented in task 08.
- QA impact: new scenarios — add content-addressed `untested` files for per-run task isolation (policy set → run in fresh `run/` worktree → persists) and fan-out isolation via CLI; loop node environment resolution; flag only — walk in task 10.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: tool input schemas + digests change for `compozy__task_execution_profile_set`, `compozy__task_fanout_runs`, `compozy__loop_create/_configure/_run`; new `compozy__task_worktree_policy_set`; no manifest/hook changes (checked: hooks shipped in 01; loop DSL is runtime-owned).
- Agent manageability: patch primitive across PATCH/CLI/tool with deterministic errors; profile read-back identical on all surfaces; fan-out flag on CLI/API/tool.
- Config lifecycle: `task.orchestration.profile.default_worktree_mode` (struct/default/validation/matrix `Live`/example/tests UT-115); docs page in task 08.

## Deliverables

- Task policy operational end-to-end (set → enqueue snapshot → claimed run → bound worker in the right worktree), fan-out included
- Loop environments resolving with the hard-cut DSL
- Patch primitive live on all surfaces with preservation semantics
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-094–UT-103 — policy normalize/validate/apply, per_run, fingerprint, patch preservation, active-run lock
- [x] UT-104–UT-110 — loop DSL hard cut, precedence, templates, per-instance materialization, goal/non-agent rules, run-loop forwarding
- [x] UT-115 — config-backed validation options
- [x] IT-029–IT-031 — per-run runs, reuse fingerprint, fan-out attribution (real queue + git)
- [x] IT-032 — loop bind resolution against real git
- [x] IT-039 — pre_create hook deny across manual + per_run
- [x] IT-040 — snapshot authority + lease-lapse fencing
- [x] E2E-002 — per-run task journey via CLI against the daemon

## Success Criteria

- Every assigned test case implemented and passing
- `make gate` green; `make codegen-check` clean
- A post-enqueue profile edit provably never retargets a claimed run; a lapsed claimant provably never binds
- Zero occurrences of `params.cwd` anywhere in `internal/loop` or fixtures

## Completion Evidence

- Unit matrix: 4,879 tests passed across 13 affected packages.
- Integration: 720 task tests, 151 worktree tests, and 6 daemon/real-Git contract tests passed.
- `make codegen-check`: passed; generated OpenAPI, SDK, CLI reference, lifecycle, and native catalog artifacts are current.
- `git diff --check`: passed; no changed production Go file exceeds 500 lines; claim-filter SQL is unchanged.
- `make gate` is deferred to the shared QA/review tail by the accepted loop instruction; no heavy gate was run during Task 04.
- Full clause and impact mapping: `analysis/task-04-contract-parity.md`.
