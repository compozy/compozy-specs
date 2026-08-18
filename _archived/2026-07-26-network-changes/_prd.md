# PRD: Opt-In Agent Network Participation

## Overview

AGH ships an Agent Network — channels, threads, direct messages, and coordination conversations between agents. Today the runtime enrolls ordinary work into it implicitly: plain sessions resolve to a default channel, every workspace task run gets an auto-created coordination channel, loop steps and judges bind generated channels, and network tooling is pushed into every agent's context whenever the subsystem is available. Light users pay for a capability they never requested — in context overhead, storage, and potential message-triggered model activations — while the network's actual value stays invisible because nobody chose it.

This redesign makes network participation an explicit, per-execution choice with **Local as the default**, and simultaneously makes the network **discoverable and worth choosing** — anchored on one flagship experience: watching a coordinator and its workers collaborate during a task run on the kanban. The two halves are one product move: remove the unrequested cost, and give the capability a front door, an invitation at its moment of value, and truthful spend visibility so opting in is safe.

It is for three audiences: the **solo builder** (the majority persona once AGH ships as the Compozy product line) who must never pay network cost without asking; the **autonomy operator** who wants to see and steer agent collaboration on coordinated runs; and the **network operator** who deliberately builds multi-agent setups and needs durable addresses, live participation, and predictable cost. Market evidence confirms the posture: every serious agent platform models multi-agent communication as an explicit, user-constructed topology — ambient default-on agent chat is the pattern the industry is abandoning.

## Goals

Observable outcomes when this ships:

- A user who never touches network settings runs sessions, tasks, loops, and automations with **zero network artifacts, zero network context overhead, and zero network-attributable model activations** — verifiable from the outside.
- Every participating execution can answer **"why is this connected?"** — mode and origin (explicit request, or the saved setting/definition that selected it) are visible wherever the execution is inspected.
- An autonomy operator can enable coordination conversations **once per workspace or task** and thereafter watch coordinator/worker collaboration live from the run detail view.
- The network is **discoverable without being pushed**: a permanently visible area with oriented empty states, one contextual invitation on qualifying kanban runs, and a light onboarding mention. Looking at any of these enrolls nothing.
- Both **participation modes** — Local (no network) and Live (may wake within visible bounds) — are publicly selectable at release; requesting one that is unavailable fails with a clear, named reason instead of degrading silently.
- Network spend is **visible and truthful**: attributed per run, per conversation space, and per workspace in near-real-time; actual usage when the provider reports it, explicitly marked unavailable when not.
- Kanban orchestration — claiming, coordination, reviews, terminal states — **works identically with the network off**; the conversation is an additive layer, never a dependency.
- Managed agents can **inspect and configure participation through structured command surfaces** with full parity to the human path and deterministic, named errors.
- Loops, automations, extensions, and bundles **declare participation explicitly in their definitions**; installing or importing anything never silently connects agents.

## User Stories

Full catalog: [_user_stories.md](_user_stories.md). Index by area:

- **US-001–US-004 — Default-local execution:** plain sessions, task runs, loop runs, and automation targets are fully local.
- **US-005–US-008 — Explicit participation:** per-definition and per-execution opt-in, no inheritance, mode semantics, no silent degradation.
- **US-009–US-012 — Coordinated runs (flagship):** persisted scope opt-in, the kanban invitation, the run-view conversation, orchestration independence.
- **US-013 — (withdrawn):** mailbox mode left the release scope; see Non-Goals.
- **US-014 — Live mode:** message-triggered wake within visible bounds.
- **US-015–US-016 — Discoverability:** always-visible area with oriented empty states; onboarding mention.
- **US-017 — Usage visibility:** truthful, attributable network spend.
- **US-018 — Administration:** availability off/on without collateral damage.
- **US-019–US-020 — Agent manageability and extension ecosystem:** structured-surface parity; explicit declarations.

## Core Features

### 1. Default-local execution

