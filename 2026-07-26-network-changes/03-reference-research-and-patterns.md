# Reference Research: Hermes MoA, Agent Swarms, and Agent Networks

## Document status

- **Snapshot date:** 2026-07-10
- **Purpose:** extract implementation patterns that can reduce AGH Network activation, prompt, and communication cost
- **Primary requested references:** `.resources/hermes`, `second-brain/research/agent-swarm`, and `second-brain/research/agent-networks`
- **AGH target:** the opt-in participation model and protocol changes described in the sibling audit and architecture documents
- **Method:** local source inspection, local research-note inspection, and source-pinned inspection of the current Hermes Agent implementation

This document distinguishes current source evidence, synthesized notes, and research claims. Research results are hypotheses for AGH experiments, not production guarantees.

## Evidence boundary

### Current Hermes source is authoritative

The requested competitor snapshot is present at `.resources/hermes`, with remote identity `https://github.com/nousresearch/hermes-agent` and inspected commit `b8880f124537acc5a6215718dd154eadc5af1515`.

The Hermes material under `second-brain/research/harness/hermes` is outdated and was not used as technical evidence for the current Mixture-of-Agents implementation. The dedicated current-source audit is `06-hermes-moa-current-source-audit.md`.

### Source quality legend

| Label | Meaning | Appropriate use |
|---|---|---|
| Current primary | Source code or docs at a pinned upstream commit | Architectural and implementation evidence |
| Local synthesis | Second-brain concept article synthesized from cited sources | Discovery, terminology, and cross-source comparison |
| Research paper | Paper/preprint copied into the local corpus | Candidate policies and benchmark design |
| Recommendation | Inference for AGH based on the evidence | Requires implementation validation |

## Executive synthesis

The references do not support making a persistent group conversation the default orchestration primitive. They instead support five rules:

1. **Single-agent/local execution is the baseline.** Add multi-agent behavior only after the task demonstrates a specialization, context, ownership, or quality need.
2. **Fan-out/fan-in is the preferred multi-agent candidate shape.** Independent workers see the request, produce policy-bounded results in parallel, and one already-required acting agent synthesizes them. Whether it is cheaper or better for a task class must be measured end to end.
3. **Workers should be stateless and tool-light by default.** They do not need the entire runtime prompt, Network tutorial, tool catalog, or shared conversation transcript.
4. **Communication is an activation decision, not merely transport.** The runtime should route, filter, aggregate, and stop deterministically before paying for another model call.
5. **Share typed artifacts and summaries, not full conversational history.** Local working memory plus selective shared facts gives the runtime a smaller and more controllable context surface.

These rules align with AGH's pivot to an agent OS: Network becomes one optional execution facility rather than the universal substrate for sessions, tasks, and loops.

## Hermes Mixture-of-Agents

### Current-source scope

The findings below are a condensed extraction from `.resources/hermes`. The detailed end-to-end trace, management-surface defect, call-cost model, test inventory, and adopt/adapt/reject matrix are in `06-hermes-moa-current-source-audit.md`.

### Current Hermes surface is explicitly opt-in

The current feature is exposed as a virtual provider. A model-picker selection persists for the session, while `/moa <prompt>` is intended to be one-shot. The gateway restores the prior model from a `finally`-owned path, but the direct CLI and TUI implementations can leak the temporary model after an exception. A normal prompt does not implicitly enter an always-on multi-agent room. Evidence: `.resources/hermes/website/docs/user-guide/features/mixture-of-agents.md:9-60`, `.resources/hermes/apps/desktop/src/app/shell/model-menu-panel.tsx:182-196,327-346`, `.resources/hermes/gateway/run.py:9951-9984,10329-10348`, `.resources/hermes/cli.py:12338-12369`, and `.resources/hermes/tui_gateway/server.py:9031-9081`.

This is the closest product analogue for AGH:

- Network participation must be selected on the run/session/task boundary.
- A multi-agent preset can be a deliberate execution mode.
- Creating a normal session must not silently select it.

### References are stateless; one actor remains responsible

