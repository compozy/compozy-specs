# Current-State Network Enforcement Audit

## Document Control

| Field | Value |
| --- | --- |
| Snapshot commit | `01d3ec251be0a34e1b42a2393f82383c750011b7` |
| Audit date | 2026-07-10 |
| Scope | Forced AGH Network participation and its runtime/token consequences |
| Primary audience | Engineers designing and implementing the Network opt-in hard cut |
| Document type | Current-state reference and architectural explanation |
| Repository | `/home/pedronauck/Projects/agh` |

## Purpose

This document identifies where AGH currently turns ordinary sessions, tasks, task runs,
Loops, automation jobs, and child work into Network participants. It follows the value
flow from public inputs and defaults through persistence, scheduler/session behavior,
prompt and tool projection, transport subscriptions, protocol routing, and model
delivery.

The audit intentionally separates:

- **Observed fact:** behavior directly supported by code, tests, or documentation at
  the snapshot commit.
- **Static inference:** a high-confidence consequence derived from the observed call
  graph, but not reproduced in a running daemon during this audit.
- **Implementation implication:** a concise constraint for the later design. This
  document does not define the target architecture or implementation roadmap.

## Evidence Method and Limitations

The evidence was collected by static inspection of production code, tests, generated
contracts, web call sites, public documentation, the official AGH skill, and
institutional-memory documents.

Limitations:

- No daemon scenario, benchmark, packet trace, token telemetry capture, link check,
  code generation, test suite, or `make verify` run was performed. The user explicitly
  excluded link and verification commands for this research task.
- Token and model-call effects are classified as facts only where the code directly
  invokes a model-facing prompt path. Aggregate token totals are inferences because no
  runtime measurements were collected.
- The first-local-send and dead-observer defects are high-confidence static findings;
  they were not dynamically reproduced.
- Line references are exact for the stated snapshot and may drift after edits.
- Lexical blast-radius counts include textual references and generated files where the
  inventory did not explicitly exclude them. They are not semantic dependency counts.
- This audit focuses on enforcement and cost. It does not attempt a complete security
  review or a complete review of remote multi-daemon Network behavior.

## Severity

| Severity | Meaning in this audit |
| --- | --- |
| **P0** | Workspace isolation or confidentiality failure that can cross a tenant-like data boundary. |
| **P1** | Systemic correctness, availability, or cost issue on primary session/task/Loop paths. |
| **P2** | Material but bounded coupling, misleading contract, or avoidable recurring overhead. |
| **P3** | Documentation, naming, or maintainability mismatch without a direct runtime failure. |

## Executive Findings

| ID | Severity | Finding |
| --- | --- | --- |
| ENF-01 | P1 | Network is enabled by default and ordinary session creation resolves an omitted channel to a configured or bundle-projected default. |
| ENF-02 | P1 | A non-empty session channel is the de facto participation switch for lifecycle joins, environment projection, prompts, peer state, and Network delivery. |
| ENF-03 | P1 | Every queued workspace task run receives a coordination channel; when none is requested, the store synthesizes and persists one. |
| ENF-04 | P1 | Scheduler eligibility and daemon-created worker, starvation, coordinator, and reviewer sessions are coupled to the run channel; autonomy composition/docs elevate that coupling into a `bind always` contract even though lease/safe-spawn primitives are local. |
| ENF-05 | P1 | Loop public surfaces expose no Network choice, while Loop coordinator runs, run-agent sessions, and judge sessions create or use synthetic channels. |
| ENF-06 | P2 | Spawned sessions, detached work, reviews, and automation inherit Network state without an explicit per-operation participation decision. |
| ENF-07 | P1 | Network envelopes become `PromptNetwork` model turns. Some fan-out fallbacks remain O(N) in active peer count even in digest mode. |
| ENF-08 | P2 | Prompt and native-tool projection use channel presence or global dependency availability rather than one resolved participation policy. |
| PROTO-01 | P1 | The simplified `agh ch` API forces a single public thread for sends and forces replies into direct rooms, contradicting same-container guidance. |
| PROTO-02 | P0 target blocker | Durable conversation addressing uses live peer IDs and directed sends require presence, so the proposed offline mailbox cannot work through identity separation alone. |
| BUG-01 | **P0** | Task channel lookup omits `workspace_id`, allowing cross-workspace metadata selection and false channel ownership conflicts. |
| BUG-02 | P1 | Persist-before-route can make the first undirected local thread send deliver to zero local peers. |
| BUG-03 | P1 | The Network task-status observer is constructed before Network boot, resolves to nil, and is not rebound. |

The current implementation does not have one authoritative answer to "does this
execution participate in Network?" Instead, different layers infer participation from
`Network.Enabled`, a default channel, bundle settings, a task/run channel, or raw
session channel presence.

---

## 1. Configuration Makes Network the Default

### ENF-01: Global Network Defaults

**Observed facts**

- `NetworkConfig` combines the administrative runtime switch, default participation
  channel, transport settings, delivery limits, fan-out activation, digest behavior,
  and prompt budgets in one configuration object
  (`internal/config/config.go:459-473`).
- `DefaultNetworkConfig` sets `enabled = true`,
  `default_channel = "default"`, a 30-second greet interval,
  `activation_top_k = 3`, and non-zero response/delivery budgets
  (`internal/config/config.go:754-769`).
- Network validation requires a non-empty valid default channel and positive delivery
  values without first short-circuiting when `Enabled` is false
  (`internal/config/config.go:1708-1785`).
- First-run bootstrap preserves an explicit disable, but otherwise inherits the
  enabled-by-default runtime configuration. The test case is named
  `ShouldKeepNetworkEnabledByDefaultOnFirstRun`
  (`internal/config/bootstrap_test.go:198-283`).
- Config tests separately preserve explicit disable behavior
  (`internal/config/config_test.go:2366-2395`), while daemon test configuration
  asserts that Network is enabled by default
  (`internal/daemon/daemon_test.go:446-447`).

**Runtime/token effect**

- **Fact:** daemon boot constructs the Network manager whenever the global switch is
  enabled (`internal/daemon/boot.go:1333-1371`).
- **Fact:** Network manager initialization starts transport before any session is
  known to participate (`internal/network/manager.go:258-297`).
- **Fact:** transport construction starts an embedded NATS server and opens an
  in-process authenticated NATS connection
  (`internal/network/transport.go:77-161`).
- **Static inference:** a fresh default installation pays Network process, connection,
  and manager lifecycle overhead even when the operator only wants local sessions or
  local task execution.

### Configuration and Participation Are Not the Same Concern

**Observed fact:** `network.enabled` is an administrative availability switch, but
the same config object also supplies the default channel that silently opts sessions
into participation. This conflates "is the subsystem available?" with "should this
execution use it?"

