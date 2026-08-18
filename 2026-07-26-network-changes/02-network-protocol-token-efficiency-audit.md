# AGH Network Protocol and Token-Efficiency Audit

## Document status

- **Snapshot date:** 2026-07-10
- **AGH commit:** `01d3ec251be0a34e1b42a2393f82383c750011b7`
- **Scope:** `internal/network`, its session/prompt/task integrations, HTTP/UDS/native-tool projections, and the current v0 protocol contract
- **Method:** static code tracing, focused source inspection, one renderer benchmark, and cross-checking against the bundled AGH skill and RFCs
- **Not performed:** full tests, lint, link checks, `make verify`, runtime scenario QA, or production traffic profiling

This document separates observed implementation facts from cost models and recommendations. A static finding is not a measured production incident unless explicitly labeled as a benchmark or confirmed defect.

## Executive diagnosis

The main cost problem is not NATS payload size by itself. It is the fact that AGH treats a routed protocol envelope as near-direct permission to start a complete ACP/model turn.

The current implementation combines five responsibilities:

1. local event transport;
2. possible future remote/federated transport;
3. durable conversation and work state;
4. model-activation scheduling;
5. multi-agent orchestration.

That combination turns ordinary protocol behavior into inference amplification:

- control traffic can reach model prompts;
- every selected recipient normally receives its own complete turn;
- `activation_top_k` limits only full deliveries, not total model invocations;
- sparse digest traffic normally still creates one turn per envelope and recipient;
- established thread participants remain sticky full-delivery recipients;
- a model's useful output is discarded unless the model explicitly sends another Network message;
- prompt guidance repeats protocol rules and CLI examples after activation has already been paid for.

The correct architectural split is:

- **Coordination plane:** deterministic presence, routing, receipts, traces, work state, inbox/outbox, subscriptions, cursors, and budgets.
- **Activation plane:** decides whether zero, one, or several model turns are warranted.
- **Cognition plane:** runs bounded ACP/model calls for actionable content only.
- **Projection plane:** renders channels, threads, directs, task links, and audit views without activating agents.
- **Federation adapter:** optional remote transport, independent of the local durable source of truth.

The highest-leverage result is therefore: **persisting or routing a message must not imply model activation**.

## Quantitative baseline

### Existing renderer benchmark

The repository already contains `BenchmarkFormatNetworkMessageGuidanceModes` at `internal/network/perf_bench_test.go:26-54`. It renders a direct `say` envelope containing work, reply, trace, causation, intent, and two inline artifacts (`internal/network/perf_bench_test.go:129-158`).

The benchmark was run once for analysis only:

```text
BenchmarkFormatNetworkMessageGuidanceModes/verbose-12
  4193 bytes/message
  1049 estimated tokens/message

BenchmarkFormatNetworkMessageGuidanceModes/compact-12
  2296 bytes/message
  574 estimated tokens/message
```

Command used:

```bash
go test ./internal/network -run '^$' \
  -bench '^BenchmarkFormatNetworkMessageGuidanceModes$' \
  -benchtime=1x -count=1
```

Important qualifications:

- These are renderer bytes and the repository's own `bytes / 4` estimate, not provider-reported tokens.
- The estimate excludes the session system prompt, conversation history, provider tool schemas, situation context, generated model output, and any subsequent reply turn.
- The benchmark fixture is intentionally rich. A plain `say` can be smaller, while a long `say`, full catalog, or repeated tool context can be much larger.
- Even the compact mode still repeats the protocol guidance array and a compact marker (`internal/network/delivery.go:29-50,1647-1693`).

### Cost model

For one root interaction, the relevant quantity is not envelope bytes alone:

```text
total_model_cost(root) =
  sum over activated recipients and rounds (
    stable session/system context
    + current conversation context
    + Network delivery prompt
    + provider tool context
    + model output/reasoning/cache cost
  )
```

The current accounting records only the rendered Network prompt size and estimates tokens as `(bytes + 3) / 4` (`internal/network/delivery.go:729-742`). Thread peer totals receive those estimates at `internal/network/manager.go:1930-1968`. It does not attribute actual input, output, thought, cache, or monetary cost, and it omits equivalent aggregate accounting for channels and direct rooms.

ACP already exposes the needed usage fields at `internal/acp/types.go:505-518`. The delivery loop discards every event at `internal/network/delivery.go:885-899`, leaving those fields unused for Network cost attribution.

## Current end-to-end path

For a normal actionable message, the effective path is:

1. The sender constructs a full v0 envelope.
2. `Manager.Send` persists the conversation record before publication (`internal/network/manager.go:680-712`).
3. The router marshals JSON and publishes to a NATS subject (`internal/network/router.go:1166-1191`).
4. `Transport.Publish` synchronously flushes the NATS connection (`internal/network/transport.go:234-259`).
5. The subscription callback discards the subject and passes only bytes to the manager (`internal/network/manager.go:1655-1661,1712-1719`).
6. The manager parses, validates, routes, applies subscriptions, persists/audits, and accepts deliveries.
7. The delivery coordinator creates one worker per target session and calls `PromptNetwork` for each queued delivery (`internal/network/delivery.go:616-688`).
8. The normal exclusive session prompt path runs with a Network turn-source marker (`internal/session/manager_prompt.go:101-130`).
9. The delivery coordinator drains and discards all ACP events (`internal/network/delivery.go:885-899`).
10. If the agent wants its answer to reach another peer, it must issue another Network tool/CLI send, beginning another route and activation cycle.

This is closer to model-activated group chat than a bounded swarm execution fabric.

## Detailed findings

### P0-1: one envelope normally means one full model turn per recipient

**Observed.** `processQueuedItem` renders the message, calls `PromptNetwork`, drains the whole event stream, and then advances to the next item (`internal/network/delivery.go:660-688`). Full deliveries do not coalesce.

**Control traffic is not isolated.** Directed `receipt` and `trace` envelopes are delivered to the target model (`internal/network/router.go:676-686`). Whois responses can also be delivered to the requesting model (`internal/network/router.go:815-835`). Capability messages follow the same general content-delivery path.

**Effect.** Presence, discovery, acknowledgements, progress state, and useful content share the same cognition queue. A large part of the cost exists even when deterministic runtime code already has enough information to update state.

**Required change.** Greet, whois, receipt, trace, task status, work transitions, catalog changes,
presence, subscriptions, and delivery acknowledgements must be runtime-owned control events. They
can update stores, hooks, SSE, and inbox state, but they are ineligible for model activation under
every Network policy. A human reaction starts a new explicit local prompt; it does not promote the
control envelope.

### P0-2: useful model output is paid for and then discarded

**Observed.** `drainPromptEvents` ignores event contents and treats a closed channel as success (`internal/network/delivery.go:885-899`). It does not inspect `EventTypeError`, capture final text, or aggregate usage.

**Effects.**

- A prompt can be marked delivered even when the agent reports an error.
- Final text that is not followed by `agh__network_send` is invisible to Network participants.
- Agents need an additional tool call and outbound envelope to make the paid output useful.
- Actual usage and monetary cost are unavailable at the envelope/root-trace level.

**Required change.** Define interaction contracts that state whether a response is required. For required responses, capture the final ACP result directly and materialize a response/result artifact or message under runtime authority. Inspect terminal/error events and attribute provider usage to the originating delivery batch.

### P0-3: `activation_top_k` does not cap activations

**Observed.** For an empty thread using capability matching, up to K positive-score peers receive full delivery (`internal/network/router.go:1422-1448`). Remaining peers can become `digestPeers` (`internal/network/router.go:1292-1329,1450-1464`). Digest delivery still enters `PromptNetwork` (`internal/network/delivery.go:660-688`).

**Effect.** `activation_top_k = 3` can still generate O(N) model turns. K bounds payload richness for some recipients, not cognition count.

**Required change.** Non-selected recipients receive store/mailbox state only. `top_k` must be enforced by the activation plane as a hard maximum number of model calls. The default routed policy should select one responder, expand only on explicit disagreement, low confidence, missing capability, or operator choice, and allow zero responders.

### P0-4: thread participation permanently expands the full-delivery set

**Observed.** Senders, targets, and mentions are upserted as thread participants (`internal/store/globaldb/global_db_network_conversations.go:152-168`; `internal/store/globaldb/global_db_network_conversation_summary.go:12-56`). Established threads deliver to all persisted participants and current mentions (`internal/network/router.go:1250-1265`). There is no participant leave, activity expiry, watcher/speaker distinction, or closed-thread delivery state.

**Effect.** A one-time mention can cause future unrelated messages in the same thread to wake that model indefinitely. Cost grows with thread history rather than current intent.

**Required change.** Separate membership, subscription, participant history, active speaker, watcher, and activation target. Mentions should be one-shot wake hints. Add inactive/leave/close state and an expiry policy for active participation.

### P0-5: digest batching usually misses the only useful debounce window

**Observed.** `collectDigestBatch` waits for `digestFlushInterval` only when a second compatible item is already queued (`internal/network/delivery.go:691-708`). The first item is normally dequeued and processed immediately. Batches are further constrained to contiguous items from the same sender (`internal/network/delivery.go:912-937,1180-1212`).

