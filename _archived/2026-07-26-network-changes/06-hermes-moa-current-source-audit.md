# Hermes Mixture-of-Agents Current-Source Audit

## Document control

- **Snapshot date:** 2026-07-10
- **Repository:** `.resources/hermes`
- **Inspected commit:** `b8880f124537acc5a6215718dd154eadc5af1515`
- **Remote identity:** `https://github.com/nousresearch/hermes-agent`
- **Scope:** current Mixture-of-Agents product surface, runtime path, prompt shaping, fan-out, caching, accounting, tracing, tests, and management UI
- **Authority:** this local snapshot, not the outdated Hermes notes under `second-brain/research/harness/hermes`
- **Method:** static end-to-end code tracing; no Hermes tests or live provider calls were run

All Hermes references in this document are repository-relative paths under `.resources/hermes` and line numbers from the inspected commit.

## Executive findings

Hermes MoA is useful to AGH primarily because it is **an explicitly selected execution mode**, not
an always-on peer network. It keeps one acting model in the normal tool loop and obtains parallel,
tool-free advice from reference models. Static inspection proves that this topology avoids durable
advisor sessions, Network tools, presence lifecycle, and worker-to-worker protocol traffic. It does
not prove lower total provider cost or higher quality; those depend on reference count, cadence,
history size, retries/fallbacks, models, and task-specific evaluation.

The implementation contains seven patterns worth adopting:

1. MoA is selected explicitly through a virtual provider/model preset or a one-shot `/moa` command.
2. References are raw advisory calls, not durable agent sessions.
3. References run concurrently and fail independently.
4. The advisory view removes the main system prompt and real tool schemas.
5. Each tool-result preview is bounded in the advisory view.
6. Reference usage and cost are attributed to configured advisor/aggregator slots in the
   no-fallback path and folded into acting-turn accounting.
7. Full traces are opt-in and separate from conversation history.

It also contains cost and contract risks AGH should not copy:

1. The default fan-out cadence is `per_iteration`, multiplying reference calls across tool iterations.
2. Reference output is uncapped by default.
3. References/model calls have no explicit aggregate MoA/root ceiling. Reference concurrency is
   capped at eight, while finite configured slots plus outer iteration/retry/timeout limits bound
   components indirectly.
4. Fan-in waits for every reference and has no root deadline, quorum, or early completion.
5. Every successful result is concatenated in full without ranking, deduplication, or a typed return schema.
6. A reference sees the entire flattened conversation, with each tool result independently allowed up to 4,000 characters.
7. No MoA root ledger spans actor, references, retries/fallbacks, tokens, cost, duration, and rounds;
   no marginal-value stopping policy exists.
8. Management API and UI schemas omit `reference_max_tokens` and `fanout`, so saving through those surfaces can erase both cost controls.
9. A legacy, separate synthesize-then-main-agent path remains alongside the virtual-provider path.
10. Benchmark numbers are documented but their raw benchmark evidence is not present in the inspected source tree.
11. Advisor projection drops non-string multimodal/structured content and replicates broad history
    across each executed reference provider without a MoA-specific egress policy.
12. Auxiliary fallback can change the actual provider/model while accounting retains the configured
    slot and can collapse intended advisor diversity.

The correct AGH lesson is: **copy the explicit mode boundary, stateless advisor topology, and
context-minimization intent; harden budgets, data policy, cancellation, actual-attempt accounting,
and cross-surface contracts before adopting it.**

## 1. Product and opt-in model

### 1.1 Virtual provider and named presets

Hermes exposes MoA as a virtual model provider. A named preset contains reference provider/model slots and one aggregator provider/model slot. Selecting a preset makes the aggregator the acting model; references run first and provide private context. Evidence:

- `.resources/hermes/website/docs/user-guide/features/mixture-of-agents.md:9-13`
- `.resources/hermes/website/docs/user-guide/features/mixture-of-agents.md:50-62`
- `.resources/hermes/hermes_cli/runtime_provider.py:1526-1536`
- `.resources/hermes/agent/agent_init.py:816-864`

This is genuine opt-in at the interaction/session boundary:

- normal models remain normal single-model sessions;
- a user can select a MoA preset persistently through the same model picker used for any provider;
- a user can invoke a single MoA turn with `/moa <prompt>`;
- `/moa` without a prompt is usage-only and does not switch the model.

Evidence:

- `.resources/hermes/website/docs/user-guide/features/mixture-of-agents.md:15-48`
- `.resources/hermes/apps/desktop/src/app/shell/model-menu-panel.tsx:108-129`
- `.resources/hermes/apps/desktop/src/app/shell/model-menu-panel.tsx:182-196`
- `.resources/hermes/apps/desktop/src/app/shell/model-menu-panel.tsx:325-349`
- `.resources/hermes/cli.py:8871-8906`
- `.resources/hermes/gateway/run.py:9951-9984`
- `.resources/hermes/tui_gateway/server.py:11907-11968`
- `.resources/hermes/tests/cli/test_moa_command.py:41-73`

### 1.2 One-shot and persistent choices are distinct

The product deliberately distinguishes:

| User intent | Hermes surface | Lifetime |
|---|---|---|
| Use multiple perspectives for this prompt | `/moa <prompt>` | Intended as one turn; gateway restore is finally-safe, CLI/TUI restore is not |
| Use MoA throughout this work | Select `moa:<preset>` in model picker or `/model ... --provider moa` | Session/model selection |
| Configure the topology | Dashboard, Desktop settings, `hermes moa`, or `config.yaml` | Persisted preset |

The gateway restores a one-shot override from a `finally`-owned path, including error/interruption cases. Evidence: `.resources/hermes/gateway/run.py:10329-10348` and `.resources/hermes/tests/gateway/test_moa_one_shot_restore.py:1-103`.

The direct CLI and TUI are not equivalent. CLI restoration occurs only after `run_conversation` returns inside the `try`, at `.resources/hermes/cli.py:12338-12358`. TUI restoration similarly follows the call rather than owning it in `finally`, at `.resources/hermes/tui_gateway/server.py:9031-9081`. A thrown exception can leave the live/model state on MoA for the next turn. Existing CLI tests cover command state but not exceptional restoration at `.resources/hermes/tests/cli/test_moa_command.py:41-82`.

AGH should use the same semantic separation:

- `off` is the default execution mode;
- an explicit run/session preset selects durable Network participation;
- a one-shot collaboration action does not silently mutate the session default;
- configuration defines available presets but does not enroll every run in them.

### 1.3 Configuration availability is not activation