**Implementation implication:** later design work must treat infrastructure
availability independently from per-execution participation. Disabling transport is
not a substitute for a local execution mode.

---

## 2. Session Creation Silently Resolves to Network

### ENF-02A: Public Contract and Server Defaulting

**Observed facts**

- The shared create-session contract has one optional string,
  `channel`, and no explicit participation policy
  (`internal/api/contract/contract.go:35-45`).
- The CLI describes `--channel` as an "Optional network channel opt-in for the
  session" (`internal/cli/session.go:82-149`, especially line 140).
- The HTTP/UDS handler does not pass an omitted channel through unchanged. It calls
  `defaultSessionChannel` before session creation
  (`internal/api/core/handlers.go:128-160`).
- Resolution order is:
  1. explicit request channel;
  2. bundle service effective default;
  3. global `network.default_channel` when Network is enabled;
  4. blank only if none of the above applies
  (`internal/api/core/bundles.go:177-195`).
- Bundle effective default resolution occurs before the
  `Config.Network.Enabled` check. Therefore a bundle-projected effective default can
  produce a channel even when the global Network runtime is disabled
  (`internal/api/core/bundles.go:182-193`).
- Bundle activation explicitly exposes
  `BindPrimaryChannelAsDefault`
  (`internal/api/core/bundles.go:143-161`), and the bundle service maintains an
  effective default channel
  (`internal/bundles/service.go:679-723`).

**Contract mismatch**

The CLI says channel is opt-in, but omission is interpreted by the server as a request
for an implicit default. A caller cannot express "use no channel" with the current
optional string because blank and omitted both enter default resolution.

### ENF-02B: Web and Onboarding Always Omit the Choice

**Observed facts**

- The main session-create dialog draft contains agent/provider/model/reasoning fields,
  but no Network field
  (`web/src/systems/session/hooks/use-session-create-dialog.ts:24-29`).
- Its mutation omits `channel`, so normal UI creation always delegates to server
  defaulting
  (`web/src/systems/session/hooks/use-session-create-dialog.ts:271-291`).
- The onboarding chat also creates its session without a channel choice
  (`web/src/systems/onboarding/hooks/use-onboarding-chat.ts:96-115`).
- The Network settings UI accurately describes the current behavior as "Channel new
  sessions join when none is specified"
  (`web/src/routes/_app/settings/network.tsx:279-298`).

**Runtime/token effect**

- **Fact:** with default configuration, the most common web session path produces a
  channel-bound session.
- **Static inference:** operators seeking a single local conversation incur Network
  prompt, join, heartbeat, and delivery surfaces without making a Network decision.

### ENF-02C: Channel Presence Becomes the Runtime Switch

**Observed facts**

- Session activation calls `joinNetworkPeer` on the critical startup path. A join
  failure rolls back session activation
  (`internal/session/manager_helpers.go:77-120`).
- `joinNetworkPeer` treats a non-empty `Info.Channel` as sufficient intent. It
  returns early only for blank channels or a missing lifecycle binding
  (`internal/session/manager_helpers.go:123-151`).
- A participating session acquires a channel broadcast subscription, a peer-direct
  subscription, and a heartbeat
  (`internal/network/manager.go:554-576`).
- Session startup exports `AGH_SESSION_CHANNEL` and `AGH_PEER_ID` whenever the
  channel string is non-empty
  (`internal/session/manager_start.go:682-713`).
- Session shutdown includes Network leave in lifecycle finalization
  (`internal/session/manager_lifecycle.go:155-199`).

**Runtime/token effect**

- **Fact:** Network join is not a sidecar after successful session startup; it is part
  of activation success.
- **Fact:** each joined session creates subscription and heartbeat work.
- **Static inference:** the channel string has accumulated authority beyond identity
  or labeling. It controls transport, environment, prompt, scheduling, and routing
  behavior across packages.

---

## 3. Onboarding Frames Network as Mandatory Setup

### ENF-02D: Managed Onboarding Agent

**Observed facts**

- The onboarding agent is described as provisioning channels and agents through
  coordination and workspace toolsets
  (`internal/config/bootstrap.go:116-124`).
- Its exact grant includes `agh__network_channels` and
  `agh__network_channel_create`
  (`internal/config/bootstrap.go:190-197`).
- Its system prompt says its job is to configure "two things, one at a time," with
  channels first and agents second. It asks the operator which channels to create and
  calls the Network creation tool for each confirmed channel
  (`internal/config/bootstrap.go:200-219`).
- The prompt permits the operator to stop without creating anything, so channel
  creation is not absolutely deterministic
  (`internal/config/bootstrap.go:214-219`).
- Bootstrap tests freeze the exact Network tool grant
  (`internal/config/bootstrap_test.go:337-377`).
- The official skill documents the same exact grant and channel-provisioning role
  (`skills/agh/references/agent-definitions.md:59`;
  `skills/agh/references/native-tools.md:61`).

**Runtime/token effect**

- **Fact:** the onboarding chat session itself omits a channel and therefore normally
  receives the default channel through session handler resolution.
- **Static inference:** the first-run product journey establishes Network as the
  expected operating model even for operators whose intended workload is local
  sessions, tasks, or Loops.

---

## 4. Spawn, Detached Work, and Reviews Inherit Participation

### ENF-06A: Spawned Sessions

**Observed facts**

- `AgentSpawnRequest` has no channel or participation selection
  (`internal/api/contract/agents.go:339-351`).
- The API mapper consequently supplies no channel override
  (`internal/api/core/agent_spawn.go:80-92`).
- `spawnChannel` returns an explicit internal option if present; otherwise it
  inherits the parent channel. Only the memory-extractor role is explicitly local
  (`internal/session/spawn.go:480-490`).
- Spawn tests assert inherited channel behavior
  (`internal/session/spawn_test.go:142-143`).
- Spawn permission policy separately carries a set of allowed Network channels
  (`internal/session/spawn.go:454-477`), but that permission set is not an explicit
  participation decision.

**Runtime/token effect**

- **Fact:** a child of a channel-bound parent normally becomes a new Network peer,
  with its own join, subscriptions, heartbeat, prompts, and potential deliveries.
- **Static inference:** using subagents for local decomposition can multiply Network
  overhead by the child count even if the parent only needs to aggregate child
  results.

### ENF-06B: Detached Work

**Observed facts**

- Detached harness work accepts a requested channel internally and otherwise falls
  back to the owner session channel
  (`internal/daemon/harness_detached_work.go:246-265,613-619`).
- Tests freeze owner-channel fallback
  (`internal/daemon/task_runtime_test.go:2110-2111`).

**Static inference:** detaching work changes lifetime but not collaboration intent;
ordinary detached execution from a default-bound owner remains Network-bound.

