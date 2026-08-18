# AGH Network Changes Research Package

## Status

- **Snapshot date:** 2026-07-10
- **AGH source snapshot:** `01d3ec251be0a34e1b42a2393f82383c750011b7`
- **Hermes source snapshot:** `.resources/hermes` at `b8880f124537acc5a6215718dd154eadc5af1515`
- **Second-brain topics:** `/home/pedronauck/Projects/second-brain/research/agent-swarm` and `/home/pedronauck/Projects/second-brain/research/agent-networks`
- **Artifact status:** architecture research and implementation recommendations; not an approved PRD or TechSpec
- **Code changes:** none
- **Validation boundary:** static source analysis plus one focused renderer benchmark; no link check, lint, full tests, build, runtime QA, or `make verify`

This package answers two questions:

1. Where does AGH currently force Network, channels, and conversation containers into sessions, tasks, loops, autonomy, prompts, tools, and UI?
2. How should AGH make Network opt-in and reduce the token/model-call cost of explicit multi-agent collaboration?

## Document map

| Document | Scope | Use it for |
|---|---|---|
| [01 - Current-state enforcement audit](./01-current-state-enforcement-audit.md) | Complete enforcement trace across config, session, task, loop, automation, scheduler, coordinator, prompt, tools, protocol, UI, tests, and docs | Finding every current coupling and delete target |
| [02 - Protocol and token-efficiency audit](./02-network-protocol-token-efficiency-audit.md) | Delivery/activation pipeline, stable mailbox-address gap, prompt cost, broker/storage work, correctness, performance tiers, metrics, and file decomposition | Designing the Network internals and cost controls |
| [03 - Reference research and patterns](./03-reference-research-and-patterns.md) | Current Hermes summary plus second-brain network/swarm synthesis and candidate AGH swarm contract | Understanding external patterns and research hypotheses |
| [04 - Opt-in architecture](./04-opt-in-architecture.md) | Normative `NetworkParticipationSpec`, mode matrix, resolution, lifecycle, every domain surface, council record, and hard delete list | Authoring the participation TechSpec |
| [05 - Implementation roadmap](./05-implementation-roadmap.md) | P0/P1/P2 program, proven bugs, stable-address/durability boundary, vertical slices, TechSpec boundaries, tests, benchmarks, rollout, and definitions of done | Sequencing implementation work |
| [06 - Hermes MoA current-source audit](./06-hermes-moa-current-source-audit.md) | End-to-end audit of the actual `.resources/hermes` snapshot, including config/UI drift and accounting/cancellation risks | Implementing bounded fan-out/fan-in without copying expensive defaults |

Recommended reading order:

```text
README
  -> 01 current enforcement
  -> 04 target architecture
  -> 05 implementation sequence
  -> 02 protocol/cost internals
  -> 06 current Hermes implementation
  -> 03 broader research corpus
```

## Executive decision

AGH should make Network participation an explicit, resolved execution policy with a built-in local default:

```text
off | mailbox | active
```

The public contract must not infer participation from a channel string. It must resolve the policy once at session/run creation, persist the immutable effective snapshot, and project behavior from that snapshot across session lifecycle, prompts, tools, scheduler, coordinator, hooks, events, UI, and recovery.

The three target semantics are:

| Mode | Durable Network state | Subscriptions and roster | Network tools | Inbound model activation |
|---|---:|---:|---:|---:|
| `off` | None | None | None except separately authorized administration | Never |
| `mailbox` | Durable address/cursor survives process exit | Passive membership/subscriptions/roster while live; offline addressability is separate | Persistence-safe subset | Never |
| `active` | Durable address/cursor plus live lease | Explicit active posture while live | Policy-limited subset | Only for a capable ACP adapter, after deterministic admission plus finite turn/attempt/token/depth/time reservation |

