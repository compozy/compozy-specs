# UI/UX Change Map: Native Worktree Support

Input for the design-reference phase: every UI surface this feature touches, where it lives today, what changes, which states must be designed, and the reference artboard (HTML) each surface needs. The generated references should land under `docs/design/opendesign/worktree/` and become the visual contracts the implementation tasks cite.

Companions: `_prd.md` (behavior authority), `_user_stories.md` (states come from ACs/ECs), `analysis/06_analysis_worktree-entry-points.md` + `analysis/07_analysis_assisted-exit-ui.md` (code anchors and competitor screenshots-in-prose).

## Design constraints (apply to every artboard)

- **Grammar**: `DESIGN.md` + `packages/ui/src/tokens.css` only — flat depth, no invented tokens. Reuse `@compozy/ui` primitives (`packages/ui/src/index.ts` inventory) before creating anything; new generic primitives land in `packages/ui`, domain composites in `web/src/systems/<domain>/`.
- **Signal palette is information, never decoration.** Candidate semantic mapping (final call belongs to the design pass): success `#5FBF85` = merged / bootstrap-ok · warning `#D6A647` = dirty / pending / setup-flagged · danger `#E0635A` = destructive (force-remove) / error states · info `#8E8EB5` = discovered / stale / missing.
- **Truthful UI**: unknown renders as unknown (never zero/clean); pending ≠ ready; stale is marked; forge affordances absent (not disabled) when no extension/credential supports the remote; agent activity only from real runtime state.
- **Copy**: `COPY.md` register; blocked-state reasons are functional labels (they prevent wrong actions) — no subtitles/helper prose under headings. The reusable copy inventory is in `analysis/07_analysis_assisted-exit-ui.md` §Transferable Patterns 8.
- **Nesting depth is 2** (workspace → worktrees). Keyboard order follows the visual tree.
- Execution note: the design/HTML pass runs through the `designer` agent with `eng-design` + `ui-craft` active; this document is the inventory it consumes, not a substitute for it.

## Surface map

| # | Surface | Kind | Core change | Stories |
| --- | --- | --- | --- | --- |
| S1 | Workspace command switcher | modify | nested worktree rows per workspace | US-008, US-010, US-011 |
| S2 | OS menubar workspace menu + chip | modify | nested rendering; selected-worktree chip state | US-010, US-011 |
| S3 | Workspaces overview dialog | modify | worktree groups under each workspace | US-010 |
| S4 | Worktree create dialog | new | name-first create + advanced options + refusals | US-001, US-002 |
| S5 | Worktree row + status chip vocabulary | new | shared row/chips for every list surface | US-008, US-012, US-015 |
| S6 | Worktree context (detail header) | new | status strip + exit control + lifecycle actions | US-019 – US-024, US-026 |
| S7 | Session create dialog (advanced) | modify | environment select under Workspace | US-003 |
| S8 | Session composer | modify | environment chip (draft) + pending materialization + `/worktree` | US-004, US-005 |
| S9 | Fork-to-worktree confirmation | new | explicit fork dialog for live sessions | US-005 |
| S10 | Task setup sheet | modify | Worktree policy fieldset (`inherit\|none\|ref\|per-run`) | US-014, US-015 |
| S11 | Task fan-out dialog | modify | per-run isolation checkbox + count statement | US-016 |
| S12 | Loop configure dialog | modify | loop-level worktree default | US-017 |
| S13 | Loop node inspector (run-agent / goal) | modify | single Environment control absorbing "Working dir" | US-018 |
| S14 | Commit sheet / PR sheet / progress toast | new | assisted-exit steps | US-021 – US-023 |
| S15 | Removal + missing-resolution dialogs | new | two-step removal; out-of-band resolution | US-026 – US-028 |
| S16 | Session view worktree binding | modify | bound-worktree indication + exit control mount | US-011, US-019 |

---

### S1. Workspace command switcher — nested worktrees

- **Today**: flat list, one `CommandSelectGroup heading="Workspaces"`, home workspace pinned via `splitHomeWorkspace` — `web/src/systems/workspace/components/workspace-command-select.tsx:45-221`, `web/src/systems/workspace/lib/home-workspace.ts:11-32` (the only existing partition precedent), data from `web/src/systems/workspace/lib/query-options.ts:6-24`.
- **Change**: each workspace row can expand a nested block of its worktrees (indent + rail, not box-drawing glyphs); worktree rows use the S5 vocabulary; a "New worktree" action rides the workspace row's context. Selecting a worktree closes the switcher and scopes context (US-011).
- **States to design**: workspace w/o git (no worktree affordance at all) · zero worktrees (no group noise) · expanded with 2–3 adopted · discovered entries mixed in · pending entry · missing entry · many worktrees (scroll/collapse) · collapsed parent with aggregate agent-activity signal.
- **Artboard**: `worktree-nav-switcher.html` — one screen per state above.

