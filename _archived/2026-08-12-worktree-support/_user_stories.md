# User Stories: Native Worktree Support

Canonical behavior catalog for native worktree support. Companion to `_prd.md`; consumed by
`_techspec.md` (component mapping) and `_tests.md` (coverage matrix).

## Personas

- **Operator** — the developer running CompozyOS on their machine. Manages workspaces and parallel work through the web UI and the CLI; owns the git repositories involved and the decision of what merges where.
- **Agent** — an AI agent session running inside a workspace. Creates and uses worktrees to parallelize work, reports status, and stages exit actions for the operator; every mutating action is subject to the session's permission mode.
- **Coordinator** — the background orchestration role that plans and routes task runs across the workspace. Never confined to a worktree; needs accurate awareness of where each worker operates.
- **Extension developer** — builds on the extension surfaces: forge integrations (pull-request state and creation) and provisioning hooks that react to worktree lifecycle events.

## Story Index

| ID | Feature Area | Persona | Story |
| --- | --- | --- | --- |
| US-001 | Creation | Operator | Create a worktree from the workspace with just a name |
| US-002 | Creation | Operator | Control branch and base when creating |
| US-003 | Creation | Operator | Choose a worktree when starting a session |
| US-004 | Creation | Operator | Choose the environment in the prompt box of a new session |
| US-005 | Creation | Operator | Fork a live session into a worktree |
| US-006 | Creation | Operator | Create and target worktrees from the command line |
| US-007 | Creation | Agent | Create a worktree with a native tool, permission-gated |
| US-008 | Discovery | Operator | See every worktree git knows about, nested under the workspace |
| US-009 | Discovery | Operator | Adopt an external worktree by selecting it |
| US-010 | Navigation | Operator | Navigate worktrees nested under their parent workspace |
| US-011 | Navigation | Operator | Select a worktree and get scoped views |
| US-012 | Navigation | Operator | Trust every status shown on a worktree row |
| US-013 | Working | Agent | Work in a worktree with the parent workspace's memory and skills |
| US-014 | Task isolation | Operator | Set a task's worktree policy in the task setup |
| US-015 | Task isolation | Operator | Get a fresh worktree per task run automatically |
| US-016 | Task isolation | Operator | Fan out parallel runs, each in its own worktree |
| US-017 | Loop isolation | Operator | Set a loop-level worktree default |
| US-018 | Loop isolation | Operator | Choose the environment per agent-executing loop node |
| US-019 | Assisted exit | Operator | Read the worktree's exit status at a glance |
| US-020 | Assisted exit | Operator | Follow one computed next action to finish work |
| US-021 | Assisted exit | Operator | Commit from the worktree context |
| US-022 | Assisted exit | Operator | Open a pull request from the worktree context |
| US-023 | Assisted exit | Operator | Watch progress and get per-step results |
| US-024 | Assisted exit | Operator | Get told when a worktree's branch merged, with a cleanup suggestion |
| US-025 | Assisted exit | Agent | Stage exit actions as reviewable prompts |
| US-026 | Removal | Operator | Remove a clean worktree safely |
| US-027 | Removal | Operator | Be protected from losing dirty or unpushed work |
| US-028 | Removal | Operator | Keep runtime history when a worktree disappears out-of-band |
| US-029 | Bootstrap | Operator | Get a working environment in every new worktree |
| US-030 | Configuration | Operator | Configure placement and naming defaults |
| US-031 | Agent surface | Agent | Inspect worktrees with structured output on every surface |
| US-032 | Extensibility | Extension developer | Ship forge capabilities as an extension |
| US-033 | Extensibility | Extension developer | React to worktree lifecycle events |

## Creation

### US-001: Create a worktree from the workspace with just a name

**As an** Operator, **I want** to create a worktree from a git-backed workspace by giving it only a name, **so that** starting parallel work never requires composing branch names or paths by hand.

Acceptance criteria:

- AC-1: Given a workspace whose root is a git repository, when I invoke the create-worktree action on that workspace and enter a name, then a worktree is created with a branch and directory derived from the name, and it appears nested under the workspace.
- AC-2: Given I omit the name, when I confirm creation, then a readable name is generated for me.
- AC-3: Given creation is in progress, when I look at the new entry, then it shows a pending state and becomes ready only after the checkout and bootstrap actually completed.
- AC-4: Given a workspace whose root is not a git repository, when I look at its actions, then no create-worktree affordance is shown.

Edge cases:

- EC-1: Name collides with an existing worktree of the same workspace → creation is refused with the collision named; nothing is created.
- EC-2: Name contains characters invalid for branches or directories → the derived branch/directory are sanitized predictably, shown before confirmation.
- EC-3: Creation fails midway (checkout error, disk full) → no half-created entry remains; the failure reason is shown; retrying with the same name works.
- EC-4: The daemon restarts while a creation is pending → the entry resolves to either ready (if materialization completed) or a failed state with retry; never a permanent pending.
- EC-5: Two creations with the same name are submitted concurrently → exactly one succeeds; the other receives the collision refusal.

