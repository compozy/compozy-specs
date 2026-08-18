# Opt-In Network Participation Architecture

- **Status:** proposed target architecture
- **Audience:** runtime, Network, task, session, loop, automation, API, CLI, Web, extension, and documentation owners
- **Decision scope:** execution participation plus the minimum stable-address wire/storage hard cut required for a durable mailbox; broader transport/protocol optimization remains P0-C/P1
- **Default:** local execution with no Network participation

## Executive Decision

AGH must replace channel-presence inference with one explicit, resolved execution contract:
NetworkParticipationSpec. The contract separates two independent planes:

1. Participation: whether an execution owns a durable Network mailbox binding and may realize a
   live membership lease.
2. Activation: whether a Network envelope is allowed to start a model turn.

The target contract contains three presets. They become user-selectable only after the release gates
defined below pass:

| Preset | Participation plane | Activation plane | Intended use |
|---|---|---|---|
| off | disabled | disabled | Local sessions, tasks, loops, reviews, and automations |
| mailbox | joined | manual only | Durable asynchronous exchange without envelope-triggered inference |
| active | joined | automatic, policy-gated | Live agent-to-agent coordination |

The built-in default is off. Network availability remains an administrative capability switch.
P0-B exposes only off while it proves the hard-cut participation boundary, minimum active admission,
actual-usage settlement, and lazy startup of the current transport. Mailbox and active remain
internal testable values until P0-C establishes stable mailbox addressing/fencing, durable
delivery/recovery, and each preset's own release gate passes. Transport replacement,
transactional-outbox implementation, and Network v1 durable-identity migration belong to that
separate P0-C TechSpec.

This is a greenfield hard cut. The old NetworkChannel and CoordinationChannelID execution fields,
the interpretation of a non-empty channel as an activation request, and every compatibility alias
must be deleted in the same change.

## Problem Statement

AGH currently uses one nullable channel string to trigger several unrelated behaviors:

- Task and run domain records expose NetworkChannel and CoordinationChannelID
  (internal/task/types.go:409-444, 468-499).
- Execution, enqueue, and store reservation requests propagate that string
  (internal/task/types.go:703-709, 727-750).
- Every workspace run derives a coordination channel when the caller supplied none, creates a
  network_channels row, and persists the derived id
  (internal/store/globaldb/global_db_task_aux.go:741-793, 804-877).
- Tests freeze that behavior by requiring a new channel after a plain task start
  (internal/task/manager_integration_test.go:690-736) and by requiring
  coord-run-derived-channel from a blank reservation
  (internal/store/globaldb/global_db_task_claim_test.go:1739-1792).
- Session startup translates a channel into AGH_SESSION_CHANNEL and AGH_PEER_ID
  (internal/session/manager_start.go:678-713), then joins a peer as part of activation
  (internal/session/manager_helpers.go:80-151).
- Harness context treats channel != empty as ChannelBound and injects the Network prompt section
  (internal/daemon/harness_context.go:310-330, 518-538).
- Accepted deliveries enqueue and immediately trigger PromptNetwork when the target is idle
  (internal/network/delivery.go:285-348, 616-688).
- The delivery prompt repeats reply commands, protocol rules, causation fields, response-register
  guidance, and examples (internal/network/delivery.go:1647-1689, 1696-1720, 1743-1868), in addition
  to the startup and turn response-register prompts
  (internal/daemon/network_response_register_prompt.go:20-29, 47-71).
- Coordinator bootstrap rejects runs without a channel and its permissions and prompt are
  Network-specific (internal/coordinator/coordinator.go:16-61, 97-175, 233-257).
- Loop action and judge sessions always receive generated channels
  (internal/daemon/loop_runtime_adapters.go:37-81, 118-161, 340-392).
- Starting a loop reserves a workspace coordinator task/run through the same auto-channel path
  (internal/store/globaldb/global_db_loop.go:26-81;
  internal/store/globaldb/global_db_loop_coordinator_seed.go:14-80).
- Review sessions inherit preferred, allowed, run, or task channels
  (internal/daemon/review_router.go:561-595, 779-797).
- Detached work inherits the explicit request or owner session channel and copies it into its task
  and run (internal/daemon/harness_detached_work.go:70-112, 172-203, 235-265, 360-386).
- Automation job task configuration persists NetworkChannel and forwards it through task creation
  and enqueue (internal/automation/model/types.go:115-121;
  internal/automation/dispatch.go:650-691, 1165-1189).
- The Network manager and embedded transport start at daemon boot whenever network.enabled is true
  (internal/daemon/boot.go:1333-1371; internal/network/manager.go:172-204).

This coupling makes three materially different user intents indistinguishable:

1. Run locally and never participate in Network.
2. Maintain a durable mailbox but do not spend a model call when messages arrive.
3. Participate in live Network coordination and allow bounded automatic activation.

Changing blank-channel behavior alone would address only the first intent. It would retain the
architectural ambiguity for the other two.

## Normative Language

The words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are normative.

The following terms have precise meanings:

- Requested participation: user, profile, definition, or config input before resolution.
- Resolved participation: the immutable NetworkParticipationSpec snapshot used by an execution.
- Mailbox binding: a durable workspace/channel/recipient address and read cursor owned by an
  execution definition or durable execution record. It survives process and presence-lease exit
  until explicit deletion or retention expiry.
- Mailbox ID: the opaque, stable protocol recipient identity for that binding. It is used by durable
  envelope addressing, direct rooms, thread participants, mentions, work ownership, subscription
  policy, cursors, and usage attribution. It is never derived from a process peer ID.
- Live membership lease: a process-lifetime workspace/channel/peer attachment used for liveness,
  live transport subscription handles, and immediate activation delivery. It is not the durable
  mailbox address or durable subscription owner.
- Peer ID: an ephemeral live-lease/process identifier used for presence and transport diagnostics.
  It may rotate on restart and is never a durable recipient, conversation participant, or work owner.
- Mailbox ingress: durable storage of envelopes addressed or routed to a mailbox binding, whether
  or not a live membership lease exists.
- Model activation: starting a provider/ACP prompt turn because of Network ingress.
- Local turn: a user, task, coordinator, review, loop, automation, or explicit session prompt whose
  origin is not a Network envelope.
- Public preset: the stable user-facing off, mailbox, or active choice.

## Architectural Principles

### Explicit participation

No channel name, task scope, session role, loop node type, coordinator state, hook origin, bundle
default, or automation trigger may implicitly enable Network participation.

### Orthogonal planes

Transport membership and model activation are separate. Persisting or receiving an envelope does
not imply that the target model should run.

### Default-local execution

The built-in resolved preset is off. Only a concrete invocation or an explicitly saved owning
profile/definition may request mailbox or active. Workspace and daemon configuration define
availability, activation defaults/limits, conditional channel resolution, and ceilings; they never enroll an
otherwise plain execution.

### Resolve once, snapshot once

Every session lifetime, task-run attempt, loop run, review session, spawned child, and automation
fire receives an immutable resolved snapshot. Later configuration edits affect future executions,
not live ones.

### No silent degradation

If Network is administratively disabled and an execution requests mailbox or active, the request
fails with a typed unavailable diagnostic. AGH MUST NOT silently downgrade it to off.

### Least activation

A descendant may inherit authority to use a channel, but authority is not participation. Child,
review, detached, and automation executions default to off unless their own resolver input selects
mailbox or active.

### State is authoritative; conversation is evidence

Task, run, review, loop, and automation state remains in the owning services. Network messages may
reference authoritative ids but never replace ownership, leases, status, or verdicts. This retains
the existing task guidance in skills/agh/references/tasks-and-orchestration.md:17-29 and 142-150.

### Network is not the workflow engine

Loop execution, MoA aggregation, task scheduling, and coordinator control remain runtime concerns.
The project glossary already states that loop execution stays off the Network wire and that AGH
Network is not a workflow engine (docs/_memory/glossary.md:311-319).

## Canonical Domain Model

### Requested contract

Public write surfaces accept a NetworkParticipationRequest:

~~~go
type NetworkParticipationPreset string

const (
    NetworkParticipationOff     NetworkParticipationPreset = "off"
    NetworkParticipationMailbox NetworkParticipationPreset = "mailbox"
    NetworkParticipationActive  NetworkParticipationPreset = "active"
)

type NetworkChannelStrategy string

const (
    NetworkChannelNamed   NetworkChannelStrategy = "named"
    NetworkChannelDefault NetworkChannelStrategy = "default"
    NetworkChannelRun     NetworkChannelStrategy = "run"
    NetworkChannelLoopRun NetworkChannelStrategy = "loop_run"
)

type NetworkParticipationRequest struct {
    Preset          *NetworkParticipationPreset `json:"preset,omitempty"`
    ChannelStrategy *NetworkChannelStrategy     `json:"channel_strategy,omitempty"`
    ChannelID       *string                     `json:"channel_id,omitempty"`
    Activation      *NetworkActivationRequest   `json:"activation,omitempty"`
}
~~~

Pointer presence means explicit input. An omitted object means inherit according to the resolution
rules. A present object with preset off is an explicit local choice and clears lower-precedence
channel and activation defaults.

Callers MUST NOT set the internal participation or activation planes directly. Public presets are
the supported policy vocabulary.

### Resolved contract

The resolver produces a complete NetworkParticipationSpec:

~~~go
type NetworkParticipationPlane string

const (
    NetworkParticipationDisabled NetworkParticipationPlane = "disabled"
    NetworkParticipationJoined   NetworkParticipationPlane = "joined"
)

type NetworkActivationMode string

const (
    NetworkActivationDisabled  NetworkActivationMode = "disabled"
    NetworkActivationManual    NetworkActivationMode = "manual"
    NetworkActivationAutomatic NetworkActivationMode = "automatic"
)

type NetworkParticipationSpec struct {
    Version       string                     `json:"version"`
    Preset        NetworkParticipationPreset `json:"preset"`
    Participation NetworkParticipationPlane  `json:"participation"`
    Activation    NetworkActivationSpec      `json:"activation"`
    WorkspaceID   string                     `json:"workspace_id,omitempty"`
    ChannelStrategy NetworkChannelStrategy   `json:"channel_strategy,omitempty"`
    ChannelID     string                     `json:"channel_id,omitempty"`
    MailboxID     string                     `json:"mailbox_id,omitempty"`
    Resolution    NetworkResolutionTrace     `json:"resolution"`
}
~~~

Version is a schema identifier, not an optional compatibility discriminator. A schema change is a
hard contract update across all consumers.

The resolved spec, not a session Channel string or run CoordinationChannelID, is the sole source of
truth for runtime effects.

### Activation contract

~~~go
// NetworkDuration is a validated canonical Go-duration string on every wire
// surface, for example "250ms". Runtime code parses it after schema validation.
type NetworkDuration string

type NetworkActivationTrigger string

const (
    NetworkTriggerDirect   NetworkActivationTrigger = "direct"
    NetworkTriggerMention  NetworkActivationTrigger = "mention"
    NetworkTriggerExplicit NetworkActivationTrigger = "explicit"
    NetworkTriggerRouted   NetworkActivationTrigger = "routed"
)

type NetworkBudgetExhaustionPolicy string

const (
    NetworkBudgetExhaustionMailbox NetworkBudgetExhaustionPolicy = "mailbox"
)

// This server-resolved contract comes only from a trusted adapter registry.
// Callers cannot assert it in NetworkActivationRequest.
type NetworkProviderAttemptContract struct {
    CapabilityID   string `json:"capability_id"`
    ProtocolVersion string `json:"protocol_version"`
    AdapterID      string `json:"adapter_id"`
    AdapterVersion string `json:"adapter_version"`
    ArtifactDigest string `json:"artifact_digest"`
    ManifestDigest string `json:"manifest_digest"`
}

// Pointer presence preserves omitted-versus-explicit input for field-level
// defaulting, narrowing, validation, and provenance.
type NetworkActivationRequest struct {
    EligibleTriggers     *[]NetworkActivationTrigger      `json:"eligible_triggers,omitempty"`
    MaxModelTurns        *int                             `json:"max_model_turns,omitempty"`
    MaxProviderAttempts  *int                             `json:"max_provider_attempts,omitempty"`
    MaxInputTokens       *int64                           `json:"max_input_tokens,omitempty"`
    MaxOutputTokens      *int64                           `json:"max_output_tokens,omitempty"`
    MaxFanoutPerEnvelope *int                             `json:"max_fanout_per_envelope,omitempty"`
    MaxActivationDepth   *int                             `json:"max_activation_depth,omitempty"`
    MaxWallTime          *NetworkDuration                 `json:"max_wall_time,omitempty"`
    CoalesceWindow       *NetworkDuration                 `json:"coalesce_window,omitempty"`
    OnBudgetExhausted    *NetworkBudgetExhaustionPolicy   `json:"on_budget_exhausted,omitempty"`
}

type NetworkActivationSpec struct {
    Mode                 NetworkActivationMode            `json:"mode"`
    EligibleTriggers     []NetworkActivationTrigger       `json:"eligible_triggers,omitempty"`
    MaxModelTurns        int                              `json:"max_model_turns,omitempty"`
    MaxProviderAttempts  int                              `json:"max_provider_attempts,omitempty"`
    MaxInputTokens       int64                            `json:"max_input_tokens,omitempty"`
    MaxOutputTokens      int64                            `json:"max_output_tokens,omitempty"`
    MaxFanoutPerEnvelope int                              `json:"max_fanout_per_envelope,omitempty"`
    MaxActivationDepth   int                              `json:"max_activation_depth,omitempty"`
    MaxWallTime          NetworkDuration                  `json:"max_wall_time,omitempty"`
    CoalesceWindow       NetworkDuration                  `json:"coalesce_window,omitempty"`
    OnBudgetExhausted    NetworkBudgetExhaustionPolicy    `json:"on_budget_exhausted,omitempty"`
    ProviderAttemptContract *NetworkProviderAttemptContract `json:"provider_attempt_contract,omitempty"`
}
~~~