Every execution surface — session, task run, loop run, automation target — resolves to Local unless an explicit request or a saved owning definition selects otherwise. Local means *nothing*: no network identity, no memberships, no conversation artifacts, no network instructions or tools in the agent's context, no message-triggered activations. This is the cost guarantee for the majority persona and the foundation every other feature builds on.

### 2. The explicit participation model

Two independent planes govern the network:

- **Availability** (installation-level): does this installation offer the Agent Network at all. On by default, costs nothing while nobody participates, never enrolls anyone.
- **Participation** (execution-level): each execution resolves its own mode — Local or Live — from the nearest explicit intent. The scope of any opt-in is the smallest unit that asked.

Participation carries provenance (what selected it), validates against what the execution actually does (a definition using network capabilities while Local is rejected at authoring/start), and never degrades silently (unavailable + participating request = clear failure).

### 3. Coordinated-run conversations (flagship)

The anchor experience: a coordinator and its workers exchanging messages in a conversation attached to a task run, watchable in near-real-time from the run detail. Enrollment is a persisted opt-in per workspace or per task ("coordination conversations"), invited contextually by the kanban when a qualifying run is active and the scope hasn't opted in. Task state remains authoritative — the conversation is collaboration evidence, and orchestration functions fully without it.

### 4. Live participation

Messages addressed to a Live participant (direct messages, mentions) may wake it — within finite, visible bounds. Exhausted bounds mean messages accumulate durably in the conversation, readable on the agent's own later turns, with the exhaustion and its reason visible. Control and status notifications never wake anyone; message bursts wake once, not per message; exchanges between live agents are bounded so they cannot ping-pong indefinitely.

### 5. Discoverability surfaces

Three on-ramps, nothing else: the always-visible network area whose empty states orient (what this is, what full looks like, one clear first action); the kanban invitation (the single behavior-triggered reveal); and a light onboarding mention. Explicitly excluded: in-session nudges, timed popups, and any surface that enrolls as a side effect of viewing.

### 6. Usage visibility

Network-attributed usage is observable in near-real-time, attributed per run, per conversation space, and per workspace. Actual provider-reported usage is shown as actual; anything else is explicitly marked unavailable. Zero participation reads as zero usage. (Configurable spend limits are deferred — see Non-Goals.)

### 7. Administration and availability

The administrative switch controls whether the capability exists. Disabling it fails new participation clearly, leaves all local work untouched, and preserves network data (visible as disabled, not vanished). Re-enabling restores without loss.

### 8. Agent manageability and extension declarations

Everything above is agent-operable: structured command surfaces expose participation state (mode, origin, bounds, usage) and accept configuration with the same rules and validations as the human path, returning deterministic named errors. Loops, automations, extensions, and bundles carry participation as an explicit, documented field of their definitions; extension-declared requirements are confirmed by the operator at activation, never auto-applied.

**Feature interactions:** the invitation (5) writes the persisted setting (3); Live (4) is what executions select through the participation model (2); usage visibility (6) covers every activation produced by (4); availability (7) gates all of it; agent surfaces (8) reach everything the human surfaces reach.

## Business Rules