### US-002: Control branch and base when creating

**As an** Operator, **I want** to optionally pick the branch name, the base to branch from, or an existing branch, **so that** worktrees fit my team's branching conventions.

Acceptance criteria:

- AC-1: Given the advanced creation options, when I set a branch name, then the worktree uses exactly that branch name with no forced prefix.
- AC-2: Given I pick an existing branch not checked out anywhere, when I create, then the worktree checks out that branch instead of creating a new one.
- AC-3: Given I don't choose a base, when I create a new branch, then it branches from the repository's default branch.

Edge cases:

- EC-1: The chosen branch is already checked out in another worktree → creation is refused with a message naming which worktree holds it and offering to select that worktree instead.
- EC-2: The chosen branch is checked out in the workspace root itself → creation is refused with a message telling me to switch the root off that branch first.
- EC-3: The base reference does not exist → refusal naming the missing reference; nothing is created.
- EC-4: The repository has no commits yet → creation is refused with a plain-language reason.

### US-003: Choose a worktree when starting a session

**As an** Operator, **I want** the session-creation flow to let me pick a worktree of the chosen workspace — or create a new one inline, **so that** a session starts in the right checkout from its first message.

Acceptance criteria:

- AC-1: Given the advanced session-creation options, when I open the environment choice, then I see "Workspace root" (default), every ready worktree of the selected workspace, and "New worktree".
- AC-2: Given I select a worktree, when the session starts, then it runs inside that worktree's directory and is visibly bound to it.
- AC-3: Given I change the selected workspace, when I return to the environment choice, then it has reset to that workspace's options.

Edge cases:

- EC-1: The workspace has no worktrees → the choice shows only "Workspace root" and "New worktree".
- EC-2: The selected worktree is removed between selection and submission → session creation fails with the missing worktree named; nothing starts at the root silently.
- EC-3: The workspace is not git-backed → the environment choice is absent entirely.

### US-004: Choose the environment in the prompt box of a new session

**As an** Operator, **I want** an environment control in the composer of a new/draft session, **so that** I can decide where the work runs at the moment I describe it — the way agent IDEs do.

Acceptance criteria:

- AC-1: Given a draft session with no work started, when I look at the composer controls, then an environment control shows the current target (workspace root by default) and lets me pick an existing worktree or a new one.
- AC-2: Given I chose "New worktree", when I send the first message, then the worktree materializes first (visible pending state), and the session starts inside it only after it is ready.
- AC-3: Given materialization fails, when I read the composer area, then the failure is stated and I can retry, pick another worktree, or continue at the workspace root — my message is not lost.

Edge cases:

- EC-1: I switch the environment after typing but before sending → the draft text is preserved; only the target changes.
- EC-2: I cancel a pending materialization → the worktree creation is rolled back; the draft stays; the target reverts to the previous choice.
- EC-3: The session already ran its first turn → the environment control no longer offers in-place switching (see US-005 for the fork path).

### US-005: Fork a live session into a worktree

**As an** Operator, **I want** to fork a running session into a worktree instead of mutating it in place, **so that** the transcript's file references stay truthful and the original session is untouched.

Acceptance criteria:

- AC-1: Given a live session, when I invoke the fork-to-worktree action (menu or slash command), then a confirmation states that a new session will be created in the worktree and the current one stays where it is.
- AC-2: Given I confirm, when the fork completes, then a new session exists bound to the chosen/new worktree, and the original session continues unchanged at its original location.
- AC-3: Given the fork target is a new worktree, when it materializes, then the same pending/failure rules as US-004 apply.

Edge cases:

- EC-1: Uncommitted changes exist in the original location → they stay there; the fork's worktree starts from its base; the confirmation says exactly that.
- EC-2: The original session is mid-prompt → the fork action is unavailable until the turn finishes, with the reason shown.
- EC-3: Fork is invoked twice quickly → two distinct sessions/worktrees result only if I confirmed twice; each confirmation creates exactly one.

### US-006: Create and target worktrees from the command line

**As an** Operator, **I want** worktree verbs and a worktree flag on session start in the CLI with structured output, **so that** scripts and terminal workflows have full parity with the UI.

Acceptance criteria:

- AC-1: Given the CLI, when I list/create/remove worktrees for a workspace, then each verb supports structured output whose fields match what the UI shows (name, branch, path, state, dirty, ahead/behind when known).
- AC-2: Given session start, when I pass a worktree by name, then the session starts inside it; passing a new-worktree flag creates one first and then starts.
- AC-3: Given any refusal (collision, dirty removal, validation), when I read the output, then the error is deterministic and machine-readable, with a distinct code per cause.