### S2. OS menubar workspace menu + chip

- **Today**: flat `MenubarItem` list + overview + add — `web/src/systems/os/components/menubar/workspace-menu.tsx:21-61`; workspace chip (monogram + name) in shell chrome — `web/src/systems/os/components/os-menubar.tsx:135-142`.
- **Change**: same nested projection as S1 (both surfaces read one query — they must not diverge); the menubar chip states parent + selected worktree when one is active (e.g. monogram + `name / worktree`) so the shell always says where new work will run.
- **States**: menu nested (mirror S1 core states) · chip with workspace only · chip with worktree selected · chip when selected worktree went missing (fallback notice per US-011 EC-1).
- **Artboard**: `worktree-menubar-menu.html`.

### S3. Workspaces overview dialog

- **Today**: `web/src/systems/os/components/os-workspaces-overview.tsx:23-38`.
- **Change**: each workspace card/row exposes its worktree count and expandable nested list (read-only vocabulary from S5); entry point for create.
- **States**: with/without worktrees · discovered-only repo.
- **Artboard**: `worktree-overview.html`.

### S4. Worktree create dialog (new)

- **Behavior source**: US-001/US-002; herdr's dialog flow is the interaction reference (`analysis/03_analysis_herdr-worktrees.md`, `.resources/herdr/src/ui/dialogs.rs:259-344` — create prefill, branch defaulting), t3code's env selector for vocabulary.
- **Shape**: name-first (single field, generated-name hint via placeholder); progressive advanced section: branch name (no forced prefix), base ref, "use existing branch" picker showing which branches already have worktrees; derived branch/path preview before confirm (US-001 EC-2).
- **States**: simple · advanced expanded · name collision refusal · branch-held-elsewhere refusal with "select that worktree instead" action (US-002 EC-1/EC-2) · base-ref-missing refusal · submitting/pending.
- **Artboard**: `worktree-create-dialog.html`.

### S5. Worktree row + status chip vocabulary (new, shared)

- **Purpose**: one component family used by S1/S2/S3/S6 and any list. Old-Compozy precedent for the row anatomy: `compozy-code/packages/tauri/src/renderer/systems/worktrees/components/worktree-list-item.tsx:99-242` (mono branch title, status badge, setup badge, relative activity, guarded actions).
- **Content**: name (record label, never basename) · branch (mono, copyable) · state chip (`ready · pending · discovered · missing · error`) · dirty `+ins −del` · ahead/behind (only when known) · agent activity (running/idle from runtime) · origin badge for run-created (`per-run`) · bootstrap-flagged marker (US-029 AC-3) · merged indicator (from S6 logic).
- **States sheet**: every chip alone + realistic combinations (ready+dirty+running · discovered · pending · missing · error · stale remote values · detached "pinned at <short-sha>" label per US-012 EC-1).
- **Artboard**: `worktree-row-states.html` (vocabulary sheet — the anchor artboard the others reuse).

### S6. Worktree context / detail header (new)

- **Purpose**: the home of the assisted exit — status strip + computed split-button + lifecycle actions. Mount decision to resolve in design (slice 07 OQ7): worktree row inline actions vs a detail header when a worktree is selected; the session header variant is S16.
- **Behavior source**: US-019/US-020/US-024/US-026; state machine reference `compozy-code/.../pull-requests/hooks/use-open-pr-button-action.ts:220-327` (auto-advancing primary + per-action `disabledReason`), split button anatomy `open-pr-button.tsx:69-158`.
- **Content**: five-field status strip (branch copyable · dirty ± · ahead/behind · PR state when known · explicit read-failure) · primary action (Commit → Commit & push → Push → Open PR → View PR) + menu with per-row reasons · merged indicator + "Clean up" suggestion · Remove action.
- **States**: each primary-action position · blocked with reason (tooltip + menu row) · paused-while-session-running · status-read-failed (whole control blocked) · diverged/behind blocked · no-remote (commit only) · zero-credential variant (browser-compare only, **no** PR-state widgets rendered) · merged (forge) · safe-to-clean (local evidence) · stale-remote marker.
- **Artboards**: `worktree-detail-exit.html` (strip + button states) · `worktree-merged-cleanup.html` (merged/cleanup variants).

