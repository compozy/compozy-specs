# AGH Network Changes: Implementation Research Roadmap

- **Status:** research recommendation, not an approved TechSpec
- **Evidence snapshot:** 2026-07-10
- **Hermes reference snapshot:** `.resources/hermes` at `b8880f1`

## 1. Purpose

This document converts the Network enforcement and token-cost audits into an executable program. It deliberately does not settle SQL layouts, public schema names, migration numbers, or provider-specific accounting details that require an approved TechSpec.

The program has three separate outcomes:

1. Ordinary sessions, tasks, loops, and automations run locally by default and incur zero Network participation cost.
2. Explicit Network participation separates durable mailbox delivery from model activation, with bounded and observable cost.
3. Multi-agent reasoning becomes an explicit execution strategy, not an accidental consequence of joining a channel.

The program must not become a compatibility bridge. AGH is a greenfield alpha: replace ambiguous fields and behavior in one co-shipped cut, delete their consumers, and do not add aliases, dual reads, fallback parsing, or inferred legacy modes.

## 2. Recommended Program

| Priority | Program | Outcome | Approval boundary |
|---|---|---|---|
| P0-A | Proven correctness remediation | Fix workspace isolation and delivery outcomes; replace dead conversational status-back with deterministic zero-activation projection | Narrow bug-fix batch |
| P0-B | Participation hard cut and minimum safe active admission | Public `off` default; internal mailbox/active semantics; deterministic finite active admission; no automatic run channels; lazy current transport | TechSpec 1 |
| P0-C | Stable mailbox addressing, durable delivery, single work authority, and non-off release gate | Mailbox-address wire/storage hard cut, fenced lease resolution, correct offline/first delivery, publish recovery, deduplication, work lifecycle authority, fixed coalescing/provider cancellation, then gated public `mailbox`/`active` exposure | TechSpec 2, separately approved |
| P1-A | Prompt and activation optimization | Fewer model turns, better coalescing/routing, compact context, and richer cost reconciliation over the P0-C-safe non-off boundary | TechSpec 3 |
| P1-B | Presence and control-plane diet | Stop treating presence and status chatter as model-facing conversation | May share TechSpec 3 if the ownership boundary stays small |
| P2 | MoA and swarm execution | Explicit advisor fanout plus one acting aggregator, with quality/cost gates | TechSpec 4 |

The dependency order is:

```text
P0-A correctness
      |
      v
P0-B participation contract -----> local/off value ships
      |
      +--------------------+
      v                    v
P0-C durability       P1 measurement baseline
      |                    |
      +----------+---------+
                 v
         P1 activation diet
                 |
                 v
         P2 MoA/swarm strategy
```

P0-B must not absorb a transport rewrite. P0-C must not be postponed as a vague optimization: the audit found correctness defects that satisfy the evidence gate for a dedicated durability TechSpec. P2 must not use Network chat as its internal orchestration bus.

## 3. Proven Correctness and Target-Contract Defects

These are observed code defects or mechanically proven blockers to the target mailbox contract, not
speculative optimization opportunities.

| ID | Defect and evidence | Required disposition |
|---|---|---|
| C-01 | Workspace-unqualified channel lookup. `coordinationChannelMetadata` knows the task workspace, but `networkChannelEntry` queries only `WHERE channel = ?` at `internal/store/globaldb/global_db_task_claim_helpers.go:145-201`. Two workspaces may legally use the same channel name, so the first matching row can project foreign metadata. | Independent P0-A fix. Qualify every channel lookup by `(workspace_id, channel)` and add a two-workspace same-name test. Do not wait for P0-B. |
| C-02 | First local public-thread send can activate nobody. `Manager.Send` persists before publish at `internal/network/manager.go:680-712`; persistence immediately adds the sender as a participant at `internal/store/globaldb/global_db_network_conversations.go:71-150` and `:152-168`; inbound routing then sees a non-empty participant set and filters the sender at `internal/network/router.go:1241-1265`. Empty-thread activation is skipped even though no receiver has participated. | P0-C structural fix. The routing decision for an envelope must use the participant snapshot that existed before that envelope's participant mutation, or the outbox dispatcher must carry a persisted routing decision. Add a first-message local fanout invariant. |
| C-03 | Network delivery discards ACP outcome and usage. `drainPromptEvents` ignores every event at `internal/network/delivery.go:885-899`, then the coordinator marks the message delivered. ACP exposes aggregate per-turn input, output, total, thought, cache, context, and monetary usage at `internal/acp/types.go:505-518`, but no provider-attempt/retry/fallback lifecycle. Fatal/error events are therefore not reflected in delivery success and real attempt usage is not attributable. | Split disposition. P0-A classifies terminal/error outcomes and never marks a failed prompt delivered. P0-B adds the minimum capability-negotiated provider-attempt budget/event extension, trusted adapter preflight, deterministic conformance, and one pinned real-adapter integration; P0-C makes it durable and restart/cancellation safe before active release. P1 adds price analytics and richer attribution, not baseline budget truth. |
| C-04 | Network work has two authorities. `Router` owns an in-memory `works` map at `internal/network/router.go:202-216` and mutates it at `:1066-1102`; SQLite independently opens and advances `network_work` at `internal/store/globaldb/global_db_network_work_mutation.go:14-62` and `:65-177`. No hydration or pruning path was found for the router map. Restart and persistence failures can make validation disagree with durable state. | P0-C structural fix. Make SQLite the authority and make routing consume a canonical transition result. Delete the in-memory lifecycle authority rather than synchronizing two state machines. |
| C-05 | Persist-before-publish can create a phantom accepted send. `Manager.Send` commits the conversation row before `PublishPrepared` at `internal/network/manager.go:690-710`. On publish failure, a retry with the same ID encounters the durable duplicate and returns success at `:699-704` without publishing. This contradicts the same-ID retry instruction at `skills/agh/references/network.md:137-140`. | P0-C structural fix. Use a transactional outbox or another durable accepted/published state machine. Do not patch this with delete-on-error or a second best-effort publish. |
| C-06 | The task Network status observer is constructed before Network exists. Boot calls tasks before Network at `internal/daemon/boot.go:210-225`; task boot composes the observer at `internal/daemon/task_runtime_boot.go:43-60`; composition captures `state.network` at `internal/daemon/task_event_bridge_notifier.go:106-138`; `newNetworkTaskStatusObserver` returns nil for a nil runtime at `internal/daemon/network_task_status_observer.go:75-83`. No later rebind was found. The observer would publish conversational `say` messages at `internal/daemon/network_task_status_observer.go:180-241,413-444`. | Independent P0-A fix. Delete or replace the dead observer with a typed deterministic status projection structurally unable to invoke `PromptNetwork`; do not merely bind the existing conversational path and make its token cost live. Wire the replacement only after dependencies exist and add a normal-daemon-boot integration test proving state projection plus zero model activation. |
| C-07 | Durable mailbox identity is not addressable under Network v0. Directed sends require live peer presence at `internal/network/router.go:354-379`; envelopes persist peer-shaped `From`/`To`/`Mentions` at `internal/network/envelope.go:196-218`; direct-room identity hashes process peer ids at `internal/network/validate.go:212-241`; thread participants and subscription preferences are also keyed by peer id at `internal/store/globaldb/global_db.go:163-176,349-402`. A rotating process peer therefore cannot be both ephemeral and the stable offline recipient. | P0-C protocol/storage/routing hard cut. Introduce stable mailbox addressing, derive direct rooms and durable relations from mailbox ids, map mailbox to a fenced live lease only after durable acceptance, and delete the presence-as-existence gate and all durable peer-id foreign-key semantics before mailbox or active is public. |

### P0-A completion rule

P0-A is complete only when all three narrow invariants are true:

- Channel metadata cannot cross workspace boundaries.
- Network prompt errors are observable and cannot increment successful delivery counters.
- A Network-origin task produces its typed status projection after normal daemon boot, and the
  projection path cannot emit a conversational envelope or invoke `PromptNetwork`.

Minimum actual usage capture and budget settlement belong to P0-B because `active` cannot be truthful without them. Rich cost attribution and analytics belong to P1. First local send, publish recovery, and work authority belong to P0-C because narrow patches would preserve the defective ownership model.

## 4. Target Participation Contract

### 4.1 One intent, one resolved snapshot

Introduce one domain type, provisionally named `NetworkParticipationSpec`, and one resolved execution snapshot. The TechSpec may improve the name, but it must preserve these semantics:

| Mode | Durable channel/mailbox | Network tools | Inbound model activation | Required resolved channel |
|---|---:|---:|---:|---:|
| `off` | No | No, except administrative status through an explicitly requested tool | Never | No; a channel is invalid |
| `mailbox` | Yes | Inbox/read/send tools allowed by policy | Never | Yes |
| `active` | Yes | Allowed by policy | Only when an activation rule and remaining budget both permit it | Yes |

Intent for `mailbox` or `active` must contain an explicit `channel_strategy`: `named` plus
`channel_id`, `default`, `run`, or `loop_run`. `default` asks the canonical resolver to consult the
configured effective channel only after mode selection; `run` and `loop_run` are valid only for
their owning execution kinds. The persisted snapshot always contains the selected strategy, a
concrete workspace-qualified channel, and a durable mailbox id for non-`off` modes. No downstream
package may independently infer a channel, derive `coord-<run_id>`, consult a task fallback, or
reinterpret an empty string.

The mailbox id and recipient cursor outlive a session process/live presence lease. Stop releases
subscriptions, heartbeat, and process peer identity; it does not delete the address, unread state,
or accepted offline envelopes. Explicit deletion or deterministic retention owns mailbox cleanup,
and restart reattaches to the same id/cursor.

Do not persist process `peer_id` inside the immutable participation snapshot. A separate mutable
live-lease projection owns lease id, current peer id, session/process id, presence, subscriptions,
expiry, and state. Restart can replace that projection while preserving the historical spec and
mailbox id.

The active-mode portion must carry a typed activation policy and budget snapshot. Minimum semantics:

- Activation triggers: directed, mentioned, explicitly requested capability match, or explicitly selected all-members policy.
- Recipient cap per envelope.
- Model-turn cap per run/session activation scope.
- Provider-attempt cap counting every initial, retry, fallback, and post-tool upstream call. The
  current ACP aggregate turn usage cannot prove this; active requires a capability-negotiated
  adapter extension that accepts a delegated sub-budget and emits attempt lifecycle records.
- Server-resolved adapter id/version/artifact digest/manifest digest/protocol version in the
  immutable active snapshot. Request/config input cannot self-assert this contract.
- Estimated prompt-token admission cap before a call.
- Actual input/output/cache/cost accounting after a call.
- If provider usage is unavailable, retain the conservative pre-call reservation as consumed and
  mark usage unavailable; never refund an unmeasured call or report the estimate as actual.
- Root activation id, maximum activation depth, wall-time deadline, and idempotent attempt id.
- Control/status envelopes are ineligible under every policy.
- Deterministic exhaustion behavior: remain durable in mailbox, record `budget_exhausted`, and do not prompt.

`network.enabled` remains an administrative availability switch. It is not an execution default. A non-`off` request while Network is unavailable must fail with a structured diagnostic before creating a session or run. Keeping `internal/config/config.go:754-769` enabled by default is acceptable only if an `off` execution creates no peer, subscription, greet, Network prompt section, Network tool projection, or model activation.

### 4.2 Resolution ownership

Resolve participation once at the execution boundary and persist the result:

| Surface | Intent owner | Resolution point | Snapshot owner |
|---|---|---|---|
| User session | Create request | Session creation before process start | Session record |
| Task | Task execution profile default plus optional run request override | Queue reservation transaction | Task run |
| Loop | Loop config plus run override | Loop start transaction | Loop run; node execution derives from that explicitly selected loop-run contract and may only narrow it |
| Automation task target | Job definition | Automation dispatch before task/run creation | Created task profile and task run |
| Automation loop target | Job/loop target | Automation dispatch before loop start | Loop run |
| Coordinator | Explicit coordinator profile plus optional concrete coordinator-session override | Coordinator session bootstrap | Coordinator session; never copied from an observed task/run |

The existing task execution profile is the natural task-owned home for intent: `internal/task/profile.go:52-63` already centralizes coordinator, worker, review, participant, sandbox, and runtime selection. Add the Network participation policy there rather than creating another detached task settings record. Persist the resolved value on `task.Run`, not only in profile JSON, so retries, recovery, hooks, and audit observe the exact execution contract.

### 4.3 Separate peer ingress from worker participation

`task.NetworkChannel` currently conflates two responsibilities:

- It binds Network peer ingress in `internal/network/tasks.go:290-342` and `:444-505`.
- It is copied into execution requests and runs, eventually forcing worker participation.

The hard cut must split them:

