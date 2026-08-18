---
status: pending
title: P10 bell composition (S9, post-herdr seam)
type: frontend
complexity: medium
---

# Task 9: P10 bell composition (S9, post-herdr seam)

## Overview

Composes loop requests into the merged herdr attention bell through the named seam only: `use-os-attention.ts` gains a loops request source (per-workspace `aggregates.pending` + rows), the bell's Needs-you section renders loop-request rows following the task-approval-row pattern (workspace label, kind + loop + age, jump lands on the run page's request form), and the badge/title count adds the loops-owned projection to the session `needs_you` count. Session-attention contracts stay untouched (ADR-006). The loop-runs list gains its needs-you indicator.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Only the composition seam MAY change: `web/src/systems/os` composition modules (`use-os-attention.ts`, the bell row model) — never the `attention-summary` endpoint/consumption, session badge derivation, `session_attention_changed` handling, or the notifier (ADR-006; verify against the merged post-herdr file state).
2. The count MUST compose two daemon-owned projections (sessions `needs_you` + loops `aggregates.pending`) with per-workspace error isolation; never a row-page count; a stale loops source contributes zero and never fires notifications (herdr staleness rule inherited).
3. Rows MUST follow the merged bell grammar (`herdr.css`; needs-you = danger in bell contexts) and leave the section on resolution/cancel/expiry (no dismiss affordances — inbox-drain behavior).
4. The jump MUST land on the run page's request form (S1), switching workspace when needed, mirroring the land-on-session behavior.
5. The loop-runs list needs-you indicator binds to pending requests from the same projection.
</requirements>

## Visual Contract

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | -------- | ---------------------------------- |
| VC-01 | `docs/design/opendesign/graph-eng/graph-eng-bell-requests.html` (iterates `docs/design/opendesign/herdr-parity/herdr-parity-bell.html` §01 on `herdr.css`) — Needs-you with a loop-request row | merged bell popover, loop-request fixture | 1440×900 | normative | Only the loop-request row kind is new; every other bell element defers to the merged herdr implementation (ADR-006) |
| VC-02 | same — near-expiry row + composed badge count | bell + menubar badge fixtures | 1440×900 | normative | None |

Evidence: `.compozy/tasks/graph-eng/evidence/visual/task_09/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [ ] 9.1 Design iteration: `graph-eng-bell-requests.html` iterating the herdr bell artboard (never a parallel bell design)
- [ ] 9.2 Loops attention source in `use-os-attention.ts` (per-workspace `aggregates.pending` + pending-request rows, staleness-isolated)
- [ ] 9.3 Bell row rendering + jump-to-request behavior + composed badge/title count
- [ ] 9.4 Loop-runs list needs-you indicator
- [ ] 9.5 Playwright journey + `eng-ui-screenshot` bundles; run `make gate` then `make gate-full` (workstream close)

## Implementation Details

Reference `_uiux.md` S9, ADR-006, and the merged post-herdr state of `web/src/systems/os`.

### Relevant Files

- `web/src/systems/os/` composition modules as merged post-herdr: `use-os-attention.ts` (source composition), the bell sections component (herdr `AttentionBellSections` per their `_uiux.md`), `os-menubar` badge
- `web/src/systems/loops/adapters/` requests adapter + `requestsByWorkspace` keys (from task_08)
- `docs/design/opendesign/herdr-parity/herdr-parity-bell.html` + `herdr.css` — the visual grammar to iterate

### Dependent Files

- `web/src/systems/loops/components/run-page/loop-run-needs-you-card.tsx` — jump target (task_08)
- Loop-runs list components — indicator

### Related ADRs

- [ADR-006: Bell integration rides the herdr-parity attention pipeline](adrs/adr-006.md) — the seam, count composition, contract non-modification

### References

- `docs/design/opendesign/herdr-parity/herdr-parity-bell.html` §01 + `herdr.css` — the bell grammar this surface iterates (never regenerate; needs-you = danger in bell contexts)
- `_worktrees/herdr-parity/.compozy/tasks/herdr-parity/adrs/adr-005.md` (as merged) — the attention pipeline contracts this task must not modify
- No `.resources/` reference applies — the bell composition derives from ADR-006 and the merged herdr implementation (checked: no competitor material was drawn on for S9)

## Web/Docs Impact

- `web/`: this task. `packages/site`: one paragraph in the loops operating docs pointing at the bell behavior (co-ship).
- QA impact: UI-bearing → add an `untested` bell scenario (pending request appears, jump lands, resolution drains) in `docs/qa/scenarios/`; walk before completion (browser).

## Extensibility / Agent Manageability / Config Lifecycle

- none — checked: composition-only frontend change; counts already daemon-owned; no config.

## Deliverables

- Loop requests visible and jumpable in the merged bell with an honestly composed count; list indicator; evidence bundles
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**
- Every Visual Contract row has a durable passing `eng-ui-screenshot` evidence bundle **(REQUIRED)**

## Tests

- [ ] E2E-030 — bell shows the loop-request row (workspace label, jump lands on the request form); badge equals sessions `needs_you` + `aggregates.pending`

## Success Criteria

- Every assigned test case implemented and passing; both VC rows `PASS`
- `git diff` over `web/src/systems/os` touches only the composition seam files (session-attention contracts untouched — reviewed explicitly)
- `make gate-full` passes (workstream close for the implementation chain)