P0-B defines and internally tests all three modes but publicly exposes only Local/`off`. It also
owns minimum finite active admission, a capability-negotiated ACP provider-attempt budget/event
extension, a trusted pre-persistence adapter manifest/artifact check, at least one AGH-controlled
pinned real adapter, conformance/real-provider evidence, and lazy startup of the current transport.
The pre-persistence adapter check is process-free; live ACP startup remains post-commit under
explicit cleanup ownership and must match the verified contract before any Network lease.
P0-C makes attempt state restart/cancellation safe. `mailbox` and `active` become publicly selectable
only after P0-C hard-cuts stable mailbox addressing/fencing and fixes durable first-send/publish/
restart semantics, and after the respective zero-activation or finite-activation release gate passes.
Neither unfinished mode silently degrades to another.

`network.enabled` remains the daemon-level administrative availability/kill switch. It does not select a session/run mode. `network.default_channel` may resolve a channel only after the caller has explicitly selected a participating mode.

## Why this change is necessary

### Current behavior is not opt-in

The current product presents some channel fields as optional, but the backend often resolves omission to the default channel. Other paths deterministically create a channel regardless of caller intent.

High-impact enforcement points:

| Surface | Current behavior | Primary evidence |
|---|---|---|
| Config | Network enabled and `default` channel configured by default | `internal/config/config.go:754-769` |
| Session API | Empty channel resolves through bundle/global default | `internal/api/core/handlers.go:128-160`; `internal/api/core/bundles.go:177-195` |
| Session UI | Create and onboarding have no Network/local choice | `web/src/systems/session/hooks/use-session-create-dialog.ts:271-291`; `web/src/systems/onboarding/hooks/use-onboarding-chat.ts:96-115` |
| Spawn | Child inherits parent channel unless it is a memory extractor | `internal/session/spawn.go:480-490` |
| Detached work | Owner channel propagates to detached sessions | `internal/daemon/harness_detached_work.go:246-265,613-619` |
| Reviews | Reviewer sessions inherit run coordination channel | `internal/daemon/review_router.go:561-595,779-797` |
| Task run | Store derives `coord-<run>` and inserts a Network channel | `internal/store/globaldb/global_db_task_aux.go:741-877` |
| Scheduler | Candidate session must match run channel | `internal/scheduler/scheduler.go:720-779` |
| Coordinator | Bootstrap requires a channel | `internal/coordinator/coordinator.go:100-142` |
| Autonomy/kernel contract | Lease/safe-spawn tools are local, but agent-kernel CLI and docs make task-bound channels the normal communication path and declare `bind always` | `internal/tools/builtin/autonomy.go:13-94`; `internal/cli/agent_kernel.go:101-247,324-375`; `packages/site/content/runtime/core/autonomy/coordination-channels.mdx:6-28` |
| Loop | Coordinator and node runs synthesize channels; run-agent/judge sessions bind them | `internal/store/globaldb/global_db_loop.go:26-81`; `internal/daemon/loop_runtime_adapters.go:37-66,118-145,340-393` |
| Automation | Task/loop dispatch inherits those paths | `internal/automation/dispatch.go:666-677,1185`; `internal/daemon/automation_loop_starter.go:98-131` |
| Prompt | Non-empty channel selects Network prompt machinery | `internal/daemon/harness_context.go:310-330,518-539` |
| Task-role prompt | Blank channel is replaced with fictional `default` in session identity and `Coordination channel` prompt text | `internal/daemon/task_role_sessions.go:260-262,286-300` |
| Tools | Coordination toolset is broad and default policy may be unrestricted | `internal/tools/builtin/toolsets.go:25-46`; `internal/tools/policy.go:174-237,323-325` |
| CLI protocol | `agh ch send` uses one fixed thread; `reply` changes to direct room | `internal/api/core/agent_channels.go:23-32,121-185,283-315` |

The full evidence and frozen tests/docs are in Document 01.

### Channel presence is overloaded

Today, a non-empty channel can mean all of the following:

- durable conversation storage;
- participant membership;
- transport subscription;
- environment identity;
- prompt section selection;
- native-tool availability;
- scheduler eligibility;
- coordinator bootstrap;
- permission for an inbound envelope to start a model turn.

These are independent concerns. A string must not be the control plane for all of them.

### Network is also an implicit model scheduler

