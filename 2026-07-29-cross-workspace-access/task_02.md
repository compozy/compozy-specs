---
status: completed
title: Seam enforcement wiring, ACP prompt at the tool seam, delete sweep
type: backend
complexity: critical
---

# Task 2: Seam enforcement wiring, ACP prompt at the tool seam, delete sweep

## Overview

Make every workspace-isolation seam consult the task_01 policy instead of denying inline — agent identity, task domain (authorizer, run-read, claims), tool dispatch + native binder, spawn, and coordination — and add the ask-UX: on a `PromptEligible` deny at the native-tool seam with a live session, the daemon prompts via the existing approval-bridge layer (`allow_once | allow_session | reject_once | reject_session`). Includes the hard-cut delete sweep (CLI pre-flight, two-string validate, inline deny branches) and the full runtime E2E matrix.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST wire the policy at exactly the TechSpec §Enforcement Seam Changes table's seams, preserving each seam's error shape (`ErrIdentityUnauthorized` + exit 77, `task.ErrPermissionDenied`, 404-masking) and honoring the NOT-seams list (`requireSessionInWorkspace`, window manager, network participation stay untouched).
- MUST implement the prompt on the same daemon dispatch layer tool approvals use (`internal/daemon/tool_approval_bridge.go` as template): daemon-computed option set, 120s default timeout, `context.WithoutCancel` + bridge deadline (L-001/SD-010); timeout, transport failure, or out-of-set answers deny and persist nothing; `*_session` answers write the task_01 consent cache.
- MUST fire the prompt ONLY from the native-tool dispatch path on `PromptEligible` denies with a live session; every other seam treats `PromptEligible` as a plain deny (invariant 8, UT-085). `deny-all` never prompts anywhere (invariant 7).
- MUST add `ReasonWorkspaceAccessDenied` to the tools reason codes for workspace-axis policy denials; `ReasonScopeMismatch` stays for identity-field conflicts (`session_id`/`agent_name` mismatches never consult the policy).
- MUST ship the denial hint copy exactly as TechSpec §Error conventions defines it.
- MUST judge cross-workspace spawns by the parent session's mode, keep both `prepareSpawn` validation phases (two audit records), and leave same-workspace spawn behavior untouched.
- MUST complete the delete sweep in the same change: pure two-string `ValidateWorkspaceAccess` (all callers updated), `validateResolutionAgainstSessionIdentity` + call sites in `internal/cli/workspace_resolution.go` (daemon becomes the sole authority; provenance notes `cross_workspace_attempt`), inline workspace-mismatch deny branches in the native binder/scope files, and the old identity denial copy. No compat shims, no dual paths.
- MUST keep existing isolation suites green with wiring-only edits plus the named deltas of invariant 1 (IT-016 regression floor) — never weaken a test to pass.
- Skills: `eng-code-guidelines`, `golang-pro`, `eng-cleanup-failure-paths` (prompt/deny failure paths), `eng-consolidate-test-suites` (isolation-suite edits stay in their canonical suites), `eng-test-conventions`, `testing-boss`.
</requirements>

## Subtasks

- [x] 2.1 Identity seam: policy-aware `ValidateWorkspaceAccess` (ctx + policy + request from the validated snapshot), `Resolve` consults `opts.WorkspaceAccess`, nil policy = deny on mismatch; thread through `internal/api/core/agent_identity.go`.
- [x] 2.2 Task seam: `WithWorkspaceAccessPolicy` option (pattern `WithWorkAdmissionChecker`); consults in the resource authorizer, run-read authorizer (mask preserved), and claim criteria normalization; operator short-circuits before the policy.
- [x] 2.3 Tool seam: binder/scope workspace-mismatch branches consult the policy via the bound-session actor; delete the inline deny branches; add `ReasonWorkspaceAccessDenied`.
- [x] 2.4 Prompt: workspace-access bridge extension on the approval-bridge layer with the daemon-computed option set and consent-cache writes; detached lifetime.
- [x] 2.5 Spawn + coordination seams: policy consults with parent-session actor (spawn, both phases) and caller scope (coordination reads; writes still operator-only).
- [x] 2.6 Delete sweep: CLI pre-flight block, old copy, dead branches — all callers updated in the same change.
- [x] 2.7 Daemon composition: inject the policy into every seam constructor from the root.
- [x] 2.8 Implement all assigned unit + integration cases; extend the canonical isolation suites in place (no tombstone tests).
- [x] 2.9 Implement the acpmock E2E matrix (mode outcomes, prompt outcomes, consent reuse, CLI/spawn parity) and the cross-seam consent journey.