Hermes builds reference calls as raw model requests rather than full `AIAgent` instances. Reference calls do not receive the normal tool loop. The aggregator remains the acting agent and owns tool use and the final response. Evidence: `.resources/hermes/agent/moa_loop.py:93-118,220-279,960-1022`.

This avoids several costs present in AGH's current Network path:

- no session lifecycle per reference model;
- no join/greet/presence traffic;
- no Network tool catalog for references;
- no repeated channel/thread guidance;
- no worker-to-worker conversation;
- no expectation that a worker must send a second protocol message for its output to become useful.

For AGH, a Network-backed swarm should therefore use **ephemeral worker invocations with structured returns**. Durable sessions should be an explicit requirement, not the implementation default.

### Fan-out is parallel, but no aggregate MoA/root ceiling exists

Hermes caps concurrent reference calls at eight, runs references concurrently, and isolates
individual failures. It exposes no `max_references` or aggregate MoA/root-call ceiling; every valid
configured slot is finite by list length, calls beyond eight queue, and outer iteration plus
auxiliary retry/timeout limits bound components indirectly. Fan-in waits for all submitted
references. Evidence: `.resources/hermes/agent/moa_loop.py:21-27,220-249,336-385`,
`.resources/hermes/agent/agent_init.py:271,385-388`, and
`.resources/hermes/agent/conversation_loop.py:643-670`.

The resulting model-call shape is preferable to conversational broadcast:

| Shape | Approximate calls for `R` references and `I` acting iterations | Communication topology |
|---|---:|---|
| References once per user turn | `R + I` | user input -> references; results -> actor |
| References on every acting iteration | `I * (R + 1)` | repeated fan-out/fan-in |
| Peer group conversation | data-dependent and potentially superlinear | peers repeatedly activate peers |

AGH should make `fanout=user_turn` the default swarm policy. Per-iteration refresh should require an explicit budget and a demonstrated stale-reference problem.

### Advisor prompts are deliberately smaller

Hermes creates an advisor view that removes the normal system prompt, flattens tool actions, and trims each tool result to 4,000 characters before sending it to references. It still traverses the full conversation, has no total advisor-input budget, and drops non-string multimodal/structured message content during projection. It also replicates that projected history across each executed valid reference provider, creating a data-egress policy surface. Evidence: `.resources/hermes/agent/moa_loop.py:83-91,388-522`.

This is directly applicable to AGH. A worker context should contain only:

```text
worker role + task slice + necessary input facts + bounded evidence pointers
+ return schema + token/deadline budget
```

It should not inherit the actor's full:

- tool registry;
- AGH skill reference set;
- Network protocol manual;
- unrelated task/channel history;
- presence roster;
- environment and lifecycle narration.

Removing the actor system prompt is not itself a safe default. An AGH advisory projection must retain a compact, provider-independent safety and authority envelope, enforce tenant/workspace data policy, allowlist eligible providers, redact or omit disallowed content classes, and preserve multimodal parts only through an explicit typed projection. Tool results and advisor text remain untrusted evidence, not instructions.

### Stable prefixes improve cacheability

Hermes applies provider-aware cache control to advisor/legacy-synthesis calls and resolves the
virtual provider to the configured acting-aggregator slot for the normal agent cache policy. It
appends changing guidance at the prompt tail so the preceding conversation remains cacheable.
Evidence: `.resources/hermes/agent/moa_loop.py:176-217,261-272,661-680`,
`.resources/hermes/agent/agent_runtime_helpers.py:1524-1562`,
`.resources/hermes/agent/agent_init.py:600-608`, and
`.resources/hermes/agent/conversation_loop.py:888-899`.

AGH should use the same ordering for the contexts that remain:

1. stable agent/provider instructions;
2. stable worker contract;
3. cacheable task background;
4. changing artifact summaries;
5. current message and exact requested output.

Dynamic channel rosters, timestamps, transport metadata, and protocol examples at the front of a prompt reduce provider prefix-cache reuse.

### Reuse exists, but the default reruns per tool iteration