The largest cost is not the NATS payload. A delivered envelope can authorize a full ACP/model turn. Control traffic, digest traffic, participant fan-out, and repeated guidance can therefore multiply inference.

Current amplification shape:

```text
envelope
  -> route recipients
  -> full or digest delivery per recipient
  -> PromptNetwork ACP turn
  -> repeated Network guidance and tools
  -> possible new Network envelopes
```

`activation_top_k` limits full delivery, not total model turns. Sparse recipients still receive digest prompts. Established participants remain sticky recipients. The acting model's final text and usage are discarded by Network delivery unless the model separately sends a protocol message.

## Quantitative evidence

A focused benchmark ran the existing Network guidance renderer once in each mode:

```text
go test ./internal/network -run '^$' \
  -bench '^BenchmarkFormatNetworkMessageGuidanceModes$' \
  -benchtime=1x -count=1
```

Observed renderer output:

| Mode | Bytes | Rough `bytes / 4` token estimate |
|---|---:|---:|
| Verbose | 4,193 | 1,049 |
| Compact | 2,296 | 574 |
| Difference | -1,897 (-45.2%) | -475 |

This measures only the rendered Network guidance. It excludes the base system prompt, conversation history, tools/descriptors, situation context, model output, retries, and peer fan-out. Prompt compression is useful but cannot replace activation avoidance.

Lexical blast-radius inventory from Document 01:

| Symbol family | Internal production Go files | Web production TS/TSX files | Meaning |
|---|---:|---:|---|
| `coordination_channel_id` / `CoordinationChannelID` | 52 | 9 web files total | Task/run/coordinator coupling |
| `network_channel` / `NetworkChannel` | 104 | 47 | Broader Network contract and UI coupling |

These are lexical counts, not semantic dependency counts.

## Proven correctness and target-contract defects

The audit found correctness defects plus one mechanically proven blocker to the target durable
mailbox. They must not be hidden as token optimizations.

| ID | Severity | Defect | Disposition |
|---|---|---|---|
| C-01 | P0 | Channel metadata lookup uses `WHERE channel = ?` without workspace qualification | Independent P0 fix before semantic migration |
| C-02 | P0 durability | First undirected local thread send can reach zero local peers because persistence adds the sender before routing | Durable delivery TechSpec |
| C-03 | P0/P1 | Network delivery discards ACP errors/final outcome/aggregate usage yet can mark delivery successful; current ACP has no provider-attempt/retry/fallback lifecycle | Outcome correctness in P0-A; minimum capability-negotiated attempt contract in P0-B, durable reconciliation in P0-C, rich pricing/analytics in P1 |
| C-04 | P0 durability | Network work lifecycle has in-memory and SQLite authorities without hydration/pruning | Make SQLite the only authority |
| C-05 | P0 durability | Persist-before-publish plus duplicate-success retry can create a phantom accepted send | Transactional outbox/state machine TechSpec |
| C-06 | P0 | Task status observer captures nil Network because tasks boot before Network and never rebinds; its dormant implementation would publish conversational `say` traffic | Replace with dependency-complete typed status projection structurally unable to activate a model; do not merely rebind it |
| C-07 | P0 durability | Directed sends, direct rooms, mentions, thread participants, and subscriptions use live process peer ids; directed send rejects an offline target | P0-C stable mailbox-address protocol/storage/routing hard cut plus fenced mailbox-to-lease resolution before mailbox or active release |

Critical workspace evidence:

- schema identity is `(workspace_id, channel)` at `internal/store/globaldb/global_db.go:549-565`;
- lookup drops `workspace_id` at `internal/store/globaldb/global_db_task_claim_helpers.go:189-201`;
- metadata can be overwritten from the wrong row at `:145-186`.

## Target contract

### Resolve once

Every execution boundary accepts a requested participation object, resolves it exactly once, and persists both effective policy and provenance.

Conceptual request:

```json
{
  "preset": "off",
  "channel_strategy": null,
  "channel_id": null,
  "activation": null
}
```

Conceptual resolved snapshot:

```json
{
  "version": "agh.network-participation.v1",
  "preset": "off",
  "participation": "disabled",
  "workspace_id": null,
  "channel_strategy": null,
  "channel_id": null,
  "mailbox_id": null,
  "activation": {
    "mode": "disabled"
  },
  "resolution": {
    "preset_source": "built_in_off"
  }
}
```

The approved TechSpec must settle exact names and schemas. It must preserve these invariants:

1. Blank channel is not an opt-out signal.
2. A non-empty channel does not imply model activation.
3. `off` creates no channel, membership, subscription, greet, roster, Network prompt, Network environment, or Network tool projection.
4. `mailbox` persists and reads without any inbound `PromptNetwork` call.
5. `active` reserves a finite activation budget before every model call.
6. Children, reviews, detached work, loops, and automation do not inherit participation implicitly.
7. Coordinator and scheduler work locally without Network.
8. Every participating datum and cache key includes `workspace_id`.
9. Mailbox id is the stable recipient identity; process peer/lease id may rotate and is never a
   durable address or foreign key.
10. Directed recipients, direct-room pairs, resolved mentions, thread participants, subscription
    policy, work ownership/targeting, delivery keys, and usage are mailbox-keyed.
11. A registered offline mailbox accepts durable ingress without live presence; mailbox-to-live-lease
    resolution occurs only after acceptance and is protected by a lease epoch/fencing token.
12. Remote home runtime identity persists per `AGH_HOME` across daemon restart; endpoint epochs rotate,
    and only immutable accepted/rejected dispositions re-attested by the authoritative current
    endpoint settle sender outbox.
13. Mailbox-home transaction order makes delivery-before-tombstone stay accepted and
    tombstone-before-delivery terminate rejected across retry/restart, avoiding infinite retry.
14. Every remote envelope has one authenticated immutable deadline: late home arrival yields
    home-authenticated `envelope_expired`, while an unreachable home yields sender-local
    `delivery_dead_lettered_unconfirmed` without fabricating recipient authority.
15. Network config exposes activation defaults/limits, not an undeclared named-policy selector;
    owning requests/profiles persist the canonical inline participation object.

### Surface behavior

The same policy must be manageable through every public surface:

| Surface | Required behavior |
|---|---|
| HTTP/UDS | Typed request, resolved snapshot, provenance, structured diagnostics |
| CLI | Explicit preset flag/option and structured output; no negative `--no-network` default model |
| Web | P0-B shows only `Local`; after P0-C, add `Active` and `Mailbox` independently when their gates pass; conditional strategy/channel/budget fields |
| Native tools | Participation fields on owning management tools; descriptors unavailable when execution is `off` |
| Config | Administrative availability, activation defaults/limits, and a conditional channel default; built-in execution default remains `off` |
| Hooks/extensions | Requested/resolved policy and source; hooks may narrow but never widen without explicit authority |
| Events/SSE | Workspace-scoped effective mode, activation decisions, budgets, and usage |
| Official skill/docs | Local-first selection and conditional Network guidance |

## Protocol optimization order

The order matters because avoiding a call saves more than shrinking its prompt.

### Tier 0: correctness and measurement

1. Fix workspace channel identity.
2. Replace peer-address Network v0 with stable mailbox sender/recipient/mention/direct/subscription/
   work identity and fenced mailbox-to-live-lease resolution; offline delivery cannot depend on
   presence. Remote delivery requires persistent authenticated home runtime identity, endpoint
   re-registration, immutable ACK/NACK plus current-endpoint re-attestation, and home-ordered
   tombstone settlement. An authenticated immutable deadline separates home `envelope_expired` from
   sender-local `delivery_dead_lettered_unconfirmed` and prevents acceptance of delayed copies.
3. Capture ACP outcome and aggregate usage; add a negotiated adapter extension for bounded
   provider-attempt identity, retry/fallback accounting, and per-attempt settlement before active is
   public. Ship one AGH-controlled pinned real adapter, verify its manifest/artifact before owner
   persistence, and reject unsupported profiles before creation.
