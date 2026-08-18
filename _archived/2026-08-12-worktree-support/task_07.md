---
status: completed
title: Web: environment surfaces and assisted exit
type: frontend
complexity: high
---

# Task 7: Web: environment surfaces and assisted exit

## Overview

Mounts the environment choice everywhere work starts and delivers the assisted-exit surface: session-create environment select, composer environment chip with materialization and the fork confirmation, session binding chip, task setup worktree policy fieldset + fan-out isolation, loop configure default + node Environment control, the worktree detail context (status strip + exit ladder via the new `SplitButton` primitive), commit/PR dialogs, streamed progress, and merged/cleanup evidence.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST follow `_uiux.md` §Component plan verbatim (names, composition, insertion points) with `eng-design` + `ui-craft` + `impeccable` active; the locked environment-mode vocabulary (`Workspace root · Inherit · Named worktree · Per-run · Directory`) is the only mode labeling.
2. MUST add the one new `@compozy/ui` primitive — `SplitButton` — with colocated story + test in the same PR (reuse gate for everything else).
3. MUST implement S7 (`SessionEnvironmentField` under Workspace, reset-on-workspace-change, absent on non-git), S8 (`SessionEnvironmentChip` via a new `environmentControl` composer slot: draft-only switching, visible materialization phases, failure keeps the draft with three exits, live-session locked → fork), S9 (`SessionWorktreeForkDialog` three-fact confirmation), S16 (`SessionWorktreeBindingChip` in the topbar status slot; exit control never mounts in the session header — locked OQ7).
4. MUST implement S10 (`TaskWorktreePolicyFields` in the Environment fieldset mirroring sandbox grammar + read row; locked-during-active-run; invalid-ref flag), S11 (fan-out isolation row with count statement + per-run result attribution), S12/S13 (loop worktree default + node Environment descriptor replacing the deleted "Working dir" FieldSpec — one control, retired-`cwd` validation error state).
5. MUST implement S6 (`WorktreeDetailHeader`, five-field `WorktreeStatusStrip` with `data-unknown` cells and read-failure blocking, `WorktreeExitControl` + `useWorktreeExitLadder` rendering the daemon plan verbatim — reasons are payload literals, PR affordances absent without capability, forge labels from the capabilities vocabulary), S14 (commit dialog with counts/± + named untracked list + honest default placeholder + agent-staged affordance; PR dialog with editable fields/template prefill/draft peer/`opened_existing`/browser tier; progress toast with tensed phases, skip reasons, one CTA), and merged/cleanup evidence rows incl. not-safe blocker suppression.
6. MUST wire per-worktree stream consumption for status/exit progress with named listeners + workspace-qualified invalidation + cleanup, and refresh triggers per the TechSpec (post-action, bound-turn end, explicit `forge=true`).
7. MUST produce the `eng-ui-screenshot` evidence bundle for every Visual Contract row.
</requirements>

## Visual Contract

Reference artboards: `docs/design/opendesign/worktree/` (iterate, never regenerate; generation states were removed from the S14 contract per B-013 — capture the amended states). All rows 1440×900, fidelity normative. Authorized differences: `DESIGN-NOTES.md` lossiness rules; divergences recorded in `review.md`.

