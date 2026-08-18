---
status: completed
title: Session binding, containment, spawn inheritance, fork
type: backend
complexity: high
---

# Task 3: Session binding, containment, spawn inheritance, fork

## Overview

Extends the session boundary to attached worktrees: `CreateOpts.Worktree` with the resolver interface, worktree-aware containment at all four gates (create, resume ×1, sandbox ×2 + tool-host root), parent-brain preservation (config/agents/skills/memory from the parent workspace), structural spawn inheritance, the `/worktree` fork command, and the worktree fields on the session-create contract across API/CLI/native tool. This makes a session run *inside* a worktree while remaining a citizen of its parent workspace.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST define `SessionWorktreeResolver` in `internal/session` (interface where consumed) and inject the `worktree.Service`-backed implementation from `internal/daemon`; binding refs resolve only attached `ready` worktrees with deterministic errors (`worktree_not_found`/`worktree_not_ready`/`worktree_missing`) — never a silent root fallback.
2. MUST make containment binding-driven at every gate: bound session ⇒ contained by the worktree root; unbound ⇒ workspace root — in `ResolveSessionCWD`, `resumeSessionCWD`, both `sandboxRuntimeCWD` checks, and the sandbox tool-host root (`LocalRootDir` = worktree root when bound); registry-evaluated, never path-prefix heuristics.
3. MUST persist the binding in session metadata + `sessions.worktree_id`, re-resolve it at resume (missing ⇒ resume fails `worktree_missing`), and keep the memory store constructed with the **parent** workspace root and catalog `workspace_id` (one workspace brain).
4. MUST add the internal inherited `worktree_id` to the canonical spawn request: children of a bound parent resolve through the same authority with the same refusals; no user-facing choice dimension in v1 (invariant 20).
5. MUST extend the session-create contract with `worktree` ref and `new_worktree{name?}` (materialize-then-start; cancel rolls back per task-01 `CancelCreate`) across HTTP/UDS, `compozy session new --worktree <ref> | --new-worktree [name]` (mutually exclusive with `--cwd`, whose auto-register meaning is unchanged and documented), and `compozy__session_create` (schema + digest via `eng-contract-codegen-coship`).
6. MUST add the `/worktree` builtin to the command catalog: on live sessions it surfaces the fork flow — confirmation → fresh clean session bound to the target worktree; origin session untouched; unavailable mid-turn with the reason.
7. MUST add the `worktree` filter param to session list queries (contract + handlers + generated types).
8. MUST keep coordinator sessions unbound (no CWD) while `HealthySession` treats worktree-bound workers as same-workspace.
</requirements>

## Subtasks

- [x] 3.1 Resolver interface + daemon injection + `CreateOpts.Worktree` + metadata persistence + `sessions.worktree_id` writes
- [x] 3.2 Containment gates: create, resume, sandbox ×2, tool-host root — one binding-derived root source
- [x] 3.3 Memory/parent-brain preservation (store roots + catalog `workspace_id` assertions)
- [x] 3.4 Spawn inheritance (internal field, same refusals, containment)
- [x] 3.5 Session-create contract fields + CLI flags + native tool schema + codegen co-ship
- [x] 3.6 New-worktree-then-start flow with visible pending + cancel rollback
- [x] 3.7 `/worktree` builtin + fork flow (fresh session, confirmation semantics)
- [x] 3.8 Session list `worktree` filter + coordinator awareness guard

## Implementation Details

Follow `_techspec.md` §Key flows Session binding, invariants 1, 2, 7, 20. The four gates and their exact lines are mapped in the runtime exploration; change them through one shared resolution helper.

### Relevant Files

- `internal/session/manager_create_prepare.go:29,160-171` + `path_containment.go:11-41` + `manager_start.go:67-75` — the containment gates
- `internal/session/sandbox_start_opts.go:13-84` + `internal/sandbox/local/provider.go:85-133` — sandbox cwd + tool-host root
- `internal/session/manager_types.go:23-59` (CreateOpts), `manager_workspace.go:40-71`, `creation_profile_builder.go`
- `internal/session/spawn.go:39-75` — spawn request + narrowing discipline
- `internal/memory/store_scope.go:21-88` + `catalog.go` — parent-root/memory keying to preserve
- `internal/command/catalog.go:89-116` + `internal/session/command_catalog.go:13-33` — `/worktree` builtin home
- `internal/tools/builtin/sessions.go:246-274` — `session_create` schema (`additionalProperties:false` ⇒ contract change)
- `internal/cli/session_create.go:18-68` — `--cwd` disambiguation site
- `internal/daemon/coordinator_runtime_session.go:53-78` + `internal/coordinator/coordinator.go:222-282` — unbound coordinator + `HealthySession`/`PromptOverlay`

### Dependent Files

- `internal/api/contract` session payloads + OpenAPI/TS (codegen)
- `internal/session/manager_prompt_runtime.go:210-256` — resume path re-validation
- `web/src/systems/session` types (generated) — consumed by task 07

### Related ADRs

- [ADR-001](adrs/adr-001.md) — parent-brain inheritance
- [ADR-003](adrs/adr-003.md) — session surfaces, fork-not-mutate
- [ADR-005](adrs/adr-005.md) — boundary extension to attached worktrees

### Competitor References

- `.resources/synara/apps/web/src/composerSlashCommands.ts:457-467` (`/fork [local|worktree]`) and `.resources/synara/apps/web/src/hooks/useThreadWorkspaceHandoff.ts:87-94` (stop-then-move ordering — the in-place path we deliberately reject per ADR-003)
- `.resources/t3code/apps/server/src/terminal/Manager.ts:2250-2267` (launch-context drift detection — the failure mode structural binding prevents)

### Web/Docs Impact

- `web/`: generated session types gain worktree fields (consumed by task 07's S7/S8/S9 surfaces); no component work here.
- `packages/site`: session CLI reference regenerates (`session new` flags); prose in task 08.
- QA impact: new scenarios — add content-addressed `untested` files for session-in-worktree lifecycle (start bound via CLI/API, resume-after-removal refusal, fork) — flag only; walk in task 10.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — no manifest/hook/tool-surface additions beyond the `session_create` input change (checked: hooks family shipped in task 01; command catalog builtin is runtime-owned).
- Agent manageability: `compozy session new --worktree/--new-worktree`, `compozy__session_create.worktree`, session list `worktree` filter, deterministic binding errors — structured parity across all three surfaces.
- Config lifecycle: none — no new keys (checked: `internal/config` untouched by this task).

## Deliverables

- Worktree-bound sessions run, resume, and spawn children correctly contained, with parent-brain intact
- Session-create surfaces (API/CLI/tool) accept worktree targets incl. materialize-then-start
- `/worktree` fork flow live
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-088–UT-093 — containment matrix, binding refusals, resume validation, sandbox/memory roots
- [x] UT-151 — spawn inheritance + refusal
- [x] IT-026 — bound sessions against real git (isolation, context payload, resume-after-removal, child spawn)
- [x] IT-027 — sandbox tool-host containment
- [x] IT-028 — one workspace brain across worktrees
- [x] IT-041 — remove-vs-new-session-bind interleaving

## Success Criteria

- Every assigned test case implemented and passing
- `make gate` green; `make codegen-check` clean (session contract change)
- A bound session's ACP subprocess `cmd.Dir` equals the worktree root; escape attempts fail at every gate with the documented errors
- Workspace-scoped memory written in a worktree recalls identically at the root (and vice versa)