### ENF-06C: Reviewer Sessions

**Observed facts**

- Reviewer routing scores preferred channels
  (`internal/daemon/review_router.go:540-558`).
- When it creates a reviewer session, it assigns the selected review channel directly
  to `session.CreateOpts.Channel`
  (`internal/daemon/review_router.go:561-595`).
- Channel selection prefers profile channels and otherwise falls through to run
  coordination channel, run Network channel, then task Network channel
  (`internal/daemon/review_router.go:779-797`).

**Static inference:** because workspace runs always receive a coordination channel,
daemon-created reviewer sessions are normally Network participants even when the task
author did not request Network.

---

## 5. Tasks and Runs Deterministically Create Network State

### ENF-03A: Public Task Contracts Have No Local/Network Mode

**Observed facts**

- Task create, child create, update, enqueue, start/publish/approval execution requests
  expose optional `network_channel` fields
  (`internal/api/contract/tasks.go:680-797`).
- The task domain repeats these optional fields across task creation, patch,
  execution, enqueue, and queued-run reservation
  (`internal/task/types.go:650-750`).
- The task execution profile models coordinator, worker, review, participant,
  sandbox, and runtime policies, but contains no collaboration or Network
  participation field
  (`internal/task/profile.go:52-122`).
- Native task tool schemas expose `network_channel` for create, child create,
  update, and fan-out
  (`internal/tools/builtin/tasks.go:374-435,637-660`).

**Observed UX mismatch**

- The task editor calls Network channel an optional "peer ingress" value
  (`web/src/systems/tasks/components/task-form/ingress-identity-section.tsx:10-35`).
- Fan-out UI says the channel is optional
  (`web/src/systems/tasks/components/tasks-fan-out-runs-card.tsx:86-99`).
- Blank does not mean local at run time; it means the store derives a channel.

### ENF-03B: Store Synthesis on Every Workspace Run

**Observed facts**

- Queue creation resolves a stored Network channel from the request/task and then
  computes `coordinationChannelIDForQueuedRun`
  (`internal/store/globaldb/global_db_task_aux.go:741-779`).
- For a workspace task:
  - a non-empty Network channel is reused;
  - otherwise `derivedRunCoordinationChannelID(runID)` is returned.
  Global task runs are the only scope that returns blank
  (`internal/store/globaldb/global_db_task_aux.go:804-812`).
- Before inserting the run, the store ensures that this ID exists in
  `network_channels`
  (`internal/store/globaldb/global_db_task_aux.go:814-877`).
- The derived format is normalized and prefixed with `coord-`
  (`internal/store/globaldb/global_db_task_aux.go:880-913`).
- The run persists both `NetworkChannel` and
  `CoordinationChannelID`
  (`internal/store/globaldb/global_db_task_aux.go:768-782`).

**Runtime/token effect**

- **Fact:** starting a workspace task with no channel creates durable Network channel
  metadata and a channel-bound run.
- **Fact:** this occurs below HTTP/CLI/UI intent, so all callers, including automation
  and internal daemon paths, inherit the behavior.
- **Static inference:** channel rows created solely for execution correlation increase
  channel-list noise, persistence volume, and the probability that later runtime
  layers treat local work as Network work.

### ENF-04A: Scheduler Eligibility Requires the Channel

**Observed facts**

- Scheduler eligibility calls `coordinationChannelMatches` for every queued candidate
  (`internal/scheduler/scheduler.go:720-746`).
- A non-empty run coordination channel requires exact equality with the session
  channel (`internal/scheduler/scheduler.go:769-779`).
- Since workspace runs always receive a coordination channel, an otherwise eligible
  local session with a blank channel cannot be selected.

**Runtime effect**

- **Fact:** Network-derived identity is part of work scheduling, not only messaging.
- **Static inference:** removing session default channels without first separating
  local scheduling would strand existing workspace runs.

### ENF-04B: Worker and Starvation Sessions Are Forced onto the Run Channel

**Observed facts**

- `taskRunSessionChannel` prefers `CoordinationChannelID` and falls back to
  `NetworkChannel`
  (`internal/daemon/task_runtime_recovery.go:345-350`).
- The general task session bridge passes this result into
  `session.CreateOpts.Channel`
  (`internal/daemon/task_session_bridge.go:95-125`).
- Task-role activation copies the same channel into its activation
  (`internal/daemon/task_role_runtime.go:215-259`).
- Role-session creation binds that channel
  (`internal/daemon/task_role_sessions.go:34-70`).
- Starvation recovery uses the same channel and creates a bounded child session
  (`internal/daemon/task_role_sessions.go:169-221`).
- `taskRoleSessionName` substitutes the literal `default` when activation has no
  channel, making the session identity itself channel-shaped even in a nominally local
  path (`internal/daemon/task_role_sessions.go:260-262`).
- `taskRolePromptOverlay` repeats the same fallback and unconditionally injects
  `Coordination channel: default` into the worker prompt
  (`internal/daemon/task_role_sessions.go:286-300`).

**Runtime/token effect**

- **Fact:** daemon-created workers for workspace tasks become Network peers.
- **Fact:** even a blank-channel task-role activation is described to the model as if a
  `default` coordination channel existed.
- **Static inference:** a single task can create channel metadata, a worker peer,
  transport subscriptions, heartbeat traffic, Network startup prompt content, and
  envelope-triggered model turns without explicit operator opt-in.
- **Static inference:** deleting runtime joins without deleting the task-role name and
  prompt fallback would preserve fictional Network authority and may cause unnecessary
  channel/tool discovery attempts. Off mode therefore needs a channel-independent role
  identity and no coordination-channel prompt line.

### ENF-04C: Coordinator and Review Coupling

**Observed facts**

- Coordinator bootstrap is disabled by default
  (`internal/config/autonomy.go:76-95`).
- When enabled, coordinator decision logic refuses to bootstrap without a run
  coordination channel
  (`internal/coordinator/coordinator.go:100-142`).
- Coordinator session creation grants and binds the decision channel and embeds it in
  the prompt overlay
  (`internal/daemon/coordinator_runtime.go:639-680`).
- Reviewer sessions inherit run/task channels as described in ENF-06C.

**Runtime effect**

- **Fact:** coordinator orchestration is designed around channel-bound runs.
- **Static inference:** a local-mode hard cut must supply non-Network correlation and
  event paths; simply blanking channels would disable or misroute coordinator behavior.

### ENF-04D: The Autonomy Kernel Is Locally Capable, but Its Composition and Product Contract Are Channel-First

**Observed facts**

- The autonomy native-tool family claims, heartbeats, completes, fails, releases, and
  reviews task runs directly through task-service tools; its descriptors do not require
  Network (`internal/tools/builtin/autonomy.go:13-94`). Lease ownership in
  `internal/task/autonomy.go` is likewise task/session state rather than channel state.