**Content is not necessarily compact.** A digest preview for `say` is the full message text (`internal/network/delivery.go:1924-1950`). Plain `say` bypasses the structured body cap (`internal/network/delivery.go:1336-1345`).

**Effect.** Sparse status traffic still creates a model turn for every event. Multi-sender swarm updates, the most valuable batching case, are not accumulated together.

**Required change.** Use a per-recipient, per-container/root-trace accumulator that always opens a short bounded window, batches across senders, and applies total byte/token/item limits. Coalesce deterministic state before optional summarization.

### P0-6: response guidance is duplicated after the expensive decision

The same behavioral rule appears in three locations:

1. startup Network response register (`internal/daemon/network_response_register_prompt.go:47-59`);
2. Network turn augmenter (`internal/daemon/network_response_register_prompt.go:62-93`);
3. delivery reply guidance (`internal/network/delivery.go:1647-1693`).

The delivery guidance additionally repeats reply flags, causation, work and trace preservation, four protocol rules, and first-delivery Bash examples (`internal/network/delivery.go:1743-1868`). Compact mode removes examples but retains protocol rules and adds another instruction line.

**Effect.** "Reply only when useful" is evaluated by a model only after the full activation cost has occurred. Each ephemeral spawned/task/loop session pays verbose guidance again because guidance state is session-scoped (`internal/network/delivery.go:532-604`).

**Required change.** Move the silence/activation decision before `PromptNetwork`. Put stable protocol behavior in one compact startup/tool contract, use an opaque runtime-resolved `reply_handle`, and send only actionable per-message data.

### P0-7: XML and base64 inflate and obscure structured content

**Observed.** Structured JSON bodies are canonicalized, capped, then base64-encoded inside XML (`internal/network/delivery.go:1386-1447`). The wrapper repeats envelope metadata as XML attributes (`internal/network/delivery.go:1584-1625`).

**Effect.** Base64 adds roughly one third in bytes, tokenizes poorly, duplicates preview/body information, and forces the model to recover JSON it could have received directly. The 4 KiB structured body cap limits one component but does not cap the complete prompt or plain text.

**Required change.** Deliver bounded structured JSON/content parts directly. Large bodies become content-addressed, versioned artifact references with a short preview, digest, media type, byte size, and authorized fetch tool.

### P0-8: there is no root interaction budget or loop breaker

The v0 envelope has `reply_to`, `trace_id`, and `causation_id`, but the runtime does not use them to enforce:

- a maximum hop count;
- a maximum response count;
- root token/cost/call budgets;
- a deadline or cancellation state;
- quorum or aggregation completion;
- cycle rejection.

The envelope fields are defined at `internal/network/envelope.go:196-218`; public sends lack budget/response semantics at `internal/api/contract/contract.go:812-831`.

**Effect.** With N active participants and r rounds, unconstrained reply fan-out can grow far faster than the useful information in the root request. `(N-1)^r` is a risk upper-bound model, not an observed benchmark, but no current invariant prevents that pattern.

**Required change.** Every active interaction needs a root ID, monotonic hop/sequence, response policy, maximum responders, call/input/output/cost/time budget, expiry, and terminal reason enforced by the runtime.

### P0-9: Network accounting discards actual ACP usage

**Observed.** ACP events can carry input, output, thought, cache, context, and monetary usage (`internal/acp/types.go:505-518`). Delivery discards the events (`internal/network/delivery.go:885-899`) and records rendered bytes/4 instead (`internal/network/delivery.go:729-755`). Only thread peer estimates are persisted (`internal/network/manager.go:1930-1968`).

**Required change.** Correlate `root_interaction_id -> envelope -> target delivery -> ACP turn -> reply/result`. Retain exact batch totals and provider/model attribution. Allocation of batched totals to individual messages can be an explicitly labeled estimate.

### P0-10: Network task status should not be conversational inference

**Observed.** The status observer builds textual started/completed/failed/canceled updates and sends a `say` into the origin thread (`internal/daemon/network_task_status_observer.go:180-313,413-445`). That thread message can activate participants.

**Adjacent wiring finding.** `bootTasks` runs before `bootNetwork` (`internal/daemon/boot.go:210-225`). The observer is constructed with the then-current `state.network` (`internal/daemon/task_runtime_boot.go:43-60`; `internal/daemon/task_event_bridge_notifier.go:106-138`), and its constructor returns nil without a Network runtime (`internal/daemon/network_task_status_observer.go:75-83`). No later rebind was found. This is a high-confidence static finding, not a runtime reproduction.

**Required change.** Task state stays in task events/read models/SSE. Replace or delete the dead
observer with a typed deterministic projection structurally unable to call `PromptNetwork`. Do not
merely rebind the existing conversational `say` implementation and make its latent token cost live.