### S7. Session create dialog — environment select

- **Today**: simple/advanced toolbar; advanced = Workspace (`WorkspaceCommandSelect`) + name + network — `web/src/systems/session/components/session-create-dialog.tsx:102-182`, `session-create-advanced-section.tsx:38-84`; store reset precedent `session-create-store.ts:96-119`.
- **Change**: an environment select directly under Workspace: `Workspace root` (default) · each ready worktree · `New worktree…` (opens S4 inline or after create). Resets when workspace changes.
- **States**: default · open with worktrees · zero worktrees (root + new only) · selected worktree · workspace not git-backed (control absent) · selected worktree went missing before submit (error per US-003 EC-2).
- **Artboard**: `worktree-session-create.html`.

### S8. Session composer — environment chip + pending + `/worktree`

- **Today**: generic `runtimeControl` slot in the control row — `web/src/components/assistant-ui/session-composer.tsx:63,203-206`; pattern to mirror: `session-prompt-runtime-selector.tsx:11-48` ("next prompt only" chip); native slash-command menu exists (builtins today: `/goal`, `/run`).
- **Change**: an environment chip sibling to the runtime chip on **draft/new** sessions (root / worktree / new worktree); on first send with "new worktree", a visible materialization progress (pending → ready, cancellable, failure keeps the draft — US-004); `/worktree` appears in the command menu and triggers the S9 fork on live sessions.
- **States**: chip root · chip worktree · chip new-worktree · materializing (phase progress) · materialization failed (retry / pick other / continue at root) · live session (chip shows binding, switching disabled → points to fork).
- **Artboard**: `worktree-composer-environment.html`.

### S9. Fork-to-worktree confirmation (new)

- **Precedent**: the deep-link workspace-switch confirm — `web/src/routes/_app/-session-workspace-switch.tsx:19-43`; behavior US-005.
- **Content**: states plainly that a **new session** will be created in the worktree, the current session stays untouched, and uncommitted changes stay where they are; target picker (existing/new worktree).
- **States**: confirm default · new-worktree target (chains S4 pending) · blocked mid-prompt (reason shown).
- **Artboard**: included in `worktree-composer-environment.html` (fork panel) — or split if the design pass prefers.

### S10. Task setup sheet — Worktree policy fieldset

- **Today**: "Task setup" sheet with Worker/Coordinator/Review/"Sandbox and participants" fieldsets; sandbox = mode select `inherit|none|ref` + ref input disabled unless `ref` — `web/src/systems/tasks/components/task-setup-sheet.tsx:44-203` (lock banner `:81-90`), `task-setup-form.tsx:220-279`; read view `task-setup-profile-view.tsx`.
- **Change**: a Worktree policy mirroring the sandbox control grammar with the extra mode: `inherit | none | ref | per-run`; `ref` mode gets a worktree picker (same workspace only); consider regrouping the fieldset as "Environment" (sandbox + worktree) — design decision. Read view gains the worktree row.
- **States**: each mode selected · ref picker open · invalid/removed ref flagged (US-014 EC-1) · locked-during-active-run banner (existing pattern) · read view with policy set.
- **Artboard**: `worktree-task-setup.html`.

### S11. Task fan-out dialog — per-run isolation

- **Today**: N briefs → N runs — `web/src/systems/tasks/components/task-fan-out-dialog.tsx:33-80`.
- **Change**: "Isolate each run in its own worktree" checkbox; confirmation line states how many worktrees will be created (US-016 EC-1).
- **States**: off (default) · on with count statement · partial-failure result reference (per-run attribution).
- **Artboard**: `worktree-fanout.html`.

### S12. Loop configure dialog — loop-level default

- **Today**: `web/src/systems/loops/components/configure/loop-configure-dialog.tsx`; layering precedent = `RuntimeDefaults` (loop default → node override → per-run).
- **Change**: worktree default (none / specific / per-run) in configure; per-run override surface on the run form (`lib/loop-run-form.ts`).
- **States**: unset · specific worktree · per-run · run-form override.
- **Artboard**: `worktree-loop-config.html` (shared with S13).

### S13. Loop node inspector — single Environment control

