# PRD: Native Worktree Support

> Domain-vocabulary note: this PRD is *about* git worktrees — git, branch, commit, push, pull request, and checkout are the user-observable domain of the feature, not implementation choices. GitHub is named only as the first forge integration target, itself delivered through the extension ecosystem. No other technology naming is intended; where this document says "structured output", "configuration", or "management surfaces", the concrete shapes belong to the TechSpec.

## Overview

CompozyOS workspaces map one project root to one runtime scope — and one checkout. Every session, task run, and loop node that touches the repository works in the same working tree, so parallel work collides: an agent refactoring module A and an agent fixing module B overwrite each other's uncommitted state, and the operator's own editor sits in the middle. Users already escape this with hand-rolled `git worktree` checkouts, but CompozyOS cannot see them, select them, run work in them, or clean them up — they are invisible to navigation, to agents, and to the runtime.

Native worktree support makes the worktree a first-class, nested sub-context of its parent workspace. Operators and agents create worktrees from any git-backed workspace with a single name; CompozyOS discovers every worktree the repository already has (whoever created it); navigation shows worktrees nested under their parent and selecting one is the same gesture as selecting a workspace; tasks and loops can isolate each run in a fresh worktree automatically; and an assisted exit walks the work back out — commit, push, pull request, merged-detection, and a safe cleanup suggestion.

It is for two users at once: the **operator** who wants parallel agent work without checkout contention or babysitting git, and the **agent** itself, which must be able to create, target, inspect, and finish worktrees through structured surfaces — because in CompozyOS a capability that agents cannot operate is incomplete. The differentiator over every reference product studied: CompozyOS's daemon already owns the runtime state (which worktree has a running session, which is idle, which finished) that worktree tooling elsewhere cannot show.

## Goals

- An operator can create a worktree from any git-backed workspace by providing at most a name — branch and location are derived; a name is generated when omitted.
- CompozyOS identifies every worktree of a workspace's repository, including ones created outside CompozyOS, and shows them nested under that workspace; selecting one adopts it after a safety validation.
- Selecting a worktree is the same gesture as selecting a workspace: views scope to it, and new work started while it is selected runs inside it by default.
- Work in a worktree keeps the parent workspace's memory, agents, skills, configuration, and permissions — parallel work never costs project knowledge, and the worktree is not a separate trust domain.
- Where work starts, the environment can be chosen: session creation, the prompt box of a new session, task execution setup (including an automatic fresh-worktree-per-run mode and per-run fan-out isolation), and loop configuration with per-node overrides.
- Work leaves through an assisted exit: truthful status (branch, dirty, ahead/behind, pull-request state when known), one computed next action with per-action blocked reasons, a lean commit step, an idempotent pull-request step with generated text staged for review, and merged-branch detection that suggests — never performs — cleanup.
- No gesture in the product can silently destroy uncommitted work: removal of a dirty worktree always refuses first and requires a second, loss-naming confirmation; branches are never deleted by worktree removal; worktrees removed outside CompozyOS become "missing" with all runtime history preserved.
- Agents can operate the entire lifecycle — create, list, inspect, target, exit, remove — through command-line, programmatic, and agent-tool surfaces with structured output and deterministic errors, under the session's permission mode.
- After this ships, running N parallel task runs against one repository without checkout contention is automatic (`per-run` isolation), and hand-rolled worktree management outside the product becomes unnecessary.

## User Stories

Index into the canonical catalog — [Full user stories](_user_stories.md):

- Creation (US-001 – US-007): manual creation across workspace navigation, session creation, the composer, live-session fork, command line, and agent tools.
- Discovery (US-008 – US-009): git-truth listing of every worktree; adopt-on-select with identity validation.
- Navigation (US-010 – US-012): nested rendering, scoped selection, truthful status.
- Working (US-013): parent-brain inheritance and concurrent sessions.
- Task isolation (US-014 – US-016): worktree policy on task setup, automatic per-run isolation, fan-out.
- Loop isolation (US-017 – US-018): loop-level default and per-node environment control.
- Assisted exit (US-019 – US-025): status strip, computed actions, commit, pull request, progress/results, merged-detection, agent-staged prompts.
- Removal (US-026 – US-028): safe removal, dirty two-step protection, out-of-band reconciliation.
- Bootstrap (US-029) and Configuration (US-030): provisioning contract; placement and naming defaults.
- Agent surface (US-031) and Extensibility (US-032 – US-033): structured parity, forge extensions, lifecycle events.