Hermes ships a default MoA preset in global default configuration, but explicit-only model pickers hide the virtual provider unless the raw user config contains an enabled preset. Evidence:

- `.resources/hermes/hermes_cli/config.py:2287-2312`
- `.resources/hermes/hermes_cli/inventory.py:330-395`

This is a useful distinction for AGH:

```text
Network installed/enabled at daemon level != this execution participates
```

AGH's global `network.enabled` should remain an administrative capability/kill switch. It must not be the implicit per-session or per-run participation request.

### 1.4 `enabled` has narrower semantics than the name suggests

Within a Hermes preset, `enabled: false` skips reference fan-out but still lets the configured aggregator act. Explicitly selecting a disabled preset is therefore an aggregator-only mode, while implicit bare-name matching excludes disabled presets. Evidence:

- `.resources/hermes/agent/moa_loop.py:836-840`
- `.resources/hermes/hermes_cli/moa_config.py:213-233`
- `.resources/hermes/website/docs/user-guide/features/mixture-of-agents.md:167-173`
- `.resources/hermes/tests/hermes_cli/test_moa_config.py:126-155`

AGH should avoid an overloaded boolean like this. Participation, persistence, subscription, tool visibility, and model activation need separate resolved fields or tested presets over those fields.

Hermes also normalizes and edits `active_preset`, but execution selects a preset through provider/model or the default preset. Repository search found management/display reads, not an execution read. Definition: `.resources/hermes/hermes_cli/moa_config.py:178-196`; UI mutation: `.resources/hermes/web/src/pages/ModelsPage.tsx:762-775` and `.resources/hermes/apps/desktop/src/app/settings/model-settings.tsx:883-902`. AGH should not expose state whose runtime effect is undefined.

## 2. End-to-end runtime path

### 2.1 Provider selection installs a facade, not another agent graph

The virtual provider resolves to `moa://local`, and agent initialization installs `MoAClient` while forcing the outer API mode to `chat_completions`. The facade exposes an OpenAI-compatible `chat.completions.create` surface. Evidence:

- `.resources/hermes/hermes_cli/runtime_provider.py:1526-1536`
- `.resources/hermes/agent/agent_init.py:816-864`
- `.resources/hermes/agent/moa_loop.py:683-725`
- `.resources/hermes/agent/moa_loop.py:1041-1060`
- `.resources/hermes/tests/agent/test_moa_switch_api_mode.py:1-84`

The design lets the existing agent loop treat the aggregator response as the real model response. The aggregator can stream, call tools, receive tool results, and continue normally. Evidence:

- `.resources/hermes/agent/moa_loop.py:977-1038`
- `.resources/hermes/tests/run_agent/test_moa_streaming.py:1-140`

AGH analogue:

- keep one authoritative acting session/coordinator;
- implement swarm advice behind a bounded invocation interface;
- return structured results directly to that actor;
- do not require each reference to join a channel or send a protocol reply.

### 2.2 References are raw, tool-free model calls

Each reference slot resolves through the normal provider resolver, but `_run_reference` calls the model as an auxiliary request with no tool schemas. A dedicated reference system prompt tells the model that it is an advisor, cannot act, and must return private guidance. Evidence:

- `.resources/hermes/agent/moa_loop.py:93-118`
- `.resources/hermes/agent/moa_loop.py:126-173`
- `.resources/hermes/agent/moa_loop.py:220-279`
- `.resources/hermes/tests/run_agent/test_moa_loop_mode.py:395-486`

This avoids:

- session creation and persistence for advisors;
- presence/join/greeting traffic;
- per-advisor tool catalogs;
- tool-execution loops in references;
- worker-to-worker chat;
- a second protocol action to return the result.

AGH's current Network delivery path pays most of those costs. A swarm execution path should not be implemented as a collection of normal channel-bound sessions unless durability or remote execution explicitly requires them.

### 2.3 The acting aggregator retains tools

Hermes appends reference context to a copy of the aggregator's normal message array and calls the real aggregator runtime with the normal tools. The aggregator is both synthesizer and actor; there is no mandatory extra synthesis model followed by a separate main actor. Evidence:

- `.resources/hermes/agent/moa_loop.py:960-1022`
- `.resources/hermes/tests/run_agent/test_moa_loop_mode.py:15-71`
- `.resources/hermes/tests/run_agent/test_moa_loop_mode.py:424-489`

For `R` references and `I` acting/tool iterations, this main path costs:

```text
per_iteration fan-out: I * (R + 1) model calls
user_turn fan-out:     R + I model calls
single actor:          I model calls
```

The formula excludes retries, provider-internal work, and any other auxiliary models in the session.

## 3. Advisory context shaping

### 3.1 Main system prompt is removed

`_reference_messages` drops the normal Hermes system message. The source comment calls it roughly
`8K` of boilerplate without specifying bytes versus tokens. References receive only the dedicated
advisor system prompt plus a transformed conversation view. Evidence:
`.resources/hermes/agent/moa_loop.py:437-466`.

This is the strongest prompt optimization to carry into AGH. A Network/swarm worker does not need:

- the full AGH skill/tool manual;
- general Network protocol instructions;
- unrelated workspace/session context;
- the acting agent's full provider-specific permission/tool narrative;
- transport metadata or a full roster.

### 3.2 Tool schemas are removed and actions become text

Real `tool_calls` arrays are flattened into readable action lines. Tool-role results are folded into assistant text. This prevents strict providers from rejecting orphan tool calls and avoids sending every advisor the full tool schema. Evidence:

- `.resources/hermes/agent/moa_loop.py:402-426`
- `.resources/hermes/agent/moa_loop.py:437-522`

For AGH, a worker input should be a typed projection of the task state, not a replay of the actor's protocol transcript.

### 3.3 Tool results use head/tail truncation

Each tool result is limited to 4,000 characters in the advisory view, preserving the first and last halves with an omission marker. Evidence:

- `.resources/hermes/agent/moa_loop.py:83-91`
- `.resources/hermes/agent/moa_loop.py:388-399`
- `.resources/hermes/tests/run_agent/test_moa_loop_mode.py:358-377`

This protects against a single huge result, but it does not impose a total advisory-context budget. A long task can contain many tool results, each contributing up to 4,000 characters, while rendered tool arguments are not separately bounded at `.resources/hermes/agent/moa_loop.py:402-426`. AGH should use both:

1. per-artifact/result caps;
2. a total worker input-token budget with relevance selection.

### 3.4 The whole conversation remains eligible

The advisory view iterates across the full message list, retaining user and assistant text and truncated tool results. It does not select only the current task slice, last `N` turns, or relevant artifacts. Evidence: `.resources/hermes/agent/moa_loop.py:468-522`.