Edge cases:

- EC-1: Structured-output listing for a workspace with zero worktrees → an empty list, not an error.
- EC-2: A worktree name that doesn't exist is passed to session start → deterministic not-found error; nothing starts at the root as a fallback.
- EC-3: Two CLI removals of the same worktree race → one succeeds, the other gets a deterministic already-removed result; both exit cleanly.

### US-007: Create a worktree with a native tool, permission-gated

**As an** Agent, **I want** to create and list worktrees through native tools, **so that** I can parallelize my own work — while the operator's permission mode governs every mutation.

Acceptance criteria:

- AC-1: Given a session whose permission mode allows it (or after an approval prompt), when I call the worktree-creation tool, then the worktree is created and the tool returns its identity, branch, and path.
- AC-2: Given listing/inspection tools, when I call them, then I receive the same truthful state the operator sees, scoped to my session's workspace only.
- AC-3: Given worktree removal via tool, when invoked, then it is always treated as a destructive action per the permission mode — never silently allowed, and the dirty two-step refusal (US-027) applies identically.

Edge cases:

- EC-1: Permission mode denies mutations → the tool returns a deterministic denial; no prompt loops, no partial creation.
- EC-2: The tool is called with a workspace the session doesn't belong to → denied by the cross-workspace boundary; no information leaks.
- EC-3: The same create call is retried after a timeout → idempotent behavior: the earlier result is returned or the collision is named; two worktrees never result from one intent.

## Discovery

### US-008: See every worktree git knows about, nested under the workspace

**As an** Operator, **I want** Compozy to list all of the repository's worktrees — including ones created outside Compozy — nested under the workspace, **so that** the navigation answers "what worktrees does this workspace have" completely.

Acceptance criteria:

- AC-1: Given worktrees created by other tools or plain git, when I look at the workspace's nested list, then they appear, visually marked as discovered (not yet adopted).
- AC-2: Given git reports a worktree as gone/prunable, when I look at the list, then it is either absent or explicitly marked stale — never shown as a healthy selectable entry.
- AC-3: Given the repository has no worktrees, when I expand the workspace, then no worktree group noise appears — only the create affordance.

Edge cases:

- EC-1: A discovered directory sits on a network path or unreadable location → shown with an explicit unavailable state, not silently dropped.
- EC-2: Dozens of stale external worktrees exist → the list stays scannable: adopted and active entries are not drowned by discovered ones.
- EC-3: Discovery itself fails (git unavailable/broken repo) → the workspace shows a discovery-error state; the rest of navigation still works.

### US-009: Adopt an external worktree by selecting it

**As an** Operator, **I want** selecting a discovered worktree to adopt it after a safety check, **so that** migrating from other tools is a single gesture.

Acceptance criteria:

- AC-1: Given a discovered worktree, when I select it, then Compozy validates that its git metadata resolves to this workspace's repository and, on success, adopts it — it becomes a first-class worktree with a stable identity.
- AC-2: Given adoption succeeds, when I look at the entry, then bootstrap was **not** re-run (the tree is assumed provisioned) and its existing branch and state are shown truthfully.
- AC-3: Given validation fails (metadata resolves into the main checkout, unreadable git entry, foreign repository), when I read the result, then the refusal names the reason and the directory on disk was not modified in any way.

Edge cases:

- EC-1: The same external worktree is selected twice (double-click, two clients) → one adoption; the second attempt lands on the already-adopted entry.
- EC-2: The directory disappears between listing and selection → deterministic not-found refusal; the list refreshes.
- EC-3: A discovered worktree's checkout is on a branch already claimed by an adopted worktree → adoption still succeeds (git itself guarantees one checkout per branch); the listing reflects reality.

## Navigation

### US-010: Navigate worktrees nested under their parent workspace

**As an** Operator, **I want** worktrees rendered as children of their workspace in every surface that lists workspaces, **so that** the parent-child relationship is always visible.

Acceptance criteria:

- AC-1: Given a workspace with worktrees, when I open the workspace switcher or the shell's workspace menu, then its worktrees render nested beneath it — never as sibling workspaces.
- AC-2: Given a worktree entry, when I read its label, then it shows the worktree name (operator-given name wins; otherwise derived), never a stale directory basename.
- AC-3: Given both navigation surfaces, when worktrees change (created, adopted, removed), then both reflect the change without divergence.

Edge cases:

- EC-1: Deep nesting is requested (worktree of a worktree) → not offered; creation entry points exist only on the parent workspace (two levels is the contract).
- EC-2: A workspace with many worktrees → the nested group stays navigable (collapse/scroll); the parent row remains selectable on its own.
- EC-3: Keyboard-only navigation → nested entries are reachable and selectable in a predictable order.

