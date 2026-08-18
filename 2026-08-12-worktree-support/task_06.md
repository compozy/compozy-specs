---
status: completed
title: "Web: navigation, lifecycle dialogs, worktree data layer"
type: frontend
complexity: high
---

# Task 6: Web: navigation, lifecycle dialogs, worktree data layer

## Overview

Delivers the web foundation for worktrees: the data layer in the workspace system (adapters, query keys/options, hooks, catalog-stream consumption, list reconciliation), the shared row/state vocabulary components, nested navigation across all three workspace-listing surfaces, the create and adopt dialogs, removal/missing-resolution dialogs, and worktree-aware selection scoping (store v3). Everything task 07 mounts reuses this vocabulary.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST follow `_uiux.md` §Component plan verbatim — component names, composition from `@compozy/ui`, insertion points, and the reuse gate (no shadowed primitives); activate `eng-design` + `ui-craft` + `impeccable` (+ dimension skills as scoped) and run through the designer-agent execution mode for design-system work.
2. MUST implement the workspace-system data layer: `adapters/worktree-api.ts` (typed errors + AbortSignal), `workspaceKeys.worktrees(id)`, `worktreesListOptions`, `hooks/use-worktrees.ts`, `lib/worktree-list-reconciliation.ts`; lists server-scoped per workspace, never client-filtered across workspaces.
3. MUST consume the catalog stream with named `addEventListener` handlers, workspace-qualified invalidation keys, and cleanup on unmount/disconnect (UT-141 contract).
4. MUST implement the S5 vocabulary components (`WorktreeStateChip`, `WorktreeStateDot`, `WorktreeSignals`, `WorktreeRow`, `WorktreeNestList` + aggregate/overflow/inert-reason/adopt affordances) with the locked rules: chip only when state ≠ ready, NULL-unknown → absent/em-dash (never `+0 −0`), stale stamping, detached "pinned at <sha>", locked nest sort + truncate-at-5, adopted-only counts.
5. MUST nest worktrees under their parent in all three surfaces (S1 command switcher, S2 menubar menu + compound chip, S3 overview) reading one query — no divergence; keyboard traversal follows the visual tree; discovered rows selectable as the adoption gesture.
6. MUST implement `WorktreeCreateDialog` (name-first, generated-name placeholder, live `branch → path` preview, the three refusals incl. "Select that worktree instead", cancel-during-submit) wired to create + create-cancel routes, and `WorktreeAdoptDialog` (confirm naming the validation + bootstrap-not-re-run; refusal naming the metadata cause).
7. MUST implement the removal dialog set (clean / bound-idle / bound-running refusal / dirty refusal with risk rows / force with re-quantified loss / remote-downgrade note) and `WorktreeMissingResolutionDialog` (dismiss / it's back, history-preserved copy, idempotent outcome).
8. MUST extend selection: `active-workspace-store` v3 (`compozy:active-workspace:v2` → `v3`, v2 state discarded) with worktree context; selecting a worktree scopes session/task views (worktree filter param) and the menubar chip; removed-selection falls back to parent with notice.
9. MUST produce the `eng-ui-screenshot` visual-contract evidence bundle for every Visual Contract row — implementation-only captures are invalid.
</requirements>

## Visual Contract

Reference artboards: `docs/design/opendesign/worktree/` (iterate, never regenerate). All rows viewport 1440×900, fidelity normative unless noted. Authorized differences: design-reference lossiness rules (`DESIGN-NOTES.md` locked facts; runtime truth + `COPY.md` own content/copy; `@compozy/ui` owns component identity) — record each divergence as an authorized delta in `review.md`.

| ID | Reference artifact + state | Implementation target + state | Viewport | Fidelity |
| --- | --- | --- | --- | --- |
| VC-01 | `worktree-row-states.html` — state chips inventory (§01) | Storybook `WorktreeStateChip` all-states story | 1440×900 | normative |
| VC-02 | `worktree-row-states.html` — signals inventory (§02: dirty/ahead-behind/agent/origin/setup-flag/merged/stale) | Storybook `WorktreeSignals` story | 1440×900 | normative |
| VC-03 | `worktree-row-states.html` — row ready+dirty+running (§03) | Storybook `WorktreeRow` fixture | 1440×900 | normative |
| VC-04 | `worktree-row-states.html` — discovered · pending · missing · error rows (§04) | Storybook `WorktreeRow` state fixtures | 1440×900 | normative |
| VC-05 | `worktree-row-states.html` — nest dots + detached "pinned at <sha>" (§05) | Storybook `WorktreeStateDot` + detached row | 1440×900 | normative |
| VC-06 | `worktree-nav-switcher.html` — workspace without git (no affordance) | Switcher with non-git workspace fixture | 1440×900 | normative |
| VC-07 | `worktree-nav-switcher.html` — zero worktrees (no group noise) | Switcher, git workspace, empty | 1440×900 | normative |
| VC-08 | `worktree-nav-switcher.html` — expanded nest, 2-3 adopted (§02) | Switcher expanded fixture | 1440×900 | normative |
| VC-09 | `worktree-nav-switcher.html` — discovered mixed in (§03) | Switcher with discovered rows | 1440×900 | normative |
| VC-10 | `worktree-nav-switcher.html` — pending + missing inert rows w/ reason lane | Switcher pending/missing fixture | 1440×900 | normative |
| VC-11 | `worktree-nav-switcher.html` — many worktrees: truncation + "All N" overflow (§07) | Switcher overflow fixture | 1440×900 | normative |
| VC-12 | `worktree-nav-switcher.html` — collapsed parent w/ aggregate activity signal | Switcher collapsed fixture | 1440×900 | normative |
| VC-13 | `worktree-nav-switcher.html` — adoption confirm (select discovered) | `WorktreeAdoptDialog` confirm | 1440×900 | normative |
| VC-14 | `worktree-nav-switcher.html` — adoption refusal (main-checkout metadata) | `WorktreeAdoptDialog` refusal | 1440×900 | normative |
| VC-15 | `worktree-menubar-menu.html` — nested menu | `WorkspaceMenu` nested fixture | 1440×900 | normative |
| VC-16 | `worktree-menubar-menu.html` — chip workspace-only | `DesktopMenubar` chip | 1440×900 | normative |
| VC-17 | `worktree-menubar-menu.html` — chip `workspace / worktree` selected | Chip with worktree selection | 1440×900 | normative |
| VC-18 | `worktree-menubar-menu.html` — missing-fallback notice (§03) | Chip fallback state | 1440×900 | normative |
| VC-19 | `worktree-overview.html` — workspace card with worktree count + expanded nest | Overview expanded fixture | 1440×900 | normative |
| VC-20 | `worktree-overview.html` — workspace without worktrees | Overview plain card | 1440×900 | normative |
| VC-21 | `worktree-overview.html` — discovered-external row keeping foreign path | Overview discovered fixture | 1440×900 | normative |
| VC-22 | `worktree-create-dialog.html` — simple (name-first + generated placeholder + preview) | `WorktreeCreateDialog` default | 1440×900 | normative |
| VC-23 | `worktree-create-dialog.html` — advanced expanded (branch/base/existing picker w/ holder badges) | Create dialog advanced | 1440×900 | normative |
| VC-24 | `worktree-create-dialog.html` — name-collision refusal | Create dialog collision | 1440×900 | normative |
| VC-25 | `worktree-create-dialog.html` — branch-held refusal + "Select that worktree instead" (§03) | Create dialog held-branch | 1440×900 | normative |
| VC-26 | `worktree-create-dialog.html` — base-ref-missing refusal | Create dialog base missing | 1440×900 | normative |
| VC-27 | `worktree-create-dialog.html` — submitting/pending (Cancel stays live) | Create dialog submitting | 1440×900 | normative |
| VC-28 | `worktree-remove-dialogs.html` — clean confirm (record-label title, branch-not-deleted line) | `WorktreeRemoveDialog` clean | 1440×900 | normative |
| VC-29 | `worktree-remove-dialogs.html` — bound-session-idle note | Remove dialog bound-idle | 1440×900 | normative |
| VC-30 | `worktree-remove-dialogs.html` — bound-session-running refusal | Remove dialog bound-running | 1440×900 | normative |
| VC-31 | `worktree-remove-dialogs.html` — dirty refusal w/ risk rows + exit-flow primary (§02) | Remove dialog dirty refusal | 1440×900 | normative |
| VC-32 | `worktree-remove-dialogs.html` — force confirm re-stating quantities | Remove dialog force | 1440×900 | normative |
| VC-33 | `worktree-remove-dialogs.html` — remote-exists downgrade note | Remove dialog downgrade | 1440×900 | normative |
| VC-34 | `worktree-remove-dialogs.html` — missing resolution (dismiss / it's back / idempotent outcome, §03) | `WorktreeMissingResolutionDialog` | 1440×900 | normative |

Evidence for each row: `.compozy/tasks/worktree-support/evidence/visual/task_06/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 6.1 Data layer (adapter, keys, options, hooks, reconciliation) + catalog-stream consumption
- [x] 6.2 S5 vocabulary components with Storybook stories + tests
- [x] 6.3 Nested nav: command switcher (partition + nest list + footer action)
- [x] 6.4 Nested nav: menubar menu + compound chip; overview cards
- [x] 6.5 Create dialog + adopt dialog wired to routes (pending/cancel flows)
- [x] 6.6 Removal dialog set + missing resolution
- [x] 6.7 Store v3 selection scoping + session/task list `worktree` filter wiring + command palette group
- [x] 6.8 Visual-contract capture bundles for VC-01..VC-34
- [x] 6.9 Playwright: E2E-005/006/007/016

Execution note: the workflow QA tail captured and validated every VC-01..VC-34
bundle at 1440×900, including reference, implementation, diff, and review artifacts.

## Implementation Details

Insertion points are enumerated with line numbers in `_uiux.md` §Existing components to MODIFY (verified 2026-08-11) — follow that table exactly. Domain components live in `web/src/systems/workspace/`; cross-system consumers import from `@/systems/workspace`.

### Relevant Files

- `_uiux.md` §Component plan + §Existing components to MODIFY — the binding contract (all paths + line anchors)
- `web/src/systems/workspace/{components/workspace-command-select.tsx,lib/home-workspace.ts,lib/query-options.ts,stores/active-workspace-store.ts}` — partition precedent, query layer, store v2
- `web/src/systems/os/components/{menubar/workspace-menu.tsx,os-menubar.tsx,desktop-menubar.tsx,os-workspaces-overview.tsx,desktop-shell.tsx,os-command-palette-results.tsx}` — nav surfaces
- `web/src/systems/session/hooks/use-session-catalog-streams.ts` — named-SSE consumption pattern to mirror
- `packages/ui/src/index.ts` + `exports/*` — primitive inventory (reuse gate)
- `docs/design/opendesign/worktree/DESIGN-NOTES.md` — locked visual facts (path convention, counts rule, sort, vocabulary)

### Dependent Files

- `web/src/generated/compozy-openapi.d.ts` — worktree types (from task 02)
- `web/src/systems/{session,tasks}` list views — `worktree` filter + meta labels
- `web/e2e/` — new Playwright specs

### Related ADRs

- [ADR-001](adrs/adr-001.md) — nesting/selection parity; [ADR-002](adrs/adr-002.md) — adopt-on-select; [ADR-006](adrs/adr-006.md) — removal safety copy

### Competitor References

- `.resources/herdr/src/ui/sidebar.rs:284-436` (render-time nesting projection, child labeling, collapsed aggregate)

### Web/Docs Impact

- `web/`: everything above — this task IS the web impact for navigation/lifecycle; MSW fixtures + Storybook stories for every new component.
- `packages/site`: none here (task 08 owns docs; checked: no site imports from web).
- QA impact: new scenarios — add content-addressed `untested` files for nested navigation/selection scoping, create/adopt via UI, removal two-step via UI, missing resolution; flag only — walk in task 10.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — presentational consumption of daemon truth (checked: no extension surfaces, no new tools).
- Agent manageability: none — web-only slice; agent paths shipped in tasks 02-05.
- Config lifecycle: none — no config keys (checked: store persist key is client-local state, not config).

## Deliverables

- Worktree data layer + vocabulary + nested nav + lifecycle dialogs live in the SPA
- Store v3 selection scoping with fallback semantics
- Visual-contract evidence bundles for all 34 rows
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-128, UT-129, UT-131, UT-134 — partition, chip/signal truthfulness, store v3, nest list rules
- [x] UT-141 — named-SSE listener contract + workspace-qualified invalidation + cleanup
- [x] E2E-005 — nested nav parity + selection scoping + keyboard traversal
- [x] E2E-006 — create dialog flows incl. refusals and pending
- [x] E2E-007 — adopt-on-select confirm + refusal
- [x] E2E-016 — dirty removal two-step + missing resolution

## Success Criteria

- Every assigned test case implemented and passing; `make gate` green (web lanes via Turbo)
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
- Both nav surfaces render from one query with no divergence; counts follow the adopted-only rule
- Zero `compozy-ui-reuse/no-shadow-ui-primitive` violations