## Core Features

### 1. Worktree creation

Create a worktree from a git-backed workspace by name; branch and directory derive from the name (generated when omitted). Advanced options: explicit branch name (no forced prefix), base reference, or checking out an existing branch. Creation shows a pending state until the checkout and bootstrap really completed, and rolls back cleanly on failure. Offered from workspace navigation, session creation, the new-session prompt box, the command line, and agent tools. A live session never changes directory in place — the affordance is an explicit fork into a worktree.

### 2. Discovery and adoption

Git is the source of truth: the daemon lists every worktree of the workspace's repository — CompozyOS-created and external alike. External ones render as *discovered*; selecting one adopts it after validating that its git metadata resolves to this repository (a directory whose metadata resolves into the main checkout is refused and left untouched). Adoption never re-runs bootstrap by default. Discovery never mutates git state.

### 3. Nested navigation and selection

Every surface that lists workspaces renders that workspace's worktrees nested beneath it — never as siblings. Labels come from the worktree record (operator-given name wins), never from directory basenames. Selecting a worktree scopes session/task views to it and makes it the default target for new work; selecting the parent shows the whole picture with each item's worktree binding visible. Hierarchy is two levels: workspaces have worktrees; worktrees do not.

### 4. Working inside a worktree

Sessions bound to a worktree run in its directory with the parent workspace's memory, agents, skills, configuration, and permissions. Workspace-scoped memory written from one worktree is visible from every other and from the root — one workspace brain. Multiple sessions per worktree are allowed, with no false isolation claims between them. The coordinator is never confined to a worktree; it gains awareness (it can see and name where each worker operates).

### 5. Task and loop isolation

The task execution setup gains a worktree policy — `inherit`, `none`, a specific worktree, or `per-run` — managed with the same completeness as the existing execution options (web form, programmatic surface, command line, agent tool, configurable default). `per-run` gives every task run a fresh worktree on a namespaced branch; fan-out offers the same isolation per parallel run. Policy is decided before a run is enqueued (setup is locked during active runs). Loops carry a loop-level worktree default plus a single per-node **Environment** control on agent-executing nodes that absorbs the existing raw working-directory field — one control, never two competing directory fields.

### 6. Assisted exit

In the worktree context: a five-field status strip (branch, dirty with change counts, ahead/behind, pull-request state when known, explicit read-failure state); one computed primary action that auto-advances (Commit → Commit & push → Push → Open PR → View PR) plus a menu where every action carries its own blocked reason; a commit step showing scope (file count, insertions/deletions) with honest message generation; a pull-request step with a real base-resolution chain, template-aware generated text staged for review, draft as a peer choice, idempotent creation, and a browser path as the zero-credential fallback; streamed progress with per-step outcomes and skip reasons and exactly one next-step call-to-action; merged-branch detection (forge state when available, local evidence always) that suggests cleanup. Exit actions pause, with the reason stated, while the worktree's agent session is executing. The product never auto-resolves divergence. Agent participation stages reviewable prompts into the composer — it never fires actions directly.

### 7. Safe removal and reconciliation

Clean worktrees remove with one confirmation that names the target. Dirty worktrees — or ones with unpushed unique commits — refuse with a machine-readable reason and require a second confirmation that names the loss. Branches are never deleted by removal; only CompozyOS-namespaced, auto-created branches are ever eligible for automatic reclamation, and only when verifiably unchanged. Sessions stop before their worktree is removed; their history remains in the parent workspace. Worktrees removed outside CompozyOS become *missing* — records preserved, resolution explicit. No archive/restore in v1.

### 8. Bootstrap and configuration

Every created worktree runs the bootstrap contract: a declarative copy list carries designated ignored files into the new tree, and a post-create lifecycle event lets hooks, extensions, and a configurable setup command provision it. Setup failure produces a usable-but-flagged worktree — visible, never silent. Placement defaults to a central CompozyOS-owned location grouped per workspace, configurable globally and per workspace, with per-creation overrides; configuration changes affect future creations only.

### 9. Agent manageability and the extension surface