| ID | Reference artifact + state | Implementation target + state | Viewport | Fidelity |
| --- | --- | --- | --- | --- |
| VC-01 | `worktree-session-create.html` — default (root selected) | `SessionEnvironmentField` default | 1440×900 | normative |
| VC-02 | `worktree-session-create.html` — open with ready worktrees + "New worktree…" | Field open | 1440×900 | normative |
| VC-03 | `worktree-session-create.html` — zero worktrees (root + new only) | Field empty-workspace | 1440×900 | normative |
| VC-04 | `worktree-session-create.html` — worktree selected (chip on trigger) | Field selected | 1440×900 | normative |
| VC-05 | `worktree-session-create.html` — non-git workspace (control absent) | Dialog without field | 1440×900 | normative |
| VC-06 | `worktree-session-create.html` — selected worktree went missing (error) | Field error state | 1440×900 | normative |
| VC-07 | `worktree-composer-environment.html` — chip root (§01) | `SessionEnvironmentChip` root | 1440×900 | normative |
| VC-08 | `worktree-composer-environment.html` — chip worktree (§02) | Chip worktree | 1440×900 | normative |
| VC-09 | `worktree-composer-environment.html` — chip new-worktree (§03) | Chip new | 1440×900 | normative |
| VC-10 | `worktree-composer-environment.html` — materializing phases (§04) | Chip pending + `WorktreePhaseSteps` | 1440×900 | normative |
| VC-11 | `worktree-composer-environment.html` — materialization failed, three exits, draft kept | Chip failure | 1440×900 | normative |
| VC-12 | `worktree-composer-environment.html` — live session locked chip → fork affordance | Chip locked | 1440×900 | normative |
| VC-13 | `worktree-composer-environment.html` — fork confirm, three facts (§05) | `SessionWorktreeForkDialog` | 1440×900 | normative |
| VC-14 | `worktree-composer-environment.html` — fork new-worktree target | Fork dialog target picker | 1440×900 | normative |
| VC-15 | `worktree-composer-environment.html` — fork blocked mid-turn | Fork blocked state | 1440×900 | normative |
| VC-16 | `worktree-detail-exit.html` — session header binding chip bound (§S16) | `SessionWorktreeBindingChip` | 1440×900 | normative |
| VC-17 | `worktree-detail-exit.html` — binding chip missing + Resolve… | Binding chip missing | 1440×900 | normative |
| VC-18 | `worktree-task-setup.html` — mode Inherit (§01) | `TaskWorktreePolicyFields` inherit | 1440×900 | normative |
| VC-19 | `worktree-task-setup.html` — mode Workspace root | Fields none | 1440×900 | normative |
| VC-20 | `worktree-task-setup.html` — mode Named worktree + ref picker open (§02) | Fields ref | 1440×900 | normative |
| VC-21 | `worktree-task-setup.html` — mode Per-run (§03) | Fields per-run | 1440×900 | normative |
| VC-22 | `worktree-task-setup.html` — invalid/removed ref flagged (§04) | Fields invalid ref | 1440×900 | normative |
| VC-23 | `worktree-task-setup.html` — locked-during-active-run banner | Sheet locked | 1440×900 | normative |
| VC-24 | `worktree-task-setup.html` — read view with policy row (§06) | `TaskWorktreePolicyReadRow` | 1440×900 | normative |
| VC-25 | `worktree-fanout.html` — isolation off (default) | `TaskFanOutIsolationRow` off | 1440×900 | normative |
| VC-26 | `worktree-fanout.html` — isolation on + count statement | Row on | 1440×900 | normative |
| VC-27 | `worktree-fanout.html` — per-run results w/ attributed failure (run id + danger line) | `TaskFanOutRunResults` | 1440×900 | normative |
| VC-28 | `worktree-loop-config.html` — loop default unset (§01) | `LoopWorktreeSection` unset | 1440×900 | normative |
| VC-29 | `worktree-loop-config.html` — loop default named worktree (§02) | Section ref | 1440×900 | normative |
| VC-30 | `worktree-loop-config.html` — loop default per-run (§03) | Section per-run | 1440×900 | normative |
| VC-31 | `worktree-loop-config.html` — node Environment root (§04) | Node descriptor root | 1440×900 | normative |
| VC-32 | `worktree-loop-config.html` — node worktree picked (§05) | Node descriptor worktree | 1440×900 | normative |
| VC-33 | `worktree-loop-config.html` — node Directory mode + templated value (§06) | Node descriptor directory | 1440×900 | normative |
| VC-34 | `worktree-loop-config.html` — effective-environment readout on node card (§07) | Node card `env` row | 1440×900 | normative |
| VC-35 | `worktree-loop-config.html` — retired-`cwd` validation error (B-009) | Node error state | 1440×900 | normative |
| VC-36 | `worktree-detail-exit.html` — hero: strip + primary Commit (awaiting-input agent) | `WorktreeDetailHeader` + strip + control | 1440×900 | normative |
| VC-37 | `worktree-detail-exit.html` — ladder Commit & push position | Exit control state | 1440×900 | normative |
| VC-38 | `worktree-detail-exit.html` — ladder Push (incl. publish variant) | Exit control state | 1440×900 | normative |
| VC-39 | `worktree-detail-exit.html` — ladder Open PR | Exit control state | 1440×900 | normative |
| VC-40 | `worktree-detail-exit.html` — ladder View PR + PR pill | Exit control state | 1440×900 | normative |
| VC-41 | `worktree-detail-exit.html` — menu with per-row blocked reasons (§02 literals) | Exit menu | 1440×900 | normative |
| VC-42 | `worktree-detail-exit.html` — paused-while-session-running (§03) | Control paused | 1440×900 | normative |
| VC-43 | `worktree-detail-exit.html` — status-read-failed blocks whole control, unknowns as em-dash (§04) | Strip + control blocked | 1440×900 | normative |
| VC-44 | `worktree-detail-exit.html` — diverged/behind blocked w/ strip (§05) | Control diverged | 1440×900 | normative |
| VC-45 | `worktree-detail-exit.html` — no-remote (commit only) | Control no-remote | 1440×900 | normative |
| VC-46 | `worktree-detail-exit.html` — zero-credential: PR cell/rows absent, browser path present | Strip + control zero-cred | 1440×900 | normative |
| VC-47 | `worktree-commit-sheet.html` — scope counts/± + named untracked list (B-011) | `WorktreeCommitDialog` scope | 1440×900 | normative |
| VC-48 | `worktree-commit-sheet.html` — honest default-message placeholder | Commit dialog empty message | 1440×900 | normative |
| VC-49 | `worktree-commit-sheet.html` — agent-staged prompt affordance | Commit dialog staged path | 1440×900 | normative |
| VC-50 | `worktree-commit-sheet.html` — nothing-to-commit | Commit dialog empty | 1440×900 | normative |
| VC-51 | `worktree-commit-sheet.html` — hook failure output block | `WorktreeExitProgress` hook stderr after the commit dialog closes | 1440×900 | normative |
| VC-52 | `worktree-pr-sheet.html` — base resolved header + editable title/body + template prefill | `WorktreePrDialog` default | 1440×900 | normative |
| VC-53 | `worktree-pr-sheet.html` — draft as peer row | PR dialog draft row | 1440×900 | normative |
| VC-54 | `worktree-pr-sheet.html` — `opened_existing` → single View row | PR dialog existing | 1440×900 | normative |
| VC-55 | `worktree-pr-sheet.html` — zero-credential: browser row only, inputs absent | PR dialog zero-cred | 1440×900 | normative |
| VC-56 | `worktree-pr-sheet.html` — template-ambiguous quiet degrade | PR dialog ambiguous | 1440×900 | normative |
| VC-57 | `worktree-pr-sheet.html` — no-changes blocked | PR dialog blocked | 1440×900 | normative |
| VC-58 | `worktree-exit-progress.html` — phases announced up front | `WorktreeExitProgress` start | 1440×900 | normative |
| VC-59 | `worktree-exit-progress.html` — advancing w/ hook output stream | Progress mid | 1440×900 | normative |
| VC-60 | `worktree-exit-progress.html` — success w/ tensed labels + skip reason + one CTA | Progress success | 1440×900 | normative |
| VC-61 | `worktree-exit-progress.html` — mid-chain failure attributed to its step | Progress failure | 1440×900 | normative |
| VC-62 | `worktree-merged-cleanup.html` — forge merged indicator + Clean up | `WorktreeMergedEvidence` merged | 1440×900 | normative |
| VC-63 | `worktree-merged-cleanup.html` — closed-without-merge flip (badge flips, never disappears) | Evidence closed | 1440×900 | normative |
| VC-64 | `worktree-merged-cleanup.html` — local safe-to-clean evidence stated | Evidence local | 1440×900 | normative |
| VC-65 | `worktree-merged-cleanup.html` — remote-exists downgrade (info tone) | Evidence downgrade | 1440×900 | normative |
| VC-66 | `worktree-merged-cleanup.html` — not-safe blocker (Clean up suppressed) | Evidence blocked | 1440×900 | normative |
| VC-67 | `worktree-merged-cleanup.html` — stale remote marker | Evidence stale | 1440×900 | normative |

