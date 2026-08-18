# User Stories: Opt-In Agent Network Participation

Canonical behavior catalog for the Agent Network participation redesign. Companion to `_prd.md`; consumed by `_techspec.md` (component mapping) and `_tests.md` (coverage matrix).

Participation mode names (`Local`, `Live`) are working names pending the product-language pass (see PRD Open Questions). `Live` corresponds to the mode the research corpus calls `active`. Mailbox mode was withdrawn from this release (PRD Non-Goals); US-013 is retained below as a withdrawn number.

## Personas

- **Solo builder** — the light user and majority persona post product-line merge: runs sessions, tasks, and loops locally to get their own work done; never asked for multi-agent networking and must never pay for it.
- **Autonomy operator** — runs the kanban with a coordinator and multiple workers; wants to watch and steer how their agents collaborate during a run.
- **Network operator** — power user who deliberately builds multi-agent setups: channels, threads, direct messages, long-lived fleets.
- **Workflow author** — authors reusable loops and automations, sometimes including steps that use network capabilities; needs participation to be declarable and validated on the definition.
- **Administrator** — owns the installation: controls whether the network is available at all and oversees usage.
- **Managed agent** — an AI session operating the runtime through structured command surfaces; must be able to inspect and configure participation without any visual interface.

## Story Index

| ID     | Feature Area                 | Persona           | Story                                                                 |
|--------|------------------------------|-------------------|-----------------------------------------------------------------------|
| US-001 | Default-local execution      | Solo builder      | A plain session is fully local with zero network presence            |
| US-002 | Default-local execution      | Solo builder      | A plain task run creates no coordination artifacts                   |
| US-003 | Default-local execution      | Workflow author   | A plain loop run creates no network artifacts for any step           |
| US-004 | Default-local execution      | Workflow author   | Automations never enroll their targets implicitly                    |
| US-005 | Explicit participation       | Workflow author   | Declare participation on one loop without affecting anything else    |
| US-006 | Explicit participation       | Network operator  | Override participation for one concrete execution at start           |
| US-007 | Explicit participation       | Network operator  | Child and derivative work never inherits participation               |
| US-008 | Explicit participation       | Network operator  | Choose a mode with clear semantics and no silent degradation         |
| US-009 | Coordinated runs (flagship)  | Autonomy operator | Turn on coordination conversations once for a scope                  |
| US-010 | Coordinated runs (flagship)  | Autonomy operator | Get invited at the right moment from the kanban                      |
| US-011 | Coordinated runs (flagship)  | Autonomy operator | Watch the run conversation without it becoming authority             |
| US-012 | Coordinated runs (flagship)  | Autonomy operator | Orchestration works fully without the network                        |
| US-013 | (withdrawn)                  | —                 | Mailbox mode withdrawn from release scope (PRD Non-Goals)            |
| US-014 | Live mode                    | Network operator  | Let messages wake an agent within visible bounds                     |
| US-015 | Discoverability              | Solo builder      | Find and understand the network from its always-visible area         |
| US-016 | Discoverability              | Solo builder      | Meet the network in onboarding without anything turning on           |
| US-017 | Usage visibility             | Administrator     | See what the network spends, where, truthfully                       |
| US-018 | Administration               | Administrator     | Turn network availability off and on without breaking local work     |
| US-019 | Agent manageability          | Managed agent     | Inspect and configure participation through structured surfaces      |
| US-020 | Extension ecosystem          | Workflow author   | Extensions and definitions declare participation explicitly          |

## Default-Local Execution

### US-001: A plain session is fully local

**As a** solo builder, **I want** a session created without any network choice to be completely local, **so that** I never pay network cost I did not ask for.

Acceptance criteria:

- AC-1: Given a fresh installation with default configuration, when I create a session without expressing any network choice, then the session has no network identity, no network instructions in its agent context, no network tools offered to the agent, and nothing about it appears in the network area.
- AC-2: Given a workspace or bundle that has a default channel configured, when I create a plain session, then configuration alone does not enroll it — the session is still local.
- AC-3: Given a plain session, when I inspect it, then its participation reads as Local with its origin shown (built-in default).