- `TaskNetworkIngressPolicy`: whether a remote Network peer may mutate/enqueue a task and from which workspace-qualified channel.
- `NetworkParticipationSpec`: whether the local worker/coordinator joins Network and whether inbound messages may activate it.

A task promoted from a thread may preserve `NetworkTaskThreadOrigin` for status-back without forcing its worker into `mailbox` or `active`. Status-back is a runtime notification link, not worker participation. A task with Network ingress enabled may still execute locally with mode `off`.

### 4.4 Coordinator independence

The coordinator is an orchestration role, not a Network role. Current bootstrap rejects a run without `CoordinationChannelID` at `internal/coordinator/coordinator.go:100-141`, grants Network channels at `:168-175`, injects Network commands at `:233-257`, and creates the session on that channel at `internal/daemon/coordinator_runtime.go:639-680`.

Required hard cut:

- Coordinator enablement remains explicit under autonomy/task profile settings.
- A coordinator may bootstrap for an `off` run and use task, spawn, context, and review surfaces without Network.
- The coordinator resolves its own participation from its coordinator profile/session override. A task run's `mailbox` or `active` mode never enrolls the singleton coordinator.
- Coordinator `mailbox` and `active` add only the corresponding Network permissions and prompt material authorized by that coordinator profile.
- Split the coordinator tool allowlist into a transport-independent base and a conditional Network extension.
- Never create a channel merely to satisfy coordinator bootstrap.

## 5. P0-B Vertical Slices

All slices belong to one approved hard-cut TechSpec and must co-ship before merge. The ordering below is an implementation order, not a compatibility period.

### Slice 0: Characterize the cost boundary

Before production edits, extend canonical suites to prove the current and target boundaries:

- A workspace task with no explicit Network selection currently creates a channel, as frozen by `internal/task/manager_integration_test.go:711-734`.
- A blank-channel ordinary session already avoids peer join via `internal/session/manager_helpers.go:123-151`; preserve that zero-participation behavior under an explicit `off` snapshot.
- Loop worker and judge sessions currently bind generated channels at `internal/daemon/loop_runtime_adapters.go:37-81` and `:118-161`.
- Channel binding currently selects the startup Network prompt section at `internal/daemon/harness_context.go:518-538`.

Replace the obsolete assertions in the same canonical suites when the hard cut lands. Do not create a parallel `network_opt_in_test.go` suite.

### Slice 1: Domain types and validation

Change the owning domain types first so compiler failures enumerate consumers:

- Add intent/resolved participation types and strict normalization in a dedicated task-independent package or a small Network contract file.
- Represent `coalesce_window` on JSON/UDS/hooks/generated-client wires as a validated canonical
  duration string such as `"250ms"`, not raw `time.Duration` nanoseconds; parse to runtime duration
  only after schema validation.
- Add the task profile field at `internal/task/profile.go:52-63`.
- Replace `Task.NetworkChannel`, `Run.NetworkChannel`, and `Run.CoordinationChannelID` at `internal/task/types.go:409-443` and `:468-499`.
- Replace request fields in `ExecutionRequest`, `EnqueueRun`, and `QueueRunReservation` at `internal/task/types.go:703-749`.
- Replace `session.CreateOpts.Channel` at `internal/session/manager_types.go:17-36` with the resolved participation snapshot.
- Extend loop config/request types at `internal/api/contract/loops.go:179-230` and the loop domain config.
- Replace `automation.JobTaskConfig.NetworkChannel` at `internal/automation/model/types.go:115-121`; add participation to loop targets too.

Validation must reject:

- `off` with channel, activation policy, or activation budget.
- Any participating request without a strategy, `named` without a channel, a non-`named` strategy
  with a channel, or a domain-incompatible strategy.
- `mailbox` with activation rules.
- `active` without a resolved channel, positive server-bounded logical-turn/provider-attempt/
  input/output/fanout/depth/wall-time/coalescing limits, or an agent profile advertising the
  required provider-attempt budget capability.
- `active` whose resolved provider has no administrator-trusted, exactly pinned adapter manifest,
  whose artifact fails pre-persistence verification, or whose live handshake differs from the
  resolved manifest. Do not downgrade it to mailbox.
- `active` with a late-bound/unknown worker provider. P0 requires one concrete adapter contract
  before owner persistence; dynamic pools remain off/mailbox or materialize separately resolved
  child runs.
- Any channel not qualified by the execution workspace.
- A run override that broadens task/workspace policy beyond configured maxima.
- A loop using `agh__network_send` or `channel_result` while resolved `off`; current Network-specific harvesting is visible in `internal/loop/action_channel_result.go` and must remain statically explicit.

### Slice 2: Persistence and destructive alpha migration

Append a new migration at the tail of `globalSchemaMigrations`; the current append-only registry is at `internal/store/globaldb/global_db.go:1037-1405`. Never edit, reorder, rename, or reuse an earlier migration.

The migration must rebuild affected SQLite tables as needed to:

- Remove `tasks.network_channel` and its index, currently declared at `internal/store/globaldb/global_db.go:675-721` and `:84`.
- Remove `task_runs.network_channel`, `task_runs.coordination_channel_id`, and their indexes, currently at `internal/store/globaldb/global_db.go:765-841` and `:1024-1025`.
- Replace `sessions.channel`, currently at `internal/store/globaldb/global_db.go:450-457`, with normalized participation columns or one validated structured representation plus required lookup indexes.
- Persist the resolved snapshot on loop runs and any automation-owned durable target that can initiate execution.
- Persist the resolved mailbox id in the internal participation snapshot, separate from live peer
  leases. P0-B does not claim that id is yet an addressable durable mailbox; P0-C owns the mailbox
  binding/cursor repository, v1 addressing, fenced lease resolver, and deletion/retention semantics.
- Add an outbox only in P0-C, not opportunistically in this migration.

Greenfield rule: do not infer `active` from old non-empty channel columns. The alpha migration should reset old ambiguous execution participation to `off`, remove obsolete auto-created `purpose='task_run_coordination'` channels that are no longer referenced by explicit Network artifacts, and document the destructive change. There must be no legacy read path.

### Slice 3: Queue and run creation

Replace automatic channel creation in `internal/store/globaldb/global_db_task_aux.go:741-793` and delete the derivation/insertion path at `:804-909`.

Queue reservation must:

1. Load the task's profile intent.
2. Apply a supplied run override if authorized.
3. Resolve Network availability, workspace, effective default channel if explicitly requested, policy, and caps.
4. Persist the concrete run snapshot in the same transaction as the run.
5. Create or validate a `network_channels` row only for `mailbox` or `active`.

For `off`, the transaction must not query or mutate Network channel tables.

Loop coordinator bookkeeping must not manufacture a Network channel. A loop with N nodes must not create N derived channels. If a loop is explicitly Network-participating, child runs inherit the one loop-run snapshot unless the approved DSL supports a narrower node-level policy.

### Slice 4: Session process and harness projection

Gate every Network effect on resolved mode, not a non-empty string:

- Environment projection currently sets `AGH_SESSION_CHANNEL` and `AGH_PEER_ID` from `session.Channel` at `internal/session/manager_start.go:678-713`.
- Activation currently calls peer join during session start at `internal/session/manager_helpers.go:77-104` and `:123-151`.
- Prompt sections currently use `ChannelBound` at `internal/daemon/harness_context.go:518-538`.
- Network turns currently add response-register guidance at `internal/daemon/harness_context.go:541-556` and `internal/daemon/network_response_register_prompt.go:47-92`.

Target behavior:

- `off`: no Network env, lifecycle join, heartbeat, Network prompt section, response augmenter, delivery queue, or Network tool descriptors.
- `mailbox`: durable mailbox identity and mailbox tools are available, but incoming envelopes never
  call the model. A live process lease may attach/detach without deleting the mailbox. Startup copy
  describes mailbox semantics only if that context is necessary.
- `active`: mailbox plus bounded activation. Do not repeat CLI tutorials in each turn.

Before any internal active invariant test calls `PromptNetwork`, a dedicated admission boundary must:

1. Reject control/status envelope kinds absolutely.
2. Validate trigger eligibility, root causation/depth, deduplication key, recipient ceiling, and
   deadline.
3. Atomically reserve positive logical-turn, provider-attempt, input/output, fanout, and time budget.
4. Resolve `network_provider_attempt_budget_v1` through a trusted adapter registry before owning
   execution state is persisted. Materialize and verify the exact adapter artifact; a raw command or
   self-declared config flag is not evidence. The live handshake must match the resolved
   adapter/version/protocol/artifact/manifest contract.
5. Pass an activation/turn/deadline/sub-budget envelope to the adapter. The adapter consumes that
   sub-budget before every upstream provider call and emits stable attempt-start/terminal evidence
   for initial, retry, fallback, and post-tool calls.
6. Classify ACP terminal/error events instead of discarding them at
   `internal/network/delivery.go:885-899`.
7. Settle each attempt's outcome and actual usage; an explicit unavailable marker or missing
   terminal evidence keeps the conservative allocation consumed and never masquerades as actual.

This is minimum P0-B safety, not the P1 routing/coalescing optimization.

### Slice 5: Loop runtime

Remove unconditional generated channels from loop action and judge sessions at `internal/daemon/loop_runtime_adapters.go:48-55` and `:126-133`.

Required behavior:

- Local loop action/judge sessions inherit `off` and never join Network.
- Explicitly participating loops receive the loop-run resolved snapshot.
- `channel_result` is available only when the definition and run are non-`off`.
- Child `run-loop` execution inherits or narrows the parent policy; it cannot silently broaden it.
- Cancellation and reaping close live mailbox/active leases without leaving subscriptions or
  heartbeats; durable mailbox binding/cursor deletion remains an explicit retention operation.

### Slice 6: Coordinator, scheduler, and task roles

- Remove `DecisionMissingChannel` as a bootstrap requirement in `internal/coordinator/coordinator.go:100-141`.
- Build coordinator permissions and prompt overlay from the coordinator session's own resolved mode, never from an observed run.
- Ensure scheduler-spawned workers and task-role sessions inherit the run snapshot rather than a task/channel fallback.
- Delete the channel-derived task-role session name and unconditional fictional coordination line at
  `internal/daemon/task_role_sessions.go:260-262,286-300`. Off task roles derive identity from
  role/run/profile and render no Network line; participating roles render only the resolved channel.
- Remove fallback channel selection in `internal/task/manager_run_hooks.go:180-188`.
- Replace `coordination_channel_id` in situation context at `internal/situation/task_context.go:508-526` with the structured participation snapshot only when non-`off`.
- Keep task/run IDs as local coordination correlation. Do not invent a replacement local channel ID.

### Slice 7: Public and extension surfaces

Co-ship all inputs and outputs:

- HTTP and UDS contracts.
- CLI session/task/loop/automation flags for preset, strategy/channel, repeatable triggers, logical
  turns, provider attempts, input/output tokens, fanout, depth, wall time, coalescing, and exhaustion
  behavior. A canonical structured activation-policy input is acceptable only if it is generated
  from and validated against the same contract; the CLI may not expose a smaller policy than
  HTTP/Web.
- Native task, loop, session, and Network tool schemas.
- Web forms and read models.
- Hook matchers and payloads.
- Bundle/resource projections and bridge SDKs.
- Official AGH skill and site docs.
- Generated OpenAPI and TypeScript clients.

No surface may accept `network_channel` or `coordination_channel_id` as an alias after the cut.

### Slice 8: Removal and proof

Run repository-wide searches for every delete target in Section 11. Any remaining occurrence must be either:

- A Network protocol/channel concept that is still valid.
- A migration statement that intentionally removes the old column.
- Historical research outside shipped runtime/docs.

Anything else blocks completion.

### P0-B release gates

The domain model contains all three target modes, but public exposure is gated independently:

| Mode | Earliest public exposure | Required proof |
|---|---|---|
| `off` | P0-B | No channel, peer, transport lease, prompt, environment, tool projection, or model activation for every ordinary execution surface |
| `active` | After P0-C and the P0-B active-admission gate | Stable mailbox-address routing and fenced live lease; deterministic eligible triggers; control/status events ineligible; finite recipient/turn/attempt/input/output/depth/time ceilings; at least one AGH-controlled pinned real adapter with pre-persistence manifest/artifact verification, live handshake, deterministic conformance, and credentialed provider smoke evidence; per-attempt terminal outcome/usage; durable delivery/recovery; fixed bounded coalescing; provider cancellation or definitive late-work accounting |
| `mailbox` | After P0-C and the zero-activation gate | Stable mailbox-address/direct/mention/subscription/work identity, correct ordering/dedup/cursors, offline delivery without presence across process-lease exit/restart, stale-lease fencing, and proof that every envelope kind causes zero `PromptNetwork` calls |