Hermes supports `user_turn`, which hashes the prefix through the last real user message and reuses advisor output across later tool iterations. The configured default is instead `per_iteration`: a new tool result changes the advisory view and reruns every reference. Evidence: `.resources/hermes/hermes_cli/moa_config.py:70-74,93-105,125-148` and `.resources/hermes/agent/moa_loop.py:844-907`.

The analogous AGH key should include:

```text
workspace_id
+ root_activation_id
+ task/input digest
+ worker role/model/config digest
+ artifact cursor
```

Reuse must be scoped to the root activation and workspace. A global or channel-only cache would violate isolation and can return stale cross-run advice.

### Output caps and usage attribution are first-class

Hermes exposes `reference_max_tokens`, documents a latency/output-token correlation of approximately `0.88`, and reports a sample latency reduction near 44% at a 600-token reference cap. The default is uncapped, and the inspected tree does not contain the raw benchmark artifact behind those source comments. Evidence: `.resources/hermes/hermes_cli/moa_config.py:52-67,125-147`, `.resources/hermes/agent/moa_loop.py:815-825`, and `.resources/hermes/website/docs/user-guide/features/mixture-of-agents.md:103-133`.

It records usage and cost against each configured reference slot and the configured aggregator slot. That is explicit in the no-fallback path, but auxiliary fallback can execute a different provider/model while accounting retains the configured identity, so it is not reliable actual-attempt accounting. Evidence: `.resources/hermes/agent/moa_loop.py:30-81,280-323,804-814,908-925`, `.resources/hermes/agent/auxiliary_client.py:6905-6952`, and `.resources/hermes/agent/conversation_loop.py:2049-2178`.

AGH needs the same primitive at the root activation level:

- input, cached-input, output, and reasoning tokens by invocation;
- actual provider cost where available;
- latency and queue delay;
- useful/selected/discarded worker result;
- termination reason;
- aggregate budget consumed and remaining.

The 600-token result is a Hermes sample, not a universal AGH default. AGH should benchmark caps by role and task class.

### Tracing creates a privacy obligation

Hermes can retain raw MoA prompts and responses in opt-in side-channel traces that do not enter conversation history. Evidence: `.resources/hermes/hermes_cli/config.py:2290-2300` and `.resources/hermes/agent/moa_trace.py:1-21,67-166`.

AGH should not copy raw tracing by default. Store numeric usage, digests, decisions, and redacted previews by default; require a scoped diagnostic flag and retention policy for full prompt bodies.

### Hermes defaults that AGH should not copy

Current Hermes code also exposes risks:

- A configured preset's reference fan-out defaults to enabled, although MoA execution still requires selecting the virtual provider or one-shot command.
- Reference count can be effectively uncapped beyond the parallelism batch size.
- Per-iteration fan-out is available.
- The fan-in waits for the slowest reference in a batch.
- Reference models can receive a broad flattened history.
- Aggregation concatenates results without a deterministic relevance rank or deduplication stage.
- There is no obvious root-level total token/cost budget in the cited configuration.
- `reference_max_tokens` and `fanout` are absent from the management API and UI schemas, and the PUT handler reconstructs presets without them; a UI save can therefore restore uncapped/per-iteration defaults.
- A legacy `aggregate_moa_context` path performs separate reference, synthesis, and main-agent calls alongside the virtual-provider path.

Evidence: `.resources/hermes/hermes_cli/moa_config.py:13-21,93-148`; `.resources/hermes/hermes_cli/web_server.py:968-996,5505-5541`; `.resources/hermes/web/src/lib/api.ts:2282-2304`; and `.resources/hermes/agent/moa_loop.py:571-658`.

AGH should copy the topology and context discipline, not those defaults.

## Second-Brain Agent-Network Findings

### Start with one agent

The synthesized orchestration survey states the common rule explicitly: start with one agent, then add multi-agent orchestration only when empirical limits appear. It distinguishes direct model calls, tool-using single agents, and multi-agent execution as increasing levels of complexity. Evidence: `/home/pedronauck/Projects/second-brain/research/agent-networks/wiki/concepts/Multi-Agent Orchestration Patterns.md:31-50`.

For AGH, that means:

- sessions default to local;
- tasks default to local scheduler/coordinator state;
- loops default to local node execution;
- Network is selected only for cross-session, cross-agent, durable mailbox, or distributed collaboration needs.