The supported automatic triggers are:

- direct: a validated direct-room envelope addressed to the execution's stable mailbox id.
- mention: a validated mention resolved to the execution's stable mailbox id.
- explicit: a validated envelope carrying an explicit activation request authorized by channel
  policy.
- routed: an envelope selected by the bounded runtime router.

Using a wire-level string is intentional: raw `time.Duration` JSON encodes nanoseconds and the
current TypeScript generator maps it to `number`, which makes units implicit
(`internal/codegen/sdkts/generate.go:298-300,365-367`). HTTP, UDS, hooks, generated clients, CLI,
config, and Web therefore exchange the same canonical duration text and reject non-canonical or
out-of-range values.

The built-in active policy SHOULD admit direct, mention, and explicit. Routed capability matching
is opt-in and MUST have a positive fanout limit. Control envelopes such as greet, whois, receipt,
trace, presence, and deterministic task-status projections are never eligible triggers.

Budgets are required for active. Zero MUST NOT mean unlimited. Administrative config defines
ceilings; a request may narrow them but may not exceed them. On exhaustion, the only initial policy
is mailbox: persist subsequent eligible envelopes without starting model turns.

`max_model_turns` limits admitted logical Network turns. One admitted turn owns one idempotent ACP
prompt dispatch; replay of that dispatch may recover state but may not silently create another
logical turn. `max_provider_attempts` is a separate root budget for every actual upstream provider
call inside the ACP agent, including initial, retry, fallback, and post-tool calls. `max_wall_time` is
the root activation deadline, and `max_activation_depth` is enforced from the persisted
root/causation chain.

The current ACP surface cannot enforce that provider-attempt promise. It reports aggregate
per-turn `TokenUsage`, but no attempt id, resolved provider/model, retry/fallback event, or
pre-attempt admission boundary (`internal/acp/types.go:505-518`). Public active therefore requires a
new capability-negotiated AGH ACP extension, not an inference from existing `TokenUsage`:

1. Before the ACP prompt dispatch, AGH atomically reserves a bounded sub-budget of provider
   attempts, input/output tokens, and wall time from the root activation ledger.
2. The prompt metadata carries the activation id, ACP turn id, deadline, and reserved sub-budget.
3. An adapter advertising `network_provider_attempt_budget_v1` MUST atomically consume that local
   sub-budget before every upstream provider call. It MUST stop before an attempt that would exceed
   the cap.
4. The adapter emits attempt-start and attempt-terminal records containing stable attempt id,
   ordinal, initial/retry/fallback/post-tool kind, resolved provider/model, outcome, and actual usage
   or an explicit `usage_unavailable` marker. Its terminal ACP result returns the unused allocation
   for reconciliation.
5. AGH durably settles those records against the root ledger. Missing terminal evidence retains the
   whole allocation as consumed; it is never refunded from an estimate.

An agent profile that does not advertise and honor this capability cannot satisfy public active and
returns `network_active_provider_accounting_unsupported`. It may still use off or mailbox. P0-B owns
the extension schema, trusted adapter preflight, capability negotiation, reservation, deterministic
conformance tests, and one pinned real-adapter integration; P0-C
owns durable attempt events, restart reconciliation, cancellation, and late-result handling. P1 may
add price analytics and richer attribution, but cannot be the phase that makes the budget truthful.

Concrete integration points are `Caps` and initialize-handshake projection at
`internal/acp/types.go:448-453` and `internal/acp/client.go:415-430`; structured prompt metadata at
`internal/acp/types.go:196-225`; prompt dispatch/terminal handling at
`internal/acp/client.go:756-779,935-991`; and session-update/usage translation at
`internal/acp/handlers.go:271-310,663-693`. The TechSpec must decide whether the extension is carried
as negotiated ACP metadata/notifications or through an AGH-owned adapter side contract, but it may
not encode unsupported fields into ordinary prompt prose and call that enforcement.

### Real adapter ownership and preflight

A mock-only extension is not a release path. Most built-in providers are external commands launched
through the generic ACP harness; Pi/OpenRouter and related providers use the external `pi-acp`
harness (`internal/config/provider.go:75-83,370-520`). Hermes and OpenClaw are also direct external
ACP commands (`internal/config/provider.go:446-465`). A generic stdio proxy cannot truthfully count
retries hidden inside those processes, so merely wrapping their ACP stream is insufficient.

TechSpec 1 MUST select, ship, and own at least one real active-capable adapter with interception at
the upstream provider-call boundary. The preferred initial shape is an AGH-controlled, exactly
pinned adapter artifact derived from a provider harness AGH can modify and test, such as a pinned
first-party variant of `pi-acp`; a first-party provider adapter is also valid. The unpinned current
`npx -y pi-acp@latest` command at `internal/config/provider.go:284-286` cannot be the proof artifact.
Direct third-party ACP commands remain eligible for off/mailbox but not active until they implement
the contract through a verified adapter.

Before any active session/run record is persisted, a provider-adapter registry MUST resolve a
trusted manifest keyed by adapter id, exact version, artifact digest, manifest digest, and protocol
version. Preflight materializes/verifies the exact executable artifact and returns the immutable
`NetworkProviderAttemptContract` stored in the resolved activation snapshot. A raw user-configured
command or self-asserted config capability is not trusted evidence. Extension adapters may register
through the same registry only with an administrator-authorized manifest, artifact verification,
and the canonical conformance suite.

The baseline MUST NOT keep a live ACP process across the owner commit. Pre-persistence preflight is a
pure lookup/verification of an immutable artifact in an adapter-registry-owned cache: it starts no
ACP process, injects no provider secret, prepares no sandbox, opens no stdio, creates no Network
lease, and returns no session-owned cleanup handle. A failed partial cache materialization is owned
and cleaned transactionally by the global adapter registry, independent of session/run creation.

After the active owner and immutable snapshot commit, normal session startup owns provider secrets,
sandbox state, subprocess, stdio, process registry record, and ACP connection. The live initialize
handshake MUST echo the same capability/protocol/adapter/digests before ACP session negotiation or
Network lease creation. A mismatch marks the already-committed owner as failed-start, runs the full
owned startup cleanup, and returns `network_active_provider_contract_mismatch`; it cannot downgrade
active to mailbox or emit a Network side effect. This preserves a pre-persistence eligibility check
without creating an unowned process interval.

The current order opens storage and persists metadata before `Driver.Start`, and `Driver.Start`
initializes before it negotiates the ACP session (`internal/session/manager_start.go:170-235`;
`internal/acp/client.go:182-208,398-435`). TechSpec 1 must insert artifact preflight before storage
open, add live contract verification between initialize and negotiation, and keep process launch
inside the existing manager/driver cleanup envelope. It must not move process launch ahead of the
cleanup defer at `internal/session/manager_start.go:198-209`. Deferred task runs validate the trusted
manifest before run persistence and repeat the live match before worker membership or activation.

Required failure ownership:

| Failure point | Durable result | Mandatory cleanup |
|---|---|---|
| Manifest absent or artifact/cache verification fails | No active owner persisted | Adapter registry removes partial cache transaction; no session resources exist |
| Owner storage/open/commit or pre-create hook fails after successful pure preflight | No active owner persisted | No ACP process/stdio/secrets/sandbox/lease exist; release request/run reservation |
| Provider secret or sandbox preparation fails after owner commit | Owner reaches failed-start | Clear redactions and temporary provider material, finalize/destroy prepared sandbox, close recorder, release reservations; no process/lease |
| Process launch or initialize/live-contract match fails | Owner reaches failed-start | Stop/wait process, close stdio/connection/process record, clear provider/sandbox/runtime material, close recorder, release reservations; no Network lease |
| ACP session negotiation fails after matching initialize | Owner reaches failed-start | Same complete owned cleanup; no membership, activation, or mailbox lease |
| Context cancellation or daemon shutdown at any post-commit stage | Deterministic failed/canceled owner state | Idempotent complete cleanup under an uncanceled bounded teardown context |

Current cleanup seams include ACP failed-start stop at `internal/acp/client.go:731-738`, process stop at
`:832-886`, manager process/recorder/session-dir cleanup at
`internal/session/manager_lifecycle.go:485-510`, provider redaction cleanup at
`internal/session/manager_start.go:198-212,253-257`, and sandbox preparation at
`internal/session/sandbox.go:69-145`. TechSpec 1 must audit and close any gap rather than assuming the
old order covers the new failure point. Fault injection must assert zero live PID/stdio/process
record, zero Network lease, no partial owner for pre-commit failures, terminal failed/canceled owner
for post-commit failures, released reservations, cleared secret/redaction/runtime material, and
destroyed temporary sandbox state.

P0 active admission requires one concrete agent/provider/adapter contract before owner persistence.
A late-bound worker pool whose provider is still unknown cannot select active; it returns
`network_active_provider_unresolved` and may be redesigned as mailbox or as separately resolved
child runs. A future multi-adapter pool would need an explicit set/intersection contract and is not
implicit in this proposal.

### Preset expansion

| Public preset | participation | activation.mode | Channel after resolution | Activation policy |
|---|---|---|---|---|
| off | disabled | disabled | empty | no triggers and zero budgets |
| mailbox | joined | manual | required | no automatic triggers and zero automatic budgets |
| active | joined | automatic | required | bounded triggers, calls, tokens, fanout, and coalescing |

Manual activation means Network ingress itself never starts a model turn. An already-running local
turn may explicitly read the inbox and respond through tools. That is ordinary local execution,
not envelope-triggered activation.

### Resolution trace

Every resolved field carries provenance:

~~~json
{
  "network_participation": {
    "version": "agh.network-participation.v1",
    "preset": "active",
    "participation": "joined",
    "workspace_id": "ws_123",
    "channel_strategy": "run",
    "channel_id": "coord-run_456",
    "mailbox_id": "mailbox_run_456",
    "activation": {
      "mode": "automatic",
      "eligible_triggers": ["direct", "mention", "explicit"],
      "max_model_turns": 4,
      "max_provider_attempts": 6,
      "max_input_tokens": 12000,
      "max_output_tokens": 4000,
      "max_fanout_per_envelope": 1,
      "max_activation_depth": 2,
      "max_wall_time": "2m",
      "coalesce_window": "250ms",
      "on_budget_exhausted": "mailbox",
      "provider_attempt_contract": {
        "capability_id": "network_provider_attempt_budget_v1",
        "protocol_version": "1",
        "adapter_id": "agh-controlled-adapter",
        "adapter_version": "1.0.0",
        "artifact_digest": "sha256:verified-adapter-artifact",
        "manifest_digest": "sha256:verified-adapter-manifest"
      }
    },
    "resolution": {
      "preset_source": "task_run_request",
      "channel_source": "derived_task_run",
      "activation_source": "workspace_config",
      "provider_adapter_source": "verified_adapter_registry",
      "administrative_network_enabled": true
    }
  }
}
~~~

Source values are stable machine-readable atoms. At minimum they distinguish:

- explicit_session_request
- task_run_request
- task_execution_profile
- coordinator_profile
- review_profile
- spawn_request
- detached_request
- loop_run_request
- loop_definition
- automation_job
- bundle_activation
- workspace_config
- daemon_config
- derived_task_run
- derived_loop_run
- built_in_off

Read APIs, CLI structured output, Situation Surface, hooks, SSE, and Web detail views MUST expose
the resolved preset and source. Operators must be able to answer why an execution joined Network
without reading config files or reconstructing defaults.

## Normative Effect Matrix

| Runtime effect | off | mailbox | active |
|---|---|---|---|
| Persist resolved spec snapshot | yes | yes | yes |
| Persist requested source/provenance | yes | yes | yes |
| Require workspace_id | no, unless required by owning execution | yes | yes |
| Resolve channel_id | never | required | required |
| Create derived execution channel | never | only for an explicit derived-channel strategy | only for an explicit derived-channel strategy |
| Validate existing named channel | not applicable | yes | yes |
| Create durable mailbox binding | no | yes | yes |
| Acquire live membership lease | no | yes, passive delivery posture | yes, active delivery posture |
| Register process peer identity | no | yes while live | yes while live |
| Appear in live channel roster | no | yes, marked mailbox | yes, marked active |
| Start transport because of execution | no | lazily | lazily |
| Acquire live broadcast transport handle | no | while lease is live | while lease is live |
| Acquire live direct/activation transport handle | no | while lease is live | while lease is live |
| Publish presence lease | no | yes | yes |
| Persist presence control traffic in conversation timeline | no | no | no |
| Set AGH_SESSION_CHANNEL | no | yes | yes |
| Set AGH_PEER_ID | no | yes, from live lease | yes, from live lease |
| Set AGH_NETWORK_PARTICIPATION | off | mailbox | active |
| Inject startup Network section | no | one compact capability line | one compact capability and budget line |
| Inject full CLI/protocol tutorial | no | no | no |
| Expose execution-scoped inbox read tool | no | yes | yes |
| Expose execution-scoped send tool | no | yes | yes |
| Expose thread/direct read tools | no | when permitted | when permitted |
| Expose channel-administration tools automatically | no | no | no |
| Accept durable ingress addressed to mailbox_id without live presence | no | yes | yes |
| Direct or mention causes model turn | no | no | policy- and budget-gated |
| Routed full delivery causes model turn | no | no | policy- and budget-gated |
| Control envelope causes model turn | no | no | no |
| Task-status projection causes model turn | no | no | no |
| Network envelope calls PromptNetwork | impossible | impossible | only after admission |
| Exhausted activation budget behavior | not applicable | not applicable | persist as mailbox |
| Actual provider token usage recorded | no Network usage | local turns only | yes, per activation/run/channel/mailbox/lease |
| Release live lease/transport handles/presence on stop | not applicable | mandatory | mandatory |
| Preserve durable mailbox binding/cursor on stop | not applicable | yes | yes |

### Roster semantics

Roster liveness, durable addressability, and activation posture MUST be separate fields. An active
heartbeat does not mean a peer accepts automatic model activation, and an offline mailbox does not
disappear merely because its process lease expired. Projections therefore expose:

- presence_state: local, active, inactive, expired, or unknown.
- delivery_posture: mailbox or active.
- mailbox_state: registered, retention_pending, or deleted.

The current official Network skill documents presence as liveness rather than ownership or security
(skills/agh/references/network.md:67-76). The new delivery_posture prevents callers from inferring
activation eligibility from presence.

### Durable recipient addressing

The target identity split requires a wire/storage hard cut. Current Network v0 addresses durable
conversation traffic with process peer ids: `Envelope.From`, `To`, and `Mentions` are peer-shaped at
`internal/network/envelope.go:196-218`; a directed send is rejected when `To` lacks live presence at
`internal/network/router.go:354-379`; and a direct-room id is derived from the two peer ids at
`internal/network/router.go:995-1027` and `internal/network/validate.go:212-241`. That cannot deliver
to an offline mailbox or preserve a direct room across a rotating live lease.

P0-C MUST introduce a new protocol version with explicit stable address fields and no compatibility
aliases:

~~~go
type NetworkEnvelopeAddressing struct {
    FromMailboxID    string   `json:"from_mailbox_id"`
    ToMailboxID      *string  `json:"to_mailbox_id,omitempty"`
    MentionMailboxIDs []string `json:"mention_mailbox_ids,omitempty"`
    SenderLeaseID    *string  `json:"sender_lease_id,omitempty"`
}
~~~

`sender_lease_id` is optional audit evidence. It is neither routing authority nor conversation
identity. Conversation/work envelopes require a durable sender mailbox. Daemon-generated control
traffic uses a separately typed runtime source, never a fabricated mailbox or peer, and remains
structurally ineligible for model activation. The ambiguous v0 `from`, `to`, and `mentions` peer-id
semantics are deleted rather than dual-read.

The full Network v1 envelope also makes `ts` and `expires_at` mandatory for remote directed delivery.
The sender runtime derives and server-clamps the deadline from the owning request and configured
delivery limit, persists both timestamps before dispatch, and includes them in the authenticated
immutable envelope digest. Every retry carries the same values. The home independently returns
terminal `envelope_invalid` for an invalid interval or one longer than its recipient/runtime maximum,
so a sender cannot widen TTL. It validates expiry before the mailbox transaction and never
accepts, stores as unread, or activates a delivery that arrives after the allowed deadline; it
persists an authenticated terminal `envelope_expired` disposition instead. Current v0 already has an
`Envelope.TS`, optional `Envelope.ExpiresAt`, and expiry validation at
`internal/network/envelope.go:213-214` and
`internal/network/validate.go:595-598`, but P0-C must define the cross-runtime clock source, maximum
clock-skew allowance, canonical time unit, and boundary comparison. Sender finalization waits through
that allowance, so a remote copy cannot be accepted after the sender reports local expiry.

At minimum, the home rejects `ts` beyond the allowed future skew, `expires_at <= ts`, and
`expires_at - ts` above its own maximum TTL; only `home_now < expires_at` may reach mailbox
acceptance. Authenticated runtime/home/route authority is checked first so a non-home endpoint cannot
mint a settlement. Envelope shape and deadline validity/expiry are then classified before mailbox
existence, permission, and tombstone state, giving retries one deterministic terminal reason without
leaking recipient state through an already invalid or expired envelope. The exact precedence is a
Network v1 conformance vector, not implementation accident.

The authoritative local/directory record is conceptually:

~~~go
type NetworkMailboxBinding struct {
    WorkspaceID  string `json:"workspace_id"`
    ChannelID    string `json:"channel_id"`
    MailboxID    string `json:"mailbox_id"`
    OwnerKind    string `json:"owner_kind"`
    OwnerID      string `json:"owner_id"`
    HomeRuntimeID string `json:"home_runtime_id"`
    RouteEpoch   uint64 `json:"route_epoch"`
    StateVersion uint64 `json:"state_version"`
    State        string `json:"state"`
    Cursor       string `json:"cursor,omitempty"`
}
~~~

P0-C may refine storage types, but these meanings are required. `home_runtime_id` and route metadata
come from an authenticated mailbox directory/binding, never an untrusted envelope field. P0-C keeps
the home binding immutable for a mailbox lifetime; transparent cross-runtime mailbox migration is a
separate protocol and MUST NOT be improvised by changing presence. A future move requires an
explicit fenced transfer/route-epoch design or a new mailbox id.

`home_runtime_id` is installation-scoped, not process-scoped. P0-C creates one opaque runtime
identity per `AGH_HOME`, persists it in a global runtime-identity record, and never derives it from
PID, hostname, port, or a transient bridge instance. The identity owns an authentication key/reference
through the existing secret boundary (`internal/vault/crypto.go:29-52`;
`internal/vault/service.go:69-108`). The exact mTLS/signature mechanism is TechSpec-owned, but an
unauthenticated runtime id string is not authority and private key material never enters public
config or mailbox rows.

A separate directory lease maps that stable identity to the current endpoint:

~~~go
type NetworkRuntimeEndpointLease struct {
    RuntimeID     string `json:"runtime_id"`
    EndpointEpoch uint64 `json:"endpoint_epoch"`
    Endpoint      string `json:"endpoint"`
    AuthKeyID     string `json:"auth_key_id"`
    ExpiresAt     string `json:"expires_at"`
}
~~~

The persistent runtime identity key authorizes endpoint registration; it is not reused as the live
endpoint credential. Each registration creates a fresh per-epoch endpoint signing key or
directory-issued lease credential referenced by `auth_key_id`, and replacing/expiring the lease
invalidates the predecessor credential. This is what makes endpoint epoch a cryptographic fence
rather than a caller-provided integer. Endpoint credential material is released with the owned live
transport lifecycle; the persistent runtime identity remains vault-backed.

Daemon restart reuses the persisted runtime id and atomically registers a higher endpoint epoch. The
mailbox `home_runtime_id` and route epoch do not change. Senders resolve the newest authenticated
endpoint lease; while none exists, the outbox remains pending. A superseded endpoint cannot produce
an acceptable disposition presentation or settle sender state, although a disposition it committed
while authoritative remains canonical and can be re-attested by the new endpoint. Runtime identity
rotation or mailbox home migration requires an explicit future protocol; it is not endpoint
re-registration.

#### Durable home disposition and tombstone ordering

Every terminal remote delivery decision is represented by an authenticated, persisted disposition:

~~~go
type NetworkMailboxDisposition struct {
    DispositionID          string `json:"disposition_id"`
    MessageID              string `json:"message_id"`
    WorkspaceID            string `json:"workspace_id"`
    ChannelID              string `json:"channel_id"`
    MailboxID              string `json:"mailbox_id"`
    HomeRuntimeID          string `json:"home_runtime_id"`
    MailboxRouteEpoch      uint64 `json:"mailbox_route_epoch"`
    CommittedEndpointEpoch uint64 `json:"committed_endpoint_epoch"`
    Status                 string `json:"status"`
    Reason                 string `json:"reason,omitempty"`
    CommittedAt            string `json:"committed_at"`
    Digest                 string `json:"digest"`
}

type NetworkMailboxDispositionPresentation struct {
    Disposition          NetworkMailboxDisposition `json:"disposition"`
    ServingEndpointEpoch uint64                    `json:"serving_endpoint_epoch"`
    ReattestationProof   string                    `json:"reattestation_proof"`
}
~~~

Minimum home settlement statuses are `home_accepted`, terminal `recipient_tombstoned`, terminal
`recipient_unknown`, terminal `recipient_forbidden`, terminal `envelope_invalid`, and terminal
`envelope_expired`. The last means the home received the authenticated envelope after its immutable
delivery deadline; it is not a sender-created substitute for an unreachable home response. A
retryable
`route_refresh_required` is an authenticated delivery response, not an immutable settlement
disposition. The home runtime persists one canonical disposition keyed by
`(workspace_id, channel_id, mailbox_id, message_id)` and returns the exact disposition id, fields,
and digest on every retry, including after restart. `digest` is computed over the canonical encoding
of every immutable field except `digest` itself; algorithm/version domain separation is part of the
Network v1 contract.

`committed_endpoint_epoch` is immutable historical evidence of the authoritative endpoint that made
the home-store decision; it is not the response-freshness gate. The currently registered endpoint
wraps the canonical record in a presentation and re-attests its exact digest. The proof binds the
home runtime id, disposition id/digest, workspace/channel/mailbox/message and mailbox route epoch,
plus `serving_endpoint_epoch`. The sender accepts a presentation only after resolving the current
authoritative directory lease and proving that the serving epoch and key match it. If that lookup is
unavailable, the outbox remains pending rather than trusting a cached endpoint. After a restart from
epoch N to N+1, the new endpoint can therefore re-attest the unchanged decision committed at N;
a delayed presentation from the superseded endpoint N fails freshness. Re-attestation MUST NOT
change status, reason, commit time, route epoch, or digest.

Mailbox deletion and delivery are serialized by one home-store transaction boundary:

- If the home commits delivery first, its idempotent result remains `home_accepted`; a later
  tombstone prevents new delivery but does not rewrite that result or erase authorized history.
- If the home commits the tombstone barrier first, a later arrival records and returns terminal
  `recipient_tombstoned`, even if the sender queued it earlier. Sender-local queue time is not home
  acceptance authority.
- A stale route or superseded-endpoint presentation is rejected until authenticated directory
  refresh. A canonical terminal disposition re-attested by the current endpoint stops outbox retry
  and remains inspectable.
- Tombstone/state version and disposition are durable across crash/restart. The delete transaction
  cannot leave an ambiguous state that causes infinite retry.
- P0-C defines server-bounded delivery age/attempts. Exhausting attempts may stop active dispatch,
  but cannot fabricate a home rejection; the sender waits through the immutable envelope deadline
  and clock-skew allowance, then records sender-local `delivery_dead_lettered_unconfirmed` if no
  authenticated home disposition exists. This is terminal for further sender retry, not proof that
  the home accepted or rejected the envelope.
- A late buffered/replayed copy still carries the same expired deadline, so a recovered home records
  canonical `envelope_expired` rather than accepting it. A genuine home disposition observed after
  local dead-letter may refine the sender's recorded ground truth but MUST NOT trigger republish.
  Tombstone, home-disposition, sender-dead-letter, and envelope-id retention MUST cover at least the
  maximum retry/replay plus settlement-grace window; a mailbox or message id is never reused after
  that retention ends.

This ordering deliberately favors a simple home-authoritative boundary over distributed timestamp
comparison. The sender can truthfully report `outbound_queued`, `home_accepted`,
`home_rejected:<reason>`, or `delivery_dead_lettered_unconfirmed` without claiming that a local
enqueue, publish, or retry exhaustion proved remote delivery state.

Addressing invariants:

1. A mailbox address is identified by `(workspace_id, channel_id, mailbox_id)` and belongs to one
   durable owner/binding. IDs are opaque, never reused after tombstone, and do not encode session or
   process identity.
2. Send validation resolves sender authority and recipient existence/visibility against durable
   mailbox bindings. It MUST NOT call presence as an existence gate. The current
   `Router.PrepareSend` `HasPresence` rejection at `internal/network/router.go:372-378` is deleted.
3. Durable acceptance means the recipient mailbox's home runtime has committed the envelope and
   delivery record. For a remote home, the sender outbox waits for an authenticated idempotent
   accepted or terminal-rejected disposition; route refresh remains retryable. Transport publish
   alone is not mailbox delivery, and tombstone rejection cannot retry forever.
4. After mailbox commit, routing resolves `(mailbox_id -> current fenced live lease)` only to decide
   immediate delivery/activation. No lease means the message remains unread in the mailbox, not a
   target-not-found error.
5. A process restart acquires a new peer/lease id and higher lease epoch for the same mailbox. The
   old fencing token cannot consume, acknowledge, mutate subscriptions, or settle activation after
   replacement.
6. Display handles are lookup conveniences. CLI/Web may accept an authorized handle, but the daemon
   resolves it to one mailbox id before send and persists the id. Renaming a display handle does not
   rewrite history or direct identity.

Direct rooms are keyed by the sorted pair of stable mailbox ids within one workspace/channel. The
target identity is derived from a versioned domain separator such as
`agh-network/direct-room/v1`, workspace id, channel id, mailbox A, and mailbox B. Current
`network_direct_rooms.peer_a/peer_b`, direct-room query filters, and `DirectRoomIdentity` v0 are
replaced with mailbox columns/logic (`internal/store/globaldb/global_db.go:382-402`;
`internal/network/validate.go:218-241`). Restarting either participant preserves the room. Deleting a
mailbox tombstones its address and prevents new home acceptance while preserving authorized historical reads;
recreation receives a new mailbox id and therefore a new direct room.

The same identity must own every durable collaboration relation:

- `network_thread_participants.peer_id` becomes `mailbox_id`
  (`internal/store/globaldb/global_db.go:349-381`).
- `network_subscriptions.peer_id` becomes `mailbox_id`; mute/digest/full and keyword filters survive
  process exit, while only their transport handles are lease-scoped
  (`internal/store/globaldb/global_db.go:163-176`; `internal/store/types.go:959-1010`).
- Envelope mentions persist resolved mailbox ids; thread routing compares mailbox participants and
  mentions, not `LocalPeer.PeerID` (`internal/network/router.go:1241-1265`).
- Work opener/target, coordinator ownership, delivery records, token/cost attribution, audit rows,
  task ingress source, and allowed/preferred Network recipients use mailbox ids. A lease/peer id may
  remain as optional observation metadata but never as the durable foreign key.
- Delivery queue/idempotency/coalescing keys use recipient mailbox id plus root/attempt identity.
  Session token or lease fencing is checked only when handing an admitted activation to a live
  process.

Mailbox-to-lease resolution is one-way and time-sensitive: durable routing selects the mailbox first;
then the lease registry may return a current local/remote live endpoint. A lease must never be used
to reconstruct a mailbox id after the envelope was accepted. Presence, whois, and roster projections
therefore return both stable `mailbox_id` and optional live `peer_id`/lease state with different
labels.