## Implementation Details

Follow TechSpec §Enforcement Seam Changes (the seam table is normative), §Implementation Design "Prompt wiring"/"Actor provenance", and §Delete Targets. Each seam builds `ActorRef` from what it already validates; `Kind` is fixed by the construction site; `AgentName` is best-effort audit attribution only.

### Relevant Files

- `internal/agentidentity/identity.go:150-163` — two-string `ValidateWorkspaceAccess` (replace with a ctx+policy entry point); callers at `identity.go:53` (inside `Resolve`), `internal/cli/workspace_resolution.go:197`, `internal/api/core/agent_identity.go:120` (`resolveAgentCallerForWorkspace`, shared HTTP+UDS).
- `internal/task/task_resource_authorizer.go:31-49` (deny at :44), `run_read_authorizer.go:19-53` (denies masked as `ErrTaskRunNotFound`), `lease_claim_hooks.go:30-38` (workspace deny at :36) — task-seam deny sites. Authorizers are hardcoded in `manager_constructor.go:17-18` — `WithWorkspaceAccessPolicy` needs a new `managerOptions` field (pattern `manager_admission.go:9-21`) in a new `manager_workspace_access.go` file, wired through the constructor.
- `internal/tools/dispatch.go:111-135` (`bindCallScopeField` + `callScopeMismatchError` — session/agent axes keep `ReasonScopeMismatch`), `internal/daemon/native_workspace_input_binder.go:111-152` (`bindNativeWorkspaceField` mismatch ladder; canonical-ID deny at :149), `internal/daemon/native_tool_scope.go:30-57` (`nativeBoundSession` via `Sessions.Status`), `:149-172` (`nativeCallerWorkspaceInput`, deny at :169), `:174-182` (shared `nativeScopeMismatchError`).
- `internal/tools/reason.go` — reason-code home for `ReasonWorkspaceAccessDenied`.
- `internal/daemon/tool_approval_bridge.go` (cascade shape local → durable → prompt; option set :192-199; timeout/error mapping :108-162) + `tool_approval_boot.go:27-49` (the lazy re-asserting `sessions func() sessionPermissionRequester` provider) — the bridge template; `boot_tool_registry.go` (`WithApprovalBridge`/`WithCallInputBinder` composition points).
- `internal/session/spawn.go:124-161` (`prepareSpawn`: `validateSpawnWorkspace` consulted at :140 and :155 — both phases) + `:229-249` (deny branches); `internal/workspace/coordination_commands.go:195-210` (`authorizeCoordinationActor`: workspace compare :202-204, writes require operator :206).
- `internal/cli/workspace_resolution.go:190-198` (`validateResolutionAgainstSessionIdentity` delete target) + call sites :116-118 and :175; canonical suite `internal/cli/workspace_test.go` (`TestWorkspaceResolutionBoundary` ~:672 — no `workspace_resolution_test.go` exists; extend this one).
- `internal/daemon/daemon_mock_agents_integration_test.go` ~:600-696 — exemplar permission-prompt E2E (acpmock `RequireDriver`, `PromptSessionHTTPWithEvents` + permission callback, `ApproveSessionPermission`, prompted+decided event assertions, diagnostics capture); `internal/testutil/acpmock` (`StepKindPermission`) + `internal/testutil/e2e` harness.

### Dependent Files

- Isolation suites to EXTEND in place (no standalone regressions): `internal/agentidentity/identity_test.go` (~:20; "Should reject workspace mismatch" ~:98), `internal/task/manager_test.go` (~:6928 workspace fencing), `internal/task/lease_test.go` (~:1576), `internal/session/spawn_test.go` (~:356 policy violations; ~:608 hooks-cannot-widen), `internal/daemon/native_tools_test.go` (~:161; scope-mismatch asserts ~:329/:956; binder ~:1204), `internal/tools/dispatch_test.go` (~:580/:659/:692), `internal/api/core/network_coordination_public_test.go` (~:77), `internal/daemon/tool_approval_bridge_test.go` (unit model for the consent prompt).
- Task_04's E2E-011 asserts the deltas this task ships.