P0-B also owns lazy startup of the existing transport. `network.enabled=true` may make Network
available, but no broker/listener/peer lifecycle starts until an administrative Network operation
or the first explicitly participating execution requires it. This is a lifecycle change only;
transport replacement and outbox semantics remain P0-C.

The hard-cut OpenAPI/domain schema may represent mailbox and active in P0-B for internal testing,
but ordinary CLI/Web create controls and public docs expose only Local until P0-C passes the
corresponding release gates. They MUST NOT alias an activating implementation as mailbox or expose
active over known first-send/publish-recovery defects.

### P0-B real-adapter decision and preflight

TechSpec 1 must select and ship one active-capable adapter controlled by AGH at the upstream
provider-call boundary. Compare at least a first-party, exactly pinned variant of the current
`pi-acp` harness with a first-party adapter for one supported provider. A generic ACP stdio proxy is
not a candidate because retries/fallbacks inside an opaque agent remain invisible. The current
provider registry launches direct external ACP commands or `pi-acp`, including an unpinned
`npx -y pi-acp@latest` (`internal/config/provider.go:75-83,284-286,370-520`); none may be marked
active-capable by assumption.

The selected design must introduce a trusted provider-adapter registry with an immutable manifest:
adapter id, exact version, executable artifact digest, manifest digest, attempt-budget protocol
version, supported provider/model boundary, and conformance revision. A raw `ProviderConfig.Command`
at `internal/config/provider.go:186-204` cannot self-declare capability. Extension adapters use the
same administrator-authorized registry, artifact verification, and conformance contract.

Resolution materializes/verifies that artifact before persisting an active session/run and stores
the resolved contract in the participation snapshot. The live initialize handshake must match it
before ACP session negotiation, Network membership, or activation. This requires reordering the
current path, which opens storage and writes session metadata before `Driver.Start`, while initialize
is where capabilities are learned (`internal/session/manager_start.go:170-235`;
`internal/acp/client.go:182-208,398-430`). Insert pure artifact preflight before storage open and live
contract verification between initialize and ACP session negotiation. Do not move process launch
ahead of owner commit or retain a staged process across that commit. Deferred runs validate the
manifest before run persistence and repeat the live match before their worker joins.

P0-B supports active only when resolution produces one concrete agent/provider/adapter contract.
Do not make a dynamic worker pool pass by checking capability after it claims the run. Such work must
use off/mailbox or materialize children whose providers are independently resolved before their
records commit.

The P0-B adapter gate has two evidence lanes: a deterministic conformance suite with a controllable
upstream that forces initial/retry/fallback/post-tool/missing-usage/over-budget cases, and a
credentialed smoke journey through the exact pinned artifact against at least one real provider.
Mocks remain useful but cannot be the only proof. Provider list/status, HTTP/UDS, native management
tools, and Web agent/provider selectors expose `network_active_support`, verified contract, last
conformance status, and typed unavailable reason without starting a Network execution.

#### P0-B startup ownership and fault matrix

The pre-persistence adapter preflight owns only a registry cache transaction. It MUST NOT launch ACP,
inject credentials, prepare a sandbox, open stdio, or acquire a Network lease. Adapter cache
materialization either publishes a complete digest-verified immutable entry or deletes its partial
state.

After owner commit, the session lifecycle owns every acquired resource from the moment it is
created. Install cleanup before each acquisition, transfer ownership explicitly, and run teardown
under an uncanceled bounded context. Required injected failures:

| Injected failure/cancel point | Required state and cleanup |
|---|---|
| Partial adapter download/verification | No owner; no cache partial; reservation released |
| Storage open or owner commit after successful preflight | No owner; no process/stdio/secrets/sandbox/lease; reservation released |
| Provider secret/runtime materialization, sandbox prepare/ready, or pre-start hook | Failed-start owner only after commit; clear redactions/runtime material; finalize sandbox; close recorder; no process/lease |
| Process launch or initialize after a valid manifest | Failed-start owner; stop/wait process; close stdio/connection/process record; clear provider/sandbox material; no lease |
| Live contract mismatch | Failed-start owner with typed mismatch; same complete cleanup; no downgrade or Network side effect |
| ACP session negotiation after matching initialize | Failed-start owner; same complete cleanup; no membership/activation |
| Context cancellation or daemon shutdown at every boundary | Deterministic failed/canceled owner; idempotent cleanup; zero orphan PID/watcher/process record/lease |

Implementation evidence must cover ACP stop/failed-start seams at
`internal/acp/client.go:731-738,832-886`, manager cleanup at
`internal/session/manager_lifecycle.go:485-510`, provider redaction cleanup at
`internal/session/manager_start.go:198-212,253-257`, and sandbox preparation/finalization beginning at
`internal/session/sandbox.go:69-145`. The TechSpec must name any additional owner needed for hosted
MCP, provider runtime directories, secret material, process registry handles, and sandbox instances.

## 6. P0-C Stable Addressing and Durable Delivery TechSpec

P0-C is a separate architecture change. The audit evidence is sufficient to author the TechSpec, but implementation must compare at least these two designs:

1. Keep NATS as the external transport, add a SQLite transactional outbox, and dispatch daemon-local recipients in process after commit.
2. Introduce a transport interface with an in-process local transport and retain NATS only for cross-process/cross-daemon delivery, backed by the same outbox.

Do not select design 2 solely because local NATS looks redundant. Select it if crash tests and the N=1/3/10/50 benchmark show a simpler correctness model or material latency/resource benefit.

### Required state machine

The design must define observable states, for example:

```text
outbound_queued (sender conversation + outbox committed)
  -> publishing
  -> published
     -> route_refresh_required -> publishing
     -> home_rejected_terminal (tombstoned | unknown | forbidden | envelope_invalid | envelope_expired)
     -> home_accepted (recipient mailbox + delivery committed)
        -> mailbox_unread | mailbox_consumed
        -> activation_pending | activation_skipped
        -> activation_running | cancel_requested
        -> activation_succeeded | activation_failed | activation_canceled | terminal_after_cancel
  -> retry_exhausted_waiting_deadline
     -> delivery_dead_lettered_unconfirmed (sender-local; no home disposition)
```

Names are TechSpec-owned. Required invariants are not:

- A send response distinguishes sender-side durable queued state from recipient-home durable
  acceptance. It never calls transport publish alone delivered/accepted-at-mailbox.
- A publish failure leaves a retryable outbox record.
- Same-ID retries return the durable current state and cannot mask an unpublished envelope.
- At-least-once dispatch plus idempotent receipt is explicit; do not promise impossible exactly-once transport.
- Local recipients receive the first thread message according to the pre-message participant snapshot.
- Timeline deduplication does not suppress delivery merely because the outbound side already persisted the envelope.
- A stopped mailbox process releases its live lease but remains addressable; an offline message is
  durable, and restart reattaches the same mailbox id/cursor without loss or duplicate consumption.
- Network v1 conversation/work addressing uses stable mailbox ids for sender, directed recipient,
  resolved mentions, direct-room pair, thread participants, subscription policy, work ownership,
  delivery/idempotency keys, and usage. Process peer/lease ids are optional observations only.
- The alpha schema migration resets v0 peer-keyed collaboration rows that cannot be authoritatively
  mapped. It does not fabricate mailbox ids from peer strings, dual-write versions, or retain a v0
  reader; only independently valid channel resource definitions may survive if the TechSpec proves
  their semantics unchanged.
- Send validation checks durable mailbox registration/authority, not live presence. An absent lease
  stores unread mailbox state; unknown or tombstoned mailbox identity returns a typed error.
- Direct-room identity is stable across peer/lease rotation because it is derived from the sorted
  mailbox pair. A deleted mailbox id is never reused.
- Durable mailbox-to-live-lease resolution happens only after mailbox acceptance and uses a monotonic
  lease epoch/fencing token. A replaced lease cannot consume, acknowledge, change subscription
  policy, or settle activation.
- For a remote mailbox home, sender outbox retries until the home runtime durably acknowledges the
  mailbox record or returns an authenticated terminal rejection. Remote transport unavailability or
  route refresh cannot be reported as mailbox delivery; a terminal recipient disposition stops retry.
- Every remote envelope has immutable authenticated `ts` and `expires_at`, server-clamped and
  persisted before dispatch. The home independently bounds the interval and validates expiry before
  mailbox commit; an invalid/overlong interval yields terminal `envelope_invalid`, while arrival
  after the valid deadline yields home-authenticated `envelope_expired`. Neither can create unread
  state or activation. The TechSpec defines the time unit, cross-runtime clock source, skew allowance,
  boundary comparison, and deterministic validation precedence: authenticated home/route authority;
  envelope/deadline validity and expiry; then mailbox permission/existence/tombstone state.
- Mailbox home serializes delivery with tombstone in one store order. Delivery committed first keeps
  its idempotent accepted disposition; tombstone committed first yields durable
  `recipient_tombstoned` even when the sender queued earlier. Sender-local time is not acceptance.
- Home persists one immutable accepted/rejected disposition keyed by mailbox/message and returns the
  exact id/digest on retry/restart. Its committed endpoint epoch is historical evidence; the current
  endpoint separately re-attests that canonical digest with its serving endpoint epoch. The sender
  validates the current authoritative directory lease, home/runtime/mailbox/message/route binding,
  immutable disposition, serving epoch, and proof before transitioning its outbox.
- Sender outbox has server-bounded retry attempts and age. Attempt exhaustion stops active dispatch
  but waits through the envelope deadline and skew allowance before recording sender-local
  `delivery_dead_lettered_unconfirmed`. That state ends retry without fabricating a home rejection;
  a later authentic disposition may refine recorded ground truth but never triggers republish.
  Envelope id, sender dead-letter, home disposition, and tombstone retention cover the retry/replay
  plus settlement-grace window, and mailbox/message ids are never reused.
- `home_runtime_id` is a persistent installation identity for one `AGH_HOME`, authenticated through
  a runtime key/reference and independent of PID/hostname/port. Restart re-registers a higher fenced
  endpoint epoch for the same id; mailbox home/route does not change. The persistent identity key
  authorizes registration, while each live epoch has a fresh endpoint credential invalidated on
  replacement so an old process cannot forge a current-epoch presentation.
- SQLite is the only `network_work` lifecycle authority.
- Daemon restart hydrates pending outbox work and does not hydrate a second work state machine.
- Generated receipts and traces use the same durable path as user sends.
- A fixed bounded coalescing window admits at most one activation attempt per target/root/batch key;
  burst envelopes remain individually durable and queryable.
- Cancellation propagates to underlying ACP/provider work when the provider contract supports it.
  Otherwise the attempt remains charged and observed until terminal/deadline, and a late result is
  stored as evidence without reactivating the actor.

### P0-C ownership map

| Concern | Current location | Target ownership |
|---|---|---|
| Outbound validation | `internal/network/router.go:354-427` | Pure routing/validation service |
| Envelope recipient identity | `internal/network/envelope.go:196-218`; `internal/network/router.go:995-1027` | Versioned stable mailbox-address contract; no v0 peer-address aliases |
| Mailbox binding and lease resolution | Current peer/presence registry in `internal/network/peer.go`; live-presence gate at `internal/network/router.go:372-378` | Durable workspace/channel/mailbox registry plus separate fenced mailbox-to-live-lease resolver; presence is never recipient existence |
| Runtime home identity/directory | No canonical installation-scoped Network runtime identity was found; current bridge/process instance ids are different scopes | Persist one authenticated runtime id per `AGH_HOME`; register fenced expiring endpoint leases; resolve remote home without deriving identity from PID/host/port |
| Home ACK/NACK and tombstone ordering | No recipient-home durable disposition state | Home-transaction-ordered, authenticated, idempotent accepted/terminal-rejected dispositions plus retryable route refresh; sender outbox settles instead of retrying forever |
| Direct-room identity | `internal/network/validate.go:212-241`; `internal/store/globaldb/global_db.go:382-402` | Versioned identity over sorted stable mailbox ids and mailbox-keyed room rows |
| Thread/mention/subscription/work ownership | `internal/network/router.go:1241-1265`; `internal/store/globaldb/global_db.go:163-176,349-430`; `internal/store/types.go:959-1010` | Stable mailbox foreign keys; optional peer/lease observation columns only |
| Conversation transaction | `internal/store/globaldb/global_db_network_conversations.go:71-150` | SQLite repository, including outbox enqueue |
| Publish | `internal/network/manager.go:680-752` | Recoverable dispatcher |
| Local delivery | NATS callback through `internal/network/manager.go:1108-1158` | Explicit local dispatch after durable acceptance |
| Deduplication | Timeline row plus router `seen` map | Durable message/delivery identities with bounded in-memory acceleration only |
| Work lifecycle | Router map plus SQLite | SQLite transition repository only |
| Delivery activation | `internal/network/delivery.go` | Durable activation-attempt scheduler consuming mailbox records with fixed coalescing, deadline, cancellation, and late-result policy |