### P1-1: embedded NATS is currently a local loopback cost

**Observed.** `NewTransport` starts an embedded NATS server and connects only an in-process token-authenticated client (`internal/network/transport.go:93-159`). The server listens on `127.0.0.1` with a random private token (`internal/network/transport.go:187-199`). No production path was found that supplies another daemon with that URL/token.

Every publish marshals JSON and synchronously flushes (`internal/network/router.go:1166-1191`; `internal/network/transport.go:234-259`). Generated envelopes publish serially.

**Effect.** Local messages pay broker, JSON, callback, and flush overhead without receiving distributed transport value.

**Recommendation.** P0-B should lazy-start the current transport and ensure `off` executions never
reach it. The first-send, persist-before-publish, and dual-work-authority defects already authorize a
P0-C durability TechSpec. That TechSpec should compare an in-process local transport plus optional
NATS remote adapter against retaining local NATS behind the same outbox. Crash tests and
N=1/3/10/50 profiling select the topology; they do not gate authoring the TechSpec. Do not add
JetStream merely to duplicate SQLite durability.

### P1-2: presence creates idle per-session and near-quadratic audit work

**Observed.** Every joined session gets a shared broadcast subscription reference, a direct subscription, and its own heartbeat (`internal/network/manager.go:554-576`). It immediately sends a full greet and repeats every 30 seconds by default (`internal/network/manager.go:821-865`; `internal/config/config.go:754-769`). Capability briefs include the catalog (`internal/network/capability_brief.go:18-50`).

Each outbound greet is persisted/audited (`internal/network/manager.go:868-881`). Local recipients also audit control receipt (`internal/network/manager.go:1492-1526,1590-1606`).

**Effect.** N peers emit N greets per interval and can approach N(N-1) local receipt audit records, even when no useful work occurs. Reconnect re-greets all sessions sequentially (`internal/network/manager.go:1788-1808`).

**Required change.** Use one daemon/channel presence lease with join/change deltas and a long jittered refresh. Do not loop local greets through NATS or persist unchanged heartbeats as conversation messages. Advertise capability digest/version, fetch details on demand.

### P1-3: channel creation eagerly creates live agent sessions

**Observed.** The channel create path starts a session for each selected agent (`internal/api/core/network_details.go:84-165,853-879`). The public request exposes agents and fan-out configuration, not lazy membership, TTL, budget, role, or reuse (`internal/api/contract/contract.go:875-883`). Sessions are created sequentially.

**Effect.** Creating a collaboration space pays cold-start prompt/tool cost for every selected agent and keeps them live/heartbeating even when no message requires them.

**Required change.** Persist channel membership as definitions/permissions, independent of live ACP sessions. Start or reuse only the responder set chosen for an active interaction. Prewarming must be explicit, bounded, and TTL-controlled.

### P1-4: routing quality is lexical and cost-blind

**Observed.** Candidate scoring lowercases message/capability text and counts substring matches for terms of length at least three (`internal/network/router.go:1514-1583`). It does not use exact input/output schema compatibility, current load, availability, cost, latency, trust, prior quality, or confidence.

**Required change.** Use deterministic filtering first: exact capability IDs/tags, schema compatibility, permissions/trust, current availability, estimated cost/latency, and budget. Rank the remaining small set using evidence-backed performance and optional semantic similarity. Select 0..K, not exactly K.

### P0 durability-1: work lifecycle has two authorities

**Observed.** The router maintains an in-memory `works` map (`internal/network/router.go:202-216`) and validates lifecycle transitions against it (`internal/network/router.go:1066-1102`). SQLite independently persists `network_work` state in `internal/store/globaldb/global_db_network_work_mutation.go`. The router does not hydrate the map on restart, and terminal entries are not visibly pruned.

**Effect.** After restart, a valid durable receipt/trace can be rejected or ignored as missing before the store can apply it. Memory can grow with terminal work IDs.

**Required change.** Make the durable store transaction the sole work-state authority, with an optional bounded coherent cache.

### P0 durability-2: persist-before-publish violates the documented retry contract

**Observed.** `Manager.Send` writes the conversation message before publishing (`internal/network/manager.go:680-712`). If publication fails, retrying the same ID sees the persisted duplicate and returns success without republishing (`internal/network/manager.go:699-704`). The official skill recommends same-ID idempotent retries (`skills/agh/references/network.md:137-140`).

**Effect.** A durable row can claim success while no recipient ever received the message.

**Required change.** A later durability TechSpec should introduce a transactional outbox with pending/published/terminal-failure states and durable per-target delivery identity. A duplicate pending ID resumes publication.

### P0 durability-3: the first local thread send can select no receiver