This is simpler but can become the dominant input cost for long-running sessions. AGH should pass:

```text
role contract
+ current task slice
+ selected facts/artifact summaries
+ exact requested output schema
+ budget/deadline
```

It should not pass the entire channel or session transcript by default. It must still carry a
compact safety/authority envelope; removing a large actor prompt is not permission to remove tenant,
privacy, tool-output-trust, or provider-egress policy.

### 3.5 Provider-shape normalization is robust

Hermes forces the advisory view to end with a user turn and emits only plain system/user/assistant text. This handles providers that reject assistant-prefill endings or orphan tool messages. Evidence: `.resources/hermes/agent/moa_loop.py:452-466` and `.resources/hermes/agent/moa_loop.py:505-522`.

AGH should similarly normalize worker inputs once at the provider adapter boundary instead of encoding provider quirks in Network prompts.

### 3.6 Non-string and multimodal content is silently lost

`_reference_messages` assigns content only when it is a string; arrays, image parts, and other
structured content become `""`. Non-string tool results are likewise truncated from the empty
string. The fallback search also accepts only string user content. Evidence:
`.resources/hermes/agent/moa_loop.py:468-521`.

The acting aggregator still receives its original multimodal messages, but advisors may judge an
empty or materially incomplete task without a diagnostic. AGH must define typed content projection:
preserve allowed text/image/document parts through provider adapters, replace excluded content with
an explicit typed omission, or reject an advisor configuration that cannot consume required input.
Tests need mixed text/image arrays, structured tool results, documents, and unsupported-provider
behavior.

### 3.7 Advisor fan-out is a data-egress multiplier

The projection drops every normal system message but sends the remaining broad user/assistant/tool
history to each valid reference slot. Different slots can resolve to different vendors. The MoA
path does not apply a separate tenant/workspace data-classification, provider allowlist, per-artifact
egress decision, or redaction layer before `_run_references_parallel`. Evidence:
`.resources/hermes/agent/moa_loop.py:437-522,844-905` and
`.resources/hermes/agent/moa_loop.py:126-173,336-385`.

AGH must treat each advisor as a new disclosure boundary. Admission includes workspace/tenant
policy, provider and region allowlists, secret/PII classification, redaction, artifact selection,
audit, and a denial reason. Removing the actor's full system prompt can reduce coordination input,
but AGH still needs a compact safety and authority envelope for every provider call.

### 3.8 Trust and visibility boundaries are inconsistent

Tool results are potentially untrusted text. Hermes folds them into advisor prompts, then appends raw
advisor output to the actor as a user-role message. Textual provenance exists: the advisor system
prompt labels the role, and the aggregate guidance labels references and says they are private
context. What is missing is a structured untrusted-data/authority boundary that provider adapters,
the actor, policy checks, and UI can enforce independently of prose. Evidence:
`.resources/hermes/agent/moa_loop.py:93-118,437-522,661-680,960-975`.

The advisor system prompt says its response is private and not shown to the user at
`.resources/hermes/agent/moa_loop.py:100-118`, but the runtime emits full advisor text to UI events.
TUI and Desktop render the full text; the direct CLI routes it through the reasoning-preview helper,
which wraps and shows five lines unless global verbose mode is enabled:

- CLI event and five-line/verbose rendering: `.resources/hermes/cli.py:10955-10983,5414-5447`.
- TUI relay: `.resources/hermes/tui_gateway/server.py:3688-3705`.
- TUI rendering even when reasoning display is off: `.resources/hermes/ui-tui/src/app/turnController.ts:693-723`.
- Desktop: `.resources/hermes/apps/desktop/src/app/session/hooks/use-message-stream/gateway-event.ts:283-305`.

The docs also say references receive user/assistant text rather than the tool-call transcript at `.resources/hermes/website/docs/user-guide/features/mixture-of-agents.md:52-60,160-165`, while code explicitly flattens tool calls and results at `.resources/hermes/agent/moa_loop.py:437-466`.

AGH should carry trust/provenance labels in typed worker artifacts, render them below system/user authority, and make advisory visibility a truthful policy. Concise status plus expandable evidence is safer than always streaming full essays.

## 4. Fan-out, failure, and latency behavior

### 4.1 Parallel references with stable output order

Hermes submits reference calls to a `ThreadPoolExecutor`, with at most eight concurrent workers, and returns results in configured slot order. A recursive MoA reference is skipped. Evidence: `.resources/hermes/agent/moa_loop.py:21-27` and `.resources/hermes/agent/moa_loop.py:336-385`.

The cap is only a concurrency cap. A preset can contain more than eight references; all are submitted and eventually run. There is no `max_references` validation in `.resources/hermes/hermes_cli/moa_config.py:108-148`.

AGH should have separate limits:

- `max_workers`: total invocations admitted;
- `max_concurrency`: simultaneously executing invocations;
- `max_model_calls`: root-call ceiling including actor/synthesis;
- `max_total_tokens` and/or `max_cost`;
- `deadline`.

### 4.2 Partial reference failure does not abort the turn

`_run_reference` catches an exception and returns a labeled failure string. The aggregator continues with other results. Evidence: `.resources/hermes/agent/moa_loop.py:246-249` and `.resources/hermes/agent/moa_loop.py:311-333`.

Failure isolation is correct, but including raw exception text in aggregator context is unnecessary token and disclosure risk. AGH should keep structured failure diagnostics outside the model prompt unless failure knowledge changes the requested decision.

### 4.3 Fan-in waits for all references

The executor scope waits for every future, and the collection loop calls `future.result()` for every submitted reference. There is no `as_completed` quorum, root deadline, early stop, or cancellation of stragglers. Evidence: `.resources/hermes/agent/moa_loop.py:359-385`.

This makes wall time approximately:

```text
slowest admitted reference + aggregator/tool-loop time
```

The slowest-reference risk is large in the inspected defaults. Auxiliary calls default to a 900-second timeout at `.resources/hermes/hermes_cli/config.py:1731-1745`; timeout resolution reaches MoA calls through `.resources/hermes/agent/auxiliary_client.py:6040-6068,6505-6521`. Transient retries default to two retries, or three total attempts, at `.resources/hermes/hermes_cli/config.py:1569-1576`, with retry/backoff at `.resources/hermes/agent/auxiliary_client.py:6542-6601` and possible provider/model fallback at `:6905-6960`. In a worst static path, one advisor can occupy roughly 45 minutes before additional fallback effects, and wait-all transfers that latency to the whole turn.