### Live lease projection

`PeerID` is not part of immutable `NetworkParticipationSpec`. It is allocated when a process
realizes a live lease and may change after restart:

~~~go
type NetworkLiveLease struct {
    LeaseID    string `json:"lease_id"`
    MailboxID  string `json:"mailbox_id"`
    PeerID     string `json:"peer_id"`
    SessionID  string `json:"session_id"`
    LeaseEpoch uint64 `json:"lease_epoch"`
    FencingToken string `json:"fencing_token"`
    State      string `json:"state"`
    ExpiresAt  string `json:"expires_at"`
}
~~~

The immutable spec owns intent, workspace/channel, durable mailbox identity, and activation policy.
The mutable lease projection owns process peer identity, liveness, transport subscription handles,
presence, fencing, and expiry. Durable subscription policy belongs to the mailbox. Restart creates a
new lease/peer projection attached to the same mailbox id; it neither
mutates the historical execution snapshot nor changes cursor ownership. `AGH_PEER_ID` is projected
from this live lease, while `AGH_SESSION_CHANNEL` and participation mode derive from the resolved
spec.

### Prompt semantics

The stable startup prompt contains identity, resolved preset, channel, and the statement that task
state remains authoritative. It does not contain command examples. Tool descriptors and the
official Network skill are the on-demand reference.

The current implementation repeats response guidance at startup and on Network turns
(internal/daemon/network_response_register_prompt.go:20-29, 47-71) and repeats command/protocol
guidance per delivery (internal/network/delivery.go:1647-1689). The target contract permits one
small activation header per admitted batch; it MUST NOT duplicate tool schemas or CLI tutorials.

## Resolution Rules

### Administrative gate

network.enabled answers only whether Network capability is available. It is not a participation
default. The current config combines availability and runtime tuning
(internal/config/config.go:459-473) and defaults availability to true
(internal/config/config.go:754-769). The target keeps the availability switch but removes all
participation inference from it.

Resolution order:

1. Resolve the requested preset.
2. Apply the network.enabled availability gate.
3. Resolve workspace scope.
4. Resolve or derive channel identity for participating presets.
5. Expand the public preset into internal planes.
6. Resolve active policy and cap it by administrative bounds.
7. For active, resolve/materialize a trusted provider-adapter manifest, verify its exact artifact,
   and attach the server-owned provider-attempt contract. Reject an unsupported or drifting adapter.
8. Validate the complete spec.
9. Persist the immutable snapshot before producing membership or prompt side effects.

If step 2 fails, AGH returns network_participation_unavailable. It does not continue with off. If
step 7 fails, it returns network_active_provider_accounting_unsupported before a session/run owner is
persisted and does not continue with mailbox.

### Preset precedence

Nearest explicit intent wins:

1. Invocation request for this concrete execution.
2. Owning persisted profile or definition.
3. Role-specific profile selected by the owning execution.
4. Built-in off.

Examples:

- A task start request overrides the task execution profile for that run only.
- A loop start request overrides the loop definition for that loop run only.
- An automation fire uses its persisted job request; webhook payloads cannot override it unless the
  automation schema explicitly authorizes and validates that input.
- A plain session create uses its explicit request and otherwise resolves off.

Workspace and daemon config define activation defaults/limits plus channel defaults used after an
explicit participating selection. This proposal deliberately has no named Network policy selector:
the owning request/profile stores the canonical preset/strategy/activation object directly. Config
MUST NOT select a non-off execution mode for an otherwise plain request. An owning
task/loop/automation/profile may carry non-off only because an operator explicitly saved that intent
on that definition.

Preset is resolved first. An explicit off clears every lower-precedence channel and activation
value. AGH MUST NOT produce a mixed spec such as preset off plus a channel inherited from a bundle.

### Channel strategy and resolution

For mailbox and active:

1. Resolve the `channel_strategy` from the same concrete request or owning profile that selected the
   participating preset. It MUST NOT come from workspace or daemon configuration.
2. `named` requires the same request/profile to contain `channel_id`.
3. `default` explicitly asks the resolver to use the configured workspace participation-channel
   default, then the daemon participation-channel default; absence is a validation failure.
4. `run` derives from the stable task-run id and is valid only for a task run.
5. `loop_run` derives from the stable loop-run id and is valid only for a loop run.
6. An omitted or domain-incompatible strategy is a validation failure.

A named channel MUST already exist in the same workspace unless the request explicitly invokes the
channel-create authority. A typo must not materialize a new channel.

No downstream task, session, daemon, coordinator, loop, review, or automation package may repeat
this derivation or fall back from an empty channel. The canonical resolver is the only owner of the
strategy-to-channel mapping.

### Whole-policy resolution

Activation policy is resolved as one object, not field-by-field across arbitrary layers. This
prevents a request fanout limit, workspace token budget, and daemon trigger set from accidentally
forming a policy no actor selected. Administrative ceilings are applied after selection and are
reported in the resolution trace.

### Authority and narrowing

Permission to address channels is a ceiling, not a default. A child request MUST satisfy both:

- the parent's delegated channel authority includes the target workspace/channel; and
- the child explicitly selected mailbox or active.

Hooks and extensions may deny or narrow a requested spec. They may widen it only through an
explicitly granted participation-policy capability before the snapshot is persisted. Post-create
hooks cannot mutate it.

## Invalid States and Deterministic Diagnostics

The resolver MUST reject:

| Invalid state | Diagnostic |
|---|---|
| preset off with channel_id | network_participation_off_has_channel |
| preset off with channel_strategy | network_participation_off_has_channel_strategy |
| preset off with activation policy | network_participation_off_has_activation |
| participating preset without channel_strategy | network_participation_channel_strategy_required |
| channel_strategy named without channel_id | network_participation_named_channel_required |
| channel_strategy other than named with channel_id | network_participation_channel_strategy_conflict |
| channel_strategy invalid for owning execution kind | network_participation_channel_strategy_invalid |
| preset mailbox with automatic triggers or budgets | network_mailbox_activation_forbidden |
| preset active without a positive activation budget | network_active_budget_required |
| active with zero/non-positive provider attempts, depth, wall time, or coalesce window | network_active_limit_required |
| invalid/non-canonical duration or value above administrative ceiling | network_activation_policy_exceeds_limit |
| active agent profile lacks provider-attempt budget capability | network_active_provider_accounting_unsupported |
| active execution has no concrete provider/adapter at resolution time | network_active_provider_unresolved |
| verified adapter artifact/manifest or live handshake differs from preflight | network_active_provider_contract_mismatch |
| activation default falls outside its administrative limit/range | network_activation_config_inconsistent |
| participating preset without resolved workspace | network_participation_workspace_required |
| participating preset without resolved channel | network_participation_channel_required |
| participating preset for a global-scoped execution | network_participation_workspace_scope_required |
| named channel belongs to another workspace | network_participation_channel_scope_mismatch |
| Network administratively disabled | network_participation_unavailable |
| request exceeds administrative budget/fanout ceiling | network_activation_policy_exceeds_limit |
| descendant requests a channel outside delegated authority | network_participation_permission_denied |
| raw legacy network_channel or coordination_channel_id field | unknown_field |
| mailbox delivery path invokes PromptNetwork | runtime invariant violation and failed delivery audit |
| active delivery cannot reserve budget | envelope persists; activation outcome budget_exhausted |

Unknown legacy fields fail schema validation. There are no aliases, dual reads, fallback metadata,
or compatibility conversion.

## Persistence and Lifecycle

### Persistence ownership

- Task execution profile stores the requested task default.
- Task run stores the resolved immutable snapshot for one attempt.
- Session stores the resolved immutable snapshot for one process lifetime; a participating durable
  session/profile or run owns the stable mailbox_id to which later process lifetimes reattach.
- Loop definition stores a requested default; loop run stores the resolved snapshot.
- Automation job stores a requested default; automation run and materialized task run store resolved
  snapshots.
- Review and spawn lineage store their own resolved snapshots and source lineage.
- Channel, mailbox-binding, and live-lease tables store only participating execution state.

The run snapshot contains channel_id. There is no second CoordinationChannelID column.

### Realization state machine

Participating execution realization follows two related lifecycles. Durable addressability follows:

    resolved -> persisted -> mailbox_registered -> dormant -> retention_expired/deleted

A live process attachment follows, with a monotonic epoch/fencing token allocated at lease join:

    mailbox_registered -> transport_ready -> lease_joined -> deliverable -> leaving -> dormant

off follows:

    resolved -> persisted -> closed

Rules:

- Persisted snapshot is the commit boundary before membership side effects.
- A live-lease join failure rolls back every live transport handle, presence lease, and session activation
  side effect, preserving the existing cleanup discipline around session network join
  (internal/session/manager_helpers.go:87-104).
- Live-lease leave is mandatory on normal stop, failed start, cancellation, provider exit, and
  daemon shutdown. It does not delete the durable mailbox binding, unread cursor, or accepted
  offline envelopes.
- Mailbox deletion is an explicit authorized operation or deterministic retention transition. A
  process stop MUST NOT serve as mailbox garbage collection.
- Restart reattaches the new process lease to the same mailbox_id and resumes from the persisted
  recipient cursor; it does not create a fresh address or mark unread envelopes consumed.
- Dormant is still addressable: directed validation targets the mailbox binding and the recipient
  home commits unread delivery without requiring a live peer. Reattach resolves the mailbox to the
  new fenced lease only after that durable step.
- Queued active envelopes that have not started a model turn become mailbox entries when the
  activation budget is exhausted.
- An in-flight provider turn is charged to the policy that admitted it even if cancellation follows.
- Control messages and presence leases are stored in their own operational tables or metrics, not
  the user conversation timeline. The current heartbeat persists every greet as a conversation
  message (internal/network/manager.go:821-881); that behavior is outside the participation
  snapshot and should be removed by the Network-efficiency design.

### Immutability and transitions

Resolved participation is immutable for a session lifetime, run attempt, loop run, review session,
or automation run.

- Updating a task profile affects future runs.
- Updating a loop definition affects future loop runs.
- Updating an automation job affects future fires.
- Updating workspace or daemon channel defaults, activation defaults, or limits affects future resolutions
  of executions that explicitly participate; it does not change the off fallback.
- Changing a live session from off to mailbox/active or back requires a controlled session restart
  or resume that creates a new resolved snapshot. AGH MUST NOT mutate process environment and tool
  grants underneath a live provider.
- Subscription mute/digest settings may change within a participating lifetime, but they do not
  change the preset.
- Cancel and retry create a new run attempt with a fresh resolution; they do not mutate historical
  evidence.

## Domain Behavior

### User sessions

Current create contracts accept only Channel (internal/session/manager_types.go:17-36;
internal/api/contract/contract.go:35-45), and session responses project it
(internal/api/contract/contract.go:105-121).

Target:

- Create accepts network_participation.
- Session response returns immutable resolved_network_participation and, only while realized, a
  separate network_live_lease projection. Peer/liveness changes never rewrite the spec.
- off creates no peer, environment channel, Network prompt section, Network turn augmenter, or
  execution-scoped Network tools.
- mailbox and active join only after the session snapshot commits.
- The old session Channel field is deleted.
- A channel address by itself is invalid; users must select mailbox or active.

### Tasks and task runs

The Task Execution Profile is already the typed orchestration overlay managed across CLI, API,
native tools, and Web (docs/_memory/glossary.md:245-249). It is the canonical persisted home for a
task's requested participation default.

Target:

- Task create may atomically initialize execution_profile.network_participation.
- Profile update replaces that requested object under existing profile replacement semantics.
- Start, publish, approve, enqueue, fan-out, retry, and recovery accept an optional concrete-run
  override.
- Run reservation resolves and persists the spec.
- off does not call ensureQueuedRunCoordinationChannel and creates no network_channels row.
- mailbox/active create a derived run channel only when channel_strategy = "run" was explicitly
  selected.
- Claim, detail, inspection, SSE, and hooks project resolved participation.
- Situation Surface includes preset and source; it does not show a phantom coordination channel for
  off runs. The current summary always carries CoordinationChannelID
  (internal/situation/task_context.go:508-527).
- Network-origin thread promotion may explicitly initialize mailbox or active; promotion does not
  make all descendant retries or reviews active unless the persisted profile says so.
- Task terminal status remains authoritative and is never inferred from Network messages.

### Worker sessions

A worker bound to a run uses the run's resolved spec. It does not independently re-resolve mutable
workspace defaults after claim.

- off worker: local task tools only.
- mailbox worker: durable inbox/send tools, no envelope wake.
- active worker: admitted Network batches may wake it within the run budget.
- A worker's session token usage and Network activation usage are attributed to both session_id and
  run_id.
- Worker identity and task-role prompt composition are not keyed by a fictional channel. Today,
  `taskRoleSessionName` substitutes `"default"` when no channel exists and
  `taskRolePromptOverlay` always prints `Coordination channel: default`
  (`internal/daemon/task_role_sessions.go:260-262,286-300`). In off, the stable identity derives
  from role/run/profile inputs and the overlay contains no Network line. In mailbox/active, one
  compact line is rendered from the resolved spec; it never invents a channel fallback.

### Coordinator

Coordinator orchestration and Network transport are separate.

The current bootstrap requires a non-empty coordination channel
(internal/coordinator/coordinator.go:97-142), the allowlist always includes Network tools
(internal/coordinator/coordinator.go:46-61), and the prompt always teaches channel communication
(internal/coordinator/coordinator.go:233-257).

Target:

- Bootstrap eligibility depends on task/run state, workspace scope, coordinator config, singleton
  rules, and spawn caps. It does not depend on Network participation.
- The coordinator always observes and manages task/run state through task APIs.
- Coordinator Network participation is resolved from its own coordinator profile, not inherited
  from each run it observes.
- A run's active preset does not silently join the workspace coordinator to its channel.
- An explicitly participating coordinator receives only its authorized channel memberships and
  conditional Network tools.