The current "bind always, speak when useful" posture reverses this rule by paying binding and context costs before need exists.

### Prefer bounded concurrent fan-out/fan-in

The survey describes the concurrent pattern as independent parallel workers merged by one synthesizer, with latency bounded by the slowest worker instead of the sum. Evidence: `/home/pedronauck/Projects/second-brain/research/agent-networks/wiki/concepts/Multi-Agent Orchestration Patterns.md:235-251`.

It also reports a cited benchmark where multi-domain router/subagent execution used roughly 9K tokens versus roughly 15K for sequential skills. Evidence: the same file at `:305-320`. These are third-party benchmark figures and should be treated as directional.

The topology maps naturally to AGH task roles:

```text
local coordinator
  -> bounded independent role invocations
  -> typed role results/artifacts
  -> one synthesis/decision step
  -> completion
```

Network transport is optional in this topology. Local workers can use direct runtime dispatch; remote/durable participants can use Network without changing the orchestration contract.

### Group chat is a specialized, high-cost mode

The survey characterizes group chat as a manager-controlled shared conversation and recommends three or fewer agents because loops escalate quickly. Evidence: the same file at `:273-285`.

Group chat is appropriate for bounded debate, maker-checker review, or consensus where peers must react to each other. It is not the appropriate substrate for:

- a one-shot task action;
- a loop node execution;
- a judge invocation;
- an implementation/review handoff with typed outputs;
- a local child session;
- status, receipt, presence, or trace control events.

If AGH offers group chat, it needs explicit `max_agents`, `max_rounds`, `max_model_calls`, `max_output_tokens`, and deterministic completion conditions.

### Handoff is not delegation

The handoff research separates three operations:

- delegation: parent retains control and receives a result;
- handoff: another agent becomes responsible for the interaction/state;
- tool call: a capability returns a bounded result without agency transfer.

It recommends transferring typed facts, a recent bounded window, a summary, and pointers rather than the whole history. Evidence: `/home/pedronauck/Projects/second-brain/research/agent-networks/wiki/concepts/Agent Handoff and Context Transfer.md:52-101`, `:201-216`, and `:235-240`.

AGH currently uses a conversation-shaped Network path for operations that are actually delegation or tool calls. The protocol should expose separate typed primitives so the runtime can choose the least-capability, lower-coordination-overhead candidate and then verify cost empirically.

### A2A is an interoperability boundary, not an internal default

The local A2A synthesis emphasizes task status and artifact events, cached agent cards/capabilities, and a distinction between external interoperability and internal subagent execution. Evidence: `/home/pedronauck/Projects/second-brain/research/agent-networks/wiki/concepts/The A2A Protocol.md:74-131`, `:219-255`, `:321-382`, `:403-437`, and `:514-525`.

The MCP-A2A composition notes similarly separate local tool/resource access from agent-to-agent delegation. Evidence: `/home/pedronauck/Projects/second-brain/research/agent-networks/wiki/concepts/The MCP-A2A Composition Pattern.md:27-45`, `:127-135`, and `:235-248`.

Recommended AGH boundary:

| Need | Cheapest appropriate primitive |
|---|---|
| Same-runtime function/tool | Direct Go/runtime call |
| Local stateless specialist | Ephemeral worker invocation |
| Local durable child | Session/task API with typed result |
| Cross-runtime agent task | Network/A2A-style task + artifacts |
| Human/agent shared discussion | Explicit Network thread |
| External event observation | Hook/SSE/event stream without model activation |

Do not route same-process control flow through an A2A-shaped conversational protocol merely because the protocol exists.

### Share artifacts selectively

The memory synthesis recommends local working memory plus selectively shared artifacts. Evidence: `/home/pedronauck/Projects/second-brain/research/agent-networks/wiki/concepts/Multi-Agent Orchestration Patterns.md:322-342` and `/home/pedronauck/Projects/second-brain/research/agent-networks/raw/articles/multi-agent-memory-computer-architecture.md`.