**Observed.** `Manager.Send` persists before publication (`internal/network/manager.go:680-712`). Conversation persistence immediately records the sender as a thread participant (`internal/store/globaldb/global_db_network_conversations.go:71-150,152-168`). Routing then sees a non-empty participant set, excludes the sender, and skips the empty-thread activation path (`internal/network/router.go:1241-1265`). A first undirected local send can therefore have no local delivery target.

**Effect.** AGH can pay persistence, broker, audit, and routing cost while delivering nothing. A later message from multiple established senders can instead expand the sticky full-delivery set. The same participant mutation is producing both under-delivery and later amplification.

**Required change.** Persist a routing/admission decision based on the pre-envelope participant snapshot, or make the outbox dispatcher derive recipients from an immutable accepted-send snapshot. Add a manager/store/router integration test; handler tests backed by a stub service cannot prove this path.

### P0 durability-4: the proposed offline mailbox is not addressable under peer-id v0

**Observed.** The v0 envelope persists peer-shaped `From`, `To`, and `Mentions`
(`internal/network/envelope.go:196-218`). `Router.PrepareSend` rejects a directed target without live
presence (`internal/network/router.go:354-379`), and direct-room identity hashes the two peer ids
(`internal/network/router.go:995-1027`; `internal/network/validate.go:212-241`). Thread participants,
subscription preferences, direct-room rows, and work ownership are also peer-keyed
(`internal/store/globaldb/global_db.go:163-176,349-430`).

**Effect.** A process peer id cannot rotate on restart and simultaneously remain the stable offline
recipient. An addressable mailbox split that changes only the participation snapshot would still
reject offline direct sends, change direct-room identity, and orphan durable mentions/subscriptions/
work relations after lease rotation.

**Required change.** P0-C must hard-cut the wire/storage/routing identity to stable mailbox ids for
sender, directed recipient, resolved mentions, direct pairs, thread participants, subscription
policy, work ownership/targeting, delivery keys, and usage. Durable mailbox validation replaces the
presence gate. Only after mailbox acceptance does a fenced `(mailbox_id -> live lease)` lookup decide
immediate delivery/activation. Peer/lease ids remain optional presence/transport observations. For a
remote home runtime, delivery requires its durable mailbox acknowledgement; publish alone is not
delivery.

### P1-7: delivery retry and queue policy are unbounded or cost-blind

**Observed.** Delivery workers are serialized per target but can run for many target sessions concurrently (`internal/network/delivery.go:285-348,616-634`). Retry uses capped exponential delay but no maximum attempts/age/dead-letter state (`internal/network/delivery.go:783-824,953-1001`). Queued expiry is not rechecked immediately before prompting. Overflow drops the oldest item regardless of priority (`internal/network/delivery.go:1117-1153`).

**Required change.** Add global/workspace/session activation semaphores, priority and expiry checks at admission and execution, maximum retry count/age, circuit breaking, dead-letter inspection, and deterministic coalescing. No control event belongs in a cognition queue.

### P1-8: discovery can create N responses, N flushes, and N model turns

**Observed.** Broadcast whois gathers every match and builds one response per peer (`internal/network/router.go:846-897`). Missing requested IDs can project a complete capability catalog (`internal/network/capability_catalog.go:62-162`). Generated responses publish serially, and whois responses can be model-delivered.

**Required change.** Local discovery queries the daemon registry directly and returns one ranked/paginated result. Remote discovery uses cached catalog digests and fetch-by-reference. Discovery never activates a recipient model.

### P1-9: synchronous persistence and audit work scales with fan-out

**Observed.** Conversation writes transact timeline/container/work/participant/summary/audit state (`internal/store/globaldb/global_db_network_conversations.go:71-149`). Thread and direct summary refreshes re-count history, participants, and work (`internal/store/globaldb/global_db_network_conversation_summary.go:74-200`). Delivery audit then writes another audit path (`internal/network/manager.go:1537-1553`); the file writer marshals, opens, writes, syncs, and closes for individual records (`internal/network/audit.go:162-230,357-386`).

**Required change.** Maintain counters incrementally, store one authoritative lifecycle record, keep per-recipient delivery state separately, and batch/asynchronously export file audit data. Hooks and noncritical projection invalidation should run after commit, outside the ingress callback.

### P1-10: hooks and UI observation amplify work

**Observed.** One logical conversation write can emit multiple lifecycle hooks (`internal/network/hooks.go:55-85,220-243`). Loop watch observers can turn them into coordinator wakes (`internal/daemon/loop_watch_events_observer.go:173-213,250-309`). The Network UI polls status, lists, messages, and work at different intervals (`web/src/systems/network/lib/query-options.ts:25-32,127-226`) because Network HTTP has no workspace stream (`internal/api/httpapi/routes.go:382-407`).

