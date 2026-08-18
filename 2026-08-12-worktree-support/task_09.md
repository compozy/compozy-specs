---
status: completed
title: QA Plan and Session Charters
type: qa-report
complexity: high
---

# Task 9: QA Plan and Session Charters

## Overview

Plans the program's verification pass over the living `docs/qa/` tree: mints/updates content-addressed scenario files for every user-visible worktree behavior shipped by tasks 01-08, updates journey flowcharts, and selects this cycle's session charters — mapping regression hot spots from the TechSpec's safety invariants and the peer-review hardening into a targeted tier plus one adjacent canary journey.

<critical>ALWAYS READ _techspec.md, every ADR, and every per-task memory file before planning.</critical>

<requirements>
1. MUST activate the `qa-report` skill with `qa-docs-path=docs/qa` (bootstrap the tree if absent).
2. MUST express coverage as scenario `entry_points` on journey-derived rows — every public surface touched by tasks 01-08: `compozy worktree` verbs (incl. exit verbs + `session new --worktree/--new-worktree` + `task profile set-worktree`), HTTP/UDS worktree + profile-patch routes, streams, native tools (`compozy__worktree_*`, `compozy__task_worktree_policy_set`, changed session/task/loop tool inputs), web surfaces S1-S16, `config.toml` keys (`[worktrees]`, `default_worktree_mode`), the `worktree` hook family, and the `forge.provider`/GitHub-extension path with its zero-credential tier.
3. MUST consolidate the per-task QA-impact flags from tasks 02-07 into content-addressed `untested` scenario files (dedup same-behavior adds; content-addressed ids; no shared counters).
4. MUST map regression hot spots from `_techspec.md` Safety Invariants 1-20 and ADR-006/007/009/011 (data-loss surfaces: removal two-step, reconcile-never-cascade, per-run rollback, branch reclamation, containment) into the charter selection — targeted tier + one adjacent canary journey (workspace lifecycle).
5. MUST update `docs/qa/journeys/` flowcharts for the new worktree journeys (create→work→exit→remove; adopt; per-run task; fork) and write this cycle's charters in `docs/qa/charters/`.
</requirements>

## Subtasks

- [x] 9.1 Inventory user-visible behaviors from tasks 01-08 diffs + QA-impact flags; dedup into scenario ids
- [x] 9.2 Mint/update `docs/qa/scenarios/*.md` rows (`untested`) with entry_points across CLI/API/web/tools/config
- [x] 9.3 Update `docs/qa/journeys/` with the worktree journeys
- [x] 9.4 Write `docs/qa/charters/` for this cycle (targeted tier + canary journey), hot-spot-weighted
- [x] 9.5 Verify registry hygiene (no dup scenarios, content-addressed ids, `state.csv` untouched as generated output)

## Implementation Details

Operates on the committed `docs/qa/` tree only (scenarios/journeys/charters/bugs/reports); never per-round `qa/` trees. Scenario granularity follows the `qa-execution` walk contract.

### Relevant Files

- `docs/qa/` tree (scenarios/, journeys/, charters/, bugs/, reports/)
- `_techspec.md` §Safety Invariants + §Web/Docs Impact (QA tracker paragraph)
- Tasks 02-07 `### Web/Docs Impact → QA impact` lines — the flags to consolidate

### Dependent Files

- `docs/qa/state.csv` — generated view only (never hand-edited)

### Related ADRs

- [ADR-006](adrs/adr-006.md), [ADR-007](adrs/adr-007.md), [ADR-009](adrs/adr-009.md), [ADR-011](adrs/adr-011.md) — the safety surfaces the charters weight

### Web/Docs Impact

- `web/` / `packages/site`: none — QA planning artifacts only (checked: writes confined to `docs/qa/`).
- QA impact: this task creates the scenario files the program's flags demand; walking them is task 10.

### Extensibility / Agent Manageability / Config Lifecycle

- none — planning artifacts only; checked surfaces: no runtime, contract, tool, or config change.

## Deliverables

- Scenario files (content-addressed, `untested`) covering every flagged behavior
- Updated journeys + this cycle's charters
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED — none assigned; planning task)**

## Tests

No `_tests.md` IDs assigned — this task produces QA planning artifacts; execution and verdicts are task 10.

## Success Criteria

- Every QA-impact flag from tasks 02-07 resolves to exactly one scenario file (no dups, no orphans)
- Charters name the safety hot spots they target and the canary journey
- `docs/qa/` tree passes the `qa-report` skill's structural checks

## Completion Note

- Consolidated every Task 02-08 QA flag into 30 canonical Worktree scenarios; four new
  content-addressed scenarios close the config/bootstrap, hook/event, Forge credential, and
  reconcile/branch-safety gaps.
- Updated the two owning journeys and added six targeted charters. The adjacent canary reuses
  `CH-add-workspace-from-root` / `J-add-workspace-by-browsing`.
- Task 10 owns every scenario walk and verdict; all Worktree scenarios intentionally remain
  `untested` until that isolated execution phase.
- The closing full gate also surfaced accumulated lint debt and two inventory gaps from Tasks
  01-08. The remediation preserves public payloads while making optional worktree state nil-safe,
  splitting oversized flows, and registering presentation metadata for all shipped Worktree tools.

Compozy Impact Audit:

- Native tools: added presentation metadata for the four shipped `compozy__worktree_*` tools and
  `compozy__task_worktree_policy_set`; tool IDs, descriptors, schemas, digests, risks, gates, and
  availability remain unchanged and are assigned to scenario entry points.
- Extensibility and hooks: kept the flat hook JSON contract while making optional session runtime
  bindings nil-safe; checked `forge.provider`, the bundled GitHub provider, the five Worktree hooks,
  canonical events, streams, and config lifecycle against the targeted scenarios and charters.
- Workspace data isolation: preserved workspace/session/task/run ownership and persisted JSON/SQL
  shapes while making optional worktree snapshots nil-safe; the plan walks workspace-qualified
  CLI/HTTP/UDS/native reads, mutations, streams, caches, filters, and events for leakage.
- Official Compozy skill: registered the already-routed `references/worktrees.md` file in the
  bundled reference contract; no tool ID, CLI path, hook, capability, memory, network, or task
  semantic changed.