- The agent-facing kernel CLI always registers `agh ch list|recv|send|reply` as the
  coordination family (`internal/cli/agent_kernel.go:101-116`). `send` requires a
  channel and presents status/blocker traffic as the normal task-run communication path
  (`internal/cli/agent_kernel.go:183-247`). Its coordination metadata defaults
  `coordination_channel_id` to the supplied channel
  (`internal/cli/agent_kernel.go:324-375`).
- The autonomy index defines autonomous work as sessions coordinating through
  task-bound channels and states that workspace runs receive a stable coordination
  channel (`packages/site/content/runtime/core/autonomy/index.mdx:6-8,77-89`).
- The dedicated page says every workspace-scoped coordinated run has a durable channel
  and makes `bind always, speak when useful` the rule
  (`packages/site/content/runtime/core/autonomy/coordination-channels.mdx:6-13`). It
  directs status, blockers, handoffs, reviews, and result exchange through `agh ch` at
  `:15-28`.
- Coordinator documentation makes channel creation/resolution a bootstrap condition and
  shows `agh task start ... --channel ...` as the execution boundary
  (`packages/site/content/runtime/core/autonomy/coordinator.mdx:11-23,76-92`).
- The autonomy-kernel article's canonical sequence binds a coordination channel before
  coordinator bootstrap and returns the worker verdict over that channel
  (`docs/articles/2026-05/06-autonomy-kernel.mdx:140-169`).

**Runtime/token effect**

- **Fact:** the lease, claim, review-verdict, and safe-spawn kernel primitives do not
  intrinsically require Network. The hard dependency is introduced by coordinator/run
  composition and the agent-facing communication contract.
- **Fact:** current guidance makes channel messages the default route for progress,
  blockers, handoffs, review requests, and intermediate results even when task events,
  typed artifacts, or direct runtime returns can carry the state.
- **Static inference:** this channel-first product contract encourages extra envelopes,
  recipient activations, prompt wrappers, and protocol replies. It also makes future
  contributors preserve mandatory channels because the public explanation calls them an
  autonomy invariant.
- **Implementation consequence:** preserve the local task lease/safe-spawn kernel. Delete
  channel existence as a coordinator/run prerequisite; project `agh ch` and its guidance
  only for an explicitly participating coordinator/worker; rewrite the `bind always`
  autonomy pages and article examples in the same hard cut.

### Tests and Institutional Contracts That Freeze Task Coupling

- Task start integration explicitly expects a derived coordination channel and one new
  `network_channels` row
  (`internal/task/manager_integration_test.go:684-761`).
- Task approval integration expects a derived channel
  (`internal/task/manager_integration_test.go:975-1020`).
- Scheduler integration requires a derived run channel in session snapshots and claim
  criteria
  (`internal/scheduler/scheduler_integration_test.go:24-142`).
- Historical-channel suites preserve Network and coordination fields through retries
  and terminal transitions
  (`internal/task/network_channel_historical_integration_test.go:755-1017`).
- The canonical glossary states that every workspace-scoped coordinated run has one
  durable Network channel and summarizes the policy as "Bind always, speak when
  useful" (`docs/_memory/glossary.md:243`).

---

## 6. Loops Have No Opt-Out but Create Multiple Channel-Bound Surfaces

### ENF-05A: No Choice at the Public Run Boundary

**Observed facts**

- `RunLoopRequest` contains inputs, parent run ID, and config overrides, but no
  participation or Network field
  (`internal/api/contract/loops.go:179-184`).
- `agh loop run` exposes no Network mode/channel flag
  (`internal/cli/loop.go:236-288`).
- The web Loop run form sends only inputs and config overrides
  (`web/src/systems/loops/hooks/use-loop-run-form.ts:49-53,81-86`).

### ENF-05B: Every Running Loop Reserves a Coordinator Task Run

**Observed facts**

- Starting a persisted Loop in running state atomically reserves a Loop coordinator
  run (`internal/store/globaldb/global_db_loop.go:26-81`).
- The store ensures a workspace-scoped synthetic coordinator task and reserves a
  coordinator run for it
  (`internal/store/globaldb/global_db_loop_coordinator_seed.go:14-80`).
- Reconciliation also reserves a missing coordinator run
  (`internal/store/globaldb/global_db_loop_reconcile_queries.go:150-194`).
- That reservation goes through the ordinary queued-run path and therefore receives a
  derived coordination channel.
- Coordinator-created node tasks inherit the parent task Network channel
  (`internal/store/globaldb/global_db_task_coordinator.go:527-613`).
- Planned node runs pass their requested Network channel into ordinary queue
  reservation, which derives a coordination channel when blank
  (`internal/store/globaldb/global_db_task_coordinator.go:897-958`).

**Runtime/token effect**

- **Fact:** even a Loop whose useful work is entirely local creates task/run Network
  metadata.
- **Static inference:** a multi-node Loop can multiply synthetic channel rows and
  channel-bound task sessions by node/run count.

### ENF-05C: Run-Agent and Judge Sessions Always Receive Synthetic Channels

**Observed facts**

- Every run-agent action session uses
  `loopRuntimeSessionChannel(workspaceID, handle)`
  (`internal/daemon/loop_runtime_adapters.go:37-66`).
- Every agent judge session uses the same helper family
  (`internal/daemon/loop_runtime_adapters.go:118-145`).
- The helper always constructs a bounded non-empty channel from
  `loop_<workspace>_<suffix>`
  (`internal/daemon/loop_runtime_adapters.go:340-393`).
- Adapter tests freeze its normalized, workspace-prefixed, at-most-64-character output
  (`internal/daemon/loop_runtime_adapters_test.go:57-85`).

**Runtime/token effect**

- **Fact:** each such ACP session is channel-bound and enters the ordinary Network
  join/prompt lifecycle.
- **Static inference:** Loop action isolation currently implies Network peer creation;
  isolation of ACP execution and collaboration intent are coupled.

---

## 7. Automation Inherits Task and Loop Enforcement

### ENF-06D: Task-Backed Jobs

**Observed facts**

- Task-backed automation creates a durable task and enqueues a run
  (`internal/automation/dispatch.go:645-683`).
- It copies `job.task.network_channel` into the task and then into enqueue input
  (`internal/automation/dispatch.go:674-678,1165-1189`).
- The automation task config exposes `network_channel`
  (`internal/automation/model/types.go:120`), and the web job form edits it
  (`web/src/systems/automation/components/job-form/task-run-step.tsx:78`).
- For a workspace job with a blank field, ordinary store reservation still derives a
  coordination channel.

### ENF-06E: Loop-Backed Jobs

**Observed facts**