### Related ADRs

- [ADR-007](adrs/adr-007.md) — mode mapping, prompt option set, session consent, deny-all hard-deny asymmetry.
- [ADR-001](adrs/adr-001.md) — seam list, no-drift funnel (L-033), NOT-seams.
- [ADR-003](adrs/adr-003.md) (superseded) — prompt-at-tool-seam-only rationale that survives.
- [ADR-006](adrs/adr-006.md) — audit-visible enforcement posture.

### Web/Docs Impact

- `web/`: none in this task — no contract/OpenAPI change (reason codes and hints ride existing error payload shapes); web consumption unchanged until task_04.
- `packages/site`: none in this task — behavior docs land in task_05.
- QA impact: user-visible behavior changes ship here (mode-gated crossings, prompt, new reason/hint, daemon-origin CLI denial) — task_05 owns the scenario flags; this task's completion notes MUST enumerate the deltas for task_05 to flag (mode matrix scenarios, prompt outcomes, agent-CLI denial copy).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: no new hooks/tools/resources/bundles/registries/bridge-SDK/MCP surfaces; Host API ceiling unchanged (actor-kind gate); claim/spawn hook semantics unchanged (hooks narrow, never widen) — checked: hooks taxonomy, `lease_claim_hooks.go`, `spawn_hooks.go`, `host_api_workspace_binding.go`.
- Agent manageability: deterministic machine-parseable denials are the agent surface — `identity_unauthorized` + hint (exit 77, daemon-origin), `ReasonWorkspaceAccessDenied` on tools, audit events on existing `GET /api/logs`, `compozy logs --type <event-type>`, `compozy__logs`, and `compozy__observe_search`; no new verbs/routes by design (ADR-007).
- Config lifecycle: none — checked surfaces: no key changes; `permissions.*` trust-root fail-close already covers `compozy__config_set` (pre-existing behavior owned by the config suite).

## Deliverables

- All five seams consulting the injected policy with preserved error shapes; NOT-seams untouched.
- Workspace-access prompt on the approval-bridge layer with session-consent writes.
- `ReasonWorkspaceAccessDenied` + hint copy shipped; delete sweep complete (no dead branches, no old copy, CLI pre-flight gone).
- Runtime E2E matrix green against acpmock.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-021, UT-022, UT-023 — identity seam: nil-policy deny, allow pass-through, deny with hint.
- [x] UT-024, UT-025, UT-026, UT-027, UT-028 — task seam: authorizer allow/deny, run-read mask, claim normalization, operator short-circuit.
- [x] UT-029, UT-030, UT-031, UT-032, UT-033, UT-036 — tool seam: fill-if-absent unchanged, allow proceed, no-session deny reason, scope-mismatch precedence, `allow_once` semantics, timeout/out-of-set denials.
- [x] UT-083, UT-085 — session-consent writes from prompt answers; non-tool seams never prompt.
- [x] UT-037, UT-038, UT-039 — spawn allow/double-validation audits; coordination read/write asymmetry.
- [x] UT-043 — CLI resolution boundary behavioral coverage post-delete.
- [x] IT-009 — HTTP/UDS identity mapping (approve-reads 403 + hint → approve-all 200).
- [x] IT-010 — claim propagation + automation kind gate at a real seam.
- [x] IT-011 — binder + bridge + acpmock `allow_session` skips next prompt.
- [x] IT-016 — regression floor: isolation suites green with explicit `approve-reads` fixtures + named deltas only.
- [x] E2E-001, E2E-002, E2E-003, E2E-004, E2E-005 — tool-seam journeys: deny-all zero-prompt, once/session/reject outcomes, approve-all promptless.
- [x] E2E-006, E2E-007 — agent-CLI exit-77/hint parity and spawn parity across modes.
- [x] E2E-013 — cross-seam consent reuse in one session.

## Success Criteria

- Every assigned test case implemented and passing
- `make lint` + scoped `go test -race` green across touched packages; `make test-e2e-runtime` green.
- Delete-target grep proof: no references to `validateResolutionAgainstSessionIdentity`, the old identity copy, or inline workspace deny branches remain.
- Isolation suites changed only by wiring + the named invariant-1 deltas (diff audit recorded in completion notes).