## 7. P1 Token and Model-Call Optimization

P1 implementation starts only after P0-B establishes minimum truthful admission/accounting and P0-C
establishes durable non-off delivery/recovery. Baseline instrumentation may be prepared earlier.

### P1.1 Separate persistence, delivery, and cognition

An envelope reaching a session must not imply a model call. The current worker invokes `PromptNetwork` for each queued batch at `internal/network/delivery.go:636-688`; only digest items can coalesce at `:691-708`.

Target pipeline:

```text
validated envelope
  -> durable conversation/mailbox
  -> deterministic routing/subscription decision
  -> activation admission (mode + trigger + budget + busy state)
  -> coalesced model turn, if admitted
```

Receipts, traces, presence, task-status transitions, subscription changes, and delivery
acknowledgements update deterministic projections and UI without waking a model under every Network
policy. They are not eligible triggers. A human may react to a projection by starting a new explicit
local prompt, but no policy may promote the control envelope itself into model authority.

P0-C already supplies fixed bounded coalescing and cancellation semantics required for safe public
active. P1 improves window selection, cross-sender digest quality, routing efficiency, and prompt
shape; it must not be the first implementation of those safety invariants.

### P1.2 Replace per-envelope prompt tutorials

Current formatting expands structured JSON into base64 inside XML at `internal/network/delivery.go:1319-1447`, then appends response-register text, reply flags, causation, trace, work rules, protocol guidance, and worked CLI examples at `:1647-1868`. Startup/turn guidance duplicates part of this at `internal/daemon/network_response_register_prompt.go:20-92`. Network tool descriptors repeat schemas again at `internal/tools/builtin/network.go:12-273` and `:279-340`.

Required redesign:

- Put stable behavior in one cached startup/tool-policy source.
- Deliver metadata through structured `PromptNetworkMeta` or a typed native event, not verbose prose.
- Deliver plain text as plain text and bounded structured JSON as JSON; do not base64 JSON for the model.
- Include only the current message body, sender/container IDs, trust, why activation happened, and a short response expectation.
- Let native tool schemas carry send arguments. Use tool discovery/help on demand rather than embedding shell commands.
- Keep the stable system/tool prefix byte-identical across turns to preserve provider prompt caching.
- Set a behavioral budget for wrapper text excluding message body; initial target: no more than 256 estimated tokens for a repeated active delivery and no worked CLI example.

### P1.3 Activation policy

Current empty-thread routing can full-activate up to three lexical capability matches and send digest delivery to the remainder at `internal/network/router.go:1268-1329` and `:1422-1464`; matching is substring scoring at `:1514-1583`. Existing participants receive full delivery at `:1241-1265` without an activity expiry.

Recommended defaults:

- `off`: zero recipients.
- `mailbox`: zero model recipients; all eligible mailboxes remain queryable.
- `active`: directed and mentioned recipients only by default.
- Capability expansion is explicit, starts at K=1, and expands only when configured confidence/disagreement rules justify it.
- All-members activation is explicitly high-cost and server-capped.
- Participant-based full activation expires or requires an active subscription; historical participation alone is insufficient.
- Lookup failures fail closed to mailbox, not open to full activation.
- Sparse digest traffic waits for the configured coalescing window; the current `hasDigestBatchCandidate` guard at `internal/network/delivery.go:691-708` prevents the first sparse item from waiting.

### P1.4 Presence and status control plane

Every joined peer immediately and periodically publishes a greet at `internal/network/manager.go:821-865`, and every greet is persisted as conversation traffic at `:868-881`. Capability briefs carry every capability ID and summary at `internal/network/capability_brief.go:18-50`. This is control-plane state, not conversation.

Change direction:

- Store peer leases/presence outside the model-facing timeline.
- Publish capability digests on change; fetch full capability detail through whois/catalog lookup.
- Do not persist unchanged heartbeat greetings as conversation messages.
- Batch audit writes and generated protocol messages where durability allows.
- Make task status-back a durable runtime notification that is ineligible for model activation under
  every policy and never activates any thread participant.

### P1.5 Rich attempt cost attribution and budget analytics

Current delivery estimates tokens as bytes divided by four at `internal/network/delivery.go:729-742`, hardcodes trust to `untrusted` at `:744-780`, and records only estimated thread prompt cost at `internal/network/manager.go:1930-1967`.

P0-B adds the minimum capability-negotiated provider-attempt events and conservative/actual
settlement needed for truthful internal admission. P0-C makes those events durable and
restart/cancellation safe before active release. P1 enriches that foundation with:

- Input, output, thought, cache-read, cache-write, and total tokens.
- Catalog/pricing metadata and normalized fallback-chain analytics for the provider/model identity
  already captured by the minimum attempt event; monetary amount/currency remains nullable unless
  trustworthy price data is available.
- Workspace, channel, thread/direct container, envelope, sender peer, target peer, session, task, run, activation reason, policy, and batch.
- Model call outcome and retry count.
- Budget decision before and after the call.

Estimates remain useful for admission before a provider call, but dashboards and budget reconciliation must use actual usage when available.

## 8. P2 MoA and Swarm Strategy

P2 is an execution strategy above sessions/tasks/loops. It is not a Network protocol extension and must default off.

### 8.1 Adopt the useful Hermes shape

Hermes provides concrete patterns worth copying, with AGH-specific budget hardening:

- MoA is intended to be one-shot or explicitly selected, not silently enabled:
  `.resources/hermes/website/docs/user-guide/features/mixture-of-agents.md:32-48`. The gateway's
  `finally`-safe restore is the reliable surface (`.resources/hermes/gateway/run.py:10329-10348`);
  direct CLI/TUI restore can leak after exceptions (`.resources/hermes/cli.py:12338-12369`,
  `.resources/hermes/tui_gateway/server.py:9031-9081`). AGH should use immutable per-run resolution,
  not mutable switch-and-restore state.
- One aggregator is the acting model; references advise and do not receive tools: `.resources/hermes/agent/moa_loop.py:93-118` and `.resources/hermes/website/docs/user-guide/features/mixture-of-agents.md:9-13`.
- References fan out in parallel with a hard concurrency cap of eight: `.resources/hermes/agent/moa_loop.py:21-27` and `:336-383`.
- Failed references degrade to labelled notes instead of aborting the acting model: `.resources/hermes/agent/moa_loop.py:245-249` and `:324-333`.
- Reference context drops the large acting system prompt, flattens tool actions, and bounds each
  tool-result preview: `.resources/hermes/agent/moa_loop.py:83-91` and `:388-502`. This reduces the
  coordination surface but does not impose a total input cap or preserve non-string multimodal
  content.
- Reference calls use provider prompt-cache controls; source comments report a prior run with zero
  cache reads, but the inspected tree does not contain the raw benchmark artifact:
  `.resources/hermes/agent/moa_loop.py:260-272`.
- Reference and aggregator usage are attributed to their configured slots in the no-fallback path,
  rather than all fan-out being priced at one rate: `.resources/hermes/agent/moa_loop.py:30-45,280-323,804-814,1013-1022`
  and `.resources/hermes/agent/conversation_loop.py:2153-2167`. Auxiliary fallback can execute a
  different provider/model while accounting retains the configured slot
  (`.resources/hermes/agent/auxiliary_client.py:6905-6952`), so AGH must record actual attempts instead
  of copying that limitation.
- A state hash avoids charging cached reference work twice, and `user_turn` cadence can avoid rerunning advisors on every tool iteration: `.resources/hermes/agent/moa_loop.py:847-925`.
- Advisor output can be capped independently; the Hermes snapshot reports latency correlation and a sample reduction at `.resources/hermes/agent/moa_loop.py:815-825`.

### 8.2 AGH-specific design

Recommended initial MoA contract:

- Explicit named MoA execution preset, in the separate model-strategy namespace, with ordered
  advisor model slots and one aggregator slot. This is not a named Network participation policy.
- Default off for sessions, tasks, loops, automations, and scheduled work.
- Advisors are stateless, tool-less, Network-off calls over a typed, bounded advisory snapshot.
- Advisor admission applies tenant/workspace data classification, provider allowlists, redaction,
  secret filtering, and egress audit. A compact safety/authority envelope remains present even when
  the actor's large provider-specific system prompt is omitted.
- Multimodal/structured parts are projected through typed content or rejected explicitly; they are
  never silently flattened to an empty string.
- Aggregator is the only acting session, tool caller, task owner, and optional Network participant.
- Bounded parallelism, advisor output tokens, total advisor input/output tokens, total monetary budget, wall clock, and failure quorum.
- `user_turn` cadence by default; per-iteration advice is an explicit high-cost choice.
- Early stop when enough advisors agree, confidence is sufficient, or marginal quality no longer justifies another advisor.
- A typed blackboard/artifact store carries facts and outputs; prompts carry compact summaries and versioned pointers.
- The final synthesizer receives independent outputs, provenance, disagreement markers, and selected evidence, not an all-to-all chat transcript.

This matches the second-brain research principle to start with one agent and add multi-agent execution only after an empirical limit at `/home/pedronauck/Projects/second-brain/research/agent-networks/wiki/concepts/Multi-Agent Orchestration Patterns.md:31-50`. It uses concurrent fan-out/fan-in for independent perspectives at `:235-251`, while avoiding supervisor translation overhead documented at `:133-140` and unbounded handoffs warned against at `:253-271`.

For context transfer, prefer typed snapshots and pointers over prose chat. The research corpus recommends structured state at `/home/pedronauck/Projects/second-brain/research/agent-networks/wiki/concepts/Agent Handoff and Context Transfer.md:89-101`, records the cost of lossy paraphrase there, and recommends pin-and-trim semantics at `:199-216`.

### 8.3 Do not copy blindly

AGH should improve on these Hermes tradeoffs:

- Do not leave advisor output uncapped by default; Hermes explicitly permits uncapped references at `.resources/hermes/website/docs/user-guide/features/mixture-of-agents.md:103-116`.
- Do not wait for every advisor when a quorum/quality threshold is satisfied; Hermes currently collects all futures at `.resources/hermes/agent/moa_loop.py:380-383`.
- Do not inject raw advisor essays into the aggregator indefinitely; persist typed outputs, summaries, and pointers.
- Do not replicate broad conversation/tool-result history to every configured vendor without a
  workspace privacy policy and per-provider egress decision.
- Do not turn advisors into Network peers. Direct invocations and task/run events avoid Network
  session, presence, routing, and protocol overhead; end-to-end cost and quality still require a
  benchmark against the single-actor baseline.
- Do not claim quality improvement without a task-specific evaluation set and cost-normalized baseline.
- Do not make safe cadence/output limits YAML-only. Hermes recognizes `reference_max_tokens` and `fanout` in `.resources/hermes/hermes_cli/moa_config.py:125-148`, but its management payload and PUT reconstruction omit them at `.resources/hermes/hermes_cli/web_server.py:968-996,5505-5541`; AGH must co-ship every budget through config, CLI, HTTP/UDS, generated clients, and UI.

## 9. Explicit File and Surface Change Map

This is the minimum expected blast radius for TechSpec planning.