1. **Availability never enrolls.** The installation-level switch answers only "is the network offered"; no configuration value — default channel, workspace setting, bundle projection — may select participation for an execution that didn't ask.
2. **Local is the built-in default** for every execution kind. Only an explicit per-execution request or a saved owning definition/setting selects Live, and the nearest explicit intent wins (execution request > owning definition/setting > default Local).
3. **Smallest-unit scope.** Enabling participation on a definition affects only executions of that definition. Enabling coordination for a task affects only that task; for a workspace, only coordinated runs in that workspace. Task-level overrides workspace-level for that task.
4. **No implicit inheritance.** Children, spawned agents, reviewers, detached work, and automation targets are Local unless their own request opts in, within the parent's delegated authority. Denials name the reason.
5. **Participation is resolved once per execution and is stable for its lifetime.** Changing a saved setting affects future executions; in-flight work keeps the participation it started with.
6. **No silent degradation, no silent upgrade.** A participating request while the network is unavailable fails with a named diagnostic (never quietly Local). A Local execution whose definition references network capabilities is rejected at authoring/start (never quietly connected). An unsupported Live request fails with a named reason and the supported alternatives.
7. **Provenance is mandatory.** Every participating execution records what selected its participation, and every inspection surface shows it.
8. **Mode semantics are fixed:**
   - *Local:* no network state, context, tools, or activations.
   - *Live:* only direct messages and mentions may wake, within finite visible bounds; control/status notifications never wake; bursts coalesce into one wake; inter-agent exchanges are depth-bounded; exhaustion converts arrivals to durable accumulation in the conversation (readable on the agent's own later turns) with a visible reason.
9. **Bounded by design.** Live activation bounds are finite and non-optional; "unlimited" is not a selectable value. Exact default values are an implementation decision, not a product one — but their existence and visibility are product requirements.
10. **Conversation is evidence, never authority.** Task statuses, claims, review verdicts, and terminal states are owned by the task system; no network message changes them by itself. A message arriving after a run's terminal state is visible history and cannot reopen the run.
11. **Invitation rules.** The kanban invitation appears only when: a coordinated run is active with visibly multi-agent shape, its scope has coordination off, and the network is available. It offers one action and states truthfully that acceptance enrolls **future** coordinated runs only — an in-flight run's participation is fixed at start and never changes underneath it. It is dismissible; dismissal persists per scope in the runtime (inspectable and resettable through the same surfaces as the setting) until explicitly reset. It never blocks, repeats after dismissal, or appears for single-agent runs.
12. **Empty states orient.** Every zero-data network view answers: where am I, why is it empty, what does it look like in use, and what is the one action that starts. Fabricated placeholder data is forbidden.
13. **Usage truthfulness.** Network usage is attributed per run, per conversation space, and per workspace; actual values are actual, unknown values are marked unavailable, and estimates are never presented as actuals. Zero participation must read as zero network usage. Cancelled work's consumed portion still appears.
14. **Workspace isolation.** All network data — conversations, addresses, memberships, usage — is workspace-scoped; no listing, reading, or attribution crosses workspaces, including when names collide.
15. **Availability toggle semantics.** Disabling preserves all durable network data, stops live delivery truthfully (affected executions show why), and leaves local work untouched. Re-enabling restores intact.
16. **One complete release.** The public product never enters a state where the old implicit model is gone and the new participation model is not fully joinable. Local default and Live ship together (ADR-004).

## User Experience

**Solo builder (zero-touch path):** installs, creates sessions, runs tasks and loops. Never sees a network decision, never pays network cost. Encounters the network only as one navigation entry (whose empty state can teach them when curious) and one onboarding line. Their agent's context contains nothing about networking.

**Autonomy operator (flagship path):** runs coordinated work on the kanban. On an active multi-agent run they meet the invitation: what coordination conversations add, one action to enable for this workspace or task. After accepting, subsequent coordinated runs carry a conversation they can watch in the run detail — coordinator and workers collaborating in near-real-time — while run state stays owned by the task system. They can check usage attributed to any run, and turn the setting off as easily as on.

**Network operator (deliberate path):** builds channels and threads from the network area, gives long-lived agents Live participation with visible bounds, sends directs, and operates a fleet. Every execution they connect shows its mode, origin, bounds, and spend.

**Managed agent (structured path):** inspects and configures all of the above through structured command surfaces with machine-readable output and deterministic errors — full parity with the human path, no visual interface required.

**Onboarding and discoverability:** exactly three on-ramps (area + invitation + onboarding mention), at most two disclosure levels between "never heard of it" and "enabled for my workspace". Accessibility follows the product's established floor for all new surfaces (empty states, invitation, conversation view, usage views).

## High-Level Technical Constraints

- **Orchestration independence is a hard constraint:** the kanban, coordinator, scheduler, and reviews must function fully — and identically — with the network off, administratively disabled, or failing mid-run. The network is additive.
- **Single-installation scope:** the network operates within one installation. Cross-installation/multi-machine federation is out of scope (see Non-Goals) and nothing in this redesign may depend on it.
- **Performance from the user's perspective:** an installation where nothing participates must be indistinguishable in behavior and responsiveness from one where the network subsystem doesn't exist.
- **Truthful surfaces:** every view renders daemon truth — no invented controls, metrics, or placeholder activity; conflicting states resolve in favor of what the runtime actually supports.
- **Cross-surface consistency:** the participation model ships coherently across the web application, structured command surfaces, agent-facing tooling, and documentation in the same release — no surface lags with the old implicit vocabulary.
- **Program sequencing:** this redesign and the AGH→Compozy product-line merge are formally interdependent — the merge's launch narrative advertises the network, and this redesign defines the network's public shape. Both programs must state the dependency and sequence releases so neither invalidates the other (see ADR-004).
- **Workspace isolation and privacy:** network data respects workspace boundaries end to end; conversation content is visible only to viewers with workspace access.
- **Vocabulary discipline:** final user-facing terms pass the product-language spec and glossary before ship; the network remains, per product positioning, a collaboration surface — never a workflow engine.

## Non-Goals (Out of Scope)

- **Multi-model execution strategies (MoA/swarm):** running advisor/aggregator model ensembles is a separate execution-strategy program; it must work without the network and is not part of this redesign.
- **Federation / multi-machine networking:** connecting agents across installations or machines remains out of scope; the product stays local-first, single-installation.
- **Mailbox mode (durable offline address):** removed from this release entirely. Within a single installation, "offline" is a runtime decision — the daemon owns every agent and decides who wakes — so a durable address for unreachable recipients only earns its cost when recipients exist outside the installation. Mailbox returns as its own program together with external interaction, in a fully separate specification. Until then, message durability inside conversations (history that agents read on their own turns) is the only asynchronous behavior, and it is not a participation mode.
- **Configurable spend limits, budgets, and alerts:** deferred to a named follow-up cost-management effort (ADR-005). This release ships usage visibility only; Live's per-participation bounds are a contract property, not a user budget system.
- **External bridge changes:** messaging bridges (chat platforms and similar) are untouched by this redesign; a bridge route is not network participation.
- **In-session network nudges:** the session surface stays free of network suggestions permanently for this release's scope; discoverability lives in the three defined on-ramps only.
- **Ambient default-on agent chat:** permanently out. No future "smart default" may re-introduce shape-inferred or configuration-inferred enrollment (ADR-001).

## Architecture Decision Records

- [ADR-001: Network Participation Is Explicit, Per-Execution, and Default-Local](adrs/adr-001.md) — two planes (availability vs participation), smallest-unit scope, no inheritance, no silent degradation.
- [ADR-002: Coordinated Task Runs Are the Flagship Use Case, Enrolled by Persisted Opt-In Plus Contextual Invitation](adrs/adr-002.md) — the aha moment, the persisted scope setting, the kanban invite, orchestration independence.
- [ADR-003: Discoverability Model — Disclosed, Not Hidden](adrs/adr-003.md) — always-visible area with oriented empty states, one behavior-triggered invitation, onboarding mention; no in-session nudges.
- [ADR-004: One Complete Release — Local and Live Ship Together, No Public "Under Construction" Window; Mailbox Removed from Scope](adrs/adr-004.md) — release packaging, the mailbox removal decision, and the merge-program dependency.
- [ADR-005: Network Usage Is Visible at Release; Configurable Spend Limits Are Deferred](adrs/adr-005.md) — truthful meter now, budget policy system later.

## Open Questions

1. **Final mode names.** `Local` and `Live` are working names (the research corpus calls the live mode `active`). The product-language pass owns the final user-facing vocabulary and must lock one pair of terms across all surfaces before ship.
2. **Spend-limits timing.** Whether the deferred cost-management effort (ADR-005) lands before or after the product-line launch is an open roadmap decision; the usage meter's scopes must compose with it either way.

*(Resolved during technical review: invitation acceptance enrolls future runs only — forced by the immutable per-execution participation rule; now Business Rule 11.)*