Everything above is operable by agents: worktree verbs with structured output on the command line, programmatic parity, and native tools for create/list/inspect/target/remove — mutations gated by the session's permission mode, removal always treated as destructive. Worktree lifecycle events (`created`, `adopted`, `removed`, plus run-created origin) feed hooks and extensions. The assisted exit's forge half (pull-request state, creation, merged detection) is an **extension surface**: the runtime core owns everything answerable from local git plus zero-credential fallbacks; forge specifics ship as extensions, GitHub first as a bundled one — the same core-plus-bundled-extension pattern the product already uses for connectivity.

## Business Rules

**Identity and hierarchy**

1. A worktree belongs to exactly one parent workspace; the hierarchy is exactly two levels deep.
2. Worktree identity is minted by the runtime at creation or adoption; the filesystem path is mutable metadata. Identity is never derived from files checked out inside the worktree's tree.
3. The canonical product term is **Worktree**. The parent term remains **Workspace**; the child unit is never called a workspace.
4. Display labels come from the worktree record; a directory basename is never shown as the name.

**Trust and inheritance**

5. A worktree is inside its parent workspace's trust domain: memory, agents, skills, configuration, and permissions inherit from the parent; cross-workspace rules are unchanged by this feature.
6. Sessions may run only inside the workspace root or an **attached** worktree of that workspace. Attachment — not path location — grants eligibility; everything else remains rejected.
7. All worktree data (records, status, caches, events) is workspace-scoped; no list, read, or event may leak a worktree across workspaces.

**Discovery and adoption**