Edge cases:

- EC-1: Network administratively disabled → identical local behavior; nothing about the session changes.
- EC-2: I explicitly choose Local → same behavior as omission, but the origin shows my explicit choice.
- EC-3: Session restart/resume → the session stays local; a restart never acquires participation.
- EC-4: Hundreds of local sessions over weeks → zero network artifacts accumulate anywhere.

### US-002: A plain task run creates no coordination artifacts

**As a** solo builder, **I want** starting a task to produce no network coordination artifacts, **so that** my kanban stays lightweight.

Acceptance criteria:

- AC-1: Given a workspace task, when I start it without any participation choice, then no coordination channel or conversation exists for the run and the run detail shows no network affordance beyond the invitation rules of US-010.
- AC-2: Given the run completes or fails, when I inspect history, then it carries no network references.

Edge cases:

- EC-1: Retrying a failed local run → the retry is local; retries never acquire participation.
- EC-2: Daemon restart with the run in flight → recovery preserves Local; recovery never re-derives a channel.
- EC-3: Runs created before this redesign → after upgrade, old auto-created coordination artifacts no longer appear attached to runs; history renders without broken links.

### US-003: A plain loop run creates no network artifacts

**As a** workflow author, **I want** loop runs with no declared participation to be fully local for every step and judge, **so that** loop cost is exactly the work I authored.

Acceptance criteria:

- AC-1: Given a loop with no participation declared, when it runs, then no channel, membership, or network prompt exists for any step, worker, or judge, and the loop completes normally.
- AC-2: Given the loop's preview/dry-run, when I inspect it, then participation reads Local explicitly.

Edge cases:

- EC-1: Loop with dozens of steps run at 100× typical volume → still zero network artifacts.
- EC-2: A loop template copied from someone else with no participation declared → behaves local; copying never smuggles participation in.

### US-004: Automations never enroll implicitly

**As a** workflow author, **I want** automation-triggered work to be local unless the automation's own definition says otherwise, **so that** background work cannot surprise me with network cost.

Acceptance criteria:

- AC-1: Given an automation job with no participation declared, when it fires a task or loop, then the resulting runs are local.
- AC-2: Given an automation job whose definition explicitly declares participation, when it fires, then only its own targets participate, with the job named as origin.

Edge cases:

- EC-1: A webhook payload attempts to inject a participation choice → ignored or rejected unless the automation's own definition explicitly authorizes that input; the outcome is visible in the fire record.
- EC-2: Retried/duplicate fires → same participation resolution every time; no drift between attempts.

## Explicit Participation

### US-005: Declare participation on one loop

**As a** workflow author, **I want** to declare network participation on a specific loop, **so that** only that loop's runs join — nothing else in my workspace changes.

Acceptance criteria:

- AC-1: Given a loop whose definition declares participation, when it runs, then that run gets one conversation space and my sessions, tasks, and other loops remain local.
- AC-2: Given a participating loop run, when I inspect it, then I see the mode and the origin ("loop definition").
- AC-3: Given I remove the declaration, when future runs start, then they are local; an in-flight run keeps the participation it started with.

Edge cases:

- EC-1: A loop step uses a network capability while the loop is Local → validation fails when saving or starting, naming the step and the fix; the run never starts half-connected.
- EC-2: Two runs of the same loop concurrently → each has its own conversation space; no cross-talk.
- EC-3: The declared conversation target does not exist → clear failure at start; a typo never silently creates a new space.

### US-006: Override participation for one execution

**As a** network operator, **I want** to override the saved participation default for a single run or session at start, **so that** I can make exceptions without changing standing intent.

Acceptance criteria:

- AC-1: Given a scope with a saved participation default, when I start one execution with an explicit different choice, then only that execution differs and its origin reads "explicit request".
- AC-2: Given a scope with coordination on, when I start one run explicitly Local, then that run is fully local.

Edge cases:

- EC-1: Override requests a conversation space outside my authority → clear permission failure before anything starts; no partial artifacts.
- EC-2: The same start request submitted twice (double-click, retry) → one execution, one resolution.

### US-007: No implicit inheritance

**As a** network operator, **I want** child and derivative work to stay local unless it opts in itself, **so that** one participating run cannot fan participation out silently.

Acceptance criteria:

- AC-1: Given a participating execution, when it spawns children, requests reviews, or hands off detached work, then those executions are local unless their own request opted in.
- AC-2: Given a child that explicitly requests participation within the parent's delegated authority, when it starts, then it participates with its own recorded origin.

Edge cases:

- EC-1: Child requests a conversation space outside the parent's delegated authority → denied with a clear reason.
- EC-2: Chains (child of child) → every level resolves independently; depth never relaxes the rule.
- EC-3: A reviewer session for a participating run → local by default; review quality features work without the network.

### US-008: Mode selection with no silent degradation

**As a** network operator, **I want** each participating execution to select Live with clear semantics, **so that** I always know whether messages can wake my agents.

Acceptance criteria:

- AC-1: Given a participation choice surface, when I choose a mode, then the surface states the semantics: Local means no network at all; Live means messages may wake the agent within visible bounds.
- AC-2: Given the network is administratively unavailable, when I request Live, then creation fails with a clear diagnostic before anything starts — it is never silently downgraded to Local.
- AC-3: Given any participating execution, when I inspect it, then I see mode, origin, and its bounds.

Edge cases:

- EC-1: Requesting Live for an agent setup that cannot honor the bounded-wake guarantees → a clear "unsupported" reason with Local offered; never a fake Live.
- EC-2: A saved definition requests a mode that later becomes unavailable → the next execution fails with the same clear diagnostic; standing intent never silently mutates.

## Coordinated Runs (Flagship)

### US-009: Turn on coordination conversations once

**As an** autonomy operator, **I want** to enable coordination conversations once for a workspace or a single task, **so that** my coordinated runs collaborate without per-run ceremony.

Acceptance criteria:

- AC-1: Given I enable coordination conversations for a workspace, when coordinated runs start there, then they participate and each run names the setting as its origin.
- AC-2: Given I enable it for one task only, when that task runs, then only its runs participate; task-level setting wins over workspace-level for that task.
- AC-3: Given I disable the setting, when future runs start, then they are local; in-flight runs keep the participation they started with.

Edge cases:

- EC-1: Enabling while the network is administratively disabled → the setting surface explains unavailability; nothing pretends to work.
- EC-2: Two operators toggle concurrently → last write wins and both see the final state.
- EC-3: The setting is inspectable at any time: current value, who changed it last, and which runs it affected.

### US-010: The kanban invites at the right moment

**As an** autonomy operator, **I want** the run view to invite me to coordination when it would actually help, **so that** I discover the feature at its moment of value — not through settings archaeology.

Acceptance criteria:

- AC-1: Given an active run with a coordinator and multiple workers, coordination off for its scope, and the network available, when I open the run detail, then I see an invitation that explains the value and offers one clear action (enable for this workspace or this task).
- AC-2: Given I accept, then the setting is enabled for the chosen scope, the change is confirmed, and **future** coordinated runs participate — the invitation itself states that the current run's participation does not change (an execution's participation is fixed at start).
- AC-3: Given I dismiss, then the dismissal persists for that scope and the invitation stops appearing.

Edge cases:

- EC-1: Network administratively disabled → the invitation never appears.
- EC-2: Single-agent runs → no invitation; the trigger requires visible multi-agent shape.
- EC-3: Accepting twice (double-click) → one enablement, one confirmation.
- EC-4: The run finishes while the invitation is open → the invitation resolves gracefully; the setting remains reachable from its normal home.

### US-011: Watch the conversation; state stays authoritative

**As an** autonomy operator, **I want** to watch the coordinator and workers converse from the run detail, **so that** I understand and steer collaboration — while run state stays owned by the task system.

Acceptance criteria:

- AC-1: Given a participating coordinated run, when I open its detail, then I see the conversation updating in near-real-time alongside run state.
- AC-2: Given anything said in the conversation, then run status, claims, and verdicts never change because of a message by itself — task state is authoritative; the conversation is evidence.
- AC-3: Given a viewer without access to that workspace, then the conversation is not visible to them.

Edge cases:

- EC-1: The agents had nothing to say → run completes normally; the empty conversation states that silence is normal.
- EC-2: Very chatty run → the conversation view stays responsive (pagination/scrolling) and never degrades the run view.
- EC-3: A message arrives after the run reaches a terminal state → it is visible as post-completion conversation; it cannot reopen or alter the run.

### US-012: Orchestration works fully without the network

**As an** autonomy operator, **I want** the kanban, coordinator, and reviews to work completely with coordination off, **so that** the network is an upgrade, never a dependency.

Acceptance criteria:

- AC-1: Given coordination off, when coordinated runs execute (coordinator + workers + reviews), then claiming, progress, review verdicts, and terminal states all function normally.
- AC-2: Given network availability is disabled installation-wide, when I use the kanban, then nothing about task orchestration degrades.

Edge cases:

- EC-1: The network fails mid-run for a participating run → task orchestration continues to completion; the conversation is marked unavailable truthfully instead of blocking the run.
- EC-2: Coordination enabled but no messages ever sent → zero model activations attributable to the network for that run.

## Mailbox Mode (withdrawn)

### US-013: (withdrawn) A durable address that never wakes the agent

Withdrawn 2026-07-11: mailbox mode left the release scope. Within one installation, "offline" is a runtime decision — the daemon owns every agent and decides who wakes — so a durable address for unreachable recipients only earns its cost with external interaction, which is a separate future program (PRD Non-Goals; ADR-004). The ID is retained and never reused.

## Live Mode

### US-014: Messages wake the agent within visible bounds

**As a** network operator, **I want** Live participants to be woken by relevant messages within bounds I can see, **so that** live collaboration is possible and its cost is never open-ended.

Acceptance criteria:

- AC-1: Given a Live execution, when a direct message or mention addressed to it arrives, then it may wake and respond, and the execution's bounds (how much waking is allowed) plus current consumption are visible on inspection.
- AC-2: Given the bounds are exhausted, when further messages arrive, then they accumulate durably in the conversation — readable on the agent's own later turns — and the exhaustion is visible with its reason. Spend never continues silently.
- AC-3: Given system/control notifications (presence, receipts, status updates), then they never wake anyone under any mode.

Edge cases:

- EC-1: A burst of messages in a short window → handled as a coalesced wake, not one activation per message.
- EC-2: Two Live agents replying to each other → the exchange is bounded (depth/turn limits); both sides stop with a visible reason instead of ping-ponging forever.
- EC-3: Cancelling the execution mid-wake → the interrupted work is accounted truthfully and the same message cannot wake it again.
- EC-4: A message arrives exactly as bounds run out → either it wakes within bounds or it accumulates; it is never half-processed.

## Discoverability

### US-015: The network area is always visible and self-explanatory

**As a** solo builder, **I want** the network area to exist in the product with an empty state that teaches me, **so that** I can discover the capability when I am curious — disclosed, not hidden.

Acceptance criteria:

- AC-1: Given zero participation anywhere, when I open the network area, then I see an oriented empty state: what the Agent Network is, what it looks like in use, and one clear first action (enable coordination conversations for a workspace).
- AC-2: Given the empty state, then nothing in it is fabricated — no fake peers, conversations, or activity.

Edge cases:

- EC-1: Network administratively disabled → the area explains the disabled state and who can change it, instead of an inexplicable blank.
- EC-2: First participation happens → the area transitions from empty state to real content without stale placeholder remnants.
- EC-3: Participation existed and was later removed → the area returns to the empty state gracefully, with history reachable where retention allows.

### US-016: Onboarding mentions the network without enabling it

**As a** solo builder, **I want** onboarding to introduce the Agent Network lightly, **so that** I know it exists without being pushed into it.