The consistency notes distinguish local, shared, and synchronized state and discuss versioned writes/conflict handling. Evidence: `/home/pedronauck/Projects/second-brain/research/agent-networks/wiki/concepts/Multi-Agent Memory Consistency.md:54-60`, `:99-111`, `:148-173`, and `:221-241`.

AGH should make the artifact the durable unit of collaboration:

```json
{
  "artifact_id": "...",
  "workspace_id": "...",
  "root_activation_id": "...",
  "producer": "...",
  "kind": "finding_set",
  "schema_version": 1,
  "content_ref": "...",
  "summary": "...",
  "input_digest": "...",
  "created_at": "..."
}
```

Prompts receive the summary and selected content, not an ever-growing channel transcript. Consumers acknowledge an artifact cursor/version rather than becoming sticky conversation participants.

### Observe causality and cost at the root

The tracing notes recommend cross-agent trace/span propagation and measuring token usage, latency, tool calls, and handoff paths. Evidence: `/home/pedronauck/Projects/second-brain/research/agent-networks/wiki/concepts/Agent Observability and Distributed Tracing.md:150-185`, `:243-256`, and `:278-360`.

AGH needs a root activation identifier that survives:

- session spawn;
- task/run/role creation;
- loop node and judge execution;
- Network delivery;
- worker fan-out;
- artifact publication;
- synthesis and final response.

Without that root, Network optimization cannot answer whether a delivered message was useful or merely generated another expensive turn.

## Second-Brain Agent-Swarm Findings

The `agent-swarm` topic is primarily a raw research corpus rather than a fully synthesized wiki. Evidence inventory: `/home/pedronauck/Projects/second-brain/research/agent-swarm/wiki/index/Dashboard.md:10-17`. Claims below therefore name the underlying local article or paper.

### Measure communication as system overhead

The May 2026 swarm survey catalogs overcommunication, repeated messages, unbounded interaction, and the absence of learned stopping policies as central scaling problems. It proposes system metrics including:

- parallel efficiency;
- useful-agent utilization;
- protocol overhead;
- message redundancy;
- error amplification;
- straggler impact.

Evidence: `/home/pedronauck/Projects/second-brain/research/agent-swarm/raw/articles/preprint-may-2026.md:368-409`, `:463-499`, `:1197-1232`, `:1292-1325`, `:1390-1420`, and `:1581-1584`.

These metrics are stronger than AGH's current message and delivery counters. The runtime should be able to report:

```text
useful_worker_results / invoked_workers
selected_artifact_bytes / delivered_context_bytes
model_activations / root_run
duplicate_or_superseded_results / results
critical_path_latency / summed_invocation_latency
control_envelopes_that_activated_models
straggler_wait_ms
```

### Deterministic stopping comes before model-based stopping

HANDRAISER studies listener-controlled interruption and reports communication-cost reductions while preserving task quality in its evaluated settings. Evidence: `/home/pedronauck/Projects/second-brain/research/agent-swarm/raw/articles/arxiv-2604-06452v1-cs-cl-7-apr-2026.md:29-47`, `:118-180`, and `:466-490`.

The caution is more valuable for AGH than the headline result:

- prompt-based interruption was overconfident;
- it interrupted too early;
- the extra back-and-forth increased total rounds and cost;
- in another setting, per-round tokens dropped while rounds increased substantially.

Evidence: the same paper at `:486-490` and `:586-598`.

Therefore AGH should first implement deterministic admission and stopping:

- enough successful workers;
- quorum reached;
- requested roles covered;
- deadline reached;
- marginal result duplicates existing evidence;
- budget exhausted;
- actor has already accepted an artifact;
- max depth/rounds reached.

Only after collecting traces should AGH experiment with learned listener interruption or adaptive topology.

### Sparse topologies are promising but must be earned by data

The local corpus cites sparse topology and agent-pruning work. HANDRAISER's related-work section points to AgentPrune and AgentDropout as methods that remove redundant communication. Evidence: the same paper at `:596-602`.

Local research summaries report:

- AgentDropout: approximately 21.6% prompt-token and 18.4% completion-token reduction in its evaluation.
- AgentPrune: approximately 28.1% to 72.8% token reduction across reported settings.