4. Replace the dead conversational status observer with a dependency-complete deterministic
   projection that cannot activate a model.
5. Add root activation IDs plus a durable attempt ledger linking admission, usage, outcome, and
   causation.
6. Instrument model calls, prompt bytes/tokens, queue time, fan-out, retries, selected/discarded results, and cost.

### Tier 1: avoid activation

1. Default ordinary execution to `off`.
2. Make the current transport lazy so `off` never starts or leases it.
3. Make receipts, presence, trace, task status, and delivery bookkeeping runtime-owned control traffic under every policy.
4. Decide recipient/admission before any model call.
5. Enforce directed/mentioned routing, finite recipient caps, call budgets, token/cost budgets, deadlines, and round limits.
6. Coalesce bursts before activation; one prompt processes a bounded batch.

### Tier 2: reduce remaining context

1. Replace repeated XML/base64/tutorial wrappers with a compact structured delivery record.
2. Inject a compact reply handle rather than command examples.
3. Omit empty situation sections rather than rendering structural shells.
4. Project Network tools only when current mode and policy allow them.
5. Use stable prompt prefixes and tail/delta placement based on artifact mutability.

### Tier 3: improve transport and storage

1. Move durable delivery to an accepted/published/delivered state machine with outbox/recovery.
2. Decide in-process local versus NATS local transport through crash tests and N=1/3/10/50 profiling.
3. Replace full summary recounts with incremental materialized counters.
4. Batch/audit asynchronously where correctness permits.
5. Add Network SSE/push to replace aggressive UI polling.

Transport replacement is not part of the participation hard cut. It belongs to a separate evidence-backed TechSpec.

## Hermes MoA lessons

The current `.resources/hermes` implementation demonstrates a lower-coordination-overhead shape; it
does not prove lower total cost or higher quality:

```text
explicit user selection
  -> parallel stateless, tool-free advisors
  -> one acting aggregator with normal tools
  -> one authoritative transcript/output
```

Adopt:

- explicit one-shot and persistent selection;
- one tool-bearing actor;
- raw/stateless advisor calls;
- smaller advisory projection without the full actor system/tool schema, while AGH retains a compact
  safety/authority and data-egress envelope;
- provider-aware prompt caching;
- configured-slot usage/cost accounting as an intent to improve into actual-attempt accounting;
- partial-failure continuation;
- trace side channel separate from conversation history.

Do not copy:

- default `per_iteration` fan-out;
- uncapped advisor output;
- wait-all fan-in with no root deadline/quorum/cancellation;
- concurrency cap without total worker/call budget;
- cumulative full-history advisor input;
- raw full-output concatenation and user-role injection;
- silent invalid-config fallback to expensive defaults;
- dual current/legacy aggregation paths;
- direct CLI/TUI mutable one-shot restore that can leak after exceptions; retain the gateway's
  explicit intent but use immutable per-run state;
- YAML-only cost controls and management API/UI field loss;
- configured-slot accounting after provider fallback;
- broad conversation/tool-result replication across multiple providers without a dedicated
  workspace egress policy;
- string-only projection that silently drops multimodal/structured content;
- streaming aggregator recovery with different validation/fallback semantics from non-streaming;
- unlimited plaintext full-body traces.

Important current-source defects:

- `reference_max_tokens` and `fanout` exist in `.resources/hermes/hermes_cli/moa_config.py:125-148` but are omitted by `.resources/hermes/hermes_cli/web_server.py:968-996,5505-5541` and both TS contracts; UI save can reset a bounded `user_turn` preset to uncapped `per_iteration`.
- Default auxiliary timeout/retry policy can make wait-all hold on one advisor for a very long time: `.resources/hermes/hermes_cli/config.py:1569-1576,1731-1745` and `.resources/hermes/agent/auxiliary_client.py:6542-6601`.
- Advisor accounting can disappear when the actor fails before usage settlement and retry takes the cache-hit path: `.resources/hermes/agent/moa_loop.py:887-899`; `.resources/hermes/agent/conversation_loop.py:2049-2093`.
- Advisor and aggregator fallback can execute a different model while retaining configured labels/pricing: `.resources/hermes/agent/auxiliary_client.py:6880-6952`; `.resources/hermes/agent/moa_loop.py:280-323,804-814`.
- Non-string content becomes empty in advisor projection: `.resources/hermes/agent/moa_loop.py:468-521`.