- Automation Loop starts resolve inputs and call the ordinary Loop service
  (`internal/daemon/automation_loop_starter.go:98-131`).
- No participation field is propagated, so all Loop coordinator/action/judge behavior
  from Section 6 applies transitively.

**Static inference:** scheduled local automation cannot reliably avoid Network by
leaving `network_channel` blank; the enforcement is below the automation layer.

---

## 8. Prompt and Situation Projection Reinforce Network Authority

### ENF-08A: Raw Channel Presence Selects the Startup Network Section

**Observed facts**

- Harness context normalizes `ChannelBound` as
  `strings.TrimSpace(input.Channel) != ""`
  (`internal/daemon/harness_context.go:310-330`).
- A channel-bound session receives the Network startup section
  (`internal/daemon/harness_context.go:518-538`).
- That section is the "AGH Network Response Register," which tells the agent how to
  behave in threads and when to reply
  (`internal/daemon/network_response_register_prompt.go:47-59`).
- The runtime envelope independently includes `channel` as a current session fact
  (`internal/daemon/runtime_prompt.go:83-115`).
- Harness tests assert that channel-bound contexts include the Network section
  (`internal/daemon/harness_context_test.go:461-470`;
  `internal/daemon/harness_context_integration_test.go:129`).

**Runtime/token effect**

- **Fact:** the default session channel adds Network-specific startup tokens to
  otherwise ordinary sessions.
- **Static inference:** the prompt also shapes agent behavior toward threads/tasks,
  reinforcing Network use beyond the deterministic runtime coupling.

### ENF-08B: Prompt Registration Does Not Follow `network.enabled`

**Observed facts**

- Prompt boot enables the Network response augmenter when
  `ResponseGuidanceMaxBytes > 0`, not when `Network.Enabled` is true
  (`internal/daemon/boot.go:342-387`).
- The default response-guidance budget is non-zero
  (`internal/config/config.go:754-769`).
- `bootNetwork` independently returns early when Network is disabled
  (`internal/daemon/boot.go:1333-1339`).
- Session join silently returns when the lifecycle binding is nil
  (`internal/session/manager_helpers.go:136-139`).

**Static inference and concrete scenario**

An explicit request channel, or a bundle effective default, can produce a
channel-bound session while Network is disabled. That session can receive Network
startup guidance and Network environment identity even though no Network manager was
booted and join is a no-op. The system can therefore present participation semantics
without a functioning transport.

### ENF-08C: Network Turns Add Another Response Register

**Observed facts**

- On `TurnSourceNetwork`, the per-turn augmenter prepends surface-specific response
  guidance
  (`internal/daemon/network_response_register_prompt.go:62-93`).
- This is in addition to the startup section and delivery-message guidance described
  in Section 10.

### ENF-08D: Situation Context Performs Network Reads

**Observed facts**

- Active-session situation assembly resolves task/channel context, then selects an
  active Network channel and calls `networkSections`
  (`internal/situation/service.go:215-295`).
- `networkSections` reads the session inbox and, when a channel is present, lists
  peers for the workspace/channel
  (`internal/situation/service.go:688-724`).
- Inbox and peer roster are rendered as prompt sections when populated
  (`internal/situation/render.go:115-145,214-216`).
- Prompt compaction tracks those sections
  (`internal/situation/prompt_compaction.go:40-105`).

**Runtime/token effect**

- **Fact:** when situation context is assembled for a channel-bound active session,
  Network state participates in prompt-context construction.
- **Static inference:** default channel binding adds database/runtime reads and can add
  roster/inbox tokens even on work that did not request collaboration.

---

## 9. Native Tool Projection Is Global Rather Than Participation-Aware

### ENF-08E: Coordination Toolset and Default Policy

**Observed facts**

- The coordination toolset contains Network status, channels, inbox, peers, send,
  channel mutation, subscriptions, threads, directs, and work tools
  (`internal/tools/builtin/toolsets.go:25-46`).
- Task tools separately include thread promotion and fan-out
  (`internal/tools/builtin/toolsets.go:144-169`).
- Effective policy starts descriptors as visible and callable, then applies explicit
  deny/restriction rules
  (`internal/tools/policy.go:174-237`).
- An agent policy is unrestricted when its tools and toolsets are both empty
  (`internal/tools/policy.go:323-325`).
- Native Network availability checks global daemon dependencies
  (`deps.Network` and sometimes `deps.NetworkStore`), not the calling session's
  channel or participation intent
  (`internal/daemon/native_tools.go:528-538`).
- Network bindings apply those dependency predicates across the Network tool family
  (`internal/daemon/native_tools.go:660-737`).

**Runtime/token effect**

- **Fact:** session participation is not an input to native Network tool availability.
- **Static inference:** a local session can still be offered Network descriptors when
  global Network is booted unless another policy happens to restrict them. Descriptor
  projection consumes prompt/tool-schema budget and makes accidental Network use more
  likely.

### Existing Positive Counterexample: Official Skill Routing

The official skill already describes Network as conditional:

- Its router loads Network guidance only for Network participation tasks
  (`skills/agh/SKILL.md:20-32`).
- The Network reference says to use it only when the current session participates in a
  Network channel
  (`skills/agh/references/network.md:16-20`).

This conditional documentation is not matched by session/task/Loop runtime defaults.

---

## 10. Protocol and Delivery Cost

### PROTO-01A: Every Conversation Message Requires a Container

**Observed facts**

- The protocol exposes only `thread` and `direct` conversation surfaces
  (`internal/network/envelope.go:43-61`).
- Discovery messages omit conversation fields, but conversation kinds must resolve
  exactly one valid conversation reference. Capability, receipt, and trace also
  require `work_id`
  (`internal/network/validate.go:271-330`).
- Task coordination metadata requires task ID, run ID, coordination channel ID,
  message kind, and correlation ID
  (`internal/api/contract/agents.go:302-318,450-470`).
- The `agh ch send` CLI exposes those correlation fields as separate required
  operational flags
  (`internal/cli/agent_kernel.go:183-247`).

**Runtime/token effect**

- **Fact:** the protocol carries useful audit/correlation information.
- **Static inference:** the required metadata and response instructions are expensive
  for short status exchanges and encourage verbose, mechanically repeated payloads.

### PROTO-01B: `agh ch send` Forces One Shared Thread

**Observed facts**

- The agent-channel API defines a constant thread,
  `thread_agent_channel`
  (`internal/api/core/agent_channels.go:23-32`).
- Every `AgentChannelSend` request is translated to a thread-surface
  `KindSay` envelope using that fixed thread ID
  (`internal/api/core/agent_channels.go:121-185`).

**Runtime effect**

- **Fact:** callers of the simplified surface cannot choose a subject-specific thread
  or direct room.
