---
status: completed
title: Domain core v3: types, invariants, persistence, config, reducers, coalescer
type: backend
complexity: critical
---

# Task 1: Domain core v3: types, invariants, persistence, config, reducers, coalescer

## Overview

Delivers the entire daemon-side tab domain in one slice: the v3 snapshot shape (floating stacks, nav stacks, pins, closed entries), its validation/normalization invariants, the discard-on-load persistence policy, the two new config keys, all new/extended reducers (group, reorder, set_active, pin, reopen, close scopes, navigate modes, stack-target open, ID reservation), and the manager-side per-client activation with the coalesced durable last-active. After this task the window manager fully models tabs; nothing is exposed publicly yet (task_02 owns the wire).

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST bump `SnapshotVersion` to 3 and add the exact domain fields from TechSpec §Data Models (`Desktop.FloatingStacks`, `Window.NavStack`, `Window.Pinned`, `Snapshot.ClosedEntries`, `ClientView.StackActive`) with the pasted signatures from §Core Interfaces — signatures are final.
2. MUST extend `ValidateSnapshot`/`NormalizeSnapshot` per Safety Invariants 1-3 and 10: floating-stack membership/size/active rules, membership-exactly-once across all containers, pinned-prefix collation, 1-member dissolution (node→leaf, floating stack→floating window at frame rect), absolute maxima only (nav 200 / closed 100) — normalize and validate stay config-free.
3. MUST implement the repository discard policy (Invariant 13 / ADR-012): version ≠ 3 or malformed decode → log + delete row + reinitialize; workspace-ID mismatch or v3 topology validation failure → hard error preserving the blob.
4. MUST add `[window_manager] nav_stack_limit` (default 50, 1..200) and `closed_entry_limit` (default 20, 1..100) with the full lifecycle (struct, defaults, validation, workspace pointer-overrides) and extend the Go shortcuts allow-list with `window.tab.new|next|previous|last|reopen|jump.1..8`.
5. MUST implement the five new commands and three extended payloads exactly per TechSpec §Core Interfaces + Reducer behavior contracts: `window.stack.group` (InsertIndex splice, pin-region clamp), `window.stack.reorder`, `window.stack.set_active`, `window.pin`, `window.reopen` (consume-on-pop, skip-live, rejoin/rebuild), `NavigateMode` push/pop/replace (pop rejects route), `CloseScope` tab/group/others/right (one ClosedEntry per command, active-neighbor rule, pinned refusal on scope ""), `window.open` stack-target + server-side ID generation with reservation across live ∪ ClosedEntries (Invariant 9).
6. MUST extend `window.move` DropCenter to floating targets and make `window.toggle_floating` on a stacked member act as ungroup.
7. MUST compose history and ClosedEntries by exclusion (Invariant 15): ClosedEntries outside history `State`; close/reopen history-eligible; `restoreStatePreservingRoutes` preserves `NavStack` with `Route`; set_active and coalescer flushes history-exempt.
8. MUST implement the activeCoalescer with the exact lifecycle from §Core Interfaces: per-workspace lock binding, `flushStackActiveLocked` for lock-holding callers, detached-context timers, tracked WaitGroup, Close cancel+flush+join, pre-durable flush failure fails the command, timer failures retain pending (Invariant 4).
9. MUST keep every new/changed production file under 500 lines using the pre-decided split in TechSpec §Architectural Boundaries (`floating_stack.go`, `closed_entries.go`, `active_coalescer.go`, one reducer file per new command).
10. MUST define the two new domain errors surfaced later as `window_manager_not_stacked` and `window_manager_window_pinned` (wire mapping lands in task_02).
</requirements>

## Subtasks

- [x] 1.1 Add v3 domain types, constants, and new-field serialization (`types.go`, new `floating_stack.go`, `closed_entries.go`)
- [x] 1.2 Extend validation and normalization with floating-stack, pin-prefix, membership, dissolution, and absolute-maxima rules
- [x] 1.3 Implement the repository discard-vs-hard-error load policy in the daemon composition root
- [x] 1.4 Add the two config keys with full lifecycle and the tab entries in the shortcuts allow-list
- [x] 1.5 Implement grouping/reorder/pin reducers (node stacks + floating stacks, shared member helpers)
- [x] 1.6 Implement close scopes with ClosedEntry pushes, active-neighbor selection, and pinned refusal
- [x] 1.7 Implement reopen with consume-on-pop, skip-live, rejoin/rebuild, and ID reservation on open
- [x] 1.8 Implement navigate modes and extend move/toggle_floating semantics
- [x] 1.9 Wire history eligibility/exemptions and the history×ClosedEntries exclusion contract
- [x] 1.10 Implement ClientView.StackActive projection/repair and the activeCoalescer lifecycle
- [x] 1.11 Implement all 59 assigned test cases in the canonical suites

## Implementation Details

Follow TechSpec §Data Models, §Core Interfaces, §Safety Invariants (1-15), and ADR-008/009/010/011/012/013. The exploration reports in the session transcript pin exact current shapes: `types.go:67-142`, `commands.go:5-25`, `tree_mutation.go:38-53/260-299`, `normalize.go:98-109`, `validate.go:213-228`, `history.go` + `service_execute.go:120-141`, `clients.go`/`client_projection.go`, `window_manager_repository.go:186-244`, `config/window_manager.go`, `shortcuts.go:104-158`.

### Relevant Files

