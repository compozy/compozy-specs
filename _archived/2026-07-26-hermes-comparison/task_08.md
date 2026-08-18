---
status: pending
title: Shadow-git workspace checkpoints for AGH-native mutating tools
type: backend
complexity: high
---

# Task 8: Shadow-git workspace checkpoints for AGH-native mutating tools

## Overview

A destructive AGH-native operation on the workspace has no undo: no snapshot, no diff, no restore.
Hermes checkpoints before mutating turns via a shadow git. The port is strictly scoped: delegated
ACP agents own their own edit paths (often in isolated worktrees) — only AGH-native mutating tools
get the checkpoint net (ADR-004). Closes the "no workspace rewind" gap for the surfaces AGH
actually owns.

**Authoritative sources:** this suite has no `_prd.md` / `_user_stories.md` / `_tests.md`.
`_techspec.md` §3.8 and `adrs/adr-004.md` are authoritative. Concrete test cases are inline below
(exact input/condition/expected).

Merges former task 23. **Standalone / parallelizable with task_07** — no graph edge between them
(both sit in W4; only shared downstream is `08→12`).

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST snapshot (shadow git under `$AGH_HOME`, workspace-keyed) before AGH-native workspace
   mutations only — never around delegated-agent edits.
2. MUST ship `agh__checkpoint_{list,diff,restore}` + CLI/HTTP/UDS parity; restore itself
   snapshots first (pre-rollback snapshot). Restore is diff-scoped (round-3 N-301): it reverts
   ONLY the paths captured in the snapshot being restored — unrelated working-tree changes
   (including delegated-agent edits made after the snapshot) survive, and overlapping paths are
   surfaced in a restore preview before applying.
3. MUST enforce retention (count/size/age config keys, full lifecycle).
4. MUST enforce worktree reconciliation mechanically (Safety Invariant 19; round-3 N-303): the
   checkpoint manager consults the worktree-isolation resolver before every `Snapshot` and skips
   native tools operating inside isolated worktrees — an enforced guard with a test, not a
   documented rule.
5. Shadow-git storage MUST respect symlink-escape hardening (security invariant).
6. MUST degrade gracefully when `git` is absent (techspec §3.8 N-006): checkpoints disabled,
   `runtime.checkpoint` doctor probe reports "git unavailable", checkpoint tools return truthful
   unavailability — never a crash.
</requirements>

## Subtasks

- [ ] 8.1 Shadow-git store + snapshot hook on native mutating-tool dispatch.
- [ ] 8.2 `agh__checkpoint_*` toolset + CLI/HTTP/UDS + codegen co-ship.
- [ ] 8.3 Retention + symlink hardening + mechanical worktree-resolver guard before `Snapshot`
      (N-303) + git-absent graceful degradation.
- [ ] 8.4 Web timeline (list/diff/restore) — screenshot; `skills/agh/` docs.

## Implementation Details

See `_techspec.md` §3.8 / ADR-004. Mutating-tool identification rides the existing descriptor
metadata (RiskClass/mutating flags) — no per-tool hardcoding. Restore blast radius is
diff-scoped (N-301); worktree skip is mechanical (N-303).

### Relevant Files

- checkpoint store package home (new file(s) near `internal/tools` dispatch or `internal/filesnap`
  / `internal/checkpoint` — decide in-code, one responsibility per file)
- `internal/tools/` dispatch — pre-mutation hook
- worktree-isolation resolver — consulted before every `Snapshot`

### Dependent Files

- `internal/daemon/` — wiring + toolset registration
- `internal/api/contract/` + TS — checkpoint payloads (codegen co-ship)
- `web/` — checkpoint timeline
- `skills/agh/` — checkpoint flow docs
- `internal/doctor/` — `runtime.checkpoint` probe (git unavailable)

### Related ADRs

- [ADR-004: Workspace checkpoint/restore scoped to AGH-native mutating tools](adrs/adr-004.md) —
  native-tool-only scope; diff-scoped restore (N-301); mechanical worktree skip (N-303);
  retention; pre-rollback snapshot

### Competitor References

- `.resources/hermes/tools/checkpoint_manager.py` — shadow-git + pre-rollback snapshot mechanics

## Deliverables

- Undo net for native mutations with list/diff/restore (diff-scoped blast radius)
- Mechanical worktree-isolation skip + symlink-escape hardening + retention
- Native toolset + CLI/HTTP/UDS + web timeline + `skills/agh/` docs
- Every test case in `## Tests` implemented and passing **(REQUIRED)**

## Tests

No `_tests.md` for this suite — concrete inline cases below. Coverage target ≥80% on touched
packages; all Go tests under `-race`.

- Unit (checkpoint store suite (new — no existing suite owns workspace-undo) + `internal/tools`
  dispatch suite extensions):
  - [ ] Native mutating tool call → snapshot exists capturing pre-state
  - [ ] Non-mutating tool → no snapshot (scope guard)
  - [ ] Restore → snapshot paths reverted; pre-rollback snapshot created first
  - [ ] Diff-scoped restore blast radius: a file edited AFTER the snapshot on a path the snapshot
        does not capture survives restore untouched; an overlapping path is listed in the restore
        preview (N-301 — delegated-edit data-loss guard)
  - [ ] Native tool inside an isolated worktree → resolver consulted, NO shadow-git snapshot
        (mechanical enforcement — N-303)
  - [ ] Retention: oldest snapshots pruned at count/size/age caps (table-driven)
  - [ ] Symlink pointing outside the workspace → snapshot refuses to follow (escape hardening)
  - [ ] Workspace A cannot list/restore workspace B checkpoints
  - [ ] `git` binary absent → doctor probe reports unavailable; tools return truthful
        unavailability (no panic)
- Integration (`make test-integration`):
  - [ ] Full cycle: mutate → list → diff shows change → restore → file content reverted
- E2E (`make test-e2e-web`):
  - [ ] Timeline renders checkpoints; restore flow works from the UI

## Success Criteria

- Every assigned test case implemented and passing
- Coverage ≥80% on touched packages
- Destructive native op is recoverable end-to-end (QA scenario reference)
- Delegated-agent paths provably not intercepted (scope tests green)
- Diff-scoped restore leaves post-snapshot unrelated edits intact (N-301)
- Worktree skip is mechanical, not documentary (N-303)
- Web screenshot via `eng-ui-screenshot` cited for the checkpoint timeline