AGH's initial swarm strategy should therefore use one or two bounded advisors, once per root/user turn, typed 256-600-token results, a total context/call/token/cost/deadline budget, collect-as-completed, quorum/role coverage, shared cancellation, and one acting aggregator. Exact defaults require AGH benchmarks.

## Second-brain synthesis

The referenced agent-network and swarm research supports these constraints:

1. Start with one agent and add multi-agent execution only after an empirical limit.
2. Prefer bounded concurrent fan-out/fan-in for independent perspectives.
3. Keep group chat to specialized, small, round-limited debate/consensus cases.
4. Distinguish delegation, handoff, and tool calls; do not encode all three as chat.
5. Share typed facts, summaries, and artifact pointers rather than complete histories.
6. Treat A2A-style protocols as interoperability boundaries, not same-process control flow.
7. Measure useful-agent utilization, protocol overhead, redundancy, error amplification, and straggler cost.
8. Implement deterministic stopping before learned interruption/routing.

Research on sparse/adaptive topologies is promising but not an immediate implementation basis. AGH first needs root traces, useful-result labels, fixed bounded baselines, and cost-normalized quality evaluation.

## Implementation program

| Phase | Outcome | TechSpec boundary |
|---|---|---|
| P0-A | Workspace isolation, delivery outcome correctness, deterministic zero-activation status projection | Narrow correctness batch |
| P0-B | Hard-cut explicit participation, public Local/`off`, internal mailbox/active, minimum finite active admission/accounting, local coordinator/scheduler, lazy current transport | Participation TechSpec |
| P0-C | First-send correctness, durable publish/restart/offline-mailbox recovery, fixed activation coalescing/cancellation, SQLite-only work authority, independent non-off release gates | Durable delivery/outbox TechSpec |
| P1 | Routing/coalescing optimization, prompt diet, control-plane split, rich attempt/cost analytics, benchmarks | Activation and token-efficiency TechSpec |
| P2 | Bounded MoA/swarm strategy independent of Network | Separate execution-strategy TechSpec |

P0-B must not absorb a NATS replacement or outbox redesign. P0-C must not be dismissed as optional performance work because the audit found concrete delivery/state correctness defects.

## Hard-cut expectations

AGH is a greenfield alpha. The participation TechSpec must enumerate and delete:

- ambiguous bare `channel` / `network_channel` mode signaling;
- task/run automatic `coord-<run>` derivation; retain derivation only inside the canonical resolver
  when the explicit `run` strategy is selected;
- channel-based scheduler and coordinator preconditions;
- implicit channel inheritance for spawn/review/detached/loop/automation;
- prompt/tool/environment gating based only on non-empty channel;
- default-channel behavior that selects participation;
- tests, fixtures, generated contracts, docs, glossary, and official-skill text that encode mandatory channels;
- compatibility aliases, dual fields, fallback reads, and old schema assumptions.

Do not grow the existing Network god files. `manager.go`, `delivery.go`, `router.go`, and `validate.go` already exceed the production 500-line cap. Split responsibilities before implementation into participation resolution, ingress, outbox/inbox, activation admission, scheduler, prompt rendering, work repository, presence, metrics, and transport adapters.

## Decisions still requiring TechSpec approval

The research settles architecture direction but intentionally leaves these implementation decisions open:

1. Exact public type and field names for requested/resolved participation.
2. Whether `active` and `mailbox` clear their independent post-P0-C release gates in the same batch
   or separate batches; neither is public in P0-B.
3. Exact finite defaults for recipients, calls, tokens, cost, deadlines, rounds, and worker concurrency.
4. Exact persistence schema and migration numbers.
5. Whether active budgets scope to session, task run, loop run, root activation, or a combination with hard ceilings.
6. Exact compact delivery/reply-handle wire schema.
7. Exact outbox state machine and transport abstraction in P0-C.
8. Initial MoA quality dataset and acceptable incremental cost threshold.