- `internal/windowmanager/types.go` — domain types; add v3 fields + version bump
- `internal/windowmanager/validate.go` — stack rules to extend to floating stacks + maxima
- `internal/windowmanager/normalize.go` — dissolution/collation/repair extensions
- `internal/windowmanager/commands.go` — command IDs + payload structs
- `internal/windowmanager/reducer.go`, `reducer_window_open.go`, `reducer_window_close.go`, `reducer_window_navigate.go`, `reducer_window_move.go`, `reducer_window_toggle_floating.go` — dispatch + extended reducers
- `internal/windowmanager/tree_mutation.go` — DropCenter machinery reused by grouping
- `internal/windowmanager/history.go`, `service_execute.go` — eligibility, exemptions, `restoreStatePreservingRoutes`
- `internal/windowmanager/clients.go`, `client_projection.go`, `client_revision.go` — StackActive + repair
- `internal/windowmanager/service.go`, `service_helpers.go` — Execute path, locks, coalescer flush ordering
- `internal/windowmanager/config.go`, `shortcuts.go` — workspace overrides + allow-list
- `internal/daemon/window_manager_repository.go` — discard policy
- `internal/config/window_manager.go` — new keys + validation
- New: `internal/windowmanager/{floating_stack,closed_entries,active_coalescer,reducer_stack_group,reducer_stack_reorder,reducer_stack_active,reducer_window_pin,reducer_window_reopen}.go`

### Dependent Files

- `internal/windowmanager/{topology,lifecycle,layout_commands,service,service_contract,config}_test.go` — canonical suites extended in place
- `internal/daemon/window_manager_repository_test.go` — discard coverage
- `internal/config/window_manager_test.go` — key lifecycle coverage
- `internal/windowmanager/resource_codec.go`, `layout.go`, `service_layout.go` — LayoutDocument v3 round-trip (export/replace carry new fields)

### Related ADRs

- [ADR-008](adrs/adr-008.md) — stack node + FloatingStack representation
- [ADR-009](adrs/adr-009.md) — focus-driven activation, coalesced last-active
- [ADR-010](adrs/adr-010.md) — generated IDs + reservation
- [ADR-011](adrs/adr-011.md) — nav stacks via navigate modes
- [ADR-012](adrs/adr-012.md) — v3 hard cut + discard classes
- [ADR-013](adrs/adr-013.md) — reopen storage, history composition

### Web/Docs Impact

- `web/`: none in this task — wire/generated types change in task_02; web adoption in task_03. Checked: no `internal/api/contract` file is touched here.
- `packages/site`: none in this task — config reference rows land in task_04 with the docs slice; CLI docs regenerate in task_02.
- QA impact: user-visible once shipped (v3 discard resets arrangements; stacked windows become reachable). Scenario resets are owned by task_05 (qa-report): `ET-window-manager-layout-recovery`, `RT-home-usage-window-persistence`, `MS-configure-window-manager` — walked in task_06.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: no hook/tool/manifest changes in this task (dispatch mapping + tools land in task_02). Checked: `internal/hooks/*`, `internal/daemon/native_tool_*` untouched here; `window_layout` resource codec extends to v3 shapes in this task (registry validation covered by UT-037..039 assigned to this task's suite scope in `_tests.md` §Reducers: navigation).
- Agent manageability: domain commands only — public exposure (CLI/HTTP/UDS/tools) is task_02's contract; deterministic domain errors (`ErrNotStacked`, `ErrWindowPinned`) defined here.
- Config lifecycle: `nav_stack_limit` + `closed_entry_limit` structs/defaults/validation/overrides + shortcuts allow-list here; settings exposure rides existing GET/PATCH; docs rows in task_04; tests UT-140..143 here.

## Deliverables

- v3 domain model compiling with all invariants enforced by validate/normalize and covered by the canonical suites
- All five new commands + three extended payloads reducing correctly with history/hook exemptions in place
- Repository discard policy active with both discard classes and both hard-error classes
- Config keys + allow-list live end-to-end in `internal/config` with workspace overrides
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-001..UT-009 — validate/normalize v3 invariants, write-time caps, absolute maxima
- [x] UT-010..UT-019 — grouping/reorder/pin reducers (both container kinds, pin regions, insert index behaviors via UT-179)
- [x] UT-020..UT-029 — close scopes, ClosedEntry shape, reopen semantics, stack-target open
- [x] UT-030..UT-039 — navigate modes, drag-out preservation, history preservation, arrange regression, export/replace v3
- [x] UT-060..UT-066 — CAS/rebase, restart round-trip seed, StackActive projection/repair, coalescer flush triggers
- [x] UT-140..UT-143 — config defaults/validation/overrides/allow-list
- [x] UT-172..UT-176, UT-179 — close scopes bulk, ID reservation, history×reopen interleavings, coalescer races, insert-index splice
- [x] IT-001 — manager + real store full v3 round-trip
- [x] IT-002 — repository discard classes vs hard errors
- [x] IT-007 — two registered clients: per-client active over shared membership

## Success Criteria

- Every assigned test case implemented and passing
- `make gate` green on the Go lanes; race detector clean on the coalescer suites
- A stacked topology created via `layout.arrange --arrangement stack` survives close-to-one (leaf collapse), regroup, restart, and reopen with byte-stable normalize output
- Loading a seeded v2 snapshot reinitializes the workspace exactly once with the documented warning; a corrupted v3 snapshot hard-fails without row deletion
- No production file introduced or grown past 500 lines