No cancellation token flows through `moa_loop.py`. `MoAClient` exposes accounting/trace accessors but
does not own or close the cached auxiliary clients used by its thread-pool calls
(`.resources/hermes/agent/moa_loop.py:1041-1073`). The main streaming interrupt machinery owns the
outer visible request path at `.resources/hermes/agent/chat_completion_helpers.py:1997-2036,2929-3015`.
Static inference: interrupting the visible turn or abandoning a future does not guarantee that an
in-flight advisor or aggregator provider request stops; it may continue until its provider timeout.

AGH should support deterministic completion policies such as:

- first `K` successful role results;
- required-role coverage;
- quorum reached;
- deadline reached;
- remaining workers canceled or ignored;
- late results stored as artifacts without reactivating the actor.

### 4.4 Fallback can erase intended model diversity

Each advisor slot is configured as a distinct provider/model, but auxiliary failure recovery may use
a configured chain, the main fallback chain, built-in discovery, or the main acting model
(`.resources/hermes/agent/auxiliary_client.py:6880-6952`). Two failing advisor slots can therefore
collapse onto the same fallback model, or an advisor can collapse onto the aggregator/main model,
while labels and accounting continue to describe configured slots.

AGH should resolve and reserve an explicit attempt plan. A fallback must satisfy role diversity,
provider-egress, model, price, and data-classification constraints; otherwise record a missing role.
The result carries both configured role/slot and every actual attempt identity.

### 4.5 No quality- or redundancy-aware admission

For an explicitly selected enabled preset, every valid non-recursive reference slot is submitted;
calls beyond eight queue. Disabled presets submit zero references, recursive MoA slots are skipped,
invalid slots may be normalized away, and caught failures still become labeled strings that are
concatenated for the actor. There is no deterministic relevance router, duplicate detection,
confidence threshold, or marginal-value stop. Evidence:
`.resources/hermes/agent/moa_loop.py:126-173,245-249,336-385,836-975` and
`.resources/hermes/hermes_cli/moa_config.py:108-148`.

AGH should start with deterministic role selection and result deduplication. Learned topology/routing should wait until root traces label which worker results changed the final outcome.

## 5. Fan-out cadence and cache behavior

### 5.1 Expensive default: `per_iteration`

Hermes accepts `per_iteration` and `user_turn`, but unknown or absent values normalize to `per_iteration`. Default presets are enabled, uncapped, and per-iteration. Evidence:

- `.resources/hermes/hermes_cli/moa_config.py:70-74`
- `.resources/hermes/hermes_cli/moa_config.py:93-105`
- `.resources/hermes/hermes_cli/moa_config.py:125-148`

In `per_iteration`, a new tool result changes the advisory view signature and reruns every reference before the next aggregator call. Evidence:

- `.resources/hermes/agent/moa_loop.py:697-708`
- `.resources/hermes/agent/moa_loop.py:844-905`
- `.resources/hermes/tests/run_agent/test_moa_loop_mode.py:634-677`

This is exactly the amplification AGH must avoid: one useful tool call can trigger another full fan-out before the actor proceeds.

### 5.2 Lower-call alternative: `user_turn`

In `user_turn`, Hermes hashes the advisory prefix only through the last real user message. Later tool iterations reuse the cached reference outputs. Evidence: `.resources/hermes/agent/moa_loop.py:847-885`.

Call-count savings for an example with `R=3`, `I=6`:

```text
per_iteration: 6 * (3 + 1) = 24 model calls
user_turn:      3 + 6       = 9 model calls
reduction:      15 calls (62.5%) before retries
```

AGH's default bounded swarm should be once per root/user turn or once per explicit phase, never once per arbitrary transport delivery or tool iteration.

### 5.3 Cache is process/facade-local

Hermes stores one cache key and output set on `MoAChatCompletions`. The key includes preset name,
advisory digest, and reference labels. It omits live-config fields that can change without rebuilding
the facade, including temperatures, reference output cap, cadence, and resolved provider runtime; a
live config change can therefore reuse stale advice. Prompt/schema version omission is future
durable-cache hardening rather than an ordinary hot-code-update defect, because a code update normally
restarts the process. The cache prevents duplicate work inside one live facade but is neither durable
nor shared. Evidence: `.resources/hermes/agent/moa_loop.py:697-725,844-907`.

AGH can use a root-scoped cache, but its key must include `workspace_id`, root activation, input/artifact digest, role, resolved model/provider/runtime, full policy digest, prompt/schema version, and trust projection. It must never be channel-only or global.

### 5.4 Prompt cache control is provider-aware

Hermes applies provider-aware cache controls to advisor calls and both aggregator paths. Advisor and
legacy synthesis decoration lives in `moa_loop.py`; the acting aggregator also resolves the virtual
MoA identity to its configured aggregator slot when selecting the normal agent cache policy. Source
comments record a prior run with zero cache reads across 1,227 advisor calls and 11.5M rebilled input
tokens before explicit Anthropic cache control. Evidence:

- `.resources/hermes/agent/moa_loop.py:176-217`
- `.resources/hermes/agent/moa_loop.py:261-272`
- `.resources/hermes/agent/agent_runtime_helpers.py:1524-1562`
- `.resources/hermes/agent/agent_init.py:600-608`
- `.resources/hermes/agent/conversation_loop.py:888-899`
- `.resources/hermes/tests/agent/test_moa_aggregator_cache_control.py:1-114`

The named test exercises the legacy synthesis call's cache-control decoration. No dedicated test was
found that proves the full virtual-provider acting-aggregator policy resolution above. The
measurements in comments are useful signals but not independently reproducible from a benchmark
artifact in the inspected tree.

### 5.5 Reference guidance is appended at the prompt tail

Hermes appends changing reference context after the stable task/tool-history prefix to avoid invalidating earlier cacheable bytes. Evidence: `.resources/hermes/agent/moa_loop.py:661-680` and `.resources/hermes/tests/run_agent/test_moa_loop_mode.py:1021-1061`.

This is correct for changing per-iteration advice. For `user_turn` advice, AGH can do better: place the stable artifact before subsequent tool-result turns so the advice itself becomes part of a reusable prefix. The renderer should choose placement based on whether the artifact is immutable for the root phase.

## 6. Aggregation behavior

### 6.1 Simple concatenation

The main path formats every reference output with its model label and concatenates all outputs in preset order. A short instruction tells the aggregator to use them as private context. Evidence: `.resources/hermes/agent/moa_loop.py:960-975`.

There is no:

- typed result schema;
- claim/evidence separation;
- relevance score;
- deduplication;
- contradiction map;
- total joined-output budget;
- selection of only the best `K` results.