- **Static inference:** unrelated task/run messages accumulate in one per-channel
  conversation container, expanding participant history and reducing routing
  precision.

### PROTO-01C: `agh ch reply` Changes the Conversation Container

**Observed facts**

- Reply construction always resolves a direct room between caller and source peer,
  sets `SurfaceDirect`, and targets the source peer
  (`internal/api/core/agent_channels.go:283-315`).
- Tests explicitly require this direct reply behavior
  (`internal/api/core/agent_channels_internal_test.go:176-224`).
- The official Network reference says to respond in the same conversation container
  by default
  (`skills/agh/references/network.md:24-30`).

**Runtime/token effect**

- **Fact:** a public thread message followed by `agh ch reply` opens/uses a direct
  room instead of preserving the thread.
- **Static inference:** this fragments context and history, requiring agents and users
  to reconstruct causality across containers despite carrying `reply_to`,
  `trace_id`, and `causation_id`.

### PROTO-02: Live Peer Identity Cannot Address a Durable Offline Mailbox

**Observed facts**

- The v0 envelope stores `From`, `To`, and `Mentions` as peer-shaped identities
  (`internal/network/envelope.go:196-218`).
- A directed send is rejected unless the target has current live presence
  (`internal/network/router.go:354-379`).
- Direct-room identity is derived from the two peer ids
  (`internal/network/router.go:995-1027`; `internal/network/validate.go:212-241`).
- Thread participants, subscription preferences, direct-room pairs, and work ownership are persisted
  with peer-id keys (`internal/store/globaldb/global_db.go:163-176,349-430`).

**Runtime/product effect**

- **Fact:** current direct addressing is live-presence-oriented, not a durable offline mailbox
  contract.
- **Static inference:** if process peer identity rotates on restart, direct rooms and other durable
  relations either change identity or must improperly retain the old process id. Merely adding a
  mailbox id to the execution snapshot cannot make offline direct delivery work.
- **Required boundary:** P0-C must hard-cut durable addressing/relations to stable mailbox ids and
  resolve mailbox to a fenced current lease only after durable acceptance. Peer ids remain optional
  live-presence/transport evidence.

---

## 11. Network Runtime and Model-Call Amplification

### COST-01: Every Delivered Envelope Can Become a Model Turn

**Observed facts**

- Delivery's model-facing interface is `PromptNetwork`
  (`internal/network/delivery.go:53-60`).
- The delivery worker formats a queued envelope or digest batch and calls
  `PromptNetwork` for the target session
  (`internal/network/delivery.go:636-688`).
- Digest mode batches multiple envelopes for one recipient, but the resulting digest
  is still a `PromptNetwork` call
  (`internal/network/delivery.go:691-726`).

**Runtime/token effect**

- **Fact:** "digest" reduces envelopes per prompt, not necessarily recipient model
  activations.
- **Static inference:** Network message count is not the correct cost unit. The
  dominant unit is recipient prompt activation, multiplied by recipients and retries.

### COST-02: Fan-Out Fallback Can Be O(N) Model Calls

**Observed facts**

- Thread routing first consults persisted participants. If none exist, it selects
  initial recipients using fan-out policy
  (`internal/network/router.go:1241-1329`).
- Capability matching activates at most `activation_top_k` positive candidates
  (`internal/network/router.go:1422-1447`).
- If capability matching selects no positive candidate, or coordinator policy has no
  available coordinator, the fallback assigns all remaining peers as digest
  recipients
  (`internal/network/router.go:1310-1328,1450-1463`).
- `all_members` selects all eligible peers for full delivery
  (`internal/network/router.go:1305-1309`).

**Runtime/token effect**

- **Fact:** the fallback recipient set can grow linearly with channel membership.
- **Fact:** each digest recipient still reaches `PromptNetwork`.
- **Static inference:** a single undirected message in a poorly matched or
  misconfigured channel can trigger N agent turns. `activation_top_k` does not bound
  this fallback.

### COST-03: Guidance Is Repeated Across Three Layers

**Observed facts**

1. Channel-bound startup receives the Network Response Register
   (`internal/daemon/network_response_register_prompt.go:47-59`).
2. Each Network-origin turn can receive another response-register line
   (`internal/daemon/network_response_register_prompt.go:62-93`).
3. Delivery formatting adds reply flags, response policy, correlation rules, protocol
   rules, and on first use several full CLI examples
   (`internal/network/delivery.go:1647-1689,1778-1887`).

- Delivery guidance state is tracked per session and persisted so later prompts can be
  compact
  (`internal/network/delivery.go:532-604`).

**Runtime/token effect**

- **Fact:** compaction exists, but each participating session still pays verbose
  guidance at least once under the applicable state.
- **Static inference:** child/session fan-out multiplies the "once per session" cost,
  and the startup/per-turn/delivery layers duplicate the same behavioral rules.

### COST-04: Joined Sessions Produce Recurring Transport and Persistence Work

**Observed facts**

- A joined session acquires a shared broadcast subscription, one direct subscription,
  and a heartbeat
  (`internal/network/manager.go:554-576,1642-1719`).
- Heartbeat starts with an immediate greet and repeats at the configured interval
  (`internal/network/manager.go:821-865`).
- Each greet is persisted as a conversation message and audited
  (`internal/network/manager.go:868-881`).
- The audit test explicitly expects three repeated greet heartbeats to produce three
  audit entries
  (`internal/network/audit_test.go:213-254`).

**Runtime/token effect**

- **Fact:** heartbeats do not themselves prove model-token consumption, but they
  consume timers, NATS traffic, SQLite writes, and audit storage.
- **Static inference:** default-bound idle sessions create recurring operational cost
  unrelated to the user's task.

---

## 12. Independent Correctness Defects

These defects should not be mistaken for optionality design choices. They are
current-state correctness issues that affect any future Network design.

### BUG-01: P0 Workspace-Isolation Failure in Task Channel Lookup

**Observed facts**

- `network_channels` is correctly keyed by
  `PRIMARY KEY (workspace_id, channel)`, so the same channel name may legally exist
  in multiple workspaces
  (`internal/store/globaldb/global_db.go:549-565`).
- The task helper `networkChannelEntry` queries only
  `WHERE channel = ?`, omitting `workspace_id`
  (`internal/store/globaldb/global_db_task_claim_helpers.go:189-201`).
- Claim metadata starts with the trusted task workspace, task ID, and run ID
  (`internal/store/globaldb/global_db_task_claim_helpers.go:145-172`).
- If the unqualified lookup returns a row, the helper overwrites channel, purpose,
  workspace ID, and last activity from that row
  (`internal/store/globaldb/global_db_task_claim_helpers.go:174-186`).
- Queue-time channel ensure uses the same unqualified helper and rejects a row whose
  returned workspace differs
  (`internal/store/globaldb/global_db_task_aux.go:835-847`).