Evidence for each row: `.compozy/tasks/worktree-support/evidence/visual/task_07/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 7.1 `SplitButton` primitive in `packages/ui` (story + test)
- [x] 7.2 S7 session-create environment field + store reset wiring
- [x] 7.3 S8 composer chip + `environmentControl` slot + materialization/failure flows; S9 fork dialog + `/worktree` wiring
- [x] 7.4 S16 binding chip in the topbar status slot
- [x] 7.5 S10 task policy fieldset + read row (patch-primitive mutation path); S11 fan-out row + results
- [x] 7.6 S12/S13 loop default section + node Environment descriptor (delete "Working dir" FieldSpec) + retired-cwd error state
- [x] 7.7 S6 detail header + status strip + exit ladder hook + menu reasons
- [x] 7.8 S14 commit/PR dialogs + progress toast; merged/cleanup evidence rows
- [x] 7.9 Per-worktree stream consumption + refresh triggers
- [x] 7.10 Visual-contract capture bundles VC-01..VC-67
- [x] 7.11 Playwright: E2E-008..E2E-015, E2E-017

Execution note: the QA tail executed the browser journeys and captured and validated every
VC-01..VC-67 bundle at 1440×900. VC-51 targets `WorktreeExitProgress`, the production owner of
terminal hook output after the commit dialog closes.

## Implementation Details

Insertion points with line anchors: `_uiux.md` §New domain components (session/tasks/loops tables) + §Existing components to MODIFY. Exit reasons/labels render payload literals from the daemon plan — the web never re-derives ladder logic.

### Relevant Files

- `web/src/components/assistant-ui/session-composer.tsx:63,203-206` — `environmentControl` slot home
- `web/src/systems/session/components/{session-create-advanced-section.tsx,session-prompt-runtime-selector.tsx}` + `stores/session-create-store.ts` + `hooks/use-session-create-dialog.ts` — S7/S8 seams
- `web/src/systems/session/hooks/use-session-topbar-slot.tsx:149-215` — S16 mount
- `web/src/systems/tasks/components/{task-setup-form.tsx,task-setup-profile-view.tsx,task-fan-out-dialog.tsx}` — S10/S11 seams
- `web/src/systems/loops/lib/{loop-node-fields.ts,loop-node-goal-fields.ts,loop-node-schema-types.ts}` + `components/editor/loop-editor-field.tsx` + `components/configure/loop-configure-dialog.tsx` — S12/S13 seams
- `packages/ui/src/` (`buttonGroupVariants`, `DropdownMenu*`, `Stepper`, `Pill`, `StatusDot`, `MonoId`, `Eyebrow`, `ConfirmDialog`, `Dialog`, `toast`) — composition inventory
- Task-06 worktree data layer + vocabulary components

### Dependent Files

- `web/src/systems/workspace` (data layer from task 06) — exit/status hooks extend it
- `web/e2e/` — new Playwright specs
- `packages/ui/src/index.ts` — `SplitButton` export

### Related ADRs

- [ADR-003](adrs/adr-003.md) — surface set; [ADR-004](adrs/adr-004.md) — exit composition (amended); [ADR-009](adrs/adr-009.md) — policy surfaces; [ADR-010](adrs/adr-010.md) — capabilities-driven vocabulary/affordances

### Competitor References

- `.resources/t3code/apps/web/src/components/BranchToolbarEnvModeSelector.tsx:31-133` (composer environment selector)
- `.resources/synara/apps/web/src/components/GitActionsControl.logic.ts:123-184` (client action resolution — replaced here by daemon-plan rendering; useful for state inventory)

### Web/Docs Impact

- `web/`: everything above + MSW fixtures + Storybook stories; this task completes the S1-S16 surface map with task 06.
- `packages/site`: none here (task 08; checked: no site changes).
- QA impact: new scenarios — add content-addressed `untested` files for composer materialization, fork, task-policy UI, loop environment UI, exit ladder + commit/PR + progress, merged/cleanup; flag only — walk in task 10.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: renders `forge/capabilities` vocabulary and affordance absence — no new extension surfaces (checked).
- Agent manageability: none — web-only; agent paths exist in tasks 02-05.
- Config lifecycle: none (checked: no config keys).

## Deliverables

- All S6-S14 + S16 surfaces live, composed per the component plan
- `SplitButton` shipped in `@compozy/ui` with story + test
- Visual-contract evidence bundles for all 67 rows
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-130 — exit ladder hook renders plan verbatim
- [x] UT-132, UT-133 — session-create field; composer chip draft/pending/failure/locked
- [x] UT-142 — `SplitButton` story + test (canonical `packages/ui` suite)
- [x] E2E-008–E2E-015 — session env, composer materialization, fork, task setup, fan-out, loop env, exit surface, merged/cleanup
- [x] E2E-017 — zero-credential absence contract

## Success Criteria

- Every assigned test case implemented and passing; `make gate` green (web lanes)
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
- One environment control per agent-executing loop node — the old "Working dir" field is gone
- Exit reasons on screen are byte-equal to daemon plan literals