### US-011: Select a worktree and get scoped views

**As an** Operator, **I want** selecting a worktree to scope my working views to it, **so that** the gesture and the result match selecting a workspace.

Acceptance criteria:

- AC-1: Given I select a worktree, when I look at session and task-run views, then they show the work bound to that worktree, and new work I start defaults to running inside it.
- AC-2: Given I select the parent workspace, when I look at the same views, then I see the workspace's whole picture, with each item's worktree binding visible.
- AC-3: Given a deep link into a worktree context, when I open it, then the same scoped state is reconstructed.

Edge cases:

- EC-1: The selected worktree is removed (by me elsewhere, or out-of-band) → the view falls back to the parent workspace with an explicit notice; nothing errors silently.
- EC-2: Selection state survives reload → returning to the app restores the last selected worktree if it still exists.
- EC-3: Two browser windows select different worktrees of the same workspace → each keeps its own selection; neither overwrites the other.

### US-012: Trust every status shown on a worktree row

**As an** Operator, **I want** the worktree's row/detail to show only states the runtime actually knows — branch, dirty, running-agent activity, ahead/behind when known, **so that** I can decide "which one needs me" without opening each.

Acceptance criteria:

- AC-1: Given a worktree row, when I read it, then I see its branch and, when known, dirty state and running/idle agent activity derived from real runtime state.
- AC-2: Given a value is unknown (no upstream, offline, not yet computed), when I read the row, then that value is absent or marked unknown — never rendered as zero or clean.
- AC-3: Given the status read failed, when I look at the row, then it shows an explicit error state that blocks dependent actions rather than pretending cleanliness.

Edge cases:

- EC-1: Detached checkout (no branch) → labeled truthfully as pinned to a commit, not shown with an invented branch.
- EC-2: Remote-derived values while offline → last-known values are visibly marked stale or hidden; never presented as current.
- EC-3: Agent activity mid-transition (session starting/stopping) → the indicator resolves within a refresh; no permanently stuck "running".

## Working

### US-013: Work in a worktree with the parent workspace's memory and skills

**As an** Agent, **I want** sessions in a worktree to inherit the parent workspace's memory, agents, skills, configuration, and permissions, **so that** parallel work never costs me project knowledge.

Acceptance criteria:

- AC-1: Given a session bound to a worktree, when I recall workspace memory or activate workspace skills, then I get exactly what a session at the workspace root would get.
- AC-2: Given my session writes workspace-scoped memory, when a session in another worktree (or the root) recalls it, then it is visible there too — one workspace brain.
- AC-3: Given permission and trust boundaries, when I operate inside the worktree, then the same rules as the parent workspace apply; the worktree is not a separate trust domain.

Edge cases:

- EC-1: Multiple sessions run concurrently in the same worktree → allowed; the product makes no false isolation claim between them.
- EC-2: Sessions in different worktrees edit the same logical file → each edits its own checkout; no cross-worktree interference, and no product claim of merged state before the exit flow runs.
- EC-3: A session asks for its own context → it can see both the parent workspace and its worktree binding truthfully.

## Task isolation

### US-014: Set a task's worktree policy in the task setup

**As an** Operator, **I want** a worktree policy on the task's execution setup — inherit, none, a specific worktree, or per-run, **so that** where task work runs is a managed, inspectable decision.

Acceptance criteria:

- AC-1: Given the task setup surface, when I open it, then the worktree policy appears alongside the existing execution options, with inherit as the default, and reads back exactly what was saved.
- AC-2: Given I pick a specific worktree, when runs start, then their worker sessions run inside it; given none, they run at the workspace root regardless of other defaults.
- AC-3: Given the policy is managed by API, CLI, or agent tool, when any of them updates it, then all surfaces read back the same state, and updating the worktree policy alone leaves the rest of the setup unchanged.
- AC-4: Given a run is active, when I try to change the policy, then the change is refused with the active run named (decide before enqueue).

Edge cases:

- EC-1: The referenced worktree was removed → the policy is flagged invalid at read and refused at run start with a deterministic error; runs do not silently fall back to the root.
- EC-2: Policy set to a worktree of a different workspace → validation refuses it.
- EC-3: Clearing the whole setup → worktree policy returns to inherit along with everything else.

### US-015: Get a fresh worktree per task run automatically

**As an** Operator, **I want** the per-run mode to give every task run its own worktree on an auto-named branch, **so that** parallel and repeated runs never contaminate each other or my checkout.

Acceptance criteria:

- AC-1: Given per-run mode, when a run starts, then a fresh worktree materializes for it (visible in navigation, marked as run-created), and the worker session runs inside it.
- AC-2: Given the run-created branch, when I inspect it, then it lives in a recognizable Compozy namespace — and only branches in that namespace are ever eligible for automatic reclamation, only when verifiably unchanged.
- AC-3: Given the run reaches a terminal state, when I look at the worktree, then it remains (work preserved) with its exit status visible, and cleanup is suggested when it is provably safe (merged/unchanged).