- Coordinator wake remains a synthetic task-origin prompt, not a Network activation
  (internal/daemon/coordinator_runtime.go:475-508, 618-628).
- PromptOverlay emits channel guidance only when the coordinator's own resolved spec participates.
- PermissionPolicy composes task tools unconditionally and Network tools/channels conditionally.
- Multi-channel coordinator communication, if enabled, is represented as explicit membership
  leases authorized by the coordinator profile. It is not modeled by copying the most recent run's
  channel into a singleton session field.

This design lets a coordinator orchestrate entirely local task runs and prevents Network from
becoming a hidden prerequisite for autonomy.

### Reviews

Current review creation selects a channel from review preferences and then falls back through run
and task channel fields (internal/daemon/review_router.go:779-797).

Target:

- Review profile contains an optional NetworkParticipationRequest.
- Default review participation is off, even when the reviewed run is active.
- Review context reads persisted task/run evidence locally.
- A reviewer joins Network only when the review profile explicitly selects mailbox or active and
  channel authority permits it.
- A review may use a participating channel as evidence, but verdict submission remains the review
  service authority.
- Continuation rounds resolve independently and preserve the selected profile provenance.
- The fallback chain through run.CoordinationChannelID, run.NetworkChannel, and
  task.NetworkChannel is deleted.

### Spawn and child sessions

Current spawn permissions carry NetworkChannels as an allowed authority set
(internal/coordinator/coordinator.go:168-175 and internal/cli/spawn.go:120-137).

Target:

- Spawn request carries a child network_participation request.
- Child default is off; parent channel authority is never participation inheritance.
- Requested mailbox/active must be within the parent's delegated workspace/channel authority.
- Child resolved spec and source are stored in lineage.
- A fork, teammate, worktree, or provider child may participate only under the same contract.
- A swarm topology may explicitly construct several participating children, but ordinary spawn
  does not.

### Detached work

Detached task submission currently inherits ownerInfo.Channel when the request has no channel
(internal/daemon/harness_detached_work.go:235-265).

Target:

- Detached request carries its own NetworkParticipationRequest.
- Default is off.
- Owner session participation is available as authority context, not a default.
- WakeTarget remains a synthetic task reentry path and does not require Network.
- Task and run snapshots retain the detached request's explicit source.
- detachedHarnessChannel and every owner-channel fallback are deleted.

### Loops

Loop runtime must remain independent of Network by default. The current unconditional channels on
action and judge sessions (internal/daemon/loop_runtime_adapters.go:37-81, 118-161) violate that
boundary.

Target:

- Loop definition and loop start request accept NetworkParticipationRequest.
- Built-in loop default is off.
- Loop coordinator task/run and every ordinary action/judge session remain off unless the loop run
  explicitly participates.
- A participating loop uses one loop-run channel by default, not one channel per node, generation,
  action, or judge.
- An action that uses agh__network_send or channel_result declares a Network requirement in the
  loop definition. Validation rejects that graph when the resolved loop preset is off.
- AGH never silently upgrades an off loop because a tool name appears at runtime.
- Node-specific membership, when truly required, is explicit in node policy and bounded by the
  loop-run participation budget.
- Loop state, gates, convergence, and artifacts stay in the Loop service and SSE model, not Network
  envelopes.

### Automation

Current JobTaskConfig exposes a raw NetworkChannel
(internal/automation/model/types.go:115-121), and dispatch forwards it to task creation and enqueue
(internal/automation/dispatch.go:674-678, 1182-1189).

Target:

- Job task and loop targets carry NetworkParticipationRequest.
- Job default is off.
- Trigger source, webhook route, bridge target, and extension origin do not imply Network
  participation.
- A concrete automation fire snapshots resolved participation before it creates a task/run or loop
  run.
- Retry uses the same persisted job request but produces a new resolved execution snapshot.
- Pre-fire hooks may narrow or deny; they cannot silently widen without the explicit capability.
- The raw network_channel automation field is deleted.

### Network-origin task status

The current task-status observer publishes Say messages to the origin thread
(internal/daemon/network_task_status_observer.go:180-241, 413-444) and is composed whenever the
Network runtime is available (internal/daemon/task_event_bridge_notifier.go:106-138).

Target:

- Status projection is enabled only for tasks with an explicit Network origin subscription.
- Projection is a typed deterministic state notification, not a conversational `say` envelope and
  not model activation. The current dead observer must be deleted or replaced with a projection
  path structurally unable to call `PromptNetwork`; merely rebinding its missing source container
  would make the token-expensive behavior live.
- mailbox and active recipients persist it; neither automatically prompts for control/status
  messages.
- The UI and API read task status from task state, not from the projected message.

## Ingress Rules and Exceptions

### Absolute Network rules

- No Network envelope activates an off execution.
- No Network envelope activates a mailbox execution, including direct messages, mentions, routed
  full delivery, receipts, traces, task status, or retry.
- An active envelope activates only after validation, subscription resolution, trigger eligibility,
  deduplication, coalescing, and budget reservation.
- greet, whois, receipt, trace, presence, roster, delivery acknowledgement, and task-status
  envelopes are deterministic runtime inputs and never model prompts.
- A Network message cannot create or mutate task/run/loop/review authority except through an
  explicit public promotion or management API with normal authorization.

### Non-Network ingress

The following are not exceptions to the Network policy because they are separate ingress types:

- User prompt to a session.
- Task scheduler or coordinator synthetic wake.
- Creator wake after a child task transition.
- Review request prompt.
- Loop action or judge prompt.
- Automation trigger prompt.
- Explicit operator session prompt.

These paths may start local model turns according to their own budgets and permissions. During such
a turn, a mailbox participant may explicitly read its inbox. The stored Network envelope is not
the cause of the turn.

### No hidden emergency bypass

There is no direct-message, mention, coordinator, extension, bridge, or administrator envelope that
bypasses off or mailbox. An operator who needs live handling must start a new active session
lifetime or execution attempt through the public participation surface.

## Lazy Transport Boundary

network.enabled remains an administrative availability switch. With it enabled:

- DB-backed channel and policy reads may operate while transport state is dormant.
- The embedded transport starts on the first mailbox/active live lease, explicit Network listener
  requirement, a durable mailbox binding configured for remote/offline ingress, or a Network
  administration operation that needs live transport. A local-only mailbox binding may remain
  DB-backed while transport is dormant.
- It stops when no live leases, remote-ingress mailbox bindings, or configured listener leases
  remain, subject to a bounded idle grace period. Process exit alone does not drop a durable
  remote-ingress requirement.
- Network status reports disabled, dormant, starting, active, stopping, or failed.
- An optional administrative listener_mode = "always" preserves deliberate remote ingress use
  cases. The built-in mode is on_demand.

This changes startup ownership but not NATS subjects, envelope semantics, delivery guarantees, or
conversation persistence.

Replacing embedded NATS and implementing the durable accepted/published state machine are deferred
to the P0-C TechSpec, not deferred for lack of evidence. The current source audit already authorizes
that TechSpec: first local send can select zero receivers
(`internal/network/manager.go:680-712`, `internal/store/globaldb/global_db_network_conversations.go:71-168`,
`internal/network/router.go:1241-1265`); work has competing in-memory and SQLite authorities
(`internal/network/router.go:202-216,1066-1102`,
`internal/store/globaldb/global_db_network_work_mutation.go:14-177`); and persist-before-publish can
turn a failed publish into a false-success duplicate retry (`internal/network/manager.go:690-710`).
A transactional outbox or equivalently rigorous durable state machine is therefore required before
non-off modes are public. Crash injection is acceptance evidence, while N=1/3/10/50 profiling and
fault results decide whether NATS remains the local transport or a shared in-process/remote
transport abstraction is simpler.

## Agent Swarm and Mixture-of-Agents Boundary

### Separate concepts

Network participation is a communication policy. Swarm and MoA are execution topologies.

| Concern | Network participation | Swarm/MoA topology |
|---|---|---|
| Primary question | Can this execution exchange messages, and can messages wake it? | How many agents/models run, in what graph, and who synthesizes? |
| Lifetime | Potentially long-lived | Bounded to one objective/round |
| Coordination state | Channel, mailbox, thread, direct room | Task graph, candidate results, aggregator input |
| Default | off | single agent |
| Model activation | Envelope policy | Explicit orchestration step |
| Output | Messages and durable conversation | Typed candidate artifacts and one aggregate |
| Presence/heartbeat | Relevant | Usually irrelevant |
| Wire protocol | AGH Network | Runtime-local unless remote participation is explicitly selected |

The local Hermes snapshot at commit b8880f1 provides a stronger opt-in precedent than an always-on
collaboration feature. MoA is a virtual provider selected through normal model surfaces; its
aggregator remains the acting model while reference models provide private analysis
(.resources/hermes/website/docs/user-guide/features/mixture-of-agents.md:9-30, 50-60). Hermes
separates two explicit user intents:

- Persistent selection: the desktop model menu switches the session to a named MoA preset through
  the same persistent provider-selection path as ordinary models
  (.resources/hermes/apps/desktop/src/app/shell/model-menu-panel.tsx:182-196, 327-346).
- One-shot selection: /moa temporarily selects the default preset for one turn. The gateway restores
  the previous model from a `finally`-owned path
  (.resources/hermes/gateway/run.py:9951-9984,10329-10348), while the direct CLI and TUI mutate and
  restore inside a success path that can leak after an exception
  (.resources/hermes/cli.py:12338-12369;
  .resources/hermes/tui_gateway/server.py:9031-9081). AGH should copy the explicit one-shot intent,
  not mutable switch-and-restore state.

This is the relevant product pattern for AGH: persistent workspace/profile selection and a
concrete-run override are separate explicit surfaces, and neither changes behavior merely because
supporting infrastructure exists.

The second-brain orchestration synthesis likewise distinguishes parallel fan-out/fan-in,
supervisor, conversational, and decentralized swarm topologies
(/home/pedronauck/Projects/second-brain/research/agent-networks/wiki/concepts/Multi-Agent Orchestration Patterns.md:52-95,
235-251). It recommends starting with one agent and adopting multi-agent execution only when
single-agent limits are observed (same file:31-50).

### AGH target behavior

- A MoA execution uses one explicit orchestration request, N bounded independent candidate calls,
  and one aggregator call.
- Candidate agents do not join a Network channel, publish greet messages, discover peers, create
  threads, or receive one another's full transcripts by default.
- Candidate outputs are typed artifacts returned directly to the aggregator.
- The aggregator receives only the original objective, candidate outputs, provenance, confidence,
  and budget evidence needed to synthesize.
- Candidate routing chatter and tool tutorials are excluded from candidate context.
- Candidates receive a compact safety/authority envelope and only data classes allowed to leave the
  workspace/tenant for the selected provider. Provider allowlists, redaction, secrets filtering,
  and egress audit are part of admission; dropping the actor's full system prompt is not permission
  to drop safety policy.
- Multimodal and structured content is projected through typed parts or explicitly rejected. Hermes'
  `_reference_messages` retains only string content, so images/structured arrays can become empty
  (`.resources/hermes/agent/moa_loop.py:437-466`).
- Parallel candidate calls share a stable prompt prefix where provider caching permits.
- Reference candidates are stateless advisors without tools or execution ownership; only the
  aggregator is the actor. Hermes makes that role split explicit in the reference prompt and
  aggregator call path (.resources/hermes/agent/moa_loop.py:93-118, 960-1022).
- Reference calls execute concurrently and preserve stable result ordering; an individual
  reference failure becomes bounded advisory evidence instead of aborting the acting aggregator
  (.resources/hermes/agent/moa_loop.py:220-385).
- Token and cost accounting records every actual attempt and resolved fallback identity. Hermes'
  per-slot accounting is a useful no-fallback shape, but auxiliary fallback may execute another
  provider/model while `_RefAccounting` retains the configured slot
  (`.resources/hermes/agent/moa_loop.py:238-249,280-323`;
  `.resources/hermes/agent/auxiliary_client.py:6905-6952`).
- Max candidates, models, wall time, input/output tokens, and aggregation rounds are explicit.
- Default rounds = 1. Additional rounds require a disagreement or confidence criterion and a
  remaining budget.
- The final answer and candidate evidence are stored under the owning task/loop/session, not as a
  channel transcript.
- A decentralized swarm uses task ownership, typed handoff artifacts, and distributed tracing. It
  does not require all-to-all Network chat.
- Network may be selected independently when candidates truly are remote long-lived peers. In that
  case each membership still has off/mailbox/active semantics and its own activation budget.

Supervisor translation overhead is a known failure mode: the second-brain synthesis records gains
from removing routing chatter from worker context and forwarding worker outputs without lossy
paraphrase (Multi-Agent Orchestration Patterns.md:131-139). AGH should therefore pass typed raw
candidate artifacts to the aggregator and summarize once, not turn every candidate result into a
Network conversation round.

### Required modeling consequence

Do not add swarm, MoA, ensemble, proposer, critic, or aggregator fields to the Network envelope.
Define execution topology under the owning runtime domain. NetworkParticipationSpec is an
independent optional communication policy on each execution node.

## Cross-Surface Contract

### HTTP and UDS

HTTP and UDS use the same generated contract:

~~~json
{
  "network_participation": {
    "preset": "off"
  }
}
~~~

Participating example:

~~~json
{
  "network_participation": {
    "preset": "active",
    "channel_strategy": "named",
    "channel_id": "builders",
    "activation": {
      "eligible_triggers": ["direct", "mention"],
      "max_model_turns": 4,
      "max_provider_attempts": 6,
      "max_input_tokens": 12000,
      "max_output_tokens": 4000,
      "max_fanout_per_envelope": 1,
      "max_activation_depth": 2,
      "max_wall_time": "2m",
      "coalesce_window": "250ms"
    }
  }
}
~~~

The object replaces:

- CreateSessionRequest.channel (internal/api/contract/contract.go:35-45).
- CreateTaskRequest.network_channel and child/update equivalents
  (internal/api/contract/tasks.go:680-730).
- EnqueueTaskRunRequest.network_channel and TaskExecutionRequest.network_channel
  (internal/api/contract/tasks.go:758-789).
- Fan-out network_channel and every response-level NetworkChannel or
  CoordinationChannelID projection.

Responses return requested input where the owning definition is read and resolved input where a
concrete execution is read. Names MUST distinguish requested_network_participation from
resolved_network_participation when both appear in one payload.

Contract, OpenAPI, generated TypeScript client, mock handlers, and CLI clients co-ship through
normal code generation. No hand-maintained parallel schema is allowed.

### CLI

The target CLI uses the following flags. P0-B advertises/accepts only `off`; non-off values return
the typed not-released diagnostic until P0-C and their preset gate pass.

    --network off|mailbox|active
    --network-channel <channel-id>
    --network-channel-strategy named|default|run|loop-run
    --network-trigger direct|mention|explicit|routed
    --network-max-turns <n>
    --network-max-provider-attempts <n>
    --network-max-input-tokens <n>
    --network-max-output-tokens <n>
    --network-max-fanout <n>
    --network-max-depth <n>
    --network-max-wall-time <duration>
    --network-coalesce-window <duration>
    --network-on-budget-exhausted mailbox

Examples:

    agh session new --network off -o json
    agh session new --network mailbox --network-channel-strategy named --network-channel builders -o json
    agh task start task-123 --network active --network-channel-strategy run -o json
    agh loop run review-cycle --network off -o json

Rules:

- --network-channel without mailbox or active is invalid.
- --network-channel requires --network-channel-strategy named; named requires --network-channel.
- --network mailbox/active without an explicit strategy is invalid.
- CLI `loop-run` serializes as canonical JSON `loop_run`.
- `--network-trigger` is repeatable and serializes as `eligible_triggers`; every activation-policy
  field accepted by HTTP/Web has a CLI representation with the same validation and provenance.
- Duration flags accept the same canonical duration strings as HTTP, UDS, config, and generated
  clients. The CLI does not reinterpret bare integers as milliseconds or nanoseconds.
- A profile whose ACP adapter lacks `network_provider_attempt_budget_v1` rejects active before
  creating the owning session/run and prints the typed unsupported-accounting diagnostic.
- `agh provider list -o json` and the corresponding HTTP/UDS/provider-status projection expose
  `network_active_support`, the verified adapter contract, and an unavailable reason. This is a
  preflight/status read and does not start a session or join Network. Current inventory seams are
  `internal/api/contract/providers.go:17-28`, `internal/api/core/providers.go:117-152`, and
  `internal/cli/provider.go:168-185,689-706`.
- --network mailbox rejects activation-budget flags.
- Structured output includes the resolution trace.
- Human output shows Local, Mailbox #channel, or Active #channel and budget.
- The old execution --channel flag is deleted from session/task/fan-out/automation contexts.
- Network-addressing commands retain --channel because it identifies the message destination, not
  execution participation.

P0-C separately hard-cuts Network-addressing commands. Directed send/direct-open/subscription/work
inputs use an explicit mailbox id or an authorized handle that the daemon resolves to one mailbox id
before acceptance. Structured output always returns the resolved mailbox id. `--peer`/peer ids remain
valid only for live presence/lease inspection; they are not accepted as durable `--to`, direct-room,
mention, subscription, or work-owner values. HTTP, UDS, native tools, generated clients, and Web use
the same v1 distinction.

Current generated CLI docs already call session --channel an opt-in
(packages/site/content/runtime/cli-reference/session/new.mdx:34), but task commands describe it as
an optional override while the store creates a channel anyway. The new flags make the behavior
truthful and consistent.

### Web UI

After their release gates pass, every create/run surface that can resolve participation uses a
segmented control containing the currently released values:

- Local: off.
- Mailbox: mailbox.
- Active network: active.

P0-B create/run surfaces expose only Local. After P0-C, Active appears only after the active gate
passes and Mailbox appears only after the mailbox gate passes. A surface MUST NOT label an
unreleased, non-durable, or activating implementation as either mode.

Even after the global active gate passes, Active is enabled only after the selected agent/provider
resolves to a verified adapter contract. Unsupported direct ACP profiles show Local/Mailbox and a
typed reason; they do not show a selectable Active value that will fail after submission. Changing
the agent/provider immediately recomputes availability from daemon truth.

Local is selected for a plain create. A task, loop, automation, coordinator, review, or reusable
session profile may show a non-off value only when that owning definition explicitly persisted it;
the UI labels its source and allows an explicit Local override for the concrete run.

Mailbox and Active reveal an explicit channel-strategy control, with domain-valid options only:

- `named`: reveal a same-workspace existing-channel selector.
- `default`: show the resolved workspace/daemon channel preview and fail if none exists.
- `run`: available only on task-run-owning surfaces; preview the stable derived id.
- `loop_run`: available only on loop-run-owning surfaces; preview the stable derived id.

Active additionally reveals bounded advanced activation settings. The UI MUST submit both preset
and strategy. It never infers `named` from a populated channel field, infers participation from field
presence, or offers a strategy invalid for the owning execution kind.

Current raw channel inputs to replace:

- Task editor ingress/channel sections
  (web/src/systems/tasks/components/task-editor-modal.tsx:230-241, 251-281).
- Automation task-run Network channel input
  (web/src/systems/automation/components/job-form/task-run-step.tsx:55-80).
- Task fan-out channel input
  (web/src/systems/tasks/components/tasks-fan-out-runs-card.tsx:86-99).

Detail views:

- Show no Network chip for off.
- Show Mailbox or Active with channel and source for participating runs.
- Show remaining/used activation calls and actual tokens only when runtime accounting supports
  them.
- Do not estimate provider cost from byte counts and present it as actual usage.
- Link to Network activity only when channel_id exists and the viewer has workspace access.
- Network roster/direct/inbox views show stable mailbox identity and delivery posture separately
  from optional live peer/lease state. An offline mailbox remains selectable as a directed recipient;
  a green presence indicator is not treated as address validity.

### Config lifecycle

Proposed configuration:

~~~toml
[network]
enabled = true
listener_mode = "on_demand"
default_channel = "default"

[network.activation.defaults]
eligible_triggers = ["direct", "mention", "explicit"]
max_model_turns = 4
max_provider_attempts = 6
max_input_tokens = 12000
max_output_tokens = 4000
max_fanout_per_envelope = 1
max_activation_depth = 2
max_wall_time = "2m"
coalesce_window = "250ms"
on_budget_exhausted = "mailbox"

[network.activation.limits]
allowed_triggers = ["direct", "mention", "explicit"]
max_model_turns = 8
max_provider_attempts = 12
max_input_tokens = 24000
max_output_tokens = 8000
max_fanout_per_envelope = 2
max_activation_depth = 3
max_wall_time = "5m"
min_coalesce_window = "100ms"
max_coalesce_window = "2s"
allowed_budget_exhaustion = ["mailbox"]
~~~

Semantics:

- enabled is availability.
- listener_mode controls transport lifetime.
- The built-in execution default is off and is not widened by daemon/workspace config.
- default_channel is consulted only after mailbox or active has been selected. It never enables
  participation.
- `network.activation.defaults` supplies omitted fields only after an owning request/profile has
  explicitly selected active. It never selects participation.
- `network.activation.limits` is administrative policy. Numeric `max_*` fields are inclusive
  ceilings, `allowed_triggers` and `allowed_budget_exhaustion` are allowlists, and coalescing has an
  explicit permitted range because both too-small and too-large windows are unsafe. A request may
  only narrow these bounds.
- Default and limit keys are distinct and provenance records each field's source. No key serves both
  roles, and reload validation rejects a default outside its corresponding limit/range.
- Provider-attempt enforcement is a non-bypassable active-mode capability requirement, not a config
  boolean that an operator can disable.
- Explicit task, loop, coordinator, review, session, and automation profiles may request a
  participating preset; administrative config may only cap or deny that request.
- Config read/write tooling exposes the new keys. The current tool surface already exposes Network
  settings at internal/config/tool_surface.go:239-249; it must add participation, listener, and
  activation-budget paths.
- Invalid config fails startup/reload. There is no compatibility read of removed keys.
- Config reload affects future resolutions and transport listener leases, not immutable live
  snapshots.

### Native tools

The current coordination toolset registers Network status, channel, inbox, peer, send,
subscription, thread, direct, and work descriptors
(internal/tools/builtin/network.go:12-105, 113-269).

Target gating:

- off: execution-scoped Network inbox/send/thread/direct/work tools are unavailable with a
  diagnostic naming preset off. Administrative Network tools remain available only through an
  independent management permission.
- mailbox: inbox, send, permitted read, and explicit subscription tools are available.
- active: mailbox tools plus activation-policy inspection are available.
- Channel create/update and global Network status are never granted merely because an execution
  participates.
- Tool availability diagnostics report required preset, current preset, channel scope, and
  administrative availability.
- Tool descriptor projection is conditional so off sessions do not pay descriptor tokens for
  Network tools.
- Agent-kernel `agh ch` commands remain installed as explicit Network commands, but an off session
  receives a typed not-participating diagnostic and no prompt/help projection that suggests using
  them. Administrative callers use a separate management permission.
- agh me context and session/task inspection expose the resolved spec; a separate setter tool is
  not required if the owning task/session/loop management tools accept the canonical object.
- Provider/agent inspection through CLI, HTTP, UDS, and native management tools exposes active
  support, adapter/protocol version, verified digests, last conformance status, and unavailable
  reason. Reading this status is administrative/local and does not grant Network participation.
- P0-C send/direct/mention/subscription/work tool schemas accept stable mailbox ids and return
  resolved mailbox identity. Peer ids are available only to presence/lease management descriptors;
  no descriptor aliases a peer id into a mailbox recipient.

Native tool ids, descriptors, I/O schemas, digests, capability gates, and fallback CLI docs must
co-ship.

### Hooks and SSE

TaskRunContext currently exports both CoordinationChannelID and NetworkChannel
(internal/hooks/payloads.go:948-957), and task hook assembly contains fallback logic across both
(internal/task/manager_run_hooks.go:131-161, 180-188).

Target:

- Hook payloads contain resolved_network_participation as one object.
- Hook matchers may match preset, channel_id, activation mode, and source.
- Pre-resolution hooks receive requested input and may deny/narrow within authority.
- Post-resolution and lifecycle hooks receive the immutable snapshot and cannot mutate it.
- SSE session, task-run, loop-run, automation-run, and Network events carry workspace_id and the
  resolved preset where relevant.
- off emits no membership or Network delivery events.
- mailbox emits stored/delivered-to-mailbox events but no activation-started event.
- active emits activation admitted, coalesced, started, completed, exhausted, and rejected events.
  The capability-negotiated ACP extension additionally emits actual attempt identity, kind,
  provider/model, outcome, and token usage or an explicit unavailable marker; monetary cost is
  nullable when the provider has no trustworthy price data.
- P0-C envelope/hooks/SSE payloads use `from_mailbox_id`, `to_mailbox_id`, and
  `mention_mailbox_ids`. Optional `sender_lease_id`/live peer evidence is separately labeled and may
  disappear without changing conversation identity.
- No hook matcher silently treats a missing legacy channel field as off.

### Bundles and extensions

Bundle payloads currently expose PrimaryChannel, BindPrimaryChannelAsDefault, and effective default
channel projections (internal/api/contract/bundles.go:5-14, 75-91, 106-123).

Target:

- Bundle channel declaration remains a resource declaration.
- Selecting a bundle primary channel may configure channel selection, but it cannot select mailbox
  or active unless bundle activation explicitly includes a participation request confirmed by the
  operator.
- BindPrimaryChannelAsDefault is replaced with an unambiguous channel-default field whose
  documentation states that it does not enable participation.
- Bundle profiles may declare a required or recommended participation preset. Required active
  participation is shown before activation and must be explicitly accepted.
- Extensions declare required Network capabilities and channel scopes in their manifest.
- Extension hooks cannot access another workspace's participation state.
- MCP sidecars and bridge SDKs receive only the scopes granted to the concrete execution.
- A bridge route named channel is not an AGH Network channel and never enables participation.
- Resource reconciliation may materialize declared channel resources, but it does not join agents.

### Official AGH skill and docs

The official skill already says to use its Network reference only when the session participates
(skills/agh/references/network.md:16-32). Update it to define off, mailbox, and active, including
the mailbox zero-activation guarantee and active budgets.

Required documentation changes:

- skills/agh/references/network.md
- skills/agh/references/tasks-and-orchestration.md
- generated CLI reference for session, task, fan-out, loop, automation, and spawn
- HTTP/UDS/OpenAPI reference
- config reference
- Network concepts and troubleshooting
- coordinator, review, detached execution, loop, and automation docs
- docs/_memory/glossary.md
- relevant RFC/ADR material that currently says bind always

The glossary's current Coordination Channel definition says every workspace run binds one and
instructs "Bind always, speak when useful" (docs/_memory/glossary.md:241-243). That sentence must be
deleted, not qualified with a compatibility note.

## Workspace Data Isolation

Network participation is workspace-scoped runtime data.

### Scope classification

| Datum | Scope |
|---|---|
| Administrative network.enabled and hard ceilings | global daemon |
| Network runtime identity/auth key reference | global installation (`AGH_HOME`); stable across daemon processes |
| Runtime endpoint lease/epoch | global installation + current authenticated endpoint lease |
| Workspace activation defaults/limits and channel defaults | workspace; cannot enroll an execution |
| Channel resource | workspace |
| Durable mailbox binding/cursor/home route/tombstone | workspace + channel + owning execution/profile + mailbox id + installation home runtime |
| Direct rooms, thread participants, mentions, subscriptions, work ownership | workspace + channel + stable mailbox id(s); never process peer id |
| Live membership lease and peer id | workspace + channel + mailbox + process/session lease/epoch; separate from resolved snapshot |
| Task profile request | task within global or workspace scope; participating values valid only for workspace tasks |
| Run resolved snapshot | task run + workspace; immutable and contains no process peer id |
| Session resolved snapshot | session + workspace; immutable and contains no live-lease peer id |
| Loop definition request | loop definition + workspace |
| Loop run resolved snapshot | loop run + workspace |
| Automation request | automation job scope |
| Automation run resolved snapshot | automation run + resolved workspace |
| Activation budget and usage | execution + workspace + channel + stable mailbox + activation/attempt; optional lease observation |
| Envelope and mailbox cursor | workspace + channel + container + stable sender/recipient mailbox |