AGH should require typed worker results and build a deterministic compact fan-in record before the actor sees them.

### 6.2 Acting aggregator avoids an extra synthesis call

The primary virtual-provider path lets the aggregator synthesize and act in one call, avoiding a
mandatory extra synthesis-to-actor call. End-to-end quality/cost still depends on which models and
caps are selected. Evidence: `.resources/hermes/agent/moa_loop.py:967-1022`.

AGH should default to `synthesis=acting_agent`. A separate synthesis call is justified only when:

- the actor cannot consume the result schema;
- a cheaper deterministic/LLM reducer materially shrinks a very large fan-in;
- quality benchmarks justify the additional call.

### 6.3 A legacy three-stage path still exists

`aggregate_moa_context` runs references, calls a separate synthesis aggregator, then returns guidance to the normal main agent. `run_conversation` can invoke that path through an encoded/explicit `moa_config`. Evidence:

- `.resources/hermes/agent/moa_loop.py:571-658`
- `.resources/hermes/agent/conversation_loop.py:523-566`
- `.resources/hermes/agent/conversation_loop.py:858-879`
- `.resources/hermes/hermes_cli/moa_config.py:245-272`

That block executes inside the outer conversation loop at
`.resources/hermes/agent/conversation_loop.py:643-670,858-879`. Without retries, `I` main-agent
iterations therefore cost `I * (R + 2)`: `R` references, one synthesis call, and one normal acting
call on every iteration. The `reference_max_tokens` value is passed as `max_tokens` to
`aggregate_moa_context`, where it caps both references and the legacy synthesis call
(`.resources/hermes/agent/conversation_loop.py:862-870`;
`.resources/hermes/agent/moa_loop.py:571-640`). Tests describe it as the one-shot synthesis path even
though current `/moa` surfaces switch to the virtual provider. Evidence:
`.resources/hermes/tests/agent/test_moa_aggregator_cache_control.py:1-13`.

The separate normalized preset field `max_tokens` appears in configuration and management schemas,
but repository search found no current MoA execution read. The virtual-provider actor uses the
caller's acting cap, while the legacy path reads `reference_max_tokens`. Evidence:
`.resources/hermes/hermes_cli/moa_config.py:93-148`,
`.resources/hermes/hermes_cli/web_server.py:973-995,5505-5541`, and
`.resources/hermes/agent/moa_loop.py:987-1018`.

AGH is greenfield. It should ship one orchestration path and delete replaced paths rather than preserving encoded compatibility markers or dual aggregation implementations.

## 7. Output caps and budgets

### 7.1 Advisors are uncapped by default

`reference_max_tokens` normalizes to `None` when absent, invalid, zero, or negative. `None` omits the provider parameter and lets each advisor use its own maximum. Evidence:

- `.resources/hermes/hermes_cli/moa_config.py:52-67`
- `.resources/hermes/hermes_cli/moa_config.py:125-147`
- `.resources/hermes/tests/hermes_cli/test_moa_config.py:252-283`

The docs recommend `600` as an example and state that advisor length dominates latency. Evidence: `.resources/hermes/website/docs/user-guide/features/mixture-of-agents.md:103-133`.

AGH should invert this default:

- every worker role has a finite output cap;
- invalid or omitted public caps resolve to a conservative server default, not unlimited;
- privileged explicit configuration is required for uncapped output;
- the root budget remains authoritative even when individual caps are higher.

### 7.2 Aggregator output follows the acting call

The current provider path intentionally does not apply the preset's historical `max_tokens=4096` to the acting aggregator; it forwards the caller's value, normally `None`. Evidence: `.resources/hermes/agent/moa_loop.py:987-1018`.

This avoids truncating a user-visible result, but it also means the normalized preset's `max_tokens` field is not the effective acting cap on the main path. The field remains in config and management schemas. Evidence: `.resources/hermes/hermes_cli/moa_config.py:93-105`, `.resources/hermes/hermes_cli/moa_config.py:125-148`, and `.resources/hermes/hermes_cli/web_server.py:973-995`.

AGH contracts should not expose fields whose meaning differs by hidden execution path.

### 7.3 No root budget

The outer Hermes actor has `max_iterations` plus an `IterationBudget`, and auxiliary calls have
finite configurable retry/timeout behavior
(`.resources/hermes/agent/agent_init.py:271,385-388`;
`.resources/hermes/agent/conversation_loop.py:643-670`;
`.resources/hermes/hermes_cli/config.py:1569-1576,1731-1745`). Those are useful component limits.
What the inspected MoA preset lacks is one aggregate root ledger spanning actor, all references,
legacy synthesis, retries, fallback attempts, input/output tokens, monetary cost, duration, and MoA
round/depth. Reference output caps and outer iteration count alone cannot stop input-history growth,
repeated fan-out, or cross-actor overspend.

AGH needs one admission ledger for the root activation:

```text
max_model_calls
max_input_tokens
max_output_tokens
max_reasoning_tokens
max_cost
deadline
max_workers
max_concurrency
max_rounds/depth
```

Each invocation reserves budget before provider activation and settles actual usage afterward.

## 8. Usage, cost, tracing, and UI

### 8.1 Configured-slot accounting is explicit in the no-fallback path

Hermes normalizes each advisor's response usage using the configured slot's resolved provider/API
mode, estimates cost against that configured slot, and sums advisor token buckets/dollar costs once
per cache miss. When no fallback occurs, this is explicit per-slot accounting. Evidence:

- `.resources/hermes/agent/moa_loop.py:238-244`
- `.resources/hermes/agent/moa_loop.py:280-323`
- `.resources/hermes/agent/moa_loop.py:908-925`
- `.resources/hermes/tests/run_agent/test_moa_loop_mode.py:772-875`

The configured identity is not always the executed identity. Auxiliary fallback can change
provider/model at `.resources/hermes/agent/auxiliary_client.py:6905-6952`, while `_RefAccounting`
normalizes, labels, and prices against the configured runtime/slot at
`.resources/hermes/agent/moa_loop.py:280-323`. Retry attempts are not individually recorded, and
missing advisor prices are omitted without a root completeness flag. AGH accounting must record
every actual attempt, retry, resolved fallback identity, usage, price status, and completion outcome.

### 8.2 Configured aggregator slot repairs the virtual identity only on the no-fallback path

Because `agent.provider=moa` and `agent.model=<preset>` are virtual identities, Hermes stores a copy
of the configured aggregator slot in `last_aggregator_slot` and uses it for pricing. This repairs the
virtual-identity hole when that slot actually executes. Source tests describe a prior roughly 50%
cost undercount when it was missing. Evidence:

- `.resources/hermes/agent/moa_loop.py:804-814`
- `.resources/hermes/agent/conversation_loop.py:2049-2178`
- `.resources/hermes/tests/agent/test_moa_aggregator_cost_slot.py:1-100`

The acting aggregator also calls the auxiliary client, whose fallback chain can execute a different
provider/model (`.resources/hermes/agent/moa_loop.py:1013-1022`;
`.resources/hermes/agent/auxiliary_client.py:6880-6952`). `last_aggregator_slot` remains the configured
dictionary, so fallback usage may be normalized/priced under the wrong identity. AGH should separate
orchestration identity, configured slot, and actual billing attempts, then aggregate all attempts by
root activation.

Hermes' session call counter still increments once for the acting response at `.resources/hermes/agent/conversation_loop.py:2124-2143`; advisor requests, retries, and fallbacks do not increment it. The log labels the virtual preset/provider rather than the real actor at `:2134-2143`. AGH must distinguish root turns from underlying provider attempts and report both.

### 8.3 Full traces are opt-in and side-channel only

When `moa.save_traces` is enabled, Hermes writes a JSONL record with each reference's full input/output/usage/cost and the aggregator's full input/output. Traces do not enter message history. Default tracing is off. Evidence:

- `.resources/hermes/hermes_cli/config.py:2290-2300`
- `.resources/hermes/agent/moa_trace.py:1-21`
- `.resources/hermes/agent/moa_trace.py:37-57`
- `.resources/hermes/agent/moa_trace.py:67-166`
- `.resources/hermes/tests/run_agent/test_moa_loop_mode.py:892-1018`

This separation is correct. The implementation has four caveats:

1. The aggregator trace stores input/output but not aggregator usage/cost, despite the module-level claim that a full turn can be audited for cost: `.resources/hermes/agent/moa_trace.py:1-17,139-163`.
2. There is no visible retention, rotation, redaction, or explicit file-permission policy in `.resources/hermes/agent/moa_trace.py:97-166`.
3. `_RefAccounting` says full inputs/outputs are populated only when tracing is on at `.resources/hermes/agent/moa_loop.py:30-45`, but `_run_reference` always copies them at `:311-323` and a pending trace is always assembled at `:926-936`.
4. Full system/transcript/tool bodies are stored as plaintext JSONL when enabled.

Raw full-body tracing creates privacy, secret, memory, disk-growth, and retention obligations. AGH should default to structured usage, hashes, decisions, and redacted previews; full bodies require an explicit diagnostic scope, permissions, and retention policy.

### 8.4 Reference outputs are visible but not transcript messages

Hermes emits `moa.reference` UI events before aggregation and a non-transcript `moa.aggregating` marker. References render like labeled thinking blocks even when normal reasoning display is disabled. Evidence:

- `.resources/hermes/agent/agent_init.py:820-858`
- `.resources/hermes/tui_gateway/server.py:3688-3705`
- `.resources/hermes/tests/tui_gateway/test_moa_reference_emit.py:1-98`
- `.resources/hermes/ui-tui/src/__tests__/createGatewayEventHandler.test.ts:420-459`

The events are not emitted as individual futures complete. `_run_references_parallel` first waits for the entire set at `.resources/hermes/agent/moa_loop.py:336-385`; only then does `.resources/hermes/agent/moa_loop.py:938-958` iterate and emit them. A slow reference therefore delays all advisor visibility as well as aggregation.

The runtime event payload carries each full `_output_text`; there is no payload truncation in
`.resources/hermes/agent/moa_loop.py:311-323,938-958`, despite a comment in `_RefAccounting` referring
to a truncated preview. TUI/Desktop render it, while the direct CLI applies its five-line preview
unless verbose as described in Section 3.8. AGH should cap event previews and let authorized users
open stored artifacts for full content.

## 9. Management-surface drift

### 9.1 Runtime fields missing from API schema

The runtime recognizes `reference_max_tokens` and `fanout`, and normalized GET output includes them. The management Pydantic payloads omit both fields. They also omit root `save_traces` and `trace_dir`. The PUT handler reconstructs every preset without the runtime cost controls. Evidence:

- runtime: `.resources/hermes/hermes_cli/moa_config.py:125-148`
- GET: `.resources/hermes/hermes_cli/web_server.py:5489-5502`
- schema omission: `.resources/hermes/hermes_cli/web_server.py:968-996`
- destructive reconstruction: `.resources/hermes/hermes_cli/web_server.py:5505-5541`

Canonical normalization itself returns only flattened/preset fields at `.resources/hermes/hermes_cli/moa_config.py:151-196`, dropping root trace settings. `hermes moa configure/delete` normalizes and replaces `cfg["moa"]` at `.resources/hermes/hermes_cli/moa_cmd.py:79-132`, so those CLI operations can erase `save_traces` and `trace_dir` as well.

Consequences:

1. a user can set `reference_max_tokens: 600` or `fanout: user_turn` in YAML;
2. Dashboard/Desktop loads a normalized object;
3. saving a preset sends it through a schema that ignores those fields;
4. the PUT path rebuilds config without them;
5. normalization restores the expensive defaults: uncapped and `per_iteration`.

This is a static contract finding; no live request was executed. It is sufficiently direct to treat as a likely destructive configuration bug.

### 9.2 Web and Desktop types omit cost controls

Both generated/manual TypeScript contracts omit `reference_max_tokens` and `fanout`. Their forms configure only reference models and aggregator; no output cap, cadence, or `enabled` control is visible. Evidence:

- `.resources/hermes/web/src/lib/api.ts:2282-2304`
- `.resources/hermes/web/src/pages/ModelsPage.tsx:696-865`
- `.resources/hermes/apps/desktop/src/types/hermes.ts:854-879`
- `.resources/hermes/apps/desktop/src/app/settings/model-settings.tsx:842-1079`

The CLI configurator also selects only reference slots and aggregator; it neither prints nor edits cadence/caps. Evidence: `.resources/hermes/hermes_cli/moa_cmd.py:63-113`.

The docs claim the enabled switch is surfaced in Dashboard/Desktop at `.resources/hermes/website/docs/user-guide/features/mixture-of-agents.md:167-173`; neither inspected form renders it. This is another code/product contract mismatch.

The temperature contract also drifts: backend payloads permit `None` so the provider default can be
used (`.resources/hermes/hermes_cli/web_server.py:973-980`), while Web and Desktop TypeScript declare
both temperatures as required `number`
(`.resources/hermes/web/src/lib/api.ts:2290-2303`;
`.resources/hermes/apps/desktop/src/types/hermes.ts:862-878`). The UI cannot truthfully represent the
backend's provider-default state without an unsafe assertion or a fabricated numeric value.