| Area | Current evidence | Required target change and phase |
|---|---|---|
| Task domain | `internal/task/types.go:409-499`, `:703-749`; `internal/task/profile.go:52-63` | Structured profile intent and resolved run snapshot; separate Network ingress policy |
| Task storage | `internal/store/globaldb/global_db_task_aux.go:741-909`; `internal/store/globaldb/global_db_task.go`; `internal/store/globaldb/global_db_task_insert.go` | Delete auto channel derivation; persist exact snapshot; workspace-qualified channel access |
| Task claims/hooks | `internal/store/globaldb/global_db_task_claim_helpers.go:145-201`; `internal/task/manager_run_hooks.go:131-188` | Qualified lookup; structured hook payload; no fallback |
| Session domain/store | `internal/session/manager_types.go:17-36`; `internal/store/globaldb/global_db_session.go:90-90`, `:380-402`, `:663-714` | Replace bare channel with mode/snapshot and persist it |
| Session lifecycle | `internal/session/manager_start.go:678-713`; `internal/session/manager_helpers.go:77-151` | Gate env/join/leave on mode |
| Harness prompts | `internal/daemon/harness_context.go:518-556`; `internal/daemon/network_response_register_prompt.go:20-92` | Mode-aware section/tool/augmenter selection; `off` has zero Network projection |
| Task-role sessions | `internal/daemon/task_role_sessions.go:260-262,286-300` | Remove channel/default identity fallback and unconditional `Coordination channel`; render Network context only from participating resolved state |
| Coordinator | `internal/coordinator/coordinator.go:100-175`, `:233-257`; `internal/daemon/coordinator_runtime.go:639-680` | Local coordinator path and conditional Network extension |
| Autonomy/agent kernel | `internal/tools/builtin/autonomy.go:13-94`; `internal/cli/agent_kernel.go:101-375`; autonomy site pages | Preserve transport-independent lease/safe-spawn authority; gate `agh ch` use/guidance on participation; delete `bind always` doctrine |
| Loop runtime | `internal/daemon/loop_runtime_adapters.go:37-161`; `internal/loop/action_channel_result.go` | Inherit resolved mode; no unconditional channels; validate Network-only actions |
| Loop public contract | `internal/api/contract/loops.go:179-230`; `internal/api/spec/loops.go`; `internal/api/spec/loop_schemas.go` | Add run/config participation and expose resolved snapshot |
| Automation | `internal/automation/model/types.go:115-132`; `internal/automation/dispatch.go:661-691`, `:1182-1189` | Replace copied channel with explicit task/loop participation intent |
| API task/session contract | `internal/api/contract/tasks.go:26-76`, `:179-230`; `internal/api/contract/contract.go:35-45`, `:105-116` | Delete legacy fields and add typed intent/resolved payloads |
| API handlers/spec | `internal/api/core/tasks.go`; `internal/api/core/handlers.go:128-139`; `internal/api/spec/spec.go`; HTTP/UDS route packages | Parse, validate, render, and error-map the new contract identically |
| Network v1 public addressing | `internal/api/contract/contract.go:812-873,892-910,1038-1070,1124-1194`; `internal/network/envelope.go:196-218` | P0-C hard cut to mailbox sender/recipient/mentions, mailbox-keyed subscription/direct/work projections, optional separately labeled live lease evidence, and regenerated OpenAPI/clients |
| Network durable identity schema | `internal/store/globaldb/global_db.go:163-176,349-430`; Network token/audit/delivery tables and repositories | P0-C rebuilds peer-keyed durable relations around mailbox ids, adds binding/cursor/lease-epoch tables and indexes, and removes v0 peer-key indexes/columns without compatibility reads |
| Network runtime identity/directory/dispositions | No canonical installation-scoped Network runtime identity or recipient-home disposition repository exists; daemon-local secret boundary is at `internal/vault/crypto.go:29-52` and `internal/vault/service.go:69-108` | P0-C adds append-only schema for persistent runtime identity/auth reference, per-epoch endpoint credentials/leases, mailbox home/route/tombstone versions, immutable idempotent accepted/rejected dispositions with committed endpoint epoch, and current-endpoint re-attestation presentations; persistent private keys remain vault-backed and superseded live credentials are invalidated |
| ACP provider-attempt extension | `internal/acp/types.go:196-225,448-453,505-518`; `internal/acp/client.go:415-430,756-779,935-991`; `internal/acp/handlers.go:271-310,663-693` | Negotiate capability; carry delegated activation budget; emit stable attempt lifecycle and actual/unavailable usage; reject active when unsupported |
| Active-capable adapter registry/preflight | `internal/config/provider.go:75-83,186-204,284-286,370-520`; `internal/session/manager_start.go:170-235`; `internal/acp/client.go:182-208,398-430` | Register trusted pinned manifests; pure artifact verification before owner persistence; match live initialize contract inside the owned post-commit start; ship at least one AGH-controlled real adapter |
| Provider active-support inventory | `internal/api/contract/providers.go:17-28`; `internal/api/core/providers.go:117-152`; `internal/cli/provider.go:168-185,689-706` | Project verified adapter contract, conformance status, and typed reason through HTTP/UDS/CLI/Web/native management surfaces without launching a session |
| CLI session | `internal/cli/session.go:90-140` | Replace implicit `--channel` with explicit mode plus conditional channel/default selection |
| CLI task | `internal/cli/task.go:360-430`, `:1300-1340`, `:2430-2467` | Replace create/start/publish/approve/enqueue/fan-out channel flags |
| CLI loop | `internal/cli/loop.go:236-288`; `internal/cli/config_loops.go` | Add explicit participation override and dry-run projection |
| Native tools | `internal/tools/builtin/network.go:12-340`; task/loop descriptors and `internal/daemon/native_tools.go:4335-4369` | Conditional descriptor visibility; participation fields on management tools; no legacy aliases |
| Hook/config matchers | `internal/config/hooks.go:50-79`; `internal/task/manager_run_hooks.go:131-161`; loop watch-event registries | Replace flat coordination channel fields with structured mode/channel snapshot |
| Web session create | `web/src/systems/session/hooks/use-session-create-dialog.ts:271-291` | Render Local in P0-B; prepare canonical adapter/state for gated Mailbox/Active controls after P0-C |
| Web task create/edit | `web/src/systems/tasks/components/task-form/ingress-identity-section.tsx:1-53`; `web/src/systems/tasks/lib/task-editor.ts:120-186` | Separate peer ingress from execution participation; serialize typed fields |
| Web task run/fanout | `web/src/systems/tasks/components/tasks-fan-out-runs-card.tsx:75-99`; task run detail components | Explicit Local override in P0-B; render internal resolved state for diagnostics; gate non-off create controls until P0-C |
| Web loop run | `web/src/systems/loops/hooks/use-loop-run-form.ts:35-54`; `web/src/systems/loops/lib/loop-overrides.ts:145-171` | Add participation to override draft, dry-run, and effective config |
| Web automation | `web/src/systems/automation/components/job-form/task-run-step.tsx:60-80`; `web/src/systems/automation/hooks/use-automation-job-form.ts:279-286` | Replace raw channel with participation and separate ingress where applicable |
| Web settings | `web/src/routes/_app/settings/network.tsx:168-217`, `:227-299` | Clarify administrative availability vs execution opt-in; expose caps and activation metrics |
| Official skill | `skills/agh/references/network.md:16-32`, `:78-100`; `skills/agh/references/tasks-and-orchestration.md:17-29`, `:142-146`; `skills/agh/references/loops.md` | Teach explicit modes, mailbox semantics, local defaults, budgets, and MoA separation |
| Site docs | `packages/site/content/runtime/core/network/`, `core/sessions/`, `core/loops/`, `core/autonomy/`, `core/configuration/`, CLI reference | Update every public field/flag/example and remove bind-always language |
| Generated clients | `openapi/agh.json`; `web/src/generated/agh-openapi.d.ts`; `sdk/typescript/src/generated/contracts.ts` | Regenerate in the same change |
| QA tracker | `docs/qa/state.csv` | Add/reset user-visible session/task/loop/automation/network rows to `untested`; one worktree owns edits |

## 10. Contract, Codegen, and Migration Co-ship

P0-B changes participation contracts and SQLite schema. P0-C separately changes the Network wire,
public addressing/read models, and peer-keyed durable schema. Each implementation task must activate
`eng-contract-codegen-coship` and `eng-schema-migration` before edits and co-ship its complete hard
cut; P0-C cannot leave v0 address aliases after its migration.

Mandatory co-ship set:

- `internal/api/contract/*` request and response types.
- `internal/api/spec/*` operation and schema definitions.
- `openapi/agh.json`.
- `web/src/generated/agh-openapi.d.ts`.
- `sdk/typescript/src/generated/contracts.ts` if the repository generator owns it.
- CLI client request/response structs.
- HTTP and UDS parity tests.
- The appended SQLite migration and current schema declarations.
- Web adapters, mocks, fixtures, and stories.
- Site CLI/config/core reference pages.

Generated artifacts must come from `make codegen`; do not hand-edit them. Codegen-check owns drift, so do not add prose/static tests for generated file contents.

## 11. Delete Targets

The P0-B TechSpec must list and remove at least these targets. Names may shift during implementation, but no legacy semantic may survive.

### Domain and persistence

- `task.Task.NetworkChannel`.
- `task.Run.NetworkChannel`.
- `task.Run.CoordinationChannelID` as a Network alias.
- `task.ExecutionRequest.NetworkChannel`.
- `task.EnqueueRun.NetworkChannel`.
- `task.QueueRunReservation.RequestedChannel`.
- `session.CreateOpts.Channel` and bare session-channel resolution.
- Automation `JobTaskConfig.NetworkChannel`.
- SQLite `tasks.network_channel`, `task_runs.network_channel`, and `task_runs.coordination_channel_id`.
- Obsolete channel indexes.

### Derivation and fallback

- `resolveStoredRunChannel` fallback behavior.
- `coordinationChannelIDForQueuedRun`.
- `derivedRunCoordinationChannelID`.
- Unconditional `ensureQueuedRunCoordinationChannel` calls.
- `taskRunCoordinationChannelID` fallback from metadata and `NetworkChannel` at `internal/task/manager_run_hooks.go:180-188`.
- Any read path that interprets empty mode/channel as inherit.

### Runtime enforcement

- Coordinator missing-channel bootstrap gate.
- Unconditional coordinator Network permissions, commands, and session channel.
- Unconditional loop action/judge generated channels.
- `ChannelBound` as the sole prompt/tool participation decision.
- Any default Network tool projection for `off` sessions.
- Channel-derived task-role session identity and unconditional/fabricated `Coordination channel:
  default` prompt content at `internal/daemon/task_role_sessions.go:260-262,286-300`.
- Automatic worker activation from durable receipt, trace, presence, or status messages.
- The conversational `say` implementation of task status-back; replace it with a typed projection
  structurally unable to activate a model.

### Public surface and language

- Task/session/loop/automation `network_channel` and `coordination_channel_id` JSON request aliases.
- Task/session/loop CLI flags that imply the old field. Network subcommands keep their own `--channel` because they operate on a channel directly.
- Hook matchers/payload fields that expose the old run alias.
- UI copy that calls every run coordinated or assumes every session joins a default channel.
- Glossary/docs language equivalent to "single durable network channel bound to every workspace-scoped coordinated run."
- Autonomy language and canonical sequences that say every run gets a channel or `bind always`,
  especially `packages/site/content/runtime/core/autonomy/coordination-channels.mdx:6-28`,
  `autonomy/index.mdx:6-8,77-112`, `autonomy/coordinator.mdx:11-23,76-92`, and
  `docs/articles/2026-05/06-autonomy-kernel.mdx:140-169`.
- Tests whose invariant is automatic channel creation, including `internal/task/manager_integration_test.go:711-734` and the derived-channel expectations in `internal/store/globaldb/global_db_task_claim_test.go:1726-1787`.

P0-C separately deletes:

- Router `works map[string]Work` as durable authority.
- Persist-before-publish success semantics.
- Timeline-duplicate short circuit as delivery acknowledgement.
- Any best-effort path that cannot expose accepted/published state.
- Network v0 peer-address fields/semantics (`from`, `to`, `mentions`) and directed-send live-presence
  existence checks.
- Peer-id durable keys for direct rooms, thread participants, subscription preferences, work
  ownership/targeting, delivery/idempotency/coalescing, token stats, and coordinator ownership.
- `DirectRoomIdentity` v0 and direct-room `peer_a`/`peer_b` schema/indexes.

## 12. Canonical Test Placement and Invariants

Before writing tests, name the invariant, owning layer, and canonical suite. Extend existing suites; do not add duplicate standalone regression files.