- **Today**: run-agent renders "Working dir" → `params.cwd` (`web/src/systems/loops/lib/loop-node-fields.ts:324-334`); goal has none (`loop-node-goal-fields.ts:10-70`).
- **Change**: one **Environment** control on `run-agent` and `goal` replacing/absorbing "Working dir": workspace root / existing worktree / new worktree / raw directory (escape hatch, template-interpolable). Never two directory fields (Business Rule 22). Non-agent nodes unchanged.
- **States**: default (root) · worktree picked · raw-dir escape hatch (directory mode) · templated value · effective-environment readout on the node card · retired-`cwd` validation error naming the one-line migration (B-009 hard cut).
- **Artboard**: `worktree-loop-config.html` (node inspector panels).

### S14. Commit sheet · PR sheet · progress toast (new)

- **References**: commit sheet anatomy `compozy-code/.../commit-changes-dialog.tsx:52-162` (repo/branch list → replace with scope counts; split submit Commit / Commit & push); PR sheet `\`branch → base\`` header + "leave empty to generate" + draft-as-peer-action + browser row — synara `GitCreatePrDialog.tsx:126-216`; progress = one updating toast with phases up front + per-step results + one CTA — synara contracts + t3code CTA logic (`analysis/07` §§B4-B5, C5).
- **Commit sheet states** (no daemon generation in v1 — TechSpec Key Decisions/B-013): scope shown (files, ± **plus untracked additions listed by name**, bounded/expandable — B-011) · message empty with the honest default placeholder ("Leave blank to use a default message.") · agent-staged prompt affordance (US-025, stages into the composer) · nothing-to-commit · hook failure output.
- **PR sheet states** (no daemon generation in v1): base resolved header · editable title/body with template prefill when unambiguous · draft peer action · existing-open-PR → View variant · zero-credential variant (browser row only) · template-ambiguous (prefill abstains, no error) · no-changes blocked · agent-staged prompt affordance.
- **Progress states**: phases announced → advancing · hook output streaming · success with per-step skip reasons + single CTA · mid-chain failure attributed to its step.
- **Artboards**: `worktree-commit-sheet.html` · `worktree-pr-sheet.html` · `worktree-exit-progress.html`.

### S15. Removal + missing-resolution dialogs (new)