AGH must co-ship config lifecycle across:

- domain schema;
- persisted config;
- CLI flags/config commands;
- HTTP/UDS contracts;
- generated clients;
- UI controls;
- effective-state output;
- docs and official skill.

No hidden YAML-only cost control should be required for safe defaults.

### 9.3 Test gap around the newest controls

Tests cover `reference_max_tokens` normalization, but the inspected test tree contains no MoA-specific `fanout=user_turn` test and no management API/UI persistence test for either field. The search evidence is the absence of `fanout` in `.resources/hermes/tests/**` MoA files while implementation exists at `.resources/hermes/agent/moa_loop.py:847-905`.

AGH needs contract tests proving every resolved mode and budget survives CLI/HTTP/UDS/store/web round trips.

## 10. Additional Hermes risks not to copy

### 10.1 One-shot CLI restoration is not in `finally`

The direct CLI restores prior model fields only after `run_conversation` returns successfully inside the `try`. If the call raises, the `except` builds an error result without restoring those fields. Evidence: `.resources/hermes/cli.py:12338-12369`.

The gateway has an explicit `finally`-safe restoration test, but the direct CLI path differs. AGH participation overrides should be immutable per-run resolved state, eliminating mutable switch-and-restore behavior.

The TUI has the same class of risk at `.resources/hermes/tui_gateway/server.py:9031-9081`. AGH tests must cover success, provider error, interrupt, cancellation, and process restart for one-shot policy isolation.

### 10.2 Full output concatenation can dominate aggregator input

Even with a 600-token advisor generation cap, `R` outputs add text on roughly the order of `R * 600`
generated tokens to aggregator input, plus labels/instructions. The exact input count differs because
advisor and aggregator tokenizers may differ and the text is retokenized. With per-iteration fan-out,
new full outputs are added each iteration. Hermes performs no deterministic compression before
`.resources/hermes/agent/moa_loop.py:960-975`.

AGH should ask workers for short typed fields and select/deduplicate before rendering. A separate LLM reducer should not be the first solution because it adds another model call.

### 10.3 Benchmark claims lack inspectable raw evidence

The docs report HermesBench scores `0.8202`, `0.7607`, and `0.7412`; source comments report about `0.88` latency/output correlation and about 44% sample latency reduction at 600 tokens. Evidence:

- `.resources/hermes/website/docs/user-guide/features/mixture-of-agents.md:144-165`
- `.resources/hermes/hermes_cli/moa_config.py:132-140`
- `.resources/hermes/agent/moa_loop.py:815-825`

No matching raw benchmark report/data was found by repository search. Treat these as product/source claims, not independently reproduced evidence. AGH should store benchmark manifests, prompts, models, seeds, per-call usage, wall time, quality labels, and raw summaries with any optimization claim.

### 10.4 Invalid configuration can widen cost

Invalid slots are dropped and can fall back to the default reference/aggregator set. Invalid/non-positive caps become uncapped, and an unknown cadence becomes `per_iteration`. Evidence: `.resources/hermes/hermes_cli/moa_config.py:52-90,108-147` and `.resources/hermes/tests/hermes_cli/test_moa_config.py:192-240,252-283`.

AGH must fail closed for malformed cost or activation policy. A typo must not select more expensive models, remove a cap, add activation, or increase cadence.

### 10.5 Advisor usage can disappear after actor failure

On a cache miss, Hermes stores pending advisor usage/trace. If the actor fails before response accounting and the outer loop retries the same facade/state, the cache-hit path zeroes pending usage/trace at `.resources/hermes/agent/moa_loop.py:887-899`. Consumption happens only inside the successful `if response.usage` block at `.resources/hermes/agent/conversation_loop.py:2049-2093`. Advisor spend can therefore be lost after actor failure, and a successful actor response without usage also leaves advisor usage/trace unconsumed.

AGH must settle every invocation attempt independently of the actor response. Root success/failure and provider usage settlement are separate state transitions.

### 10.6 Streaming and non-streaming aggregator recovery diverge

The acting aggregator uses the auxiliary client. When `stream=true`, that client returns the raw SDK
stream and deliberately skips response validation plus the temperature, max-token, payment, retry,
and provider/model fallback chain. The outer streaming helper has its own bounded streaming retry
loop for timeout/connection/parse failures; unsupported-streaming or Bedrock permission errors set
`_disable_streaming`, causing the next main-loop attempt to be non-streaming, while other exhausted
stream failures propagate to outer recovery. Non-streaming calls use the full auxiliary recovery
chain. Evidence: `.resources/hermes/agent/auxiliary_client.py:6528-6540,6542-6601,6880-6952`,
`.resources/hermes/agent/chat_completion_helpers.py:2591-2605,2778-2806,2832-2880`, and
`.resources/hermes/agent/moa_loop.py:995-1022`.

This split may be appropriate for transport mechanics, but it creates two attempt paths with
different validation/fallback/accounting behavior. AGH should give both one root attempt ledger:
record the failed stream and any non-stream retry separately, carry actual resolved identity, charge
both when usage exists, enforce the same deadline/budget, and cancel underlying work rather than only
closing visible output.

## 11. Adopt, adapt, reject

| Hermes pattern | AGH decision | Reason |
|---|---|---|
| Virtual, explicitly selected multi-model mode | Adopt concept | Prevents Network from becoming default execution |
| One-shot command plus persistent selection | Adopt concept | Expresses per-turn and per-session intent separately |
| One acting aggregator with tools | Adopt | Avoids an extra synthesis-to-actor call |
| Raw/stateless reference calls | Adopt | Avoids session/network lifecycle overhead |
| Tool-free advisors | Adopt by default | Reduces prompt and blast radius |
| Parallel reference fan-out | Adopt with limits | Good latency shape for independent workers |
| Partial-failure continuation | Adopt | One worker must not abort root execution |
| Drop provider-specific actor system bulk | Adapt | Keep compact safety/authority and data-egress policy; benchmark savings |
| Per-tool-result truncation | Adapt | Add total context budget and relevance selection |
| Provider-aware prompt caching | Adopt with tests | Can improve cache reuse; acting and legacy paths need coverage |
| Configured-slot usage/cost accounting | Adapt | Preserve per-slot intent but record actual retries/fallbacks |
| Opt-in full trace separate from history | Adapt | Add redaction/retention/workspace isolation |
| `per_iteration` default | Reject | Model-call multiplication |
| Uncapped advisor default | Reject | Unbounded latency/cost |
| Concurrency-only cap | Reject | Does not bound total calls |
| Wait-for-all fan-in | Adapt | Add deadline/quorum/straggler policy |
| Full-history advisor view | Reject as default | Context grows without task relevance |
| Full concatenation fan-in | Reject | No relevance/dedup/total cap |
| Failure exception text in model context | Reject | Token and disclosure noise |
| Raw advisor text injected at user authority | Reject | Prompt-injection and provenance ambiguity |
| Dual provider and legacy aggregation paths | Reject | Contract drift and extra call risk |
| Mutable model switch/restore for one-shot | Reject | Error-path state leakage |
| YAML-only safety controls | Reject | Unsafe config lifecycle |
| Silent fallback from invalid policy to expensive defaults | Reject | Configuration errors widen spend |
| Configured-slot rather than actual-attempt accounting | Reject | Retry/fallback cost becomes false |
| Broad cross-provider transcript replication | Reject as default | Multiplies privacy and data-egress surface |
| String-only advisory projection | Reject | Silently loses multimodal/structured task evidence |