| Invariant | Owning layer | Canonical suite |
|---|---|---|
| Channel lookup is workspace-qualified | Global DB task claim projection | `internal/store/globaldb/global_db_task_claim_test.go` |
| Schema hard cut and append-only migration | Global DB schema | `internal/store/globaldb/global_db_task_test.go` and `global_db_task_orchestration_schema_test.go` |
| Default workspace task run is `off` and creates no channel | Task service/store integration | `internal/task/manager_integration_test.go` |
| Run override resolves once and is stable across retry/recovery | Task service/store integration | `internal/task/manager_integration_test.go` plus existing recovery cases |
| `off` session has zero peer lifecycle and prompt projection | Session/daemon integration | `internal/session/manager_integration_test.go` and `internal/daemon/harness_context_integration_test.go` |
| `mailbox` persists but never calls `PromptNetwork` | Network delivery integration | `internal/network/delivery_integration_test.go` |
| Prompt error does not mark delivered | Network delivery | `internal/network/delivery_test.go` |
| Adapter attempt budget counts initial/retry/fallback/post-tool calls and rejects overrun | ACP adapter/runtime contract | Existing ACP protocol/conformance suite plus `internal/network/delivery_integration_test.go`; use the canonical suite selected by the implementation TechSpec |
| Unsupported ACP adapter rejects active before session/run creation | Session/task resolution boundary | Existing session/task manager integration suites plus HTTP/UDS/CLI parity cases |
| Adapter manifest/artifact mismatch leaves no active session/run owner state | Provider registry plus session/task preflight | Existing `internal/config` provider suites and session/task manager integration suites |
| Failure/cancel after owner commit but before/during initialize/negotiation leaves no process, stdio, process record, secret/runtime material, sandbox, reservation, or Network lease | Session/ACP/sandbox lifecycle | Existing session manager start/stop/provider/sandbox integration suites with fault injection at every acquisition boundary |
| Pure adapter preflight never launches a process and cleans a partial cache transaction | Provider adapter registry | Canonical provider registry/config suite selected by TechSpec 1 |
| Exact pinned adapter works against one real provider and reports attempt events | Release/provider integration | Credentialed provider lifecycle/real-scenario QA lane named by TechSpec 1; deterministic retry/fallback coverage remains in the conformance harness |
| First local public-thread send reaches selected peers | Network manager/router integration | `internal/network/manager_integration_test.go` and `internal/network/router_test.go` |
| Directed send to an offline registered mailbox commits without live presence and is read once after reattach | Mailbox repository/router/manager integration | Existing `internal/network/manager_integration_test.go`, `router_test.go`, and global DB Network conversation repository suite |
| Peer/lease rotation preserves mailbox/direct/thread/mention/subscription/work identity | Network identity plus repositories | Existing `internal/network/validate_test.go`, manager/router suites, and `internal/store/globaldb/global_db_network_conversation_repository_test.go` |
| Stale lease fencing cannot consume/ack/change policy/settle activation | Mailbox-to-lease resolver and activation scheduler | Existing Network manager/delivery integration suites selected by TechSpec 2 |
| Remote publish is not mailbox delivery until recipient home durable acknowledgement | Outbox/transport integration | Existing Network manager integration suite with two-runtime/faulting transport harness |
| Envelope cannot spoof mailbox home/route or bypass workspace/channel binding | Mailbox directory and routing authorization | Existing Network validation/router integration suites plus two-workspace/two-runtime negative cases |
| Tombstone committed between sender enqueue/publish and home commit returns one durable terminal rejection across retry/restart; accepted-before-tombstone stays accepted | Mailbox home repository plus outbox integration | Existing global DB Network repository and two-runtime manager integration suites with ordered transaction barriers |
| Home commits accepted or terminal-rejected disposition at endpoint N, response is lost, daemon restarts at N+1, and the current endpoint re-attests the exact immutable digest so the original outbox settles once | Runtime identity/directory plus remote transport | Two-runtime manager integration suite with restart/faulting transport harness; assert committed epoch N and serving epoch N+1 |
| Delayed presentation from superseded endpoint N, forged proof, stale runtime, or mismatched disposition cannot settle sender outbox after N+1 registration | Runtime authentication and disposition validation | Network protocol/router security suite plus two-runtime negative integration cases |
| Home bounds authenticated `ts`/`expires_at`, returns canonical `envelope_invalid` for an invalid/overlong interval, and returns canonical `envelope_expired` for arrival after a valid immutable deadline without unread state or activation | Envelope validation plus mailbox home transaction | Existing Network validation/manager/global DB suites with fake clocks at before/equal/after deadline, overlong TTL, maximum allowed skew, and conflicting invalid/expired/tombstoned precedence vectors |
| Home remains unavailable through attempt cap, deadline, and skew allowance; sender records `delivery_dead_lettered_unconfirmed` without a home disposition, while a late buffered arrival is rejected as expired and later reconciliation never republishes | Outbox/home disposition retention | Existing Network manager/global DB integration suites with two-runtime faulting transport, fake clock, restart, and delayed-delivery barriers |
| Publish failure survives restart and same-ID retry | Network durability | `internal/network/manager_integration_test.go` with a faulting transport/store harness |
| SQLite is sole work authority across restart | Network work repository/manager | `internal/store/globaldb/global_db_network_conversation_repository_test.go` and manager integration |
| Normal daemon boot wires deterministic status projection with zero model activation | Daemon composition root | `internal/daemon/daemon_integration_test.go` |
| Local coordinator starts without Network tools/channel | Coordinator runtime integration | `internal/daemon/coordinator_runtime_integration_test.go` |
| Loop local run creates no Network peers/channels | Loop runtime integration | Existing `internal/loop` service/action suites plus `internal/daemon/loop_*` integration suite |
| Network-only loop action rejects effective `off` at start/dry-run | Loop effective-config/start validation | `internal/loop/service_test.go`; `internal/loop/linter_test.go` owns only the separate static declaration-shape invariant |
| Automation preserves explicit participation | Automation dispatcher | `internal/automation/dispatch_test.go` and `manager_integration_test.go` |
| HTTP/UDS/CLI request and output parity | API/CLI | Existing `internal/api/core`, `httpapi`, `udsapi`, and `internal/cli` suites |
| Web controls serialize modes and render resolved state | Web system | Existing session/task/loop/automation component and adapter suites |
| End-to-end Network remains usable when explicitly active | Runtime E2E | `internal/daemon/daemon_network_collaboration_integration_test.go` and `web/e2e/__tests__/network.spec.ts` |

Behavioral assertions, not snapshots:

- `off`: zero `network_channels` mutations, zero local peers, zero subscriptions, zero greet publishes, zero Network wrapper bytes, zero Network-origin model calls.
- `mailbox`: an offline stable mailbox address accepts and returns exactly the
  workspace/channel/container-scoped message; peer/lease rotation does not change direct,
  mention/subscription/work identity; stale leases cannot acknowledge it; zero model calls.
- `active`: only admitted recipients are prompted; skipped recipients remain durable in mailbox; budget exhaustion is structured and queryable.
- Two workspaces using channel `ops` cannot observe each other's metadata, inbox, presence, thread/direct history, cache, SSE, events, or token usage.
- Same envelope ID is idempotent for storage and delivery state, not a silent publish skip.
- Every activation has a terminal outcome and actual or explicitly unavailable usage.
- `receipt`, `trace`, greet, presence, acknowledgement, subscription, and status projection alone do
  not prompt a model under any policy.
- An explicit active Network workflow still supports thread/direct reply, work lifecycle, task promotion, and status-back.

## 13. Benchmark and Metrics Matrix

### 13.1 Scale matrix

Run every applicable scenario with N = 1, 3, 10, and 50 local peers. Add a remote-daemon variant for N = 3 and N = 10 after P0-C selects transport topology.

| Scenario | Mode/policy | N=1 | N=3 | N=10 | N=50 | Required model-call shape |
|---|---|---:|---:|---:|---:|---|
| Session/task/loop startup, no messages | `off` | Yes | Yes | Yes | Yes | 0 Network calls |
| Public message durable history | `mailbox` | Yes | Yes | Yes | Yes | 0 calls |
| Directed message | `active`, directed | Yes | Yes | Yes | Yes | 1 target call maximum |
| Mentioned public message | `active`, mentioned | Yes | Yes | Yes | Yes | Calls equal unique admitted mentions, capped |
| Empty thread capability selection | `active`, capability K=1 | N/A | Yes | Yes | Yes | At most 1 call; other mailboxes only |
| Existing thread participants | `active`, subscription/expiry | N/A | Yes | Yes | Yes | Calls equal active subscribed participants, capped |
| Ten-message burst in one coalescing window | `active` | Yes | Yes | Yes | Yes | One call per admitted target/batch, not ten |
| Ten sparse messages outside the window | `active` | Yes | Yes | Yes | Yes | Policy-defined; report each call and tokens |
| Receipt/trace/status/greet storm | default | Yes | Yes | Yes | Yes | 0 calls |
| Publish failure plus restart | any non-off | Yes | Yes | N/A | N/A | No loss, no false success, no duplicate activation |
| MoA advisors plus aggregator | P2 explicit | 1/2/4 advisors | 1/2/4 | 1/2/4 | N/A | One acting aggregator; bounded parallel advisors |

### 13.2 Measurements

Record per scenario and aggregate P50/P95/P99 where meaningful:

- Model calls per initiating envelope, per target, and per resolved user outcome.
- Actual input/output/thought/cache-read/cache-write/total tokens.
- Estimated tokens and estimation error versus actual.
- Cost by provider/model, activation, peer, run, workspace, and MoA role.
- Prompt wrapper bytes separately from message/artifact bytes.
- Activated, mailbox-only, muted, deduplicated, coalesced, budget-exhausted, failed, and dropped counts.
- Queue wait, activation wait, model latency, end-to-end delivery latency, and slowest-advisor latency.
- SQLite statements/transactions/fsync time per envelope.
- Transport publishes/bytes and local dispatch count.
- CPU time, goroutines, RSS, open subscriptions, and sockets.
- Greet/presence writes per peer-minute.
- Crash-recovery time and pending outbox depth.
- Quality/success score for MoA versus the same aggregator alone.

### 13.3 Merge gates

Hard correctness gates:

- `off` and `mailbox` produce exactly zero Network-triggered model calls.
- Fanout grows O(K), not O(N), for bounded active policies.
- No loss or duplicate activation in the fault matrix.
- Workspace isolation failures are zero.
- Every provider-reported usage event reconciles to one activation or MoA role.

Initial efficiency targets, subject to baseline confirmation in the TechSpec:

- At least 70% reduction in repeated-delivery wrapper tokens versus the current formatter.
- No more than 256 estimated wrapper tokens for a repeated active message, excluding body/artifact content.
- K=1 capability activation by default.
- One model call for a burst batch per admitted target.
- P95 `off` execution startup overhead attributable to Network is statistically indistinguishable from a build with Network disabled.
- MoA must beat aggregator-only quality on the selected evaluation set and publish the incremental tokens, cost, and latency. A quality tie fails because the cheaper single model wins.

## 14. Operational Rollout and Observability Gates

This is a hard cut, not a dual-mode rollout. Operational phases still matter:

1. Land P0-A correctness fixes with no contract aliases.
2. Land P0-B atomically; all newly resolved executions default `off`.
3. Observe `off` publicly and internal `mailbox`/`active` invariant journeys with structured reason
   codes; do not expose non-off creation yet.
4. Land P0-C's stable-address/fencing/durable-delivery hard cut and pass the independent release
   gates before exposing mailbox or active.
5. Establish N=1/3/10/50 baseline before P1 prompt/routing changes.
6. Land P1 only when actual usage proves the intended reduction.
7. Expose P2 only behind explicit preset/run selection and a configured budget.

Required diagnostics:

- Participation resolution: requested mode, source, resolved mode/channel, caps, and rejection reason.
- Network availability: enabled, lazy-started/running, and unavailable reason.
- Routing: durable recipient set, activation candidates, admitted recipients, policy, score, cap, and skip reason.
- Activation: batch ID, envelope IDs, model, actual usage, cost, outcome, retry, and latency.
- Outbox: accepted/published/failed/retry counts, oldest age, and recovery count.
- Work state: canonical transition and rejected transition reason.
- MoA: preset, advisors, aggregator, cadence, advisor cache hits, per-model usage/cost, slowest advisor, failure quorum, and early-stop reason.

User-facing read models show `Local` in P0-B and add `Mailbox`/`Active` only when each release gate
passes. They report `Network unavailable` or `mode not released` when a requested value cannot run.
Do not show estimated savings unless backed by measured usage.

Implementation completion must update `docs/qa/state.csv`: add untested rows for the new mode controls and reset affected session/task/loop/automation/network rows. This roadmap does not perform QA or tracker edits.

## 15. Web and Docs Impact

### Web

- Session create: P0-B exposes only `Local`. After P0-C, add `Active` only after the finite-admission
  and durable-delivery gate, and add `Mailbox` only after its zero-activation/offline-durability gate.
  Reveal strategy/channel and active policy/budget fields only when relevant. Current submit omits all
  participation selection at `web/src/systems/session/hooks/use-session-create-dialog.ts:271-291`.
- Task create/edit: separate `Network peer ingress` from `Run participation`; replace the single raw channel field at `web/src/systems/tasks/components/task-form/ingress-identity-section.tsx:21-35`.
- Task start/fan-out: add explicit per-run override; current fan-out field conflates coordination and status-back at `web/src/systems/tasks/components/tasks-fan-out-runs-card.tsx:86-99`.
- Task/run detail: replace generic channel chips with resolved mode, channel, activation policy, and budget outcome.
- Loop run/configure: add participation to effective config, dry-run, and override form; current request projection is `web/src/systems/loops/hooks/use-loop-run-form.ts:49-54`.
- Automation task and loop targets: expose explicit participation; do not infer from task channel.
- Network settings: keep administrative enabled/listener/default-channel controls, clarify that they do not opt executions in, and add actual activation/outbox metrics.
- Network roster/threads/directs/inbox: show stable mailbox identity and delivery posture separately
  from optional live peer/lease presence; allow authorized offline mailbox selection; never change a
  direct room because a process restarts; show cost per activation only where truthful.
- All UI changes require `eng-design`, `ui-craft`, `impeccable`, Storybook coverage in existing stories, and `eng-ui-screenshot` evidence.