**Concrete failure scenarios**

1. Workspaces A and B both have channel `default`. A claim for a workspace-B run can
   receive workspace-A channel purpose, workspace ID, and activity metadata if SQLite
   returns A's row for the unqualified query.
2. A workspace-B task explicitly requests `default`. Queueing can fail with
   "belongs to workspace A" instead of recognizing that B may legally own its own
   `(workspace_id, default)` row.

**Classification**

- The query defect is an observed fact.
- Which duplicate row SQLite returns is not a trusted ordering contract.
- Cross-workspace metadata disclosure and false ownership rejection are direct static
  consequences. This is P0 under AGH's workspace data-isolation policy even though the
  inspected projection does not include message bodies.

**Concise implementation implication:** every channel lookup in a workspace-scoped
path must use `(workspace_id, channel)`, and trusted task/run scope must never be
overwritten by an unqualified lookup result.

### BUG-02: First Undirected Local Thread Send Can Reach Zero Local Peers

**Observed causal chain**

1. `Manager.Send` prepares the envelope, persists it, and only then publishes/routes
   it (`internal/network/manager.go:680-712`).
2. Persisting a thread message upserts the sender as a thread participant
   (`internal/store/globaldb/global_db_network_conversations.go:71-149`;
   `internal/store/globaldb/global_db_network_conversation_summary.go:12-18`).
3. Router thread delivery reads persisted participants. Any non-empty participant set
   bypasses empty-thread activation/fallback
   (`internal/network/router.go:1241-1265`).
4. The first persisted participant can be only the sender.
5. Delivery filters the sender out
   (`internal/network/router.go:1625-1637`).
6. `agh ch send` creates an undirected message in its fixed public thread unless
   other routing metadata adds a recipient
   (`internal/api/core/agent_channels.go:121-185`).

**Concrete failure scenario**

A channel has several live local peers and no history for
`thread_agent_channel`. Peer A calls `agh ch send` without a directed target or
mentions. Persistence records A as the sole participant. Routing sees a non-empty set,
selects only A, then removes A as sender. The local delivery list is empty, so peers B
and C receive no local prompt from the initial message.

**Classification:** high-confidence static P1 defect. It can make the simplified
coordination path appear accepted and persisted while failing to activate any local
recipient. A dynamic regression scenario is still required.

### BUG-03: Network Task-Status Observer Is Dead in Normal Boot Order

**Observed causal chain**

1. Daemon boot runs `bootTasks` before `bootNetwork`
   (`internal/daemon/boot.go:210-225`).
2. Task boot calls `composeTaskEventObserver`
   (`internal/daemon/task_runtime_boot.go:43-60`).
3. Composition passes `state.network` to
   `newNetworkTaskStatusObserver`
   (`internal/daemon/task_event_bridge_notifier.go:106-138`).
4. At that point `state.network` is nil because Network has not booted.
5. The constructor returns nil when the Network runtime is nil
   (`internal/daemon/network_task_status_observer.go:75-83`).
6. The nil observer is installed in task runtime state
   (`internal/daemon/task_runtime_boot.go:80-91`).
7. Static repository search found shutdown/use sites but no post-Network rebind
   (`internal/daemon/task_runtime.go:44`;
   `internal/daemon/task_runtime_recovery.go:36-37`).

**Observed test gap**

- Observer unit tests construct the observer with an explicit fake runtime
  (`internal/daemon/network_task_status_observer_test.go:138-147`).
- No inspected boot-order integration test asserts a non-nil observer after normal
  daemon boot.

**Runtime effect**

- **Static inference:** automatic task status-back messages implemented by this
  observer are absent in normal boot, despite the rest of the task/channel machinery.
- This accidentally avoids some Network messages/model turns, but it is broken
  functionality, not an optimization.
- The dormant implementation formats status as conversational `say` traffic and sends
  it into the origin thread
  (`internal/daemon/network_task_status_observer.go:180-241,413-444`). Merely rebinding
  it would restore functionality by increasing envelopes and potential model activation.
- **Required disposition:** delete or replace it with a typed deterministic state
  projection structurally unable to call `PromptNetwork`, then prove normal-boot
  delivery and zero cognition in one integration test.

---

## 13. Lexical Blast Radius

The following counts are snapshot `rg -l` inventories supplied with the audit. They
measure files containing the named identifiers/text, not the number of semantic
dependencies.

### `coordination_channel_id|CoordinationChannelID`

| Surface | Files |
| --- | ---: |
| `internal/`, all matching files | 102 |
| `internal/`, production Go excluding `*_test.go` | 52 |
| `web/`, all matching files | 9 |
| `packages/site/` | 7 |
| `skills/agh/` | 0 |
| `docs/` | 2 |

### `network_channel|NetworkChannel`

| Surface | Files |
| --- | ---: |
| `internal/`, all matching files | 207 |
| `internal/`, production Go excluding `*_test.go` | 104 |
| `web/`, all matching files | 79 |
| `web/`, TS/TSX excluding tests and stories | 47 |
| `packages/site/` | 4 |
| `skills/agh/` | 3 |
| `docs/` | 5 |

Generated files were not universally excluded. For example, the OpenAPI-generated web
types contain many task/run channel projections
(`web/src/generated/agh-openapi.d.ts:4988-6241,28269-44581`). Any contract hard cut
therefore has a code-generation blast radius in addition to the handwritten files.

---

## 14. Tests and Docs That Encode the Current Behavior

This is a targeted freeze inventory, not an exhaustive test list.

| Behavior | Freezing evidence |
| --- | --- |
| Network enabled on first run | `internal/config/bootstrap_test.go:198-283` |
| Explicit disable preserved | `internal/config/config_test.go:2366-2395` |
| Daemon test default expects Network enabled | `internal/daemon/daemon_test.go:446-447` |
| Spawn inherits parent channel | `internal/session/spawn_test.go:142-143` |
| Detached work inherits owner channel | `internal/daemon/task_runtime_test.go:2110-2111` |
| Task start creates derived channel row | `internal/task/manager_integration_test.go:684-761` |
| Task approval creates derived channel | `internal/task/manager_integration_test.go:975-1020` |
| Scheduler examples require matching channel | `internal/scheduler/scheduler_integration_test.go:24-142` |
| Loop runtime generates a bounded channel | `internal/daemon/loop_runtime_adapters_test.go:57-85` |
| Channel-bound harness includes Network section | `internal/daemon/harness_context_test.go:461-470` |
| `agh ch reply` becomes direct | `internal/api/core/agent_channels_internal_test.go:176-224` |
| Repeated greets produce repeated audit rows | `internal/network/audit_test.go:213-254` |
| Site calls default channel the default for Network-enabled sessions | `packages/site/content/protocol/nats.mdx:147-158` |
| Config reference calls it the default Network channel for sessions | `packages/site/content/runtime/core/configuration/config-toml.mdx:595,1515` |
| Glossary requires a durable channel for every coordinated workspace run | `docs/_memory/glossary.md:243` |
| QA story defines channel creation as spawning sessions | `docs/qa/_seeds/feature-stories/04_analysis_network-bridges.md:53` |