## 12. Target AGH bounded swarm shape

Hermes demonstrates a small worker-to-actor primitive with lower coordination/session overhead than
durable peer chat. Static source does not validate lower total cost or better quality, and it does not
justify putting swarm semantics inside every Network envelope.

```text
Root activation
  1. resolve participation and swarm policy once
  2. reserve root budget
  3. select at most K independent worker roles
  4. invoke workers concurrently with minimized typed contexts
  5. collect until required roles/quorum/deadline
  6. normalize, deduplicate, and rank typed results
  7. give one compact artifact set to the already-required actor
  8. settle actual usage and finish without worker-to-worker turns
```

Network can be used as a transport only when workers are remote/durable or human-visible collaboration is requested. Local workers should use direct runtime dispatch while preserving the same result schema and root trace.

### Worker request minimum

```json
{
  "workspace_id": "...",
  "root_activation_id": "...",
  "role": "reviewer",
  "task": "...",
  "artifact_refs": ["..."],
  "return_schema": "finding_set/v1",
  "data_policy": "workspace_advisory_v1",
  "allowed_provider": "...",
  "limits": {
    "input_tokens": 4000,
    "output_tokens": 500,
    "deadline_ms": 20000
  }
}
```

### Root policy minimum

```json
{
  "fanout": "root_turn",
  "max_workers": 4,
  "max_concurrency": 4,
  "min_successes": 1,
  "max_model_calls": 5,
  "max_total_tokens": 8000,
  "deadline_ms": 30000,
  "synthesis": "acting_agent"
}
```

Exact defaults require AGH benchmarks. The invariants do not: every value is finite, resolved, persisted, observable, and enforced before activation.

## 13. AGH validation scenarios derived from Hermes

1. A normal local session creates zero worker/Network activations.
2. One-shot swarm does not change the session's next-turn participation or model mode after success, failure, or interruption.
3. Persistent active selection remains active until an explicit transition.
4. `fanout=root_turn` with six actor tool iterations runs references once.
5. Invalid/omitted worker output cap resolves to a finite server default.
6. A root budget prevents the next provider call before overspend.
7. One failed worker does not abort a root with sufficient successes.
8. A straggler past the deadline does not delay synthesis and cannot reactivate the actor later.
9. Worker prompts omit Network/system/tool catalogs when not required.
10. Large tool/artifact results obey both per-item and total input budgets.
11. Duplicate worker findings are merged before actor context rendering.
12. Usage is attributed to every actual provider/model attempt, retry, fallback, role, and root activation.
13. Full traces are off by default, workspace-scoped, redacted, and retention-bound when enabled.
14. CLI, HTTP, UDS, generated clients, UI, config, and persisted effective state round-trip every policy field.
15. Saving a preset in UI cannot reset cost controls to a more expensive default.
16. Changing a cap, cadence, provider runtime, prompt, or result schema invalidates worker caches.
17. Invalid configuration fails before any provider call and never widens cost.
18. Actor failure or missing actor usage cannot erase already-incurred worker usage/traces.
19. CLI and TUI one-shot modes restore state after thrown errors and interrupts.
20. Worker cancellation stops provider work, not only visible streaming.
21. Required multimodal/structured content is preserved or fails explicitly; it never becomes an
    empty advisor prompt.
22. Workspace data classification and provider allowlists block disallowed advisor egress before a
    call and produce an auditable reason.
23. Streaming aggregator failure plus non-stream recovery creates two bounded, independently
    accounted attempts.
24. Fallback cannot silently collapse required role/model diversity or relabel one actual model as
    several configured advisors.

## AGH Impact Audit

- **Native tools:** A bounded swarm/native coordination surface must expose effective mode, finite budgets, and structured results. It must not project Network tools to local/off workers. Descriptor/schema/digest/capability tests must cover the same fields as CLI/HTTP/UDS.
- **Extensibility and hooks:** Worker selection, data-egress admission, context projection, result normalization, synthesis, and transport are extension points. Hooks observe invocation/result/budget events without implying model activation. Network bridges may implement transport behind the same bounded worker contract.
- **Workspace data isolation:** Root activations, worker requests, data classifications, provider-egress decisions, caches, results, artifacts, usage, costs, traces, UI events, and budgets are workspace-scoped. Cache keys and list/SSE paths include `workspace_id`; identical role/task/channel names across workspaces must not collide or disclose data through a shared advisor request.
- **Official AGH skill:** Teach explicit local/network/swarm selection, one-shot versus persistent behavior, bounded fan-out/fan-in, typed returns, and budget inspection. Do not expose protocol tutorials or Network tool descriptions to workers/sessions that did not opt in.

## Conclusion

The current Hermes implementation confirms a narrower claim: an explicitly selected multi-model
turn can use stateless, tool-free advisors while one actor retains tools and authority, without a
permanent conversational network. It demonstrates lower coordination/session overhead, not lower
total provider spend or a quality gain; those claims require reproducible task-specific benchmarks.

Its weaknesses are equally instructive. Per-iteration and uncapped defaults, wait-for-all fan-in,
full-history/full-output aggregation, multimodal loss, broad cross-provider egress, fallback identity
drift, missing root budgets, dual execution paths, and management-schema drift can recreate the same
cost problem at a smaller scale. AGH should adopt the topology only behind explicit selection,
finite root budgets, typed artifacts, data policy, deterministic stopping/cancellation,
actual-attempt accounting, and complete cross-surface config lifecycle.