These figures are research-specific, not transferable AGH promises. They suggest a later experiment:

1. instrument actual AGH root activation graphs;
2. label useful and redundant deliveries;
3. implement deterministic edge/participant pruning;
4. compare quality, cost, and latency against fixed fan-out;
5. only then consider learned routing.

### Hierarchy and self-organization have control costs

The corpus includes work on dynamically evolving teams and society/hivemind topologies. Evidence:

- `/home/pedronauck/Projects/second-brain/research/agent-swarm/raw/articles/evolve-as-a-team-collaborative-self-evolution-for-llm-based-multi-agent-systems.md:52-75` and `:627-633`.
- `/home/pedronauck/Projects/second-brain/research/agent-swarm/raw/articles/the-society-of-hivemind-multi-agent-optimization-of-foundation-model-swarms-to-unlock-the-potential-of.md:24-52`, `:474-500`, `:528-538`, and `:611-630`.

Those patterns may improve open-ended exploration, but they introduce topology selection, agent creation, shared-memory coherence, and stopping decisions. They should not become the default implementation for AGH tasks or loops. AGH first needs a cheap fixed topology and complete cost traces.

## Pattern Decision Matrix for AGH

| Pattern | AGH default | Explicit opt-in | Reject for now | Reason |
|---|:---:|:---:|:---:|---|
| Local single agent with tools | Yes | | | Lowest coordination overhead |
| Local deterministic task pipeline | Yes | | | Existing scheduler can operate without Network |
| Bounded fan-out/fan-in workers | | Yes | | Best initial swarm primitive |
| Durable mailbox without model wake-up | | Yes | | Useful if zero-activation invariants are implemented |
| Explicit thread/group discussion | | Yes | | High cost; cap agents and rounds |
| Per-iteration MoA refresh | | Narrowly | | Only under root budget |
| Fully connected peer broadcast | | | Yes | Sticky participants and activation amplification |
| Model handling of receipts/presence/trace | | | Yes | Runtime-owned control plane |
| Learned interruption/routing | | | Yes | Requires AGH traces and quality labels first |
| Dynamic agent/team creation | | | Yes | Control and stopping system not yet present |
| New transport replacing embedded NATS | | | Defer | Current highest cost is model activation, not broker bytes |

## Proposed AGH Swarm Primitive

The references imply a small, independent primitive rather than a behavior hidden inside channel delivery.

### Request contract

```json
{
  "mode": "fanout_fanin",
  "workers": [
    {"role": "researcher", "agent_id": "...", "max_output_tokens": 600},
    {"role": "reviewer", "agent_id": "...", "max_output_tokens": 400}
  ],
  "input": {"task": "...", "artifact_refs": ["..."]},
  "limits": {
    "max_workers": 4,
    "max_model_calls": 5,
    "max_total_tokens": 5000,
    "deadline_ms": 30000,
    "min_successes": 1
  },
  "fanout": "user_turn",
  "synthesis": "acting_agent"
}
```

### Worker result contract

```json
{
  "worker_id": "...",
  "role": "researcher",
  "status": "completed",
  "summary": "...",
  "artifact_refs": ["..."],
  "confidence": 0.78,
  "evidence_refs": ["..."],
  "usage": {
    "input_tokens": 0,
    "output_tokens": 0,
    "cost_usd": 0
  },
  "termination_reason": "completed"
}
```

The exact public schema belongs in a TechSpec. The important properties are bounded execution, typed results, one root budget, no implicit thread, and no worker-to-worker activation.

### Relation to Network participation modes

The swarm primitive and Network participation are orthogonal:

| Worker location/durability | Transport | Network participation needed? |
|---|---|---:|
| Same daemon, ephemeral | Direct runtime invocation | No |
| Same daemon, durable session | Session/task bridge | No by default |
| Same daemon, durable mailbox | Local Network persistence | `mailbox` |
| Remote/federated agent | Network bridge | `active` or a future remote-mailbox contract |
| Human-visible debate | Thread delivery | `active` |

This separation prevents AGH from recreating the current problem under a new "swarm" name.

## Prompt Changes Derived from the References