### Propagation invariants

- workspace_id is mandatory before channel resolution.
- Channel identity is the composite workspace_id + channel_id, even when channel labels are globally
  unique in current storage.
- CLI, HTTP, UDS, core services, stores, caches, SSE, hooks, metrics, and Web routes carry the same
  workspace id.
- List/read APIs always filter by authenticated workspace before channel, membership, or usage.
- Cache keys include workspace id and concrete execution id.
- Direct rooms and threads cannot resolve across workspaces.
- Spawn, review, detached, loop, and automation inheritance validates workspace equality before
  channel authority.
- A global task may resolve only off. To participate, work must be materialized in an explicit
  workspace.
- Resolution source data never exposes another workspace's config or channel label.
- Actual token/call accounting is readable only under the same workspace and execution authority.

These invariants apply to history, live SSE, recovery scans, coordinator routing, and post-restart
reconstruction. No path may reconstruct participation from an unscoped channel string.

## Non-Off Public-Release Gates

P0-B may define and internally exercise the complete enum, but ordinary CLI, HTTP, UDS, Web, docs,
and bundled defaults expose only off/Local. A non-off value on an ordinary public write returns a
typed `network_participation_not_released` diagnostic until P0-C and that preset's gate pass. This
keeps one canonical schema without claiming an unfinished behavior is available.

### Active gate

Active MUST NOT be publicly selectable until all of the following are executable invariants:

1. P0-C stable mailbox addressing, mailbox-to-fenced-lease resolution, direct/mention/subscription/
   work identity migration, ordering, deduplication, accepted/published state, first-send routing,
   cursor, restart, and crash-recovery invariants pass.
2. Every admitted activation reserves positive logical-turn, provider-attempt, input-token,
   output-token, fanout, and wall-time budgets before ACP dispatch; zero never means unlimited. The
   supporting adapter reserves from the delegated sub-budget before each upstream provider call.
3. The agent profile advertises and passes `network_provider_attempt_budget_v1`. Attempt-start and
   terminal records expose every initial, retry, fallback, and post-tool provider call; terminal
   outcome and actual/unavailable usage settle against the durable reservation. Profiles without
   the capability reject active before execution creation.
4. At least one AGH-controlled, exactly pinned real adapter intercepts the upstream provider-call
   boundary, passes the deterministic conformance suite, and passes a credentialed real-provider
   smoke journey. Its verified manifest is available before owner persistence and its live handshake
   matches the snapshot. A mock or generic ACP proxy alone cannot satisfy this item.
5. direct, mention, explicit, and routed eligibility is deterministic; greet, whois, receipt, trace,
   presence, roster, acknowledgement, subscription, and task-status envelopes produce zero model
   calls under every policy.
6. Coalescing, root causation, loop prevention, maximum activation depth, and per-envelope recipient
   ceilings are enforced server-side.
7. Cancellation stops or definitively accounts for underlying provider work, not only visible
   streaming.
8. Public reads report resolved policy, used/remaining budgets, outcome, and unavailable reason
   truthfully across CLI, HTTP, UDS, SSE, hooks, generated clients, and Web.
9. Cross-workspace same-name channel, mailbox, usage, cache, SSE, and recovery negative tests pass.

Phase ownership is explicit. P0-B owns the resolved fields, trusted adapter registry/preflight,
at least one AGH-controlled pinned real adapter, the ACP extension and live capability handshake,
finite admission, delegated sub-budget reservation, terminal classification, conformance suite, and
credentialed real-provider smoke evidence. P0-C owns the durable provider-attempt state machine,
fixed bounded coalescing, restart reconciliation, cancellation propagation, and late-result handling
required by items 1, 3, 6, and 7. P1 may optimize/adapt windows, digest quality, routing, prompt size,
and price analytics; active release does not depend on a future P1 fix because its minimum real
adapter/attempt/coalescing/cancellation contract already ships in P0-B/P0-C.

### Mailbox gate

The target public contract includes mailbox, but it MUST NOT be exposed in CLI, HTTP, UDS, or UI
until executable invariants prove its semantics.

Required evidence:

1. A direct envelope to mailbox persists and produces zero PromptNetwork calls.
2. A mention, routed full delivery, receipt, trace, task-status message, and retry each produce zero
   PromptNetwork calls.
3. Mailbox order, deduplication, cursor, unread state, and restart recovery are deterministic.
4. A process can stop, release its live lease, receive an offline envelope at the durable mailbox id,
   restart, reattach, and read it exactly once without changing address or cursor ownership.
5. Live roster reads distinguish mailbox posture from active posture and separately report offline
   durable addressability.
6. Tools available to mailbox are explicitly enumerated and permission-gated.
7. Subscription changes cannot convert mailbox ingress into automatic activation.
8. Provider/tool/session errors cannot trigger a fallback prompt.
9. Metrics can assert received_count > 0 while activation_count = 0.
10. Stop and failed-start paths release live leases/transport handles/presence while preserving the
    durable mailbox binding, cursor, and subscription policy; explicit delete/retention removes them
    deterministically.
11. Every public read surface reports mailbox and its resolution source consistently.
12. A directed send to a registered offline mailbox succeeds without live presence, commits at the
    mailbox home, and is readable after reattach; an unknown/tombstoned mailbox returns a typed
    durable-address diagnostic.
13. Restart rotates peer/lease id without changing mailbox id, direct-room id, thread participant,
    resolved mention, durable subscription preference, work owner/target, or usage attribution.
14. A stale lease epoch/fencing token cannot consume or acknowledge mailbox data, change durable
    subscription policy, or complete an activation after a replacement lease attaches.
15. For remote mailbox homes, sender success follows durable home acknowledgement; transport-only
    publish or an unavailable home before deadline remains retryable outbox state and is never
    reported delivered.
16. If mailbox tombstone commits before a queued remote delivery reaches home, the home persists an
    authenticated terminal `recipient_tombstoned` disposition; if delivery committed first, retry
    returns the original accepted disposition. Delete/publish/retry/restart cannot loop indefinitely.
17. Home daemon restart preserves installation runtime id, re-registers a higher endpoint epoch, and
    lets a pre-restart sender outbox resolve/retry to exactly one durable disposition without changing
    mailbox home or route identity.
18. If the home commits an accepted or terminal-rejected disposition at endpoint epoch N, loses the
    response, and restarts at N+1, the current endpoint re-attests the exact immutable disposition and
    the sender settles once. The committed epoch remains N and the serving epoch is N+1.
19. A delayed presentation from superseded endpoint N, an unauthenticated or stale-runtime response,
    or a mismatched disposition/message/mailbox/route cannot settle sender outbox state.
20. A persistently unavailable home exhausts finite attempts and the immutable envelope deadline
    into inspectable sender-local `delivery_dead_lettered_unconfirmed`; no home disposition is
    fabricated and no recipient acceptance/rejection is claimed.
21. A copy arriving at the home after that deadline produces authenticated terminal
    `envelope_expired`, never mailbox unread state or activation. A later genuine home disposition
    can reconcile the sender record without republishing, and all relevant retention spans the full
    retry/replay plus settlement-grace window.

After P0-C, Active and Mailbox may clear their independent gates in either order. Before a gate
passes, that preset remains unavailable rather than degrading to another preset. The internal planes
remain separate throughout; no phase may collapse the model back to channel-present-means-active or
add an alias/compatibility field.

## Hard Delete Targets

The implementation TechSpec must treat these as delete targets, not deprecated paths:

| Target | Required hard cut |
|---|---|
| internal/task/types.go:409-416 | Delete Task.NetworkChannel; persist requested policy in Task Execution Profile |
| internal/task/types.go:468-487 | Delete Run.NetworkChannel and Run.CoordinationChannelID; add resolved snapshot |
| internal/task/types.go:703-709, 727-750 | Delete NetworkChannel/RequestedChannel inputs; accept canonical request |
| internal/store/globaldb/global_db_task_aux.go:741-910 | Delete unconditional resolve/create/derive path; create membership channel only from resolved participating strategy |
| internal/task/manager_integration_test.go:690-736 | Delete expectation that plain start creates a channel; replace with off invariant |
| internal/store/globaldb/global_db_task_claim_test.go:1739-1792 | Delete blank-reservation derived-channel invariant; test explicit run strategy |
| internal/session/manager_types.go:17-36 | Delete CreateOpts.Channel; carry requested/resolved participation |
| internal/session/manager_start.go:678-713 | Delete channel-presence environment inference; project from resolved spec |
| internal/session/manager_helpers.go:123-151 | Delete channel-string join condition; realize participating spec |
| internal/daemon/harness_context.go:310-330 | Delete ChannelBound = channel != empty |
| internal/daemon/harness_context.go:518-556 | Select Network prompt/augmenters from resolved mode and turn admission |
| internal/daemon/task_role_sessions.go:260-262 | Delete channel/default fallback from task-role session identity; derive identity from role/run/profile |
| internal/daemon/task_role_sessions.go:286-300 | Delete unconditional `Coordination channel` overlay and fictional `default`; render a compact line only from a participating resolved spec |
| internal/coordinator/coordinator.go:16-31, 97-142 | Delete DecisionMissingChannel and channel bootstrap gate |
| internal/coordinator/coordinator.go:46-61, 168-175 | Split task orchestration tools from conditional Network tools |
| internal/coordinator/coordinator.go:233-257 | Delete unconditional channel guidance |
| internal/daemon/loop_runtime_adapters.go:37-81, 118-161 | Delete unconditional action/judge Channel assignment |
| internal/daemon/loop_runtime_adapters.go:340-392 | Retain derivation only behind explicit loop-run channel strategy or replace with canonical resolver |
| internal/daemon/review_router.go:779-797 | Delete run/task channel fallback |
| internal/daemon/harness_detached_work.go:235-265 | Delete owner channel inheritance |
| internal/automation/model/types.go:115-121 | Delete JobTaskConfig.NetworkChannel |
| internal/automation/dispatch.go:674-678, 1182-1189 | Delete raw channel forwarding |
| internal/api/contract/contract.go:35-45, 105-121 | Delete session Channel request/response |
| internal/api/contract/tasks.go:680-789 | Delete task/run network_channel fields |
| internal/hooks/payloads.go:948-957 | Delete dual hook fields; emit resolved object |
| internal/task/manager_run_hooks.go:180-188 | Delete metadata/network-channel fallback |
| docs/_memory/glossary.md:221-243 | Replace mandatory coordination-channel doctrine |
| packages/site/content/runtime/core/autonomy/coordination-channels.mdx:6-28 | Delete `every run` / `bind always` doctrine; document Local first and conditional Network communication |
| packages/site/content/runtime/core/autonomy/index.mdx:6-8,77-112 | Remove channel-bound autonomy as a kernel prerequisite; distinguish local task authority from optional communication |
| packages/site/content/runtime/core/autonomy/coordinator.mdx:11-23,76-92 | Delete channel bootstrap condition and mandatory `--channel` execution example |
| docs/articles/2026-05/06-autonomy-kernel.mdx:140-169 | Replace the canonical always-channel sequence while preserving leases, claim fencing, and safe-spawn semantics |
| web/src/systems/tasks/components/task-editor-modal.tsx:230-281 | Replace raw channel field with preset control and conditional selector |
| web/src/systems/automation/components/job-form/task-run-step.tsx:55-80 | Replace raw automation channel field |
| web/src/systems/tasks/components/tasks-fan-out-runs-card.tsx:86-99 | Replace raw fan-out channel field |
| packages/site/content/runtime/cli-reference task/session pages | Regenerate without legacy flags |
| generated OpenAPI and web client types | Regenerate; no dual fields |
| internal/network/envelope.go:196-218 and public envelope/send contracts | P0-C deletes v0 peer-address `from`/`to`/`mentions`; emit only versioned mailbox-address fields plus separately labeled optional lease evidence |
| internal/network/router.go:354-379 | P0-C deletes live presence as directed-recipient existence validation; validate durable mailbox binding/authority |
| internal/network/validate.go:212-241 | P0-C deletes `DirectRoomIdentity` v0 over peer ids; derive v1 identity over sorted stable mailbox ids |
| internal/store/globaldb/global_db.go:163-176,349-430 | P0-C rebuilds peer-keyed subscriptions, participants, directs, and work relations around mailbox ids; delete peer-key indexes/columns |
| internal/api/contract/contract.go:812-910,1038-1070,1124-1194 | P0-C deletes peer-address send/read/subscription/direct/work public fields; regenerate mailbox-address contracts across HTTP/UDS/CLI/Web/native tools |

SQLite schema work uses an append-only numbered migration. Old columns, indexes, constraints,
queries, scanners, fixtures, and seed data are removed in the same change. There is no read-time
fallback to metadata_json and no repair of old alpha state.

For the P0-C v1 address cut, old peer-keyed conversation/direct/participant/subscription/work/
delivery/usage rows have no trustworthy mailbox mapping. The alpha migration destructively resets
that v0 collaboration state (while retaining only channel resource definitions the TechSpec proves
identity-independent) and rebuilds the tables with mailbox keys. It MUST NOT derive mailbox ids from
old peer strings, dual-write v0/v1, or keep a compatibility reader.

## Architecture Sequencing Constraints

This is not an implementation task list. The following ordering constraints prevent partial states:

1. Define the canonical request, resolved snapshot, resolver, provenance, diagnostics, and
   persistence schema before any caller is migrated.