**Required change.** Emit one committed event bundle with monotonic workspace sequence, deduplicate loop wakes by root/run/node, and provide projection-only SSE with `Last-Event-ID`. UI observation must never imply agent activation.

### P1-11: native tool projection is broader than participation intent

**Observed.** The coordination toolset contains 18 Network tools (`internal/tools/builtin/toolsets.go:25-46`). `network_send` exposes greet, whois, receipt, and trace to models (`internal/tools/builtin/network.go:308-331`). Availability is based on daemon Network dependencies rather than participation (`internal/daemon/native_tools.go:528-539,660-737`). Hosted MCP can project the callable catalog (`internal/mcp/hosted.go:443-461`; `internal/mcp/hosted_proxy.go:138-175,199-224`).

**Effect.** Exact model token impact depends on harness/provider behavior, but catalog width, schema serialization, discovery clutter, and tool-selection risk are real.

**Required change.** Project only bootstrap discovery by default. Network tools are available only to participating sessions and are further narrowed by mode. Runtime owns greet/whois/receipt/trace; models receive a small reply/request tool with server-filled context.

### P0 security boundary: future federation cannot trust the current proof path

**Observed.** `Proof` is an opaque map (`internal/network/envelope.go:190-217`). Advertising privileged `task.write` requires only a nonempty map (`internal/network/validate.go:609-620`). Subscription callbacks discard the NATS subject, so the manager cannot bind the subscribed workspace/channel to the envelope (`internal/network/manager.go:1655-1661,1712-1719`). Task ingress trusts a presence card containing `task.write` (`internal/network/tasks.go:345-388`).

**Current mitigating fact.** The embedded server is loopback-only with a private random token. That limits current exposure but is not a federation trust model.

**Required change before federation.** Bind authenticated transport principal, persistent
installation-scoped runtime id, current endpoint epoch, subject workspace/channel, stable mailbox
sender/recipient, payload digest, issuer/audience, and expiry. Recipient-home accepted/rejected
dispositions must be immutable and correlated to runtime/mailbox/message/route plus their historical
commit endpoint epoch. Each presentation separately binds the authoritative current serving endpoint
epoch and re-attests the canonical disposition digest; a superseded endpoint, stale directory result,
or forged ACK/NACK cannot settle sender outbox. Enforce per-principal rate and activation budgets
before persistence/fan-out. Do not expose task ingress to federation before this proof exists.

## Protocol contract gaps

The v0 kinds are `greet`, `whois`, `say`, `capability`, `receipt`, and `trace` (`internal/network/envelope.go:13-38`). Conversation messages require exactly one of the only two surfaces, thread or direct (`internal/network/envelope.go:40-61`; `internal/network/validate.go:271-330`). Discovery messages omit conversation containers.

The protocol/runtime contract has no first-class fields for:

- monotonic stream sequence or resume cursor;
- priority and delivery class;
- response required/no reply;
- maximum responses and maximum hops;
- root interaction budget;
- quorum or aggregation policy;
- batch delivery;
- content-addressed context/artifact reference;
- state version/delta or summary checkpoint;
- cancellation/terminal reason;
- persistent home runtime identity and fenced endpoint registration;
- idempotent recipient-home accepted/terminal-rejected disposition and tombstone ordering;
- authoritative home expiry disposition versus sender-local unconfirmed dead-letter state;
- swarm invocation/worker/result identity.

Not every item must become a public wire field. Sender-provided activation hints cannot be trusted as authority because they can create a denial-of-wallet fan-out. The recipient/runtime policy must own activation and budgets. Some fields belong in an internal delivery record rather than the interoperable envelope.

## Optimization program

### Tier 0: avoid Network work entirely

1. Default sessions, tasks, runs, loops, automation, spawned work, and reviews to resolved Network participation `off`.
2. Do not create a channel for a local task run.
3. Do not bind local task/loop/judge/reviewer sessions to a channel.
4. Do not join peers, create subscriptions, start heartbeat, set peer environment variables, or inject Network prompts/tools in `off` mode.
5. Keep Network administrative availability separate from participation and lazy-start transport.

These changes remove the complete Network cost rather than making it cheaper.

### Tier 1: cap model calls before optimizing prompt bytes

1. Separate durable mailbox delivery from activation.
2. Handle presence, discovery, receipt, trace, task status, and work state deterministically.
3. Make non-selected peers store-only recipients.
4. Select one responder by default; support explicit expansion to 0..K.
5. Add root call/token/cost/time/response/hop budgets.
6. Enforce global/workspace/session concurrency limits.
7. Apply activation policy on every message, not only an empty thread.
8. Make group chat an explicit expensive mode with agent/turn caps.