- **References**: copy inventory from paperclip assess/blockers + old-Compozy confirm (`Remove "<branch>" from disk? This cannot be undone.`) — `analysis/07` §D and §Transferable 6; behavior US-026/US-027/US-028.
- **Clean remove**: single confirm naming target + "branch is not deleted" statement + sessions-will-stop note when applicable.
- **Dirty/unpushed refusal**: names exactly what's at risk (changed files, unpushed commits) · offers the exit flow as the alternative · separate explicit force confirm stating permanence · remote-exists downgrade note (US-027 EC-1).
- **Missing resolution**: missing state row + actions (dismiss record / it's back) with "history is preserved" statement.
- **Artboard**: `worktree-remove-dialogs.html`.

### S16. Session view — worktree binding + exit mount

- **Today**: session header/topbar (session routes); precedent for mounting exit actions in the execution header: `compozy-code/.../execution/components/execution-actions.tsx:167-210`.
- **Change**: a session bound to a worktree shows the binding (worktree name chip linking to its context); the exit control (S6) may mount here for worktree-bound sessions — design decides between header mount and detail-only.
- **States**: unbound (nothing added) · bound chip · bound + exit control mounted · binding target missing.
- **Artboard**: folded into `worktree-detail-exit.html` (header-mount variant).

---

## Component plan (design → production mapping)

Built from three verified inventories (2026-08-11): the `@compozy/ui` export surface (`packages/ui/src/index.ts` + component sources), the existing `web/` composites with line-level insertion points, and the final post-review artboard anatomy. This section is the implementation contract: **compose from what exists; the design's `worktree.css` is a visual contract, never a stylesheet to import** — production styles come from Tailwind v4 tokens and `@compozy/ui` primitives.

### Rules

1. **Reuse gate first.** Every generic need maps to an `@compozy/ui` export below before any authoring. Shadowing an exported name is a blocking lint error (`compozy-ui-reuse/no-shadow-ui-primitive`).
2. **Naming**: `<Domain><Noun><Kind>` PascalCase, kebab-case files — `WorktreeCreateDialog` (`worktree-create-dialog.tsx`), hooks `use-worktrees.ts`, adapters `worktree-api.ts`. OS-shell chrome keeps `Os*`/`Desktop*` prefixes.
3. **Placement**: domain-free + token-driven ⇒ `packages/ui` **with colocated story + test in the same PR**; anything reading worktree state/queries ⇒ `web/src/systems/<domain>/`.
4. **Artboard anchors are the per-component spec** — open the named file/section before building; gating comments in the artboards carry the behavior contract (routes, predicates, copy literals).
5. Artboard "sheet" naming ≠ primitive choice: the commit/PR "sheets" are modal **Dialogs** in production (matching their `.dialog` chrome), not `Sheet` drawers.

### New `@compozy/ui` primitive (exactly one)

| Export | Why a primitive | Composes | Spec |
| --- | --- | --- | --- |
| `SplitButton` | Generic computed-primary + chevron-menu control; no export exists today (verified). Raw material is already in the system: `buttonGroupVariants` collapses inner radii/seams; `DropdownMenu*` owns the menu. Needs: shared `variant`/`disabled` propagation, chevron trigger, menu slot, keyboard contract. Story + test mandatory. | `ButtonGroup` + `Button` + `DropdownMenu`, `DropdownMenuTrigger`, `DropdownMenuContent`, `DropdownMenuItem` | `.splitbtn` — `worktree.css:237-255`; canonical `worktree-detail-exit.html` hero + ladder |

Everything else is a domain composite — the remaining gaps (status strip, nest rail, shape-coded chips, phase toast) carry worktree contract semantics and stay in `web/`.

### Signal & state mapping cheatsheet (design glyph → existing primitive)

| Design fact | Production rendering |
| --- | --- |
| Agent **running** dot (`.d--run.d--pulse`) | `Pill.Dot tone="accent" pulse` |
| Agent **awaiting-input** (`.d--wait`, hollow accent, no pulse) | `StatusDot variant="ring" tone="accent"` + `label` |
| Agent **idle** | renders nothing in rows (contract) |
| Ready quiet dot (`.d--ok`) | `Pill.Dot tone="success" size="sm"` |
| PR pill (`PR #412`) | `Pill tone="info" mono size="xs"` |
| Branch / sha / path copyable mono | `MonoId value copy preserveCase` (never a hand-rolled copy button) |
| Strip keys / fieldset legends / group headings (uppercase micro) | `Eyebrow` (inline eyebrow styles are a lint error) |
| Field refusal (`.ferr`) | `FieldError` (+ `aria-invalid` on the control) |
| Warning/info/danger notices (`.notice`, `.mb-notice`) | `Alert variant="warning|info|danger"` (+ `AlertDescription`); menubar notice sits outside `role="menu"` |
| Blocked box (`.blockedbox`) | `Empty` (`framed`) or `Alert variant="neutral"` — reason mirrored on the disabled action's `title` |
| Toast host | existing `Toaster` + `toast.custom` content (`notifyUser` stays the store-level path for plain feedback) |
| Kbd hint (`⌘↵`) | `Kbd` |
| Fact/kv pairs (`.kv-mini`, fork `.fact`) | `MetadataList` / `ContextBox` |
| Monogram avatar wells | existing monogram span pattern (`DesktopMenubar.workspaceMonogram`) / `Avatar` |

### New domain components — `web/src/systems/workspace/`

| Component | Implements (canonical anchor) | Composes from `@compozy/ui` | Notes / contract carried |
| --- | --- | --- | --- |
| `WorktreeStateChip` | `.wt-chip[data-state]` — `worktree-row-states.html` §01 | `Pill`-style container (tone tints via tokens); shape glyph is domain CSS | 5 states, shape+color coded; renders only when state ≠ ready |
| `WorktreeStateDot` | `.wt-dot` — `row-states` §05 | shapes = domain CSS; ready is NOT this component (use `Pill.Dot tone="success"`) | single shape-truth source; never inline-styled |
| `WorktreeSignals` (dirty / aheadBehind / agent / origin / setupFlag / merged / stale / detachedPin) | `.wt-dirty` `.wt-ab` `.wt-agent` `.wt-origin` `.wt-flag` `.wt-merged` `.wt-stale` — `row-states` §02 | `Pill`, `Pill.Dot`, `StatusDot`, `Icon`, tone tokens | nullable → absent (never `+0 −0`); ahead/behind only with known upstream; max two per row; stale stamps every remote value |
| `WorktreeRow` | `.wt-row` — `row-states` §03-§05 | `Icon`, `MonoId`, `WorktreeStateChip/Signals`; row grammar per `ListingRow`/session-list rows | name = record label; branch mono+copy; path demoted; `data-inert` + reason lane; 44px full / 30px nest densities |
| `WorktreeNestList` (+ inert-reason lane, adopt affordance, `WorktreeAggregate`, overflow row) | `.wt-nest` `.wt-inert-why` `.wt-adopt` `.wt-agg` `.cs-more` — `nav-switcher` §02-§07 | wraps `CommandItem`/`MenubarItem` children; rail = domain CSS (do NOT use headless `Tree` — 2-level fixed nesting inside cmdk/menus) | locked sort (state group → activity desc); truncate at 5 + "All N worktrees" (adopted-only counts); discovered selectable = adoption gesture; pending/missing inert with reason |
| `WorktreeCreateDialog` | `worktree-create-dialog.html` (all §§) | `Dialog`+`dialogShellClass`, `EntityDialogHeader/Body/Footer`, `Field*`, `Input`, `Collapsible` (advanced fold), `CommandSelect*` (branch picker), `FieldError`, `Button`, `Spinner` | template: `WorkspaceSetupDialog`. Name-first + generated-name placeholder; live `branch → path` preview in footer; 3 refusals incl. "Select that worktree instead"; Cancel stays live while submitting |
| `WorktreeAdoptDialog` (confirm + refusal) | `nav-switcher.html:263-333` | `ConfirmDialog tone="accent"` (confirm) · `Dialog` + `Alert variant="danger"` (refusal) | bootstrap NOT re-run; refusal names the metadata reason; directory untouched |
| `WorktreeRemoveDialog` set (clean / bound-idle / bound-running refusal / dirty refusal / force / downgrade) + `WorktreeRiskRow` | `worktree-remove-dialogs.html` §01-§02 | `ConfirmDialog` (danger/warning tones), `Alert`, `MetadataList` (kv), `Button variant="destructive"`, ghost-danger doorway **after** Cancel in DOM order | titles quote the record label; risk rows quantify each blocker; force re-states quantities; assisted exit is the primary in the refusal |
| `WorktreeMissingResolutionDialog` (+ idempotent outcome) | `remove-dialogs` §03 | `Dialog`, `Item as="button"` rows (Dismiss record / It's back), `Alert` | history preserved; dismissal deletes nothing; no-op outcome verbatim |
| `WorktreeDetailHeader` | `.dhead` — `detail-exit` hero | `Icon` well, `WorktreeStateChip`, `WorktreeSignals.agent`, `MonoId` path, `Button` (Remove…) | ready chip allowed in this header (locked exception) |
| `WorktreeStatusStrip` | `.wt-strip` — `detail-exit` §strip variants | `Eyebrow` keys, `MonoId`, tone tokens; `data-unknown` cells | five fields, no more; PR cell absent with zero credential; read-failure is the fifth field and blocks everything |
| `WorktreeExitControl` + `useWorktreeExitLadder` | `.splitbtn`+`.gmenu` — `detail-exit` §01-§07 | **new `SplitButton`** + `DropdownMenuItem` two-line rows (label + reason), `Tooltip` | auto-advance ladder; per-row reason literals verbatim from the artboard; global pauses (session running, read failed); no-remote rows absent |
| `WorktreeCommitDialog` + `WorktreeScopeBlock` | `worktree-commit-sheet.html` | `Dialog`, `EntityDialogHeader/Footer`, `Textarea`, `SplitButton` (mirrors invoking ladder action), `FieldError` | scope = counts + ± plus untracked additions listed by name (B-011); honest default-message placeholder (no generation states — B-013); agent-staged prompt affordance; nothing-to-commit replaces scope; hook stderr block |
| `WorktreePrDialog` + `WorktreePrActionRows` | `worktree-pr-sheet.html` | `Dialog`, `Input`, `Textarea`, `Item as="button"` action rows, `Kbd` | actions are rows (no footer); editable title/body + template prefill only (no generation states — B-013); draft = peer row; idempotent `opened_existing` → single View row; zero-credential = browser row only, inputs absent |
| `WorktreeExitProgress` + `WorktreePhaseSteps` | `.toast`/`.tstep` — `worktree-exit-progress.html` | `toast.custom` content: `Spinner`, `Icon`, `Button` (single CTA), mono tokens | one updating toast; phases announced up front; tensed labels; skip reasons; failure attributed; `WorktreePhaseSteps` reused by the composer materialization strip |
| `WorktreeMergedEvidence` | `.mrow` — `worktree-merged-cleanup.html` | `Icon`, `Button size="sm"` (Clean up / Retry), tone tokens | two tiers; downgrade = info tone; blocker suppresses (never disables) Clean up; badge flips, never disappears |
| Data layer: `adapters/worktree-api.ts`, `workspaceKeys.worktrees(id)`, `worktreesListOptions(workspaceId)`, `hooks/use-worktrees.ts`, `lib/worktree-list-reconciliation.ts` | — | mirrors `workspace-api.ts` (typed error + AbortSignal), reconcile-then-invalidate mutation contract, server-scoped per workspace | worktree lists are never client-filtered across workspaces |

### New domain components — `web/src/systems/session/`

| Component | Implements | Composes | Mounts via |
| --- | --- | --- | --- |
| `SessionEnvironmentField` | S7 — `worktree-session-create.html` | `Field`+`FieldTitle`+`FieldDescription`, `CommandSelect*` (root group + ready worktrees + "New worktree…" foot), `WorktreeStateChip` on trigger, `FieldError` | inserts after the Workspace `Field` (`session-create-advanced-section.tsx:59`); absent when workspace not git-backed |
| `SessionEnvironmentChip` | `.env-chip` — `composer-environment` §01-§04 | chip grammar mirrors `RuntimeSelectorTrigger` composer variant (`trigger.tsx:46-56`); `Icon`, popover = `CommandSelect*` | new `environmentControl` slot in `SessionComposer`; `data-pending` dashed; `data-locked` inert on live sessions |
| `SessionWorktreeForkDialog` | S9 — `composer-environment` §05 | `ConfirmDialog tone="accent"` (template: `SessionWorkspaceSwitchDialog`), fact rows via `MetadataList`, target `CommandSelect` | `/worktree` command + locked-chip affordance; blocked mid-turn with reason |
| `SessionWorktreeBindingChip` | S16 — `detail-exit:574-632` | `Icon` + name chip (+ missing variant with Resolve… link) | `useSessionTopbarSlot` `status:` slot (`use-session-topbar-slot.tsx:149-215`); chip always mounts; exit control never mounts in the header (locked OQ7) |

### New domain components — `web/src/systems/tasks/` and `loops/`

| Component | Implements | Composes | Mounts via |
| --- | --- | --- | --- |
| `TaskWorktreePolicyFields` | `worktree-task-setup.html` §01-§04 | `PillGroup` (modes `Inherit · Workspace root · Named worktree · Per-run`), conditional `CommandSelect` ref picker (disabled-never-hidden), `FieldError`, `WorktreeStateChip` (invalid ref) | Environment fieldset in `task-setup-form.tsx` (mirror the sandbox pair at L222-252; design regroups both under one "Environment" legend) |
| `TaskWorktreePolicyReadRow` | task-setup §06 read view | `PropertyRow` + `Pill` record-label chip | `task-setup-profile-view.tsx:163-188` |
| `TaskFanOutIsolationRow` + `TaskFanOutRunResults` | `worktree-fanout.html` | `Checkbox` row + derived count statement; result rows = `Item` grammar + `WorktreeSignals` | `task-fan-out-dialog.tsx:83` + store field/trigger |
| `LoopWorktreeSection` | `worktree-loop-config.html` §01-§03 | `FormSection` + `PillGroup` + `CommandSelect` ref picker | `loop-configure-dialog.tsx:97-136` + `use-loop-configure.ts` draft + `buildLoopConfigRequest` |
| Node **Environment** field descriptors | loop-config §04-§07 | descriptor-driven: reuse `select` + `text` FieldSpecs where possible; a compound `environment` FieldSpec variant (mode + conditional ref/dir) likely needs one new union member in `loop-node-schema-types.ts:120-131` **plus** its renderer case in `loop-editor-field.tsx` | `runAgentFields` between `cwd` (L325-334, absorbed) and `isolated` (L336); same for `goalFields`; node-card `env` micro row names inherited source |

### Existing components to MODIFY (insertion points verified 2026-08-11)

| File | Point | Change |
| --- | --- | --- |
| `workspace/components/workspace-command-select.tsx` | L23-27 option type · L65-68 partition · L74-82 `handleSelect` · L167-201 group map · L202-216 footer | option gains kind/parent/branch; nested `WorktreeNestList` per workspace; worktree picks bypass `selectGlobalScope`; "New worktree…" footer action |
| `workspace/lib/workspace-command-select-options.ts` | L4-12 | projection chokepoint — carry the new fields |
| `workspace/lib/home-workspace.ts` | after L32 | `groupWorkspaceTree` partition sibling (same generic shape) |
| `workspace/lib/query-keys.ts` / `lib/query-options.ts` / `hooks/use-workspaces.ts` / `index.ts` | — | keys + options factory + mutations + banner-grouped exports |
| `os/components/menubar/workspace-menu.tsx` | L36-50 · L51-57 | nested worktree `MenubarItem`s + "New worktree…" |
| `os/components/os-menubar.tsx` | L22 `workspace` prop · L134-144 chip | chip expresses `workspace / worktree`; missing ⇒ reverts to workspace |
| `os/components/desktop-menubar.tsx` | L70-72 · L117-128 | feed worktree identity + menu handlers |
| `os/components/os-workspaces-overview.tsx` | L117-141 header · L151-211 card map | counts (adopted-only) + expandable nested read-only rows (Adopt/Resolve exceptions) |
| `os/hooks/use-desktop-shell-model.ts` | L44-59 | expose `worktrees`, `openWorktreeCreate`, dialog state |
| `os/components/desktop-shell.tsx` | L165-184 · L277-287 · L288-293 | wire props + `WorktreeCreateDialogBoundary` alongside setup boundary |
| `os/components/os-command-palette-results.tsx` | L105 | additive "Worktrees" `CommandGroup` (palette model already carries switch) |
| `session/components/session-create-advanced-section.tsx` | after L59 | mount `SessionEnvironmentField` |
| `session/components/session-create-dialog.tsx` + `-host.tsx` + `hooks/use-session-create-dialog.ts` | props · `canSubmit` L80-85 · submit body L131-142 | thread environment through props, validity, and the create request |
| `session/stores/session-create-store.ts` (+ `lib/session-create-draft.ts`) | L36-58 events · L102-111 | `worktreeSelected` event; workspace change resets environment (state the reset policy) |
| `components/assistant-ui/session-composer.tsx` + `session-thread.tsx` | L63 + L204-206 · L58 + L136 | `environmentControl?: ReactNode` slot beside `runtimeControl`, threaded through `SessionThreadProps` |
| `os/apps/session/session-window-content.tsx` | L108 | mount `SessionEnvironmentChip` beside `SessionPromptRuntimeSelector` |
| `session/hooks/use-session-topbar-slot.tsx` | L149-215 | mount `SessionWorktreeBindingChip` in `status:` |
| `os/apps/session/session-window.tsx` | L50-65 | extend cross-workspace guard for worktree deep links |
| `tasks/components/task-setup-form.tsx` / `task-setup-profile-view.tsx` / `task-fan-out-dialog.tsx` (+ fan-out store) | per rows above | policy fields, read row, isolation row |
| `loops/lib/loop-node-fields.ts` / `loop-node-goal-fields.ts` / `loop-node-schema-types.ts` / `components/editor/loop-editor-field.tsx` / `components/configure/loop-configure-dialog.tsx` / `hooks/use-loop-configure.ts` + `lib/loop-config-draft.ts` | per rows above | environment descriptors + section + draft/request |

Store/selection note: if worktree selection persists client-side alongside the workspace, `active-workspace-store.ts` gains a context field and the persist key bumps (`compozy:active-workspace:v2` → `v3`).

## Shared state vocabulary (design once, reuse everywhere)

`ready` · `pending` (materializing; never rendered ready) · `discovered` (external, not adopted; **selectable — selection is the adoption gesture**) · `missing` (out-of-band removal; history preserved) · `error` (status read failed; blocks dependent actions) · `stale` (remote-derived values unrefreshed) · `run-created` (origin badge) · `setup-flagged` (bootstrap failed, usable) · `merged` (forge) / `safe-to-clean` (local evidence) · agent activity `running / awaiting-input / idle` (idle renders nothing; the exit-pause rule applies to `running` only).

## Artboard checklist (generation order)

1. `worktree-row-states.html` — vocabulary sheet (everything else reuses it)
2. `worktree-nav-switcher.html`
3. `worktree-menubar-menu.html`
4. `worktree-overview.html`
5. `worktree-create-dialog.html`
6. `worktree-session-create.html`
7. `worktree-composer-environment.html` (incl. fork confirm)
8. `worktree-task-setup.html`
9. `worktree-fanout.html`
10. `worktree-loop-config.html` (configure + node inspector)
11. `worktree-detail-exit.html` (incl. session-header mount variant)
12. `worktree-commit-sheet.html`
13. `worktree-pr-sheet.html`
14. `worktree-exit-progress.html`
15. `worktree-merged-cleanup.html`
16. `worktree-remove-dialogs.html`

Every artboard: dark theme per `DESIGN.md` (dark mode only, warm-dark surface ramp), real (non-lorem) copy from `COPY.md` register + the §copy inventory, and one annotated variant per state listed in its surface section. Content in artboards is illustrative; runtime truth, `COPY.md`, and the brand inventory own final content per the design-reference lossiness rule.