### Remove from ordinary/local prompts

- Channel/thread terminology when participation is off.
- Network response-register instructions.
- Protocol examples and command tutorials.
- Channel roster and capability catalog.
- Instructions to create a thread or direct message.
- Presence/greeting obligations.

### Include for bounded workers

- One role sentence.
- Exact task slice.
- Minimum necessary artifact summaries.
- Output schema.
- Deadline and token cap.
- "Return result; do not coordinate with other workers."

### Include for active participants only

- Stable participant identity and participation mode.
- Compact reply handle for the current message.
- Allowed conversation operation for this turn.
- Remaining root budget and round limit.
- Small relevant roster, not full catalog.
- Runtime-generated correlation metadata outside natural-language instructions.

## Evaluation Plan

Compare the following on the same AGH workloads:

1. local single agent;
2. current Network channel behavior;
3. local bounded fan-out/fan-in;
4. Network-backed bounded fan-out/fan-in;
5. explicit three-agent group chat.

Measure:

| Dimension | Metrics |
|---|---|
| Quality | task success, judge score, accepted artifacts, regressions |
| Cost | input/cached/output/reasoning tokens, provider cost, cost per accepted result |
| Activation | model calls, calls per useful artifact, control-triggered calls |
| Latency | p50/p95 wall time, queue time, straggler wait, time to first useful artifact |
| Communication | envelopes, prompt bytes, repeated bytes, participant fan-out, rounds |
| Reliability | partial-failure completion, retry duplicates, lost delivery, stale result rate |
| Isolation | cross-workspace negative tests, cache-key scope, event/SSE scope |

Each benchmark must use root-level traces. Aggregate channel counters cannot attribute savings to an orchestration decision.

## Adoption Sequence

1. Make local/off the effective default and stop injecting Network surfaces when off.
2. Decouple tasks, loops, judges, reviews, and spawned sessions from mandatory channel binding.
3. Make control envelopes runtime-owned and block them from model activation.
4. Add root activation IDs, invocation usage, budgets, and useful-result attribution.
5. Implement local, bounded fan-out/fan-in with stateless workers and typed results.
6. Allow the same primitive to select Network transport for explicit remote/durable cases.
7. Add explicit group discussion only with hard agent/round/call/token limits.
8. Profile real runs before experimenting with adaptive routing, pruning, or interruption.

## AGH Impact Audit

- **Native tools:** A swarm primitive would require explicit descriptors and schemas only if it is exposed as an agent-manageable capability. Existing `agh__network_*` tools must remain absent in `off`, become persistence-only where permitted in `mailbox`, and become active only in `active`. Tool availability must describe the effective mode and budget.
- **Extensibility and hooks:** Worker policies, synthesis strategies, result schemas, and transport selection are natural registries/extensions. Hooks must observe invocation/artifact events without waking models. MCP sidecars and future bridges can implement worker execution behind the same bounded contract.
- **Workspace data isolation:** Root activations, caches, worker results, artifacts, usage, budgets, traces, and delivery decisions are workspace-scoped. Every key includes `workspace_id`; list/read/SSE/cache paths must prove negative isolation across identical task/channel names.
- **Official AGH skill:** `skills/agh/` must teach local-first execution, the difference between delegation/handoff/discussion, explicit participation modes, bounded swarm invocation, typed artifact returns, and the absence of Network tools in local mode. It should not teach manual thread mechanics to sessions that did not opt in.

## Conclusion

Hermes' most relevant optimization is architectural: references are stateless, tool-free advisor calls with an eight-call concurrency cap, and one normal actor owns the result. Total references, projected input, output, retries, and root spend are not comprehensively bounded in the inspected implementation. The second-brain corpus reinforces that multi-agent interaction should be added only after local execution reaches a measured limit, and suggests testing shared artifacts against persistent full-history conversation rather than assuming savings.

For AGH, the immediate objective is not a more sophisticated free-form agent society. It is a small set of explicit execution modes with deterministic admission, hard budgets, typed results, and root-level cost accounting. Once those foundations produce real traces, sparse or adaptive swarm policies can be evaluated without guessing.