8. Git is the discovery source of truth; discovery is read-only and never mutates repository state (no implicit prune, no implicit fetch).
9. An external worktree is *discovered* until explicitly selected; selection is the adoption gesture. Ambient auto-attach is forbidden.
10. Adoption validates repository identity first; refusal names its reason and never modifies the refused directory. Adoption does not run bootstrap unless explicitly requested.
11. One branch can be checked out in only one worktree (git's own constraint); the product surfaces this as an actionable choice (select the holding worktree, branch from it, or pick another branch) — never as a raw error.

**Truthful state**

12. A worktree is *pending* until its checkout and bootstrap actually completed; pending is never rendered as ready.
13. Unknown values render as unknown or absent — never as zero, never as clean. A failed status read is an explicit error state that blocks dependent actions.
14. Remote-derived values shown while unrefreshed are marked stale. The product performs no network requests the user did not ask for.

**Branches**

15. Manually created worktrees use the user's branch name or a name derived from the worktree name — no forced prefix.
16. Automatically created worktrees (per-run isolation) use branches in a recognizable CompozyOS namespace.
17. Worktree removal never deletes a branch. Only namespaced, auto-created branches are eligible for automatic reclamation, and only when verifiably unchanged since creation.

**Isolation policies**

18. The task worktree policy has exactly four modes: `inherit`, `none`, a specific worktree reference, `per-run`. Defaults are configurable; `inherit` is the default of defaults.
19. A worktree reference in any policy must belong to the same workspace and resolve to an existing, attached worktree at validation and at run start; a broken reference fails the run with a deterministic error — never a silent fallback to the root.
20. Execution setup (including the worktree policy) is decided before a run is enqueued and cannot change while a run is active.
21. Per-run isolation: each run gets a fresh worktree; materialization failure fails the run and leaves no orphan; run-created worktrees persist after terminal states with their status visible.
22. On loop nodes, exactly one environment control exists per agent-executing node; node settings win over loop defaults; loop defaults win over nothing (workspace root). Non-agent-executing nodes carry no environment control.
23. Session reuse across runs must respect worktree bindings: a pooled or reused session never executes in a different worktree than the one its work was bound to.

**Assisted exit**

24. The primary exit action is computed from actual state and auto-advances; every blocked action names its cause; all exit actions pause, with the reason stated, while the worktree's agent session executes.
25. The product never auto-resolves divergence (no automatic rebase or merge); behind/diverged states block with an explanation.
26. Pull-request creation is idempotent: an existing open pull request for the branch is surfaced, never duplicated.
27. Generated text (commit messages, pull-request title/body) is always shown before use; placeholders never promise generation that is not actually available.
28. Merged-state precedence: fresher forge state wins over cached git status; newer local commits win over an older merged verdict. Cleanup is suggested, never executed automatically.
29. Agent-assisted exit affordances stage reviewable prompts; execution happens only when the operator sends, under the session's normal permission mode.
30. Forge-dependent affordances render only where an active forge extension supports the remote; where none does, they are absent — not permanently disabled placeholders. Git-local capabilities never depend on a forge extension.

**Removal and reconciliation**

31. Removal is two-step for at-risk work: refusal with a machine-readable dirty/unpushed reason first, explicit loss-naming confirmation second. A safety-evaluation failure blocks removal; a read error never counts as clean.
32. Unique local commits that also exist on a remote downgrade the blocker to an informational note.
33. Sessions bound to a worktree stop before removal; runtime history is preserved in the parent workspace and remains readable after removal.
34. Reconciliation never cascades: a worktree missing from disk becomes a *missing* record; no session, task, or history record is deleted because a directory disappeared. Resolution (dismiss or restore) is explicit, and dismissal deletes no history.
35. Leftover-directory cleanup after a failed removal verifies the directory is really the removed worktree's before touching it.

**Bootstrap and configuration**

36. The copy list carries only ignored files; tracked files never duplicate through it.
37. Setup commands are bounded by a timeout; failure produces a usable-but-flagged worktree with the failure readable.
38. Placement and naming defaults are configurable globally and per workspace; changes affect future creations only; an invalid configured location fails creation with the offending setting named.

**Agent surface and events**

39. Every capability in this PRD is operable through command-line, programmatic, and agent-tool surfaces with structured output; the same state yields the same data on every surface, and errors are deterministic with a distinct code per cause.
40. Agent mutations follow the session's permission mode; worktree removal is always classified destructive.
41. Lifecycle events (`created`, `adopted`, `removed`, with origin: manual, per-run, adopted) fire exactly once per worktree per transition; consumer failures are reported and never block the lifecycle.

## User Experience

**Personas.** The Operator (developer running CompozyOS locally, owns the repos and the merge decisions), the Agent (works inside sessions, parallelizes via worktrees, permission-gated), the Coordinator (plans across the workspace, never pinned to a worktree), and the Extension developer (forge integrations, provisioning hooks). Full persona definitions live in the story catalog.

**Flow: create and run.** From the workspace's nested list (or the session dialog, or the new-session prompt box): create worktree → name it (or accept a generated one) → pending → ready → start work inside it. The first session in the worktree carries the workspace's full brain; the operator sees the worktree's row show live agent activity.

**Flow: parallel task runs.** Task setup → worktree policy → `per-run` → fan out N briefs → N runs, each in its own fresh worktree on a namespaced branch, each visible nested under the workspace with its own status. The coordinator narrates which worker is where.

**Flow: finish.** Select the worktree → status strip says what remains → follow the primary action: commit (scope shown, message generated for review) → push → open PR (base stated, text generated for review, draft as a peer choice) → merged indicator appears when true → cleanup suggested → two-step-safe removal. At every blocked point the reason is written where the block is.

**Flow: adopt.** Operator has worktrees from the terminal or another tool → they appear nested, marked discovered → selecting one validates and adopts it → no bootstrap re-run → work continues where it was.

**Flow: recover.** A worktree deleted in the terminal shows as missing with history intact → operator dismisses the record or restores the directory.

**UI considerations.** Nesting must be legible at a glance (indent under the parent in both workspace-listing surfaces); worktree rows carry status as information, not decoration, and only states the daemon truly knows (running agent, dirty, branch; ahead/behind when known). Pending, missing, and error states are first-class visuals. Keyboard navigation reaches nested entries in predictable order. Accessibility follows the product's established floor. All copy follows the product-language spec — blocked-state explanations are labels that prevent wrong actions, not helper prose.

**Discoverability.** The create affordance lives where the parent workspace is (navigation context, session creation, prompt box) — no separate "worktree manager" destination is required in v1; the worktree list *is* the nested navigation plus each worktree's context surface.

## High-Level Technical Constraints

- **Integration with existing systems**: workspace registry and selection, session lifecycle and permission modes, task execution setup, loop configuration and node inspector, hooks/events, the extension runtime, memory scoping, and both workspace-listing navigation surfaces. The feature extends these; it does not fork parallel mechanisms.
- **Local-first**: every git-local capability works with zero credentials and zero network. No network I/O without an explicit user gesture; offline shows truthful staleness.
- **Cross-platform**: behavior holds on macOS, Linux, and Windows, including case-insensitive filesystems, path canonicalization quirks, and platform-specific dirty-detection differences.
- **Git as an external dependency**: the feature degrades with a real diagnostic when git is unavailable or too old — capability-gated, never a raw error leak.
- **Performance from the user's perspective**: navigation renders without waiting on repository probing; status reflects reality within a short refresh of the triggering gesture; a workspace with dozens of worktrees stays responsive.
- **Safety and security**: adoption identity validation is mandatory; the session boundary extension covers attached worktrees only; forge credentials flow through the product's existing secret handling; destructive operations are auditable through the event stream.
- **Concurrency**: creation, adoption, and removal are safe under concurrent invocation (UI + CLI + agents at once); per-repository mutations serialize rather than corrupt.
- **Data isolation**: every worktree datum is workspace-scoped end to end (reads, caches, events, extension context).

## Non-Goals (Out of Scope)

- **In-product diff review, rebase, or merge tooling.** The assisted exit ends at commit/push/PR/merged-detection; reviewing diffs and resolving divergence stay with the agent, the terminal, or the forge. (Operator decision; revisit after v1.)
- **Automation jobs/triggers worktree mode.** Scheduled and triggered runs keep executing at the workspace root until an orphan-cleanup story for unattended per-fire worktrees exists. (Deliberate deferral, not an oversight.)
- **Archive/restore of worktree state.** Removal is removal; the preserving-archive model is a candidate future increment.
- **Multi-forge support in the core.** The core carries the forge extension surface and zero-credential fallbacks; only GitHub ships as a bundled extension in v1. Other forges arrive as extensions, not core work.
- **Worktrees of worktrees.** Two levels is the contract.
- **Automatic deletion of anything.** No retention sweeps, no auto-prune, no auto-reclamation beyond verifiably-unchanged namespaced branches.
- **Container/VM-based isolation.** Worktrees isolate the working tree only; the product must not claim environment isolation it does not provide (services, ports, databases, caches remain shared).
- **Live-session in-place directory switching.** Forking into a worktree is the supported path.
- **Bulk import/adoption flows.** Adoption is per-worktree selection in v1.

## Architecture Decision Records

- [ADR-001: Worktree as a Nested Sub-Context of Its Parent Workspace](adrs/adr-001.md) — nested sub-context with parent-brain inheritance; canonical term "Worktree"; full-child-workspace and session-attribute models rejected.
- [ADR-002: Discovery Lists Every Git-Known Worktree; Adoption Is Selection With Identity Validation](adrs/adr-002.md) — git as discovery truth; discovered vs adopted; refusal semantics.
- [ADR-003: Creation Surfaces — Session, Tasks (Including Per-Run), and Loops; Automation Deferred](adrs/adr-003.md) — v1 surface set; fork-not-mutate for live sessions; coordinator never pinned.
- [ADR-004: Assisted Exit With an Extensible Forge Layer](adrs/adr-004.md) — status strip, computed actions, commit/PR steps, merged detection; core git-local + bundled GitHub extension.
- [ADR-005: Central-Home Placement; the Session Boundary Extends to Attached Worktrees](adrs/adr-005.md) — placement default and the explicit boundary decision.
- [ADR-006: Lifecycle Safety — Bootstrap Contract, Two-Step Removal, Branch Preservation, Reconcile-Never-Cascade](adrs/adr-006.md) — creation and destruction safety rules.

## Open Questions

- **Disk-footprint surfacing.** The worktree list must keep footprint legible (ADR-005), but whether v1 shows per-worktree size, a count-plus-total, or only cleanup suggestions is undecided — measuring directory sizes has a real cost.
- **Task deletion vs run-created worktrees.** When a task with per-run worktrees is deleted, do its surviving run worktrees stay as ordinary worktrees, or gain a stronger cleanup suggestion? (History-preservation rules already forbid cascading deletion; the question is the suggestion's strength.)
- **Bulk cleanup.** Multiple safe-to-clean worktrees currently require one removal flow each; whether a batched "clean all provably-safe" affordance fits v1 or waits is open.
- **Generated-name style.** Whether generated worktree names follow a readable-words scheme or a task-derived slug — and whether the scheme is configurable — is a naming-polish decision for design.
- **Composer environment control on mobile/compact layouts.** Where the environment chip lands when the composer collapses is a design-phase question.