### Tier 2: minimize each remaining prompt

1. Use one stable Network contract at startup only for participating sessions.
2. Delete per-delivery CLI examples and repeated protocol tutorials.
3. Replace routing flags with a runtime-resolved `reply_handle`.
4. Deliver bounded JSON/content parts, not XML-wrapped base64.
5. Pass only decision-relevant metadata.
6. Use typed concise responses such as `{decision,evidence_refs,risks,confidence,need_more}`.
7. Default advisory output to roughly 300-600 tokens, with explicit expansion.
8. Use versioned artifact pointers, rolling summaries, last-N relevant events, and elision markers.
9. Preserve stable prompt prefixes and inject transient advice at the tail for cache reuse.
10. Dynamically narrow toolsets by participation, role, and interaction phase.

### Tier 3: make fan-out swarm-shaped

1. Gate whether a task needs more than one model before selecting agents.
2. Use typed capability/schema/permission/cost/load filtering.
3. Keep workers isolated from peer conversation by default.
4. Give each worker an immutable brief, artifact references, and independent budget.
5. Run workers in parallel with deadlines and partial-failure isolation.
6. Stop on quorum, sufficient diversity/confidence, deadline, or budget.
7. Give results once to the acting aggregator.
8. Capture worker final output directly; do not require a Network reply tool.
9. Refresh fan-out only on material new evidence or a milestone, not every tool iteration/status event.
10. Persist one root trace with actual per-model usage.

### Tier 4: finish P0-C durability before public non-off, then optimize runtime

1. Transactional outbox and durable per-target delivery ledger.
2. Single durable work authority.
3. Correct pre-message recipient snapshot for first local send.
4. Network v1 stable mailbox addressing for directs, mentions, participants, subscriptions, work,
   delivery, and usage; no durable peer-id aliases.
5. Offline mailbox identity/cursor recovery plus fenced mailbox-to-live-lease resolution.
6. Persistent authenticated home runtime identity plus fenced endpoint re-registration across daemon
   restart.
7. Recipient-home immutable idempotent accepted/terminal-rejected disposition, current-endpoint
   re-attestation across restart, home-ordered tombstone race, and authenticated sender outbox
   settlement.
8. One authenticated immutable remote-envelope deadline, home `envelope_expired` on late arrival,
   and sender-local `delivery_dead_lettered_unconfirmed` for unreachable home without forged
   recipient authority or late acceptance.
9. Select in-process local transport versus retained local NATS through fault/profile evidence.
10. One daemon/channel presence lease with deltas and capability digests.
11. Real cross-sender digest accumulation and incremental conversation summary counters.
12. Batched remote publish/flush and asynchronous audit export where correctness permits.
13. Workspace SSE instead of polling-only projections.

## Mixture-of-Agents target for AGH

A lower-coordination-overhead MoA/swarm candidate should not be implemented as agents chatting in a
channel. It should be a bounded orchestration primitive whose total cost and quality are benchmarked:

1. Resolve one immutable, receiver-minimized context snapshot.
2. Select 0..K relevant reference workers, with K explicitly bounded.
3. Use stateless/no-tool reference calls when tools are unnecessary; otherwise use isolated delegated workers with narrow tools.
4. Run them concurrently with independent input/output/call/time/cost budgets.
5. Keep references isolated from each other's messages.
6. Capture structured results and artifact references.
7. Deduplicate/rank evidence and record absences/timeouts structurally.
8. Stop early on sufficient evidence/quorum/deadline.
9. Give the compact result set once to the existing acting model.
10. Project progress/results into a thread for UI/audit only if the user opted into a Network projection.

This produces approximately K reference calls plus the acting model's existing iterations. It avoids the current feedback topology in which every response can become input to every participant.

## Required telemetry

Add a root-correlated trace with spans for:

- context snapshot;
- route/gate decision;
- mailbox directory resolution and runtime endpoint epoch;
- transport publish and flush;
- recipient-home accepted/rejected disposition and tombstone/route-refresh reason;
- ingress parse/validation;
- persistence and audit;
- mailbox admission;
- activation decision;
- queue wait;
- session cold start/reuse;
- ACP/model turn;
- tool calls;
- response/result materialization;
- aggregation and terminal reason.

Record:

- actual input/output/thought/cache/context tokens and monetary cost;
- model calls per root interaction;
- selected versus available peers;
- full/mailbox/suppressed delivery counts;
- response amplification and maximum hop;
- useful-agent utilization;
- duplicate evidence/message ratio;
- no-op/silent model turns;
- prompt cache hits and stable-prefix reuse;
- cold/warm start latency;
- queue, model, publish/flush, DB, and file-sync p50/p95;
- retry, expiry, dead-letter, overflow, and circuit-breaker outcomes;
- heartbeat bytes/audit rows;
- hook wake counts;
- CPU and heap.