### Packages/site and product copy

Update:

- Session lifecycle and CLI new-session docs.
- Task execution, orchestration, fan-out, hooks, and CLI references.
- Loop authoring, configuration, guardrails, running, visual editor, and CLI references.
- Automation target docs.
- Network index, delivery/safety, channels/peers, threads, directs, work, task ingress, and protocol boundary.
- `config.toml` reference and lifecycle matrix.
- Official AGH skill references.
- `COPY.md`-governed UI/CLI terms after confirming glossary vocabulary.

Docs must state:

- Local is the default.
- Mailbox is durable without model wakeup.
- Active can spend tokens and is bounded.
- Network availability and participation are different controls.
- Mailbox id is the stable recipient/conversation identity; peer id is ephemeral live-presence
  evidence. Directed offline delivery, direct rooms, mentions, subscriptions, and work use mailbox
  ids, and remote delivery is not complete until the mailbox home acknowledges durable commit.
- Task state remains authoritative; Network messages are evidence/communication.
- MoA/swarm is a separate explicit execution strategy.

## 16. Config Lifecycle

Recommended lifecycle:

- Keep `[network].enabled` as the administrative kill switch.
- Keep `network.default_channel` only as an effective channel resolved after explicit `mailbox`/`active` selection; it must never select the mode. Bundle effective-default semantics therefore remain usable.
- Do not add a global `default_mode = active`. Built-in execution default is `off`.
- Keep existing transport bounds in `internal/config/config.go:459-473`, but apply prompt/digest/fanout knobs only to active participation.
- Add separately named `[network.activation.defaults]` and `[network.activation.limits]` namespaces.
  Defaults fill omitted active-policy fields only after explicit selection; limits use numeric
  maxima, trigger/exhaustion allowlists, and minimum/maximum coalescing windows. Per-run values must
  remain within those bounds, and provenance records which default or request supplied each field.
- Treat provider-attempt enforcement as a mandatory negotiated capability for active, not a config
  switch. Existing per-turn `TokenUsage` at `internal/acp/types.go:505-518` is insufficient because
  it contains no provider-attempt/retry/fallback identity.
- Do not add a user-writable `supports_active = true` escape hatch. Active eligibility comes from an
  administrator-trusted adapter manifest, exact artifact verification, conformance revision, and
  matching live handshake. Config may select a registered adapter id but cannot author its claims.
- Add MoA preset defaults under a separate execution/model strategy namespace, not `[network]`.
- Resolve workspace overlays once and persist the resolved run/session snapshot.
- `agh config set`, settings HTTP/UDS, settings web UI, config tool surface, examples, and site reference must co-ship.
- Config changes that affect transport topology may require restart; execution participation changes apply to new runs/sessions and never mutate an active snapshot silently.

## 17. AGH Impact Audit

### Native tools

Changed surfaces:

- Task create/start/publish/approve/enqueue/fan-out schemas must accept participation intent and separate peer ingress.
- Loop run/config tools must accept participation.
- Session creation native/HTTP/UDS surfaces must accept participation.
- Network descriptors remain available administratively, but `off` agent sessions must not receive the coordination toolset by default.
- `agh__network_send` requires non-`off` participation plus workspace/channel permission and resolves
  directed handles to stable mailbox ids; it never accepts a live peer id as durable addressing.
- Provider/agent status schemas expose verified active support, adapter/protocol version and digests,
  conformance status, and unavailable reason without starting a session.
- Tool descriptors, input/output schemas, schema digests, availability diagnostics, risk flags, capability gates, CLI/API fallbacks, and tests must regenerate together.

No new native tool ID is required if existing management tools can carry the structured fields. A TechSpec proposing a new ID must prove why extending the owning task/loop/session surface is insufficient.

### Extensibility and hooks

Affected:

- Hook task/run context currently exports flat channel fields at `internal/task/manager_run_hooks.go:131-161`.
- Hook matchers currently include `coordination_channel_id` at `internal/config/hooks.go:69-73`.
- Loop watch-events expose coordination/network stream fields at `internal/loop/watch_events_contract.go:5-50`.
- Automation dispatch copies task channels at `internal/automation/dispatch.go:674-678`.
- Bundles declare channels and an effective default; they must not activate executions.
- Extensions, bridge SDKs, MCP sidecars, skills/capabilities, resources, registries, and job definitions that start sessions/tasks/loops must pass explicit intent or receive `off`.
- Network bridges/SDKs hard-cut to the v1 mailbox-address envelope and separately label optional live
  lease evidence; no adapter may translate v0 peer addresses through a compatibility alias.
- The provider-harness/extension registry gains an administrator-authorized adapter-manifest
  contract. Third-party adapters cannot self-assert active support through ordinary provider config;
  they must use the same artifact verification and conformance surface as the built-in adapter.

Hooks should expose one normalized snapshot (`mode`, channel when non-off, activation policy/budget when active) without legacy flat aliases. Network-origin correlation (`thread_id`, `direct_id`, `work_id`) remains separate from worker participation.

### Workspace data isolation

Scope classification:

- Administrative Network availability/config: global with workspace overlays where already supported.
- Network runtime identity/auth key reference: global installation (`AGH_HOME`), persistent across
  daemon processes; endpoint lease/epoch is live global runtime state and never workspace-derived.
- Trusted provider-adapter manifests and conformance revisions: global administrative registry;
  resolved contract copies are execution-scoped and contain no provider credentials.
- Participation intent on task/loop/job: workspace-owned when the parent artifact is workspace-scoped.
- Resolved participation snapshot: session/run/loop-run scoped and immutable for that execution.
- Channel/timeline/outbox data: workspace plus channel/container. Durable mailbox binding/cursor and
  direct/mention/participant/subscription/work/delivery/usage identity: workspace plus channel plus
  stable mailbox id(s). Live transport handles/presence: workspace plus channel plus mailbox plus
  process/session lease epoch. Activation usage: workspace plus concrete
  execution/mailbox/activation/attempt/provider identity, with lease id optional observation only.
- MoA preset: global/workspace config; invocation usage is session/run scoped.

Proof obligations:

- Propagate `workspace_id` through CLI, HTTP, UDS, core, store, web, SSE, cache keys, hooks, events, outbox, routing, and token/cost rows.
- Qualify every channel lookup by `(workspace_id, channel)`; C-01 proves this is not currently universal.
- Test the same channel, mailbox-like ID, live-peer-like ID, direct/thread-like ID, and message-like ID
  in two workspaces; neither durable nor live lookup may cross the boundary.
- `off` creates no Network-scoped datum.
- Mailbox/activation list and read paths cannot infer workspace from session/channel alone.

### Official AGH skill

Required updates:

- `skills/agh/references/network.md`: modes, explicit activation, stable mailbox versus live peer
  identity, offline direct semantics, budgets, retry/outbox state, and compact response behavior.
- `skills/agh/references/tasks-and-orchestration.md`: local task authority, separate peer ingress, local coordinator path, and when Network is useful.
- `skills/agh/references/loops.md`: local default, Network-only action validation, and MoA strategy.
- `skills/agh/references/native-tools.md`: changed schemas/tool availability.

Keep the skill lean: one source of truth per concept, route to references, and do not paste protocol tutorials into runtime prompts.

## 18. Risk Register

| Risk | Consequence | Mitigation/gate |
|---|---|---|
| `mailbox` is interpreted as a delayed model wake | Surprise cost | Contract states zero automatic activation; invariant test counts calls |
| Live peer id remains a durable recipient or foreign key | Offline direct fails and restart forks directs/mentions/subscriptions/work history | P0-C v1 mailbox-address hard cut across wire/store/API/CLI/Web; delete v0 peer aliases and presence-as-existence gate |
| Replaced lease can still consume or acknowledge | Duplicate/lost inbox state or wrong activation settlement | Monotonic lease epoch/fencing token on every consume/ack/policy/activation transition; stale-lease fault tests |
| Remote transport publish is reported as mailbox delivery | Sender sees success before recipient durability | Recipient-home durable acknowledgement plus retryable sender outbox; publish-only state is never delivered |
| Mailbox is tombstoned after sender enqueue but before home commit | Sender outbox retries forever or delivery crosses a delete barrier ambiguously | Home transaction orders tombstone versus delivery; persist authenticated idempotent terminal NACK or accepted ACK and replay it on retry/restart |
| Home runtime identity rotates with daemon process | Every remote mailbox route breaks on restart | Persist one authenticated installation-scoped runtime id per `AGH_HOME`; re-register only endpoint epoch; two-runtime restart gate |
| Sender treats committed endpoint epoch as the response-freshness gate | A valid pre-crash ACK/NACK can never settle after home restart, causing duplicate retry or dead-letter | Keep immutable `committed_endpoint_epoch` separate from current `serving_endpoint_epoch`; require the current endpoint to re-attest the unchanged disposition digest |
| Forged or superseded-endpoint disposition presentation settles outbox | Message loss or false delivery | Give every endpoint epoch a fresh credential invalidated on replacement; resolve the authoritative current directory lease and bind re-attestation proof to home runtime, disposition digest, mailbox/message/route, and serving endpoint epoch; fail closed when directory freshness is unavailable |
| Sender retry exhaustion is recorded as home `envelope_expired` | The system fabricates recipient authority and a delayed copy may still be accepted | Separate sender-local `delivery_dead_lettered_unconfirmed` from home-authenticated `envelope_expired`; authenticate one immutable deadline, wait through skew allowance, and test delayed arrival/reconciliation without republish |
| Empty channel is still treated as inherit downstream | Enforcement returns | Persist resolved snapshot; delete fallbacks; repository-wide search gate |
| Task peer ingress remains coupled to run participation | Remote task forces expensive worker | Separate types, columns, UI sections, and tests |
| Local coordinator loses needed behavior | Orchestration regression | Base non-Network allowlist and local coordinator integration journey |
| Existing alpha data is silently inferred as active | Surprise spend | Destructive migration to `off`; release note and no aliases |
| Network unavailable after user selects active | Partial session/run creation | Resolve before creation and return structured diagnostic |
| Budget admission uses inaccurate estimate | Overspend | Conservative estimate pre-call; actual reconciliation; hard model-turn cap |
| Provider omits usage | Incomplete accounting | Mark `usage_unavailable`, retain estimates, never report actual savings |
| ACP adapter exposes only aggregate turn usage | Hidden retries/fallbacks can exceed the root attempt budget | Require `network_provider_attempt_budget_v1` for active; reject unsupported profiles before creation; off/mailbox remain available |
| Attempt capability passes only with a mock | Active is nominally released but unusable by real providers | P0-B ships one AGH-controlled pinned adapter; deterministic conformance plus credentialed real-provider smoke are release gates |
| Adapter command or package drifts after resolution | Manifest promise differs from executable behavior | Pre-persistence artifact digest verification, no `@latest` proof artifact, live handshake match, typed hard failure without downgrade |
| Adapter preflight launches or retains an ACP process across owner commit | Storage/hook/cancel failure can orphan PID, stdio, secrets, sandbox, or process record | Baseline preflight is pure artifact verification only; process launch remains post-commit inside the existing owned cleanup envelope; boundary fault injection is mandatory |
| Outbox duplicates model activations | Duplicate work/cost | Separate durable delivery and activation idempotency keys |
| In-process transport diverges from remote path | Two behavior models | Shared validation/routing/outbox contracts and parity fault tests |
| SQLite becomes a throughput bottleneck at N=50 | Queue latency | Benchmark before replacing NATS; batch dispatcher/audit writes based on profile |
| Capability K=1 misses the best peer | Quality loss | Explicit escalation policy and evaluation; mailbox preserves unactivated messages |
| Sticky participants continue causing fanout | Linear token growth | Subscription/expiry requirement and bounded recipients |
| Compact prompts omit critical correlation | Broken replies | Typed metadata and behavioral reply tests, not verbose prose |
| MoA multiplies every tool iteration | Explosive cost | Default `user_turn` cadence, state cache, explicit per-iteration opt-in |
| Advisor output dominates latency/context | Slow expensive run | Per-advisor output cap, quorum/early stop, typed summaries |
| Advisor can act or mutate state | Authority confusion | No tools, Network off, aggregator is sole actor |
| UI mode and daemon truth disagree | Misleading control | API returns resolved snapshot/unavailable reason; UI renders daemon truth |
| Parallel branches collide on QA tracker/schema | Lost changes | One worktree owns `docs/qa/state.csv`; serialize migration registry edits |

## 19. Suggested TechSpec Boundaries

### TechSpec 1: Explicit Network Participation and Activation Boundary

Owns:

- `off | mailbox | active` contract, with only Local/off publicly selectable in this phase.
- Task ingress split.
- Session/task/run/loop/automation resolution and persistence.
- Coordinator decoupling.
- Prompt/tool/lifecycle gating.
- Minimum active admission scheduler: deterministic eligible triggers, absolute control-event
  exclusion, positive server-bounded logical-turn/provider-attempt/input/output/fanout/depth/time
  budgets, fixed coalescing policy snapshot, and pre-call reservation.
- `network_provider_attempt_budget_v1` schema/capability negotiation, delegated sub-budget envelope,
  trusted manifest/artifact registry, pre-persistence verification, adapter conformance harness, ACP
  terminal/error classification, and minimum actual/unavailable attempt settlement required to make
  internal active budget tests truthful.
- At least one AGH-controlled, exactly pinned real adapter plus its live handshake and credentialed
  provider smoke evidence. Generic direct ACP providers remain off/mailbox-only until verified.
- Session/task start-order changes for pure adapter artifact verification before owner persistence,
  followed by live handshake verification inside the owned post-commit start and before ACP session
  negotiation or Network lease creation. No staged process crosses the commit.
- Startup resource ownership/cleanup matrix and fault injection for adapter-cache, storage, hooks,
  provider secrets/runtime dirs, sandbox, process/stdio/connection/process record, cancellation,
  shutdown, and ACP negotiation paths.
- Lazy startup/reference counting for the current transport.
- Hard-cut public contracts and read models, migration, codegen, UI, docs, skill, and QA tracker
  impact; ordinary create controls/docs expose Local only.

Does not own:

- NATS replacement.
- Transactional outbox.
- Work lifecycle rearchitecture.
- Rich prompt formatter/routing/coalescing optimization beyond removing Network projection from
  `off`, preventing activation in mailbox, and the minimum safe active header.
- MoA.

### TechSpec 2: Stable Mailbox Addressing, Durable Network Delivery, and Canonical Work State

Owns:

- C-02, C-04, C-05, and C-07.
- Accepted/published/delivered/activated state machine.
- Transactional outbox.
- Local transport decision.
- Restart recovery and idempotency.
- SQLite-only work authority.
- Network v1 stable mailbox-address envelope hard cut, including sender/recipient/mention fields and
  deletion of v0 peer-address aliases.
- Durable mailbox binding/cursor independent of process leases, including offline directed delivery,
  recipient-home acknowledgement, restart reattachment, tombstone/no-reuse semantics, and typed
  unknown-address failures.
- Authenticated mailbox directory/binding with stable home runtime and route epoch; P0-C does not
  infer home from presence or silently migrate a mailbox between runtimes.
- Persistent installation-scoped Network runtime identity per `AGH_HOME`, authenticated key/reference,
  and fenced expiring endpoint re-registration that survives daemon restart without changing mailbox
  home identity; persistent identity authenticates registration while fresh per-epoch credentials
  authenticate live responses and are invalidated on replacement.
- Immutable idempotent recipient-home accepted/rejected dispositions with historical committed
  endpoint epoch; authenticated retryable route-refresh responses; and current-endpoint
  re-attestation presentations with independent serving epoch, sender verification, and durable
  outbox settlement.
- Mandatory authenticated immutable remote-envelope deadline, home-side expiry disposition, explicit
  clock/skew semantics, sender-local unconfirmed dead-letter after retry/deadline exhaustion, and
  late-arrival/reconciliation behavior that cannot accept after expiry or republish.
- One home-store serialization boundary for mailbox tombstone versus delivery; accepted-first remains
  accepted, tombstone-first terminates as rejected, and retry/restart returns the same disposition.
- Mailbox-keyed direct rooms, thread participants, mentions, subscription policy, work
  opener/target, delivery/idempotency/coalescing, coordinator ownership, audit, and usage attribution.
- One-way mailbox-to-live-lease resolution with monotonic epoch/fencing; live peer ids remain
  optional presence/transport observations only.
- Fixed bounded activation coalescing, durable attempt/idempotency state, provider/ACP cancellation
  propagation, deadline handling, and non-reactivating late results.
- Durable ingestion/reconciliation of the provider-attempt extension across crash/restart, including
  retaining an unclosed allocation as consumed and rejecting active when capability conformance is
  lost.
- Independent public release gates for mailbox and active, co-shipped across CLI/HTTP/UDS/native
  tools/Web/docs/official skill only when their invariants pass.

### TechSpec 3: Activation Efficiency, Prompt Diet, and Rich Cost Attribution

Owns:

- Activation routing/scheduler optimization beyond the finite P0-B admission contract.
- Adaptive coalescing/digest/routing optimization beyond the fixed P0-C safety contract.
- Prompt wrapper redesign.
- Rich attempt/provider/fallback pricing, dashboards, and analytics beyond the minimum P0-B/P0-C
  attempt identity, outcome, and token settlement contract.
- Presence/control storage, audit, and projection diet beyond the absolute P0-A/P0-B
  zero-activation boundary.
- N=1/3/10/50 benchmark gates.

### TechSpec 4: MoA Execution Strategy

Owns:

- Named MoA execution presets and explicit selection in the model-strategy namespace; no named
  Network policy selector is introduced by TechSpec 1.
- Advisor/aggregator roles.
- Typed advisory context and blackboard pointers.
- Concurrency, cadence, budgets, quorum, early stop, usage/cost tracing.
- Session/task/loop/automation/UI/CLI management surfaces.
- Evaluation datasets and cost-normalized quality gates.

## 20. Two-Touch Rule

The Network package already shows repeated tactical growth: `internal/network/manager.go`, `delivery.go`, `router.go`, and `validate.go` exceed the 500-line production-source cap. Do not append new participation, outbox, activation, or MoA behavior to these files.

Track patches by package and behavior. After two patches to the same Network delivery/routing behavior in this workstream, a third change requires the approved structural TechSpec and a file split. In practice:

- P0-A may make the narrow delivery-outcome correction.
- P0-B may gate participation at the boundary.
- P0-C must be the structural redesign, not a third patch inside `manager.go` or `delivery.go`.

Before production code, decide files by responsibility: participation contract/resolver, mailbox repository, outbox repository/dispatcher, routing policy, activation scheduler, prompt renderer, usage accounting, presence, and one implementation per transport. Existing over-cap files must shrink or remain untouched; they must not grow.

## 21. Definition of Done by Phase

### P0-A

- C-01, C-03 outcome handling, and C-06 have failing-then-passing tests in canonical suites.
- C-06's replacement emits typed state through normal boot and has no code path to conversational
  `say` or `PromptNetwork` activation.
- No error is discarded.
- No user-visible contract change unless explicitly documented and QA-tracker flagged.
- Scoped Go lint/race lanes pass, followed by one final `make verify` for the implementation batch.

### P0-B

- All ordinary creation/execution surfaces default to resolved `off`.
- `off` and `mailbox` have exactly zero Network-triggered model calls.
- Internal `active` admission has deterministic triggers, finite hard budgets, control-event
  exclusion, pre-call reservation, and a deterministic provider-attempt conformance harness that
  proves retry, fallback, post-tool, missing-usage, over-budget, and unsupported-adapter behavior.
- A trusted manifest and exact artifact are verified before active session/run persistence; live
  handshake mismatch leaves no Network lease and cannot downgrade the request.
- Adapter preflight is process-free. Every post-commit resource has an owner and cleanup installed
  before acquisition; fault injection after a valid preflight proves zero orphan process, stdio,
  process record, provider secret/runtime material, sandbox, reservation, or Network lease.
- At least one AGH-controlled pinned adapter passes the conformance harness and a credentialed
  real-provider smoke journey. A mock-only result does not satisfy P0-B or the later active gate.
- No automatic task/loop channel creation remains.
- Coordinator works locally without Network.
- Peer ingress and worker participation are separate.
- Existing transport starts lazily; `off` execution cannot start or lease it.
- Contract, schema migration, codegen, HTTP/UDS/CLI/native tools/web/docs/skill co-ship.
- All Section 11 P0-B delete targets are gone; no aliases or fallback reads.
- Workspace-isolation plus internal mailbox/active invariant journeys pass; ordinary public create surfaces expose only Local until P0-C release gates pass.
- UI screenshots cover session, task, loop, automation, detail, and disabled-runtime states.
- QA tracker rows are added/reset to `untested`.
- `make verify` passes once at completion.

### P0-C

- First local thread send reaches intended recipients.
- Publish failure and restart cannot lose or falsely acknowledge accepted work.
- Same-ID retry reports/reuses durable state and does not suppress publish.
- SQLite is the only work lifecycle authority.
- Fault matrix covers crash before/after commit, before/after publish, duplicate receipt, and restart.
- A stopped mailbox releases live transport handles/presence while preserving durable subscription
  policy, remains addressable, receives an offline envelope, and reattaches the same id/cursor after
  restart without loss or duplicate read.
- Network v1 envelope/address schema contains stable mailbox sender/recipient/resolved mentions and
  no durable peer-address alias; directed validation succeeds without presence for a registered
  mailbox and rejects unknown/tombstoned addresses deterministically.
- Peer/lease rotation preserves direct-room id, thread membership, mention target, durable
  subscription policy, work ownership/target, audit, and usage identity. Stale fencing tokens cannot
  consume, acknowledge, mutate, or settle on behalf of the mailbox.
- A remote send is only delivered after recipient-home durable acknowledgement; transport-only
  publish remains observable retryable outbox state.
- Mailbox tombstone and delivery are transactionally ordered at the home. Delete between sender
  enqueue/publish and home commit yields a durable authenticated terminal rejection; accepted-first
  remains accepted; retry/restart cannot change the result or loop forever.
- Home runtime id persists per `AGH_HOME` across daemon restart. When a response is lost after a
  disposition commits at endpoint N, endpoint N+1 re-attests the exact canonical digest with serving
  epoch N+1 and the original sender outbox completes once; a delayed presentation from N and forged
  ACK/NACK fail closed.
- Remote retry age/attempts are finite. An unavailable home reaches sender-local
  `delivery_dead_lettered_unconfirmed` only after the immutable deadline and skew allowance, without
  a fabricated disposition; a late arrival yields home `envelope_expired`, cannot activate, and any
  later reconciliation cannot republish. Relevant retention spans retry/replay plus settlement grace.
- Burst envelopes coalesce into one durable activation attempt per target/root/batch key under the
  fixed window; every source envelope remains queryable.
- Cancel propagates to underlying ACP/provider work where supported. Non-cancelable work remains
  charged/observable to terminal/deadline, and late output cannot wake or reenter the actor.
- Outbox/delivery state is agent-manageable through structured CLI/HTTP/UDS output.
- N=1/3/10/50 durability and resource baseline is recorded.
- Mailbox is exposed only after its stable-address/offline-direct/fencing/zero-activation/order/
  dedup/cursor/restart gate passes.
- Active is exposed only after the P0-B finite-admission gate and the P0-C
  stable-address/delivery/recovery/fixed-coalescing/cancellation/provider-attempt reconciliation
  gate passes.
- Even after release, Active is selectable only for agent/provider profiles whose current verified
  adapter contract matches the P0-B registry and live handshake; others expose a typed unavailable
  reason before submission.
- CLI, HTTP/UDS, native tools, Web, docs, and official skill expose the newly released modes in the same batch.
- `make verify` passes once at completion.

### P1

- Actual usage/cost is attributed or explicitly unavailable for every activation.
- Control-plane traffic causes zero model calls under every Network policy.
- Burst coalescing and bounded fanout meet Section 13 gates.
- Prompt-wrapper reduction meets the measured target without breaking reply/work correlation.
- Settings, diagnostics, web cost views, docs, skill, and QA tracker co-ship.
- `make verify` passes once at completion.

### P2

- MoA is explicit and default off on every surface, including automation/scheduled execution.
- Advisors have no tools, task authority, or Network participation.
- One aggregator owns action and final output.
- Concurrency, cadence, output, token, cost, wall-clock, and failure budgets are enforced.
- Usage/cost is attributed per actual advisor/aggregator attempt, retry, and fallback identity.
- Quality beats aggregator-only baseline at an acceptable published incremental cost; otherwise the feature does not ship.
- MoA works without Network and can opt the aggregator into Network separately.
- Public management surfaces, config lifecycle, UI, docs, official skill, QA tracker, and full verification co-ship.

## 22. Final Recommendation

Ship the local-default participation hard cut first, but do not describe it as the complete Network fix. The narrow correctness defects C-01, C-03 outcome handling, and C-06 are independent P0 prerequisites. The proven local-send, persist-before-publish, and dual-authority defects justify a separately approved P0 durability TechSpec. Only after those boundaries are truthful should AGH optimize prompts/fanout and add a Hermes-inspired MoA strategy.

This sequencing provides the immediate user value - lightweight local runs - without coupling it to a risky transport rewrite, while preventing known durability defects from being buried under token optimizations.