2. Migrate storage reservation and claim projection before enabling off as the built-in default;
   otherwise callers may report off while channels are still created.
3. Migrate session lifecycle, environment, prompt, tool, and delivery admission as one behavioral
   boundary; no intermediate build may let mailbox call PromptNetwork.
4. Decouple coordinator bootstrap from channel existence before off task runs reach coordinator
   recovery.
5. Migrate review, spawn, detached, loop, and automation composition before removing legacy
   request fields.
6. Co-ship API, OpenAPI, generated client, CLI, UDS, Web, hooks, native tools, bundles, extensions,
   official skill, docs, glossary, and tests in the same hard-cut release.
7. P0-B makes the current transport lazy only after live-lease/listener reference counting and
   cleanup are proven; no non-off mode is public in that phase.
8. P0-C implements the durable accepted/published state machine and resolves local/remote transport
   topology; the existing source defects are sufficient to authorize the TechSpec now.
9. In the same P0-C hard cut, replace v0 durable peer identity with stable mailbox addressing across
   envelope, direct, mention, participant, subscription, work, delivery, audit, and usage state, then
   add fenced mailbox-to-live-lease resolution. Mixed v0/v1 address state is forbidden.
10. Expose active and mailbox independently only after P0-C and their respective gates pass.

No phase may introduce aliases, dual writes, legacy reads, implicit conversion, or a temporary
default-active behavior.

## Alternatives Rejected

### A. Make blank channel truly local and keep channel presence as opt-in

Rejected as the target architecture. It fixes auto-created channels but leaves channel presence
coupled to membership, prompts, tools, heartbeat, and PromptNetwork. It cannot express a durable
zero-activation mailbox.

### B. Add a --no-network boolean

Rejected. A negative boolean preserves active as the conceptual default, composes poorly with
profiles/config, and cannot represent mailbox. Public presets are explicit and extensible without
boolean combinations.

### C. Keep auto-created channels but suppress Network prompts

Rejected. It still creates storage, roster, transport, heartbeat, routing, hook, SSE, and cleanup
side effects and leaves future code free to infer participation from channel presence.

### D. Use network.enabled as the per-run choice

Rejected. It is daemon-wide availability, not user execution intent. Toggling it affects unrelated
work and cannot safely express mixed local and participating executions.

### E. Let child/review/detached work inherit the parent's channel automatically

Rejected. Authority inheritance is not intent. It recreates hidden participation and can cross
role, lifetime, and workspace boundaries.

### F. Delay the participation/activation plane split until mailbox ships

Rejected. off|active implemented over channel presence would make mailbox a second cross-surface
rewrite. The internal planes are required in the first hard cut even if mailbox public exposure is
gated.

### G. Replace embedded NATS and add a transactional outbox in this change

Rejected for the P0-B hard cut. These are transport and crash-consistency decisions that require a
separate P0-C TechSpec. The source audit already supplies enough evidence to start it; it is a
prerequisite for public non-off modes, not for shipping truthful Local/off behavior.

### H. Encode MoA or swarm rounds in Network envelopes

Rejected. MoA and swarm are execution topologies owned by task/loop/session orchestration. Encoding
them in the wire protocol would contradict the stated boundary that Network is not a workflow
engine.

### I. Require a Network channel for coordinator bootstrap

Rejected. It makes task orchestration depend on communication transport and prevents local
coordinators from managing ordinary runs.

## Embedded Council Record

### Opening positions

The Architect selected the explicit-spec option. A nullable channel currently carries transport
participation, inbox availability, and model-activation permission. The recommendation was to
resolve one NetworkParticipationSpec at the composition boundary, snapshot it on the execution,
and make the coordinator transport-independent.

The Pragmatic Engineer accepted the plane split but emphasized a bounded first hard cut: change
participation, activation, and lazy startup without replacing local NATS or introducing new
durability machinery.

The Devil's Advocate steel-manned the explicit spec but challenged mailbox as an initially public
mode. Without executable definitions for ingress, roster visibility, subscriptions, tools, and
zero activation, a third preset could become a permanent ambiguity.

### Tension 1: whether mailbox is public immediately

The Architect partially conceded. An underspecified public enum is worse than omission. However,
the underlying planes must still be separate in the first release because durable receipt without
automatic inference is a credible requirement and the current delivery path directly couples
accepted envelopes to PromptNetwork (internal/network/delivery.go:285-348, 660-688).

Synthesis:

- mailbox is part of the target contract.
- Public exposure is gated by the zero-activation and durable-offline invariants in this document.
- P0-B exposes only off. After P0-C, mailbox and active clear independent release gates; neither is
  temporarily assumed safe because the other is unavailable.
- Internal participation and activation remain orthogonal.
- No temporary field, alias, or channel-presence implementation is permitted.

Evidence that would reverse this position: proof that AGH has no credible durable asynchronous
delivery use case without automatic inference. The current token-cost objective and existing inbox,
digest, mute, and durable-history features point in the opposite direction
(skills/agh/references/network.md:78-107).

### Tension 2: transport replacement and transactional outbox

The Pragmatic Engineer argued that replacing embedded NATS or adding an outbox changes durability,
recovery, and failure semantics independently and should not expand the P0-B local/off hard cut.

The Architect conceded on phase ownership, not on necessity. P0-B makes the current transport lazy
and hides it behind the participation boundary. P0-C owns the durable accepted/published state
machine and the local/remote transport decision before either non-off mode becomes public.

Evidence already authorizing P0-C:

- Persist-before-publish plus duplicate short-circuit can acknowledge an envelope that was never
  published (`internal/network/manager.go:690-710`).
- First local thread send can persist the sender before routing and then select no receiver
  (`internal/store/globaldb/global_db_network_conversations.go:71-168`;
  `internal/network/router.go:1241-1265`).
- Router memory and SQLite independently own work lifecycle
  (`internal/network/router.go:202-216,1066-1102`;
  `internal/store/globaldb/global_db_network_work_mutation.go:14-177`).

Crash injection is required acceptance evidence. Profiling selects whether embedded NATS remains on
the local path; it no longer gates writing the TechSpec.

### Tension 3: whether aggregate ACP usage is enough for active

The final architecture review rejected the initial assumption that current ACP usage could support
a provider-attempt cap. AGH receives aggregate turn usage but cannot see or stop provider retries,
fallbacks, or post-tool calls inside an opaque ACP process
(`internal/acp/types.go:505-518`; `internal/acp/client.go:935-991`). A mock event extension would prove
only AGH's ledger, not that any real provider obeys it.

Synthesis:

- Active requires a provider-call-boundary adapter contract; aggregate turn usage alone is
  insufficient.
- P0-B owns a trusted manifest/artifact registry, pre-persistence verification, a matching live
  handshake, and at least one AGH-controlled pinned real adapter.
- Pre-persistence verification is process-free; live startup occurs only after commit under explicit
  lifecycle ownership and before any Network lease.
- A generic ACP proxy and an unpinned `@latest` package cannot be the proof artifact.
- Deterministic conformance covers forced retry/fallback/post-tool/over-budget paths; a credentialed
  real-provider smoke proves the exact shipped artifact is usable.
- Direct third-party ACP providers remain valid for off/mailbox and expose a typed Active unavailable
  reason until they meet the same extension contract.
- P0-C makes provider-attempt state and reconciliation durable before public Active release.

This is intentionally stricter than a logical-turn-only cap. The product goal is predictable token
spend, and a cap that cannot observe or constrain hidden retries would be a false control.

### Position evolution

| Phase | Position |
|---|---|
| Opening | Choose explicit off/mailbox/active spec; separate membership and activation |
| After mailbox challenge | Keep mailbox in target; gate its public exposure on executable zero-activation and offline-durability invariants |
| After transport challenge | Make the current transport lazy in P0-B; move durable outbox/topology work to the already-authorized P0-C TechSpec |
| After provider-attempt challenge | Require trusted preflight and one real AGH-controlled adapter; opaque ACP profiles remain off/mailbox-only |
| Final | Ship Local/off in P0-B; expose mailbox after its P0-C zero-activation gate and expose active only after P0-C plus the finite-activation gate, with selection limited to verified adapter-backed profiles |

### Council synthesis

The stable architectural boundary is:

- Execution orchestration decides whether communication is requested.
- Network owns channels, membership, durable mailbox delivery, and routing.
- Activation admission owns whether a stored envelope may spend a model call.
- A verified provider adapter owns upstream-attempt enforcement; the runtime ledger owns admission,
  reconciliation, and workspace-scoped reporting.
- Task, review, loop, and automation services own their authoritative state.
- Swarm/MoA topology remains outside the Network protocol.

## AGH Impact Audit

- Native tools: High impact. Conditional availability must replace unconditional projection for
  agh__network_* tools. Descriptor schemas, toolset membership, digests, risk flags, capability
  gates, diagnostics, tests, and CLI/API fallbacks must reflect off/mailbox/active. Checked:
  internal/tools/builtin/network.go:12-269, coordinator allowlists in
  internal/coordinator/coordinator.go:46-73, and config tool paths in
  internal/config/tool_surface.go:239-249.
- Extensibility and hooks: High impact. Extension manifests, pre-resolution hook authority,
  TaskRunContext payloads/matchers, bundle channel defaults, bridge distinction, MCP sidecar scopes,
  registries, SDKs, and config lifecycle must carry the canonical request/resolved object and may
  not widen implicitly. Checked: internal/hooks/payloads.go:948-957,
  internal/task/manager_run_hooks.go:131-188, internal/api/contract/bundles.go:5-123,
  internal/automation/model/types.go:101-121.
- Workspace data isolation: High impact. Resolved participation, channels, memberships, mailbox
  cursors, activation budgets, token usage, hooks, SSE, caches, and Web links are workspace-scoped.
  workspace_id must propagate through CLI/HTTP/UDS/core/store/session/Network/Web/event paths and
  must be part of lookup/cache keys. Global tasks may resolve only off. Existing channel workspace
  validation is visible in internal/store/globaldb/global_db_task_aux.go:814-877, but the target
  removes unscoped reconstruction from channel strings.
- Official AGH skill: Required. Update skills/agh/references/network.md and
  skills/agh/references/tasks-and-orchestration.md for presets, mailbox zero activation, active
  budgets, conditional tool visibility, coordinator independence, and removal of mandatory
  coordination channels.

## Web and Docs Impact

Web impact is mandatory:

- Replace raw channel inputs in task, fan-out, and automation forms with explicit preset controls.
- Add the same control to session and loop start surfaces.
- Add resolved preset/source/budget projections to task, run, session, loop, automation, and
  Network detail views.
- Keep Network channel/thread/direct management views, but distinguish mailbox and active peers.
- Regenerate web API types and update mocks/stories from the hard-cut schema.
- Verify every changed surface visually through the AGH screenshot workflow.

Docs impact is mandatory:

- Regenerate CLI and API references.
- Rewrite configuration and Network participation concepts.
- Update coordinator, task, review, spawn, detached, loop, automation, extension, and bundle docs.
- Replace the glossary's mandatory coordination-channel definition.
- Update the official AGH skill.
- Record the hard-cut delete list in the implementation TechSpec.

## Final Architecture Invariants

The design is complete only when all of the following are true:

1. A plain session, task start, loop run, review, spawn, detached submission, or automation fire
   resolves off and creates no Network channel or membership.
2. No runtime code infers participation or activation from a non-empty channel string.
3. No runtime code derives a channel unless the resolved request selected a participating preset
   and an explicit derivation strategy.
4. mailbox can persist and read messages while proving zero envelope-triggered PromptNetwork calls.
5. active reserves bounded call/token/fanout budget before model activation.
6. Control and task-status envelopes never invoke a model.
7. Coordinator bootstrap and local task orchestration work with off runs.
8. Child, review, detached, loop, and automation paths do not inherit participation implicitly.
9. Off sessions receive no Network prompt section, environment identity, heartbeat, subscriptions,
   execution-scoped Network tools, or Network descriptor tokens.
10. All public and extension surfaces use one canonical schema with source transparency.
11. Workspace identity is present in every participating lookup, event, cache key, and usage record.
12. Legacy fields, aliases, fallback metadata, tests, generated types, docs, and glossary language
    are deleted.
13. MoA and swarm execution can run entirely with Network off.
14. P0-B makes the current transport lazy; P0-C proves durable accepted/published recovery and
    selects local/remote transport topology before any non-off preset is publicly selectable.
15. A durable mailbox binding, cursor, and subscription policy survive process-lease exit; live
    transport handles, presence, and peer leases do not.
16. Network v1 uses stable mailbox ids for durable envelope/direct/mention/participant/subscription/
    work/delivery/audit/usage identity; process peer ids are never durable addresses or keys.
17. An offline registered mailbox accepts directed durable ingress without presence, while an
    unknown/tombstoned address fails deterministically.
18. Restart may rotate peer/lease id without changing direct or mailbox history, and stale fencing
    tokens cannot consume, acknowledge, mutate, or settle for the replacement lease.
19. Remote mailbox delivery settles only from an authenticated idempotent recipient-home accepted or
    terminal-rejected disposition; transport publish alone is not delivery.
20. Home runtime identity persists per installation across daemon restart while endpoint leases
    advance monotonically. The current endpoint can re-attest an immutable disposition committed by
    its predecessor, while stale/forged presentations fail closed.
21. Home-store ordering makes delivery-before-tombstone remain accepted and tombstone-before-delivery
    terminally rejected across retry/restart, so outbox cannot retry the delete race forever.
22. Remote envelopes carry one authenticated immutable deadline. An unreachable home ends in
    sender-local `delivery_dead_lettered_unconfirmed`, while an envelope reaching home after the
    deadline ends in home-authenticated `envelope_expired`; neither path can cause late acceptance or
    fabricate the other's authority.
23. Active is selectable only for a concrete profile backed by a trusted, pinned, conformant real
    adapter; pre-persistence artifact verification is process-free and live startup is post-commit
    under complete cleanup ownership.