Acceptance criteria:

- AC-1: Given first-run onboarding/getting-started, when I go through it, then one item introduces the Agent Network and points to its area — visiting or completing it changes no settings and enrolls nothing.

Edge cases:

- EC-1: Skipping or dismissing onboarding → no effect on network state, now or later.

## Usage Visibility

### US-017: Truthful network spend, attributable

**As an** administrator (and as an autonomy operator for my own runs), **I want** to see what the network spends in near-real-time, attributed to where it happened, **so that** live collaboration never produces a surprise bill.

Acceptance criteria:

- AC-1: Given participating executions, when network-triggered activations occur, then usage is visible attributed per run, per conversation space, and per workspace, updating in near-real-time.
- AC-2: Given a provider that reports usage, then actual usage is shown; given one that does not, then the value is explicitly marked unavailable — estimates are never presented as actuals.
- AC-3: Given zero participation, then network usage reads zero — and that claim is verifiable (nothing hidden elsewhere).

Edge cases:

- EC-1: Usage data is scoped by workspace access; a viewer sees only workspaces they can access.
- EC-2: High-volume runs → aggregates stay responsive and consistent with per-run detail.
- EC-3: An activation is cancelled mid-flight → its consumed portion still appears; cancelled work is never free-and-invisible.

## Administration

### US-018: Availability off/on without collateral damage

**As an** administrator, **I want** to turn Agent Network availability off and on for the installation, **so that** I control whether the capability exists without breaking local work or losing data.

Acceptance criteria:

- AC-1: Given I disable availability, then new participation requests fail with a clear diagnostic, all local work continues unaffected, and existing network data is preserved (visible as disabled/read-only, not vanished).
- AC-2: Given I re-enable availability, then participation works again and preserved data is intact.

Edge cases:

- EC-1: Disabling while live conversations are active → live delivery stops cleanly and truthfully; durable state (addresses, unread messages, history) is preserved; affected executions show why.
- EC-2: A non-administrator attempts the toggle → denied by normal permission rules.
- EC-3: Disable and re-enable in quick succession → no duplicated or lost state.

## Agent Manageability

### US-019: Agents manage participation through structured surfaces

**As a** managed agent, **I want** to inspect and configure network participation through structured command surfaces, **so that** I can operate this capability without any visual interface.

Acceptance criteria:

- AC-1: Given any execution, when I query it through the structured command surface, then I get machine-readable participation state: mode, origin, bounds, and usage — the same truth the visual interface shows.
- AC-2: Given authority over the owning work, when I configure participation (settings, definitions, per-execution requests), then the same rules and validations apply as on the human path — no UI-only control exists.
- AC-3: Given an invalid request, then the error is deterministic and named (unavailable, permission denied, unknown conversation space, unsupported mode, invalid combination).

Edge cases:

- EC-1: A Local agent session asks about network execution tools → it receives a clear "not participating" answer with what would enable them, not silent absence.
- EC-2: An agent attempts to widen its own participation beyond its delegated authority → denied with the same permission rules as any child request (US-007).

## Extension Ecosystem

### US-020: Definitions and extensions declare participation explicitly

**As a** workflow author, **I want** loops, automations, and extension-provided work to carry their participation declarations in their definitions, **so that** installing or importing something never silently connects my agents.

Acceptance criteria:

- AC-1: Given any definition surface that can start work (loop, automation, extension-provided workflow), then participation is a declarable, documented field of that definition, and its absence means Local.
- AC-2: Given an extension or bundle that recommends or requires participation, when it is activated, then the requirement is shown and explicitly confirmed by the operator; it is never auto-applied.
- AC-3: Given the operator declines a required participation, then activation fails visibly naming the requirement — no partial, silently-degraded install.

Edge cases:

- EC-1: An imported definition from another installation declares participation → it is inert until the operator confirms it in this installation's context (authority and conversation targets re-validated here).
- EC-2: An extension update adds a new participation requirement → the update surfaces it for confirmation like a new activation; it never rides in silently.