These tests and documents must change with behavior. Keeping them unchanged while
altering only one runtime layer would preserve contradictory contracts.

---

## 15. Facts, Inferences, and Concise Implications

### Confirmed Current-State Facts

1. Network is administratively enabled by default.
2. Omitted session channel normally resolves to a non-empty default.
3. Non-empty session channel is the de facto participation switch.
4. Every queued workspace task run receives a coordination channel.
5. Scheduler eligibility and daemon-created task sessions consume that channel.
6. Loop public surfaces expose no Network choice while internal Loop sessions and
   coordinator runs create channel-bound state.
7. Every delivered full or digest Network prompt calls the agent-facing
   `PromptNetwork` path.
8. Network prompt/tool availability decisions are distributed across raw channel
   presence, byte budgets, global config, and global runtime dependencies.
9. The simplified `agh ch` surface changes conversation semantics rather than
   preserving caller intent.
10. Workspace-scoped task channel lookup is incorrectly unqualified.

### High-Confidence Inferences

1. Default session and child creation multiplies Network startup tokens, tool
   descriptors, subscriptions, heartbeats, and per-session first-delivery guidance.
2. Fan-out fallback can generate O(N) agent model calls from one message.
3. Synthetic task/Loop channel rows increase persistence and UI channel noise while
   making local scheduling impossible without further separation.
4. The first undirected send in a newly persisted local thread can be accepted but
   delivered to no local recipient.
5. Network task status-back is inactive in normal daemon boot due to construction
   order.

### Concise Implementation Implications

The later design should prove these invariants, regardless of its exact public shape:

- A local execution produces no Network channel row, join, subscription, heartbeat,
  Network prompt section, Network tool projection, Network situation read, or
  Network-based scheduler constraint.
- Administrative Network availability does not opt executions into participation.
- Participation is resolved once and persisted as concrete effective state; downstream
  packages do not independently infer it from channel presence.
- Workspace channel identity is always `(workspace_id, channel)`.
- Delivery admission and model activation are separate from message persistence.
- The first-message routing defect is fixed explicitly, and the dead conversational
  status observer is replaced by a deterministic zero-activation projection rather than
  rebound as-is.
- Because AGH is greenfield alpha, public contracts, persisted schemas, generated
  clients, CLI, UI, tools, hooks, docs, tests, and official skill should move in one
  hard cut without aliases or dual fields.

---

## 16. Current-State AGH Impact Audit

### AGH Impact Audit

- **Native tools:** Current participation is exposed through the full coordination
  toolset in `internal/tools/builtin/toolsets.go:25-46`; availability is gated by
  global Network dependencies in `internal/daemon/native_tools.go:528-538,660-737`,
  not session intent. Task native schemas repeat `network_channel` across create,
  child-create, update, and fan-out in
  `internal/tools/builtin/tasks.go:374-435,637-660`. A participation hard cut affects
  tool IDs only if tools are renamed, but always affects descriptors, input/output
  schemas, availability diagnostics, capability gates, schema digests, native-tool
  tests, and CLI/API fallbacks.
- **Extensibility and hooks:** Bundle activation can bind a primary channel as the
  effective default and thereby affect every later session
  (`internal/api/core/bundles.go:143-195`;
  `internal/bundles/service.go:679-723`). Automation task and Loop paths inherit the
  enforcement. Task status-back hooks/observers are intended to publish into Network
  but the observer is dead under current boot order. Extension resources, bundle
  manifests, hook payloads, task/Loop registries, bridge subscriptions, and config
  lifecycle are therefore affected surfaces, not optional follow-up work.
- **Workspace data isolation:** Session/task/run/Loop participation is workspace
  scoped, and Network channel storage correctly uses a composite workspace/channel
  key. The task helper violates that boundary with a channel-only query
  (`internal/store/globaldb/global_db_task_claim_helpers.go:189-201`). The current
  path can expose channel metadata across workspaces or falsely reject legal
  same-name channels. Future state must propagate `workspace_id` through
  CLI/HTTP/UDS, core, store, web, SSE/events, caches, hook payloads, and every lookup;
  two-workspace same-channel tests are mandatory.
- **Official AGH skill:** `skills/agh/SKILL.md:20-32` and
  `skills/agh/references/network.md:16-32` already present Network use as
  conditional, while `skills/agh/references/tasks-and-orchestration.md:70,144`
  assumes coordination channels in task work and onboarding references prescribe
  channel provisioning. The skill must co-ship with any public session/task/Loop,
  tool, environment-variable, or conversation-container change.

### Additional Cross-Surface Impact

- **Web:** Session creation and onboarding have no participation control; task and
  automation forms expose optional channel strings; Loop run has no control; Network
  settings describe global default join behavior. Generated OpenAPI types repeat the
  old fields extensively. A backend-only change would make the UI misleading or
  unable to express the new behavior.
- **CLI/HTTP/UDS:** Session, task, Loop, spawn, automation, bundle, and `agh ch`
  contracts all encode current semantics. Structured output must expose effective
  participation and source of resolution; omission cannot remain ambiguous.
- **Configuration lifecycle:** `network.enabled` currently controls manager boot and
  `network.default_channel` controls implicit participation. Settings GET/PATCH,
  config merge/load/save, restart diagnostics, bootstrap defaults, config docs, and
  bundle effective-default behavior are coupled to that split.
- **Docs and QA:** `packages/site`, `skills/agh`, institutional memory, generated
  API references, QA feature stories, and tracker rows encode "default join" and
  "bind every run." User-visible behavior changes require QA tracker rows to be added
  or reset to `untested` during implementation.

## Final Conclusion

Network is not an optional capability in the current execution model. It is a default
and, for workspace task runs and several Loop session types, a deterministic
requirement. The coupling is distributed rather than owned by one policy boundary:
configuration supplies defaults, API handlers resolve them, the store synthesizes
channels, scheduler and session runtime require them, prompt/tool systems infer from
them, and delivery turns envelopes into agent model calls.

The highest-priority correctness issue is the P0 workspace-unqualified channel lookup.
The first-send routing defect and dead status observer are also material because they
mean the expensive machinery is not consistently delivering the behavior it claims
to provide. Any opt-in redesign must therefore remove enforcement and repair Network
correctness together, while preserving an explicit, testable path for genuine
multi-agent collaboration.