Do not put high-cardinality workspace/session/user values into metric labels. Structural traces can retain IDs under normal access controls; full prompt/content capture must be explicit, redacted before storage, sampled, and retention-limited.

## Benchmark matrix

Run each relevant scenario at N = 1, 3, 10, and 50 participants:

| Scenario | What it proves |
| --- | --- |
| Five minutes idle | heartbeat, subscriptions, audit rows, CPU, heap, zero model calls |
| Channel broadcast | selected activations, store-only recipients, fan-out bound |
| First thread message | routing correctness before participants exist |
| Established thread | participant TTL/roles and per-turn activation gate |
| Direct request/reply | one-recipient call bound and response capture |
| Broadcast whois | zero cognition and catalog aggregation |
| Work lifecycle | deterministic receipt/trace/state transitions |
| 100-message multi-sender digest | actual debounce/coalescing and token savings |
| Busy target | priority, expiry, backpressure, bounded queue |
| Permanent render/prompt failure | retry cap, dead-letter, error accounting |
| Reconnect storm | jittered presence and bounded burst |
| Restart with pending publish | outbox/delivery recovery and dedupe |
| MoA K = 1, 3, 5 | K+actor call bound, quorum, straggler behavior |

Hard acceptance budgets:

- `off`: zero Network rows, peers, subscriptions, heartbeats, startup sections, projected Network tools, deliveries, and model turns attributable to Network;
- presence/discovery/receipt/trace/task status: zero model turns;
- direct request: at most one recipient turn unless explicitly continued;
- routed request: one selected responder by default and never more than the configured K;
- mailbox: zero event-driven model turns;
- MoA: no more than K reference calls plus the acting model's declared iterations;
- expired delivery: zero model turns;
- retries, queues, dedupe, work cache, and metrics maps: bounded;
- restart: pending delivery resumes without duplicate model execution;
- causal cycles: rejected at configured hop/response limit.

## Structural decomposition required before implementation

Current production files exceed the repository's 500-line limit:

| File | Lines at snapshot |
| --- | ---: |
| `internal/network/manager.go` | 2,095 |
| `internal/network/delivery.go` | 2,012 |
| `internal/network/router.go` | 1,787 |
| `internal/network/validate.go` | 1,085 |
| `internal/network/peer.go` | 919 |
| `internal/network/tasks.go` | 613 |

The redesign should split responsibilities before adding behavior:

- protocol contracts and validation;
- local transport and federation adapter;
- presence coordinator;
- ingress pipeline;
- outbox/inbox and delivery ledger;
- activation policy and budget admission;
- delivery scheduler;
- prompt renderer;
- durable work lifecycle repository;
- participant/subscription policy;
- metrics/tracing;
- swarm/MoA orchestration.

Growing the existing files would deepen the same responsibility conflation this audit identifies.

## AGH Impact Audit

- **Native tools:** Direct impact to every `agh__network_*` descriptor, `agh__coordination`, tool availability diagnostics, capability gates, schema digests, and CLI/API fallbacks. Control kinds should become runtime-owned. Network tools must be projected only for participating modes.
- **Extensibility and hooks:** Direct impact to Network hooks, loop watch observers, capabilities/catalogs, routing-policy registries, bundles, MCP-hosted projection, future bridge/federation adapters, and config lifecycle. Hook observation must not imply model activation.
- **Workspace data isolation:** New root traces, budgets, inbox/outbox rows, cursors, artifacts, summaries, delivery ledgers, SSE, caches, and metrics must be workspace-scoped. Authenticated principal, subject, envelope workspace/channel, and sender must agree before persistence. The separate enforcement audit documents a current unqualified channel lookup that can cross workspace metadata.
- **Official AGH skill:** `skills/agh/references/network.md`, `tasks-and-orchestration.md`, and `native-tools.md` must change with the public behavior: opt-in participation, mailbox semantics, automatic control handling, reply handles/result capture, budgets, retry/outbox behavior, and removal of repeated manual protocol tutorials.

## Conclusion

Prompt compression alone will not solve AGH Network cost. The ordering of work matters:

1. avoid Network for local execution;
2. decide activation before inference;
3. cap model calls and interaction depth;
4. shrink the remaining prompts;
5. make fan-out use a bounded worker-to-aggregator topology;
6. then optimize broker, storage, presence, and federation based on measured profiles.

The existing Network can remain a valuable durable collaboration surface, but it must stop being both the event bus and the implicit model scheduler for every participating peer.