Edge cases:

- EC-1: Worktree materialization fails at run start → the run fails with the reason; no orphan directory or branch remains.
- EC-2: Multiple concurrent runs of the same task → each gets a distinct worktree; names never collide.
- EC-3: Run canceled mid-flight → the worktree persists with whatever state exists; the two-step removal rules apply to it like any other worktree.
- EC-4: Many per-run worktrees accumulate → the worktree list keeps them visibly grouped and their footprint legible; cleanup suggestions surface as they become safe.

### US-016: Fan out parallel runs, each in its own worktree

**As an** Operator, **I want** the fan-out flow to offer per-run isolation, **so that** N parallel workers on one task each get their own checkout.

Acceptance criteria:

- AC-1: Given the fan-out flow, when I enable per-run isolation, then every spawned run follows US-015 semantics — distinct worktree, namespaced branch.
- AC-2: Given fan-out without the option, when runs spawn, then the task's existing worktree policy applies unchanged.

Edge cases:

- EC-1: Large fan-out counts → bounded by the existing fan-out limits; the isolation option does not change those caps, and the confirmation states how many worktrees will be created.
- EC-2: Partial failure (run 3 of 5 fails to materialize) → only that run fails; the others proceed; the failure is attributable per run.

## Loop isolation

### US-017: Set a loop-level worktree default

**As an** Operator, **I want** a worktree default on the loop's configuration, **so that** every agent-executing node of that loop runs isolated without per-node repetition.

Acceptance criteria:

- AC-1: Given the loop configuration surface, when I set the worktree default (none, a specific worktree, or per-run), then agent-executing nodes inherit it unless they override.
- AC-2: Given a per-run loop execution override, when I start a run with a different worktree setting, then that run uses the override; the stored default is unchanged.

Edge cases:

- EC-1: A referenced worktree is removed before a run starts → the run fails validation with the missing worktree named.
- EC-2: A child loop is invoked → it inherits the parent run's environment unless its own configuration declares one.

### US-018: Choose the environment per agent-executing loop node

**As an** Operator, **I want** a single environment control on agent-executing nodes (run-agent, goal), **so that** node placement is explicit and never expressed through two competing directory fields.

Acceptance criteria:

- AC-1: Given the node inspector for run-agent or goal, when I open the environment control, then I choose among workspace root, an existing worktree, a new worktree, or a raw working directory — one control, one value.
- AC-2: Given both a loop default and a node setting exist, when the node runs, then the node setting wins, and the effective environment is visible on the node.
- AC-3: Given nodes that do not execute agent sessions, when I inspect them, then no environment control appears.

Edge cases:

- EC-1: Definitions carrying the retired raw working-directory field → fail validation with a clear error naming the one-line migration to the environment control's directory mode; nothing executes with the retired field. *(Amended 2026-08-12 — greenfield hard cut, peer-review B-009.)*
- EC-2: A templated/interpolated environment value → resolved at run time; an unresolvable value fails the node with a clear reason before any session starts.
- EC-3: Fan-out branches multiplying a downstream run-agent node → each instance follows the same environment rule; per-branch isolation is only available when the loop/node declares per-run.

## Assisted exit

### US-019: Read the worktree's exit status at a glance

**As an** Operator, **I want** a compact status strip in the worktree context — branch, dirty with change counts, ahead/behind, pull-request state when known, or an explicit error, **so that** I know what stands between this worktree and done.

Acceptance criteria:

- AC-1: Given a worktree context, when I read the strip, then I see the branch (copyable), dirty state with insertions/deletions, and ahead/behind counts when an upstream exists.
- AC-2: Given a linked pull request is known, when I read the strip, then its state (open, closed, merged) and number are shown.
- AC-3: Given the status read failed, when I read the strip, then it shows an explicit failure state, and exit actions that depend on status are blocked with that reason.

Edge cases:

- EC-1: No upstream configured → ahead/behind is absent, not zero; push semantics adjust accordingly (publish rather than update).
- EC-2: Local data is fresh but remote/forge data is not → local renders immediately; remote-derived values appear when known and are marked stale when old.
- EC-3: The worktree's agent session is actively writing → dirty counts may lag; the strip refreshes when the session's turn ends.

### US-020: Follow one computed next action to finish work

**As an** Operator, **I want** a single primary exit action that always shows my next useful step — commit, push, open PR, view PR — plus a menu of all actions with per-action reasons, **so that** finishing is guided instead of remembered.

Acceptance criteria:

- AC-1: Given the exit control, when the underlying state changes (committed, pushed, PR opened), then the primary action advances to the next step automatically.
- AC-2: Given any blocked action, when I hover or attempt it, then the exact reason is stated ("No uncommitted changes to commit.", "Commit changes before pushing.", "Branch has diverged from upstream. Rebase/merge first.").
- AC-3: Given the worktree's agent session is executing, when I look at the exit control, then all actions are paused with that stated reason, and they unlock when the session stops.
- AC-4: Given a branch behind or diverged from upstream, when I look at the actions, then the product blocks and explains — it never attempts a merge or rebase on my behalf.

Edge cases:

- EC-1: No remote configured → push/PR actions are blocked with the missing-remote reason; commit still works.
- EC-2: Detached checkout → branch-dependent actions are blocked with a "create a branch first" reason.
- EC-3: Status errors → the whole control is blocked by the status failure (US-019 AC-3), not partially enabled on stale data.

### US-021: Commit from the worktree context

**As an** Operator, **I want** a lean commit step showing what will be committed and taking a message, **so that** committing agent work is one pause, not a context switch.

Acceptance criteria:

- AC-1: Given the commit step, when it opens, then it shows the file count and insertions/deletions that will be committed — enough to sanity-check scope without a diff tool.
- AC-2: Given I leave the message blank, when generation is available, then a message is generated and shown before anything is committed; when it is not, the placeholder honestly says a default message will be used.
- AC-3: Given the split submit, when I choose Commit & push, then the push follows a successful commit in the same flow.

Edge cases:

- EC-1: Nothing to commit when the step opens (state changed underneath) → the step says so and closes without creating an empty commit.
- EC-2: Commit hooks fail → the failure output is shown; nothing is retried silently.
- EC-3: Generation hangs or is cancelled → the step stays cancellable; no git operation is held open waiting for text.

### US-022: Open a pull request from the worktree context

**As an** Operator, **I want** a PR step that resolves the base sensibly, generates title/description for review, and always offers the browser as fallback, **so that** publishing work is assisted regardless of credentials.

Acceptance criteria:

- AC-1: Given the PR step, when it opens, then it states the exact `branch → base` it will propose, with the base resolved from recorded intent, upstream, or the repository default — in that order.
- AC-2: Given a forge extension with credentials is active, when I confirm, then creation is idempotent: an existing open PR for the branch is surfaced and linked instead of duplicated.
- AC-3: Given blank title/description, when generation runs, then the repository's PR template (when unambiguous) informs the body, and I see the generated text before anything is created.
- AC-4: Given no credential or no forge extension, when I use the PR step, then the browser compare page path is offered and works — with no dead PR-state affordances rendered around it.

Edge cases:

- EC-1: Multiple PR templates exist → the product abstains from choosing; generation proceeds without a template.
- EC-2: The remote is not supported by any active forge extension → PR affordances beyond the browser path are absent, not disabled-forever placeholders.
- EC-3: The branch has an upstream but zero commits ahead → the PR action is blocked with "no changes to include" rather than opening an empty PR.
- EC-4: Draft is wanted → draft creation is a peer choice at the same step, not a buried toggle.

### US-023: Watch progress and get per-step results

**As an** Operator, **I want** exit actions to stream progress and return per-step outcomes with skip reasons, **so that** I always know what actually happened.

Acceptance criteria:

- AC-1: Given a multi-step action (commit & push), when it runs, then one updating progress surface announces the phases up front and advances through them; hook output streams live when hooks run.
- AC-2: Given completion, when I read the result, then each step reports what it did or why it was skipped ("push skipped — already up to date"), and the summary leads with the highest-value outcome.
- AC-3: Given success, when I read the result, then exactly one next-step call-to-action is attached (after commit → Push; after push with an open PR → View PR; after push without one → Open PR).

Edge cases:

- EC-1: A middle step fails (push rejected) → prior completed steps are reported as done, the failure is attributed to its step, and nothing after it ran.
- EC-2: The action is cancelled mid-flight → completed steps stand (a created commit is not un-created); the report says where it stopped.

### US-024: Get told when a worktree's branch merged, with a cleanup suggestion

**As an** Operator, **I want** merged-state detection with a suggested — never automatic — cleanup, **so that** finished worktrees don't rot on disk.

Acceptance criteria:

- AC-1: Given a forge extension knows the PR merged, when I look at the worktree, then a calm merged indicator appears (and a closed-without-merge PR flips the indicator rather than removing it).
- AC-2: Given no forge data, when local evidence shows the branch holds nothing unique (or exists on a remote), then the worktree is marked safe to clean with that evidence stated.
- AC-3: Given a cleanup suggestion, when I accept it, then the standard removal flow (US-026/US-027) runs — the suggestion itself never deletes anything.

Edge cases:

- EC-1: Forge says merged but local shows new commits since → the newer local evidence wins; no "safe to clean" claim.
- EC-2: Remote state is stale (offline) → the merged indicator shows its staleness; the suggestion downgrades accordingly.
- EC-3: Detection requires network the user didn't ask for → it doesn't happen implicitly; refresh is an explicit gesture.

### US-025: Stage exit actions as reviewable prompts

**As an** Agent, **I want** "have the agent do it" exit affordances to draft a prompt for the operator's review instead of firing actions, **so that** the human stays in the loop by construction.

Acceptance criteria:

- AC-1: Given an agent-assisted affordance (write the commit message, resolve conflicts, open the PR), when the operator clicks it, then a prepared prompt appears in the session composer for review — nothing executes yet.
- AC-2: Given the operator edits and sends the staged prompt, when I execute it, then my actions run under the session's normal permission mode.

Edge cases:

- EC-1: The bound session is busy → the affordance stages into a queued/draft state or explains why it can't; it never interrupts a running turn.
- EC-2: No session is bound to the worktree → the affordance offers to start one; it never invents an agent.

## Removal

### US-026: Remove a clean worktree safely

**As an** Operator, **I want** removal of a clean worktree to be direct but explicit, **so that** cleanup is easy without being casual.

Acceptance criteria:

- AC-1: Given a clean worktree (no uncommitted changes, no unpushed unique commits), when I remove it, then one confirmation names the worktree and states the directory will be deleted, and removal proceeds on confirm.
- AC-2: Given removal, when it completes, then the branch still exists, sessions bound to the worktree were stopped first, their history remains readable in the parent workspace, and navigation updates everywhere.
- AC-3: Given the removal, when I look at past work, then records referencing the removed worktree still name it (as removed), never dangling into nothing.

Edge cases:

- EC-1: A session is running inside it → the confirmation states the session will be stopped; removal proceeds only after it stopped cleanly.
- EC-2: Removal fails partway (directory locked) → the worktree remains listed with an error state; retry is offered; no phantom "removed" state.
- EC-3: The worktree being removed is currently selected → after removal, selection falls back to the parent workspace with a notice.

### US-027: Be protected from losing dirty or unpushed work

**As an** Operator, **I want** removal of a worktree holding work to refuse first and require an explicit second confirmation naming the loss, **so that** no one-click gesture can destroy uncommitted work.

Acceptance criteria:

- AC-1: Given uncommitted changes or unpushed unique commits, when I attempt removal, then it refuses, names exactly what is at risk (changed files, unpushed commits), and offers the exit flow as the alternative.
- AC-2: Given I explicitly choose to force, when the second confirmation appears, then it states the loss is permanent, and only that confirmation proceeds.
- AC-3: Given API/CLI/tool callers, when they hit the same state, then they receive a distinct machine-readable refusal (dirty-requires-force), and forcing requires an explicit separate flag/argument.

Edge cases:

- EC-1: Unique commits exist locally but the branch also exists on a remote → downgraded to an informational note ("safe to remove locally — branch exists on the remote"), not a blocker.
- EC-2: The safety evaluation itself fails (git error) → removal is blocked by the evaluation failure; a read error never counts as "clean".
- EC-3: Force-removal leaves a leftover directory (git metadata already gone) → cleanup of the leftover happens only after verifying the directory really is the removed worktree's; an unrelated directory at that path is never touched.

### US-028: Keep runtime history when a worktree disappears out-of-band

**As an** Operator, **I want** worktrees removed outside Compozy to become "missing" instead of vanishing with their records, **so that** external git usage never destroys my runtime history.

Acceptance criteria:

- AC-1: Given a worktree was removed or pruned outside Compozy, when the daemon notices, then the entry shows a missing state; sessions, task runs, and history bound to it remain intact and readable.
- AC-2: Given a missing worktree, when I resolve it, then I can dismiss the record (acknowledging the removal) — and dismissal never deletes session/task history.
- AC-3: Given reconciliation, when it runs, then it never cascades deletion of any runtime record on its own.

Edge cases:

- EC-1: The directory reappears (restored from backup) at the same path with matching identity → the worktree returns to its normal state.
- EC-2: A different repository appears at the missing worktree's old path → it is not mistaken for the old worktree; the record stays missing.
- EC-3: The parent workspace root itself disappears → existing workspace-level behavior governs; worktree records follow their workspace's fate, and nothing new is destroyed by the worktree layer.

## Bootstrap

### US-029: Get a working environment in every new worktree

**As an** Operator, **I want** new worktrees provisioned by a declared copy list and a post-create hook, **so that** they are born usable instead of broken.

Acceptance criteria:

- AC-1: Given a declared copy list, when a worktree is created, then the matching ignored files (environment files, local configs) are carried into the new tree before it is announced ready.
- AC-2: Given a configured setup command, when creation runs, then it executes inside the new worktree with the worktree and parent paths available to it, and its success/failure is recorded and visible on the worktree. Extensions reacting to the creation event are fail-open observers whose outcomes live in their own logs/records — they never gate or mark the worktree's setup state. *(Amended 2026-08-12 — scope clarified, peer-review B-012.)*
- AC-3: Given setup failed, when I look at the worktree, then it is usable-but-flagged with the failure readable — never silently "ready", never blocked forever.
- AC-4: Given an adopted external worktree, when adoption completes, then bootstrap did not run (the tree is presumed provisioned) unless I explicitly ask for it.

Edge cases:

- EC-1: No copy list or setup configured → creation still works; the worktree is a plain checkout.
- EC-2: The setup command hangs → it is bounded by a timeout, reported as failed, and the worktree follows AC-3.
- EC-3: The copy list matches tracked files → tracked files are never duplicated by the copy mechanism; only ignored files travel.

## Configuration

### US-030: Configure placement and naming defaults

**As an** Operator, **I want** to configure where worktrees are placed and how branches are named, globally and per workspace, **so that** the defaults fit my machine and my team's conventions.

Acceptance criteria:

- AC-1: Given no configuration, when worktrees are created, then they land in the central Compozy-owned location, grouped per workspace, with derived names — and I never typed a path.
- AC-2: Given I configure a different default location (globally or for one workspace), when new worktrees are created, then they use it; existing worktrees stay where they are.
- AC-3: Given per-creation overrides, when a path or branch is passed explicitly, then it wins over the defaults for that creation only.

Edge cases:

- EC-1: The configured location is invalid or unwritable → creation fails with the configuration problem named, pointing at the offending setting.
- EC-2: Changing the default location → affects only future creations; nothing moves on disk.
- EC-3: Two workspaces derive the same worktree directory name → per-workspace grouping keeps them apart; no collision.

## Agent surface

### US-031: Inspect worktrees with structured output on every surface

**As an** Agent, **I want** worktree listing/inspection with identical structured data across the CLI, the API, and native tools, **so that** I can manage worktrees without the web UI.

Acceptance criteria:

- AC-1: Given any of the three surfaces, when I list a workspace's worktrees, then I get the same fields and values (identity, name, branch, path, state, bindings) for the same state of the world.
- AC-2: Given my session's workspace, when I query, then I see only that workspace's worktrees — cross-workspace queries follow the existing access policy.
- AC-3: Given error cases, when I hit them, then errors are deterministic and coded consistently across surfaces.

Edge cases:

- EC-1: A worktree in pending/missing/error state → the state is represented in the structured output exactly as the UI shows it.
- EC-2: High-frequency polling by an agent → answered from daemon state without degrading the UI; truthfulness rules unchanged.

## Extensibility

### US-032: Ship forge capabilities as an extension

**As an** Extension developer, **I want** the forge-specific half of the assisted exit (PR state, PR creation, merged detection) to be an extension surface with GitHub as the bundled first implementation, **so that** new forges arrive without core changes.

Acceptance criteria:

- AC-1: Given the bundled GitHub extension is enabled and credentialed, when I use the exit flow, then PR state, idempotent creation, and merged detection work as specified in US-022/US-024.
- AC-2: Given the extension is disabled or absent, when I use the exit flow, then every git-local capability still works and no forge-dependent affordance renders.
- AC-3: Given a third-party forge extension for another provider, when it is enabled, then the same exit affordances light up for repositories on that provider, with provider-appropriate terminology.

Edge cases:

- EC-1: The extension errors or rate-limits → forge-derived fields show an error/stale state; git-local actions are unaffected.
- EC-2: Two extensions claim the same remote → resolution is deterministic and visible; never double-created PRs.
- EC-3: Credentials expire mid-session → forge affordances degrade to their absent-credential behavior with the cause named.

### US-033: React to worktree lifecycle events

**As an** Extension developer, **I want** worktree lifecycle events (created, adopted, removed — and run-created for per-run isolation), **so that** provisioning, tracking, and cleanup tooling can hook the lifecycle.

Acceptance criteria:

- AC-1: Given a subscribed hook or extension, when a worktree is created, adopted, or removed, then the corresponding event fires with the worktree's identity, workspace, branch, path, and origin (manual, per-run, adopted).
- AC-2: Given the post-create bootstrap hook (US-029), when it runs, then it is this same event surface — one lifecycle, not a parallel mechanism.
- AC-3: Given event consumers fail, when lifecycle operations run, then the operations themselves succeed or fail on their own terms — consumer failures are reported, never block the lifecycle silently.

Edge cases:

- EC-1: Events during a burst (fan-out creating N worktrees) → one event per worktree, attributable to its run.
- EC-2: A consumer needs ordering → events for one worktree are ordered (created before removed); cross-worktree ordering is not promised.