## Evidence limitations

- Static tracing cannot measure real provider tokenization, cache-hit rates, latency, or quality.
- Current ACP `TokenUsage` is aggregate per turn; provider-attempt budgets in the target architecture
  require the proposed capability extension and cannot be inferred from today's events.
- The Network renderer benchmark is not an end-to-end session benchmark.
- Lexical blast-radius counts overcount comments/tests and undercount semantic behavior.
- Hermes benchmark figures in source/docs were not independently reproduced; no matching raw report was found in the snapshot.
- The first-send, workspace, boot-order, persist-before-publish, dual-authority, and live-peer versus
  offline-mailbox identity findings are static code-path findings; implementation must add focused
  integration/crash/restart/fencing tests.
- No production traffic profile was available.

## AGH Impact Audit

- **Native tools:** High impact. Network descriptors/toolsets, availability, schemas, digests, risk
  flags, capability gates, task/loop/session management inputs, stable mailbox addressing, and
  structured fallbacks must reflect effective mode. `off` must not expose execution-scoped Network
  tools; live peer ids are not valid durable recipients.
- **Extensibility and hooks:** High impact. Bundles, execution profiles, hooks, capabilities,
  registries, bridges, MCP sidecars, SDKs, and config lifecycle must carry the canonical
  requested/resolved policy. A trusted provider-adapter manifest/artifact/conformance registry gates
  active; raw provider config cannot self-assert support. Network bridges/SDKs hard-cut to v1 mailbox
  addresses with separately labeled optional lease evidence. Observation must not imply model activation.
- **Workspace data isolation:** High impact. Participation, channels, durable mailbox bindings/cursors,
  mailbox-keyed directs/mentions/participants/subscriptions/work/delivery/usage, live membership
  leases, activation budgets, actual attempts, caches, artifacts, traces, hooks, events, SSE, and UI
  queries are workspace-scoped. Every lookup/cache key includes `workspace_id`; same-name channels
  and mailbox/peer-like ids across workspaces require negative tests.
- **Official AGH skill:** Required. Rewrite `skills/agh/` for local-first execution, conditional
  Network tools, explicit modes, stable mailbox versus live peer identity, bounded collaboration,
  typed artifacts, and truthful task/loop/coordinator behavior.

## Handoff checklist

Before implementation starts:

1. Approve the target architecture in Document 04.
2. Convert P0-A defects into focused issues with canonical test placement.
3. Author TechSpec 1 from the P0-B slices and hard delete list in Document 05.
4. Include the complete AGH Impact Audit, Web/Docs Impact, config lifecycle, migration, codegen, QA tracker, and official-skill changes.
5. Keep P0-C durability and P1 activation-efficiency/rich-cost work in their own approved boundaries.
6. Capture a baseline matrix for local vs current Network at `N=1,3,10,50` before changing behavior.
7. Require an end-to-end proof that plain sessions, tasks, loops, reviews, spawn, detached work, and automation create zero Network state and zero Network-triggered model calls.
8. Make TechSpec 1 choose and own the first real active-capable adapter, pre-persistence manifest and
   artifact verification, live handshake, deterministic conformance suite, and credentialed provider
   smoke gate; a mock or raw `@latest` command is insufficient.
9. Keep adapter preflight process-free and require fault-injection proof that every post-commit
   failure/cancel path closes process/stdio/process-record, provider secret/runtime material,
   sandbox, reservation, recorder, and Network lease state.
10. Make TechSpec 2 own C-07 as a full Network v1 hard cut: stable mailbox addresses for every
    durable relation, offline directed delivery without presence, direct identity across restart,
    fenced mailbox-to-live-lease resolution, persistent authenticated home runtime identity,
    endpoint re-registration, tombstone-ordered immutable ACK/NACK, current-endpoint re-attestation,
    authenticated immutable delivery deadlines, distinct home-expiry/sender-dead-letter semantics,
    and no v0 peer aliases.
