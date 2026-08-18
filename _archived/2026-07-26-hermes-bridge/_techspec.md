# TechSpec — Bridge Usability Parity: Tool-Call Progress, One-Paste Setup, Delivery Robustness

- **Status:** Approved — cross-LLM peer review READY (round 2; findings in `qa/peer-review-findings-round1..2.md`)
- **Date:** 2026-07-05
- **Scope:** `internal/bridges/`, `internal/bridgesdk/`, `internal/extension/` (bridge lanes),
  `internal/cli/`, `internal/doctor/`, `internal/api/`, `extensions/bridges/*`,
  `web/src/systems/bridges/`, `packages/site/content/runtime/core/bridges/`
- **Companion artifacts:** `analysis/summary.md` (root-cause chains), `analysis/01..07_*.md`
  (evidence corpus), `_tasks.md` (+ 10 task files), `_tests.md` (authoritative numbered test
  inventory), `_qa.md` (QA base for the tasks 09–10 qa-report/qa-execution pair)

## 1. Context & Goals

The prior `hermes-comparison` research deliberately deferred channels/bridges. This workstream
closes that gap. Benchmarks named in the research prompt: (a) Hermes's `hermes slack manifest`
one-paste setup; (b) Hermes bots posting each tool call as live channel messages while working.
Both were confirmed and fully traced (analysis 02 §3.1, analysis 03 §4.1 T1–T16) — with one
correction: Hermes's Slack tool progress is **off by default** and renders as **one edited
accumulator bubble**, not one-message-per-tool (analysis 03 §1 scope note).

**Goals (numbered):**
1. Bridges surface agent activity live: tool-call progress, typing, completion state — per
   channel, opt-out-able, throttled, never polluting agent context.
2. Channel setup collapses to a few AGH-driven steps: manifest generation, guided wizards with
   validators, live credential/webhook verification, real send-test.
3. Outbound delivery is robust: chunking on every provider, correct markdown dialects, durable
   delivery ledger.
4. Docs reach parity and truthfulness across all 8 providers; tests gain a conformance suite +
   docs↔code drift enforcement.

**Non-goals:** outbound media upload (deferred — analysis 07 C9/R10), GChat per-user OAuth
(C14), Telegram QR managed-bot onboarding (rejected — third-party broker, C3), Discord voice
(rejected — C10), Hermes-style in-channel pairing-code store (future; AGH `dm_policy` +
`paired_user_ids` already covers the ACL need — analysis 07 §3.8-5).

**MVP boundary.** Tasks 01–08 implement the program (W1 progress + text core, W2 provider
rendering + setup, W3 robustness + hygiene); tasks 09–10 prepare and execute QA. Everything
in Non-goals above stays out of scope unless explicitly pulled in later; Open Decisions (§6)
carry recommendations and do not gate W1 start except OD-1 (progress defaults), which is
resolved at task_01 execution time.

## 2. Root Causes / Findings (evidence-backed)

Full chains in `analysis/summary.md §1`. Contract rows:

| ID | Cause | Key evidence | Owning analysis |
| --- | --- | --- | --- |
| D1 | Broker projector + replay filter accept only `agent_message/done/error`; projection struct and `MessageContent` cannot carry tool data | `internal/bridges/delivery_broker.go:913-971`; `internal/extension/host_api_bridges.go:787-791`; `internal/bridges/delivery_types.go:55-62`; `internal/bridges/types.go:490-492` | 05 §KF-1, 03 §6.1 |
| D2 | No setup/manifest/verify/doctor seam; 8–17 manual steps; dry-run-only test-delivery | `internal/cli/bridge.go:78-89`; `internal/api/core/bridges.go:620-655`; `setup.mdx:108-183` | 06 §K-1/K-2/§4.1, 05 §KF-7 |
| D3 | Chunking exists only in WhatsApp provider; others post verbatim | `extensions/bridges/whatsapp/provider.go:2150`; `extensions/bridges/slack/provider.go:1532-1536` | 07 §3.7-1/G1 |
| D4 | No markdown dialect conversion (Slack mrkdwn, Telegram MarkdownV2) | analysis 07 §3.1/§3.2 dialect rows | 07 §G3 |
| D5 | No typing/status/reaction affordances outbound | analysis 07 §3.7-4/7 | 07 §G4 |
| D6 | Docs: 5/8 providers unguided; wrong Discord README; scopes never enumerated | `setup.mdx:103-322`; `extensions/bridges/discord/README.md:8,10`; `index.mdx:165-171` | 06 §K-5/K-6/§4.8 |
| D7 | ~19.6k duplicated provider lines; `markers.go`/`main.go` byte-identical ×8 | analysis 06 §K-7/§4.5 | 06 §G-3 |
| D8 | Broker is in-memory; no durable per-turn delivery ledger or persistent metrics | `internal/bridges/delivery_broker.go:95-114` | 05 §P-3/P-7 |
| D9 | Inbound edits unrouted; no `ReplyToText`; edit/delete ops dead in contract | `extensions/bridges/telegram/provider.go:115-117`; `internal/bridges/types.go:644-664`; 05 §P-6 | 07 §G8/G9 |

## 3. Target Architecture / Design

Constraint that frames everything: Hermes is in-process Python; AGH bridges are out-of-process
Go extensions over JSON-RPC (`internal/bridgesdk/runtime.go`). Every copied pattern crosses the
daemon↔adapter boundary as typed contract data. Guardrails: 500-line file cap (Hermes's 20k-line
`run.py` is the anti-pattern — analysis 01 §5.2), SD-002 zero-legacy, SD-007 truthful UI/docs,
SD-010 detached execution, SD-011 extensibility + agent-manageability, L-025 hard-cut the
current contract version (update all in-tree adapters in the same change; no dual paths — the
canonical negative example is Hermes's unfinished typed-event seam, analysis 03 §F1).

### 3.1 Progress delivery family (D1, D5)

Daemon side renders; adapters deliver. New projection + delivery types (new file
`internal/bridges/progress.go`; extend `internal/bridges/delivery_types.go`):

```go
// DeliveryEventTypeProgress joins start/delta/final/error/resume/delete.
const DeliveryEventTypeProgress DeliveryEventType = "progress"

// ToolProgress is the typed payload carried by progress events.
type ToolProgress struct {
    ToolCallID string             `json:"tool_call_id"`
    ToolID     string             `json:"tool_id"`             // e.g. "agh__terminal"
    Phase      ToolProgressPhase  `json:"phase"`               // started | completed | failed
    Label      string             `json:"label"`               // "Running", pre-rendered, redacted
    Preview    string             `json:"preview,omitempty"`   // redacted, capped
    Emoji      string             `json:"emoji,omitempty"`
    DurationMS int64              `json:"duration_ms,omitempty"`
    Index      int                `json:"index"`               // monotonic per turn (Hermes P1)
}
```

- `DeliveryProjectionEvent` gains `Tool *ToolProgress`; `DeliveryEvent` gains
  `Progress *ToolProgress` (content stays text for final answers).
- `Broker.projectEventLocked` (`delivery_broker.go:913-971`) and the replay filter
  (`host_api_bridges.go:787-791`) route ACP `tool_call`/`tool_result` through ONE shared
  projection helper — the double-narrowing is deleted, not patched:

```go
// ProjectAgentEvent is the single projection entry point used by BOTH the live
// notifier path and the seed/replay path. mode gates progress emission.
func ProjectAgentEvent(ev DeliveryProjectionEvent, mode ProgressMode) (DeliveryEvent, bool, error)

// ResolveProgressConfig resolves instance override → platform default → global default.
func ResolveProgressConfig(instance *BridgeInstance, platform string) ProgressConfig
```
- Per-instance gating via a typed `delivery_defaults.progress` block:
  `{tool_progress: off|new|all|verbose, grouping: accumulate|separate, typing: bool,
  reactions: bool}` — resolver precedence copies Hermes `display_config.py:21-26`.
- Presentation-only invariant (analysis 03 P7): progress events are bridge-delivery types,
  distinct from ACP session events; a contract test asserts no progress chrome appears in
  session transcripts.
- Backpressure: progress events coalesce like deltas (queue capacity `delivery_types.go:40`);
  under saturation newest-wins per phase (Hermes throttle lesson, analysis 07 G10/G11).

### 3.2 Label registry + previews (feeds 3.1)

New `internal/toolmeta` (or `internal/bridges/toolmeta.go` if <500 lines): map keyed by tool ID
with `{Verb, Connector, DropsPreview}` + per-tool preview builders (terminal shell-summarizer,
read_file basename+range, delegate "N tasks: …" — port `agent/display.py:412-577`). Tool
descriptors gain optional `friendly_verb`/`preview` metadata so extension/MCP tools get
first-class labels (fixes Hermes F8).

**Redaction (security surface — peer review B-001).** Every preview/label passes ONE canonical
secret redactor before it enters any payload. Do not write a new matcher: promote
`internal/modelcatalog/redact.go`'s package-private `secretPatterns`/`RedactString` into a
shared leaf helper (e.g. `internal/redact`) consumed by both modelcatalog and toolmeta, so the
pattern set cannot drift. The enforced taxonomy is the FULL `internal/CLAUDE.md`
Security-Invariant set — raw claim tokens (`agh_claim_*`), MCP auth tokens, OAuth authorization
codes, PKCE verifiers, secret-binding refs and `*_secret`-suffixed values, plus provider token
shapes (`sk-…`, `xoxb-…`/`xapp-…`, `gh[pousr]_…`, `Bearer …`, generic
`api_key|auth_token|oauth_token|access_token|refresh_token|id_token|secret|password|credential|private_key`
key/value forms). Progress previews are channel messages; the invariant applies verbatim.
(Hermes redacts only `browser_type` — F4; AGH redacts everything.)

### 3.3 Provider delivery mechanics (bridgesdk)

`internal/bridgesdk/progress.go`: a `ProgressAccumulator` struct encapsulating Hermes's drain
loop state (9 mutable closure cells → explicit struct, analysis 03 F6): throttled edit-in-place
(1.5s default), `×N` dedup counter, reset-on-content marker, overflow roll at
`maxMessageLen - 64`. Final signatures:

```go
// ChunkMessage splits text at limit (measured by lenFn), preserving fenced code
// blocks across chunks and appending "(N/M)" indicators on multi-chunk output.
func ChunkMessage(text string, limit int, lenFn func(string) int) []string
func UTF16Len(s string) int

type ProgressSink interface { // implemented per provider
    Post(ctx context.Context, target bridges.DeliveryTarget, text string) (remoteID string, err error)
    Edit(ctx context.Context, target bridges.DeliveryTarget, remoteID, text string) error
    Typing(ctx context.Context, target bridges.DeliveryTarget, active bool) error // no-op allowed
    React(ctx context.Context, target bridges.DeliveryTarget, reaction string) error // no-op allowed
}

type ProgressAccumulator struct{ /* unexported state; constructor-injected clock */ }
func NewProgressAccumulator(cfg ProgressConfig, sink ProgressSink, now func() time.Time) *ProgressAccumulator
func (a *ProgressAccumulator) OnProgress(ctx context.Context, ev bridges.ToolProgress) error
func (a *ProgressAccumulator) OnContent(ctx context.Context) error // reset marker
func (a *ProgressAccumulator) Flush(ctx context.Context) error
```

Providers declare `maxMessageLen` (Slack 40000, Telegram 4096 UTF-16, Discord 2000, Teams
28000, GChat per API). Typing + reactions ride the progress family (`Progress.Phase`
start→👀/typing; final→✅/❌ where the platform supports it).

### 3.4 Setup & verification surface (D2)

- `agh bridge manifest slack [--write PATH]` (new `internal/cli/bridge_manifest.go` — keeps
  `bridge.go` under cap): emits the full Slack app manifest (display info, scopes, event
  subscriptions, interactivity; socket_mode=false — AGH is webhook-based) generated from the
  installed extension's `extension.toml` + the events the provider actually ingests. Tests
  assert scope↔event **relations** plus vendor-schema validity (closes Hermes's own gap,
  analysis 04 §5).
- `agh bridge setup <platform>` (new `internal/cli/bridge_setup.go`): interactive wizard AND
  non-interactive `--json` mode (SD-011 — agents must drive it). WhatsApp: port the field-shape
  validators verbatim (`setup_whatsapp_cloud.py:52-157`). Telegram: token prompt + generated
  `webhook_secret` + daemon-executed `setWebhook` (daemon holds the token post-binding). Discord:
  invite-URL builder with correct scopes/permissions. Every wizard ends with the exact verify
  command (analysis 02 §4-9).
- `agh bridge verify <id>` + doctor `CategoryBridge` (`internal/doctor/`): live identity probe
  (Slack `auth.test` + scope diff vs manifest, Telegram `getMe`, Discord `users/@me`), webhook
  reachability, structured remediation records (not logs — analysis 02 §5). `agh bridge
  send-test <id>` performs a real platform send; `test-delivery` stays dry-run.
- HTTP/UDS parity for verify/send-test/manifest (SD-011); web create-wizard surfaces the
  manifest and verify results (analysis 06 R-7).

### 3.5 Durability (D8)

`bridge_deliveries` table (append via numbered migration — `eng-schema-migration`, L-021):
delivery id, route, seq, state, `remote_message_id`, timestamps. Broker checkpoints send/ack
transitions; restart resumes or fails-open with the existing `resume` machinery
(`delivery_broker.go:651-677`). Metrics gain durable counters.

### 3.6 Structural refactor (D7 — two-touch discipline)

Hoist provider scaffolding into `internal/bridgesdk`: provider lifecycle loop, instance-config
reconciliation, route-map swap, delivery-state store, config lookups (analysis 06 §4.5 names the
blocks). Providers keep only: signature verifier, payload mappers, platform API client, delivery
executors. `markers.go` ×8 → one bridgesdk test-support package; `main.go` ×8 → shared stub.

### 3.7 Data model & config lifecycle (field rationale)

**`bridge_deliveries` side table** (task_06; numbered append-only migration). Side table, not a
JSON blob: every field below is matchable state queried by boot reconciliation and metrics —
JSON bags are for opaque metadata only (marker rule; matches `bridge_task_subscriptions`
precedent).

| Column | Purpose |
| --- | --- |
| `delivery_id` (PK) | broker delivery identity; joins ack/resume paths |
| `bridge_instance_id` | owning instance; adapter reconcile scope |
| `scope`, `workspace_id` (indexed) | direct workspace isolation — no instance join needed for scoped lists/metrics (precedent: `bridge_task_subscriptions`, migration #20; peer review N-003) |
| `session_id`, `turn_id` | source turn; boot reconciliation looks up session state |
| `routing_key` | route identity for resume ordering |
| `state` (`active\|terminal_ok\|terminal_error`) | reconciliation decision input |
| `last_sent_seq`, `last_acked_seq` | resume point; detects half-streamed messages |
| `remote_message_id` | adapter-side reconcile anchor (`FindDeliveryMessage`) |
| `terminal_error` (nullable text) | operator-visible failure cause |
| `created_at`, `updated_at` | audit + stale-row GC |

**`delivery_defaults.progress` block** (task_01) — typed struct validated at resource apply
(NOT free-form JSON; tightens analysis 05 P-9):

| Key | Shape / purpose |
| --- | --- |
| `tool_progress` | enum `off\|new\|all\|verbose` — emission gate |
| `grouping` | enum `accumulate\|separate` — bubble vs per-line messages |
| `typing` | bool — typing affordance toggle |
| `reactions` | bool — 👀/✅/❌ affordance toggle |

Progress payloads travel as typed `ToolProgress` fields on `DeliveryEvent` — explicitly NOT
stuffed into `ProviderMetadata` JSON (matchable, validated state).

### 3.8 Safety invariants (numbered)

1. Progress chrome is presentation-only: no progress event is ever written to session
   transcripts or replayed into ACP history (type boundary + contract test, task_01.4).
2. Per-route delivery ordering is preserved: progress events enqueue on the same ordered route
   queue as text events; exactly one terminal event per delivery.
3. Coalescing under saturation collapses only same-`(tool_call_id, phase)` progress items (a
   status snapshot) and cumulative text deltas; events for DISTINCT tool ids are never merged
   into each other, and terminal events are never dropped. When saturation forces dropping an
   intermediate progress line, the degradation is best-effort-visible: counted in
   `dropped_by_reason` metrics and documented in `progress.mdx` (peer review N-002).
4. Ledger checkpoints (task_06) write on register/ack-advance/terminal only — never per delta —
   and serialize through SQLite transactions.
5. Boot reconciliation completes before the broker accepts new delivery registrations
   (composition-root ordering, SD-008).
6. Every preview/label is redacted daemon-side through the ONE canonical secret redactor
   (promoted from `internal/modelcatalog/redact.go`) covering the full Security-Invariant
   taxonomy — `agh_claim_*`, MCP auth tokens, OAuth codes, PKCE verifiers, secret-binding
   refs/`*_secret` values, provider token shapes — before it enters a payload (task_01); raw
   args never cross the daemon→adapter transport.
7. The `bridges/deliver` contract change ships wired end-to-end — all 8 in-tree adapters +
   codegen in the same program; no dual event paths (L-025; Hermes F1 negative example).
8. Verify/doctor probes (task_04) are read-only with respect to the instance lifecycle state
   machine; only operators and `report_state` mutate state, per existing rules.
9. Secrets in wizards (task_04) echo masked, bind via existing vault-backed
   `secret-bindings` APIs, and never persist to disk outside the vault.

## Architectural Boundaries

Composition-root discipline (SD-008) and the downward-only import graph are load-bearing for
every task in this program:

- `internal/bridges` stays transport-free and provider-free: it defines the delivery contract
  (`ToolProgress`, `DeliveryEvent`, projection) and the broker. It MUST NOT import
  `internal/bridgesdk`, `internal/extension`, or any platform SDK.
- `internal/bridgesdk` is the provider-side scaffold. It imports `internal/bridges` contract
  types only; it MUST NOT import `internal/extension`, `internal/daemon`, or `internal/store`.
- `internal/toolmeta` (new, task_01) is a leaf package: imports nothing from bridges/extension;
  consumed by `internal/bridges` projection and (later) other renderers. It may read tool
  descriptor metadata via a narrow interface defined in `toolmeta`, implemented by
  `internal/tools` at the composition root — no upward import from `toolmeta` to `tools`.
- `internal/extension` owns the daemon↔adapter transport (Host API, `bridges/deliver`); it is
  the only package that talks to provider subprocesses.
- `internal/daemon` is the only multi-importer: it wires broker ⇄ transport ⇄ store, boot
  reconciliation (task_06), and doctor probes registration (task_04). New cross-cutting wiring
  lands ONLY here.
- `extensions/bridges/*` binaries import `internal/bridgesdk` (+ their platform client); they
  never import `internal/extension` or `internal/daemon`.
- CLI verbs (`internal/cli`) call daemon APIs; they never import `internal/bridges` broker
  internals. Doctor bridge probes ask the daemon, which asks the provider — the daemon never
  learns platform REST APIs (task_04 requirement).
- `mage Boundaries` is updated in the same commit that introduces `internal/toolmeta` (and any
  other new subpackage).

## 4. Delete Targets (zero-legacy)

| Delete | Replaced by |
| --- | --- |
| Double event-narrowing: `default` no-op switch in `delivery_broker.go:913-971` + replay filter `host_api_bridges.go:787-791` | One shared projection helper handling text + progress families |
| `markers.go` byte-identical ×8 (`extensions/bridges/*/markers.go`) | bridgesdk test-support package |
| `main.go` byte-identical ×8 | shared provider stub |
| Duplicated scaffolding blocks in 8 × `provider.go` (analysis 06 §4.5 list) | bridgesdk hoisted provider runtime |
| WhatsApp-local `splitMessage` (`whatsapp/provider.go:2150`) | `bridgesdk.ChunkMessage` |
| Wrong Discord README lines (`discord/README.md:8,10`) | corrected Ed25519/REST wording |
| curl-first setup instructions (`setup.mdx:20-22,128-183`) | CLI-first flows (`agh bridge create/setup/manifest`) |
| `index.mdx:165-171` 3-provider slots table | generated/complete 8-provider table with drift test |

No compat shims: the `bridges/deliver` contract change updates all 8 in-tree adapters in the
same program (L-025); adapters simply ignore `Progress` payloads they don't render.

## 5. Sequencing (waves)

- **W1 — Progress + text core (gates: broker/projection tests first):** task_01 (progress
  contract + projection, label registry/redaction, ProgressAccumulator), task_02 (shared
  chunking + markdown dialects).
- **W2 — Provider rendering + setup (gates: CLI structured output + vendor-schema tests):**
  task_03 (progress rendering across the six chat providers, after 01), task_04 (setup CLI:
  Slack manifest + wizards + verify/doctor/send-test), task_05 (web setup orchestrator, after
  01+04).
- **W3 — Robustness + hygiene:** task_06 (inbound edit family + ReplyToText + durable delivery
  ledger, after 01), task_07 (scaffolding hoist + conformance/drift suites — last, after all
  provider-touching tasks: 02, 03, 04, 06), task_08 (docs parity + truthfulness).
- **QA tail (SD-005):** task_09 (qa-report planning), task_10 (qa-execution).

## 6. Open Decisions

1. **Default tool-progress mode per platform.** Slice 03 recommends `new`+`accumulate` on
   edit-capable platforms; slice 07 recommends Hermes-matching `off` for Slack.
   **Recommendation: `new`+`accumulate` for Slack/Telegram/Discord, `off` for Teams/WhatsApp/
   GChat v1** — visibility is the product goal; Hermes's own comment admits accumulate is the
   spam fix (`display_config.py:115-118`).
2. **Where labels live:** `internal/toolmeta` package vs `internal/bridges/toolmeta.go`.
   Recommendation: dedicated package if reused by web/CLI transcript rendering later.
3. **Telegram `setWebhook` executed by daemon vs printed for the user.** Recommendation:
   daemon-executed with `--print-only` escape hatch (daemon already holds the token).
4. **In-channel pairing-code issuer** (analysis 01 R7): defer — existing `dm_policy` +
   `paired_user_ids` covers ACL; revisit if operator demand appears.
5. **Durable ledger scope (task_06):** full audit ledger vs checkpoint-only. Recommendation:
   checkpoint-only v1 (delivery id/seq/state/remote_message_id), audit columns later.

## 7. AGH Impact Audit

- **Native tools:** no `agh__*` IDs/toolsets change. Tool **descriptors** gain optional
  `friendly_verb`/`preview` metadata (task_01) — descriptor schema + digest surfaces re-checked;
  codegen co-ship if descriptor JSON schema changes. **Security surface:** task_01 opens a new
  outbound render path (tool previews → channel messages); it is gated by the canonical
  redactor with the full Security-Invariant taxonomy (§3.2, §3.8-6 — peer review B-001).
- **Extensibility and hooks:** `bridges/deliver` contract gains the `progress` family +
  `delivery_defaults.progress` config block (extension authors read both); bridgesdk gains
  ProgressAccumulator/ChunkMessage/hoisted provider runtime — all documented in the bridge SDK
  surface. New CLI verbs (`manifest`, `setup`, `verify`, `send-test`) with HTTP/UDS parity and
  structured output (SD-011). Config lifecycle: `delivery_defaults.progress` is typed +
  validated (tightens free-form JSON, analysis 05 P-9); no new `config.toml` keys.
- **Workspace data isolation:** progress events carry `BridgeInstanceID` + routing key with the
  same workspace scoping as existing deliveries (`ValidateScopeWorkspaceID`,
  `internal/bridges/types.go:100`); the `bridge_deliveries` table is keyed by instance
  (workspace-scoped); no cross-workspace list/SSE path changes.
- **Official AGH skill:** `skills/agh/references/native-tools.md` (and `runtime-operations.md`
  as applicable) must document the new bridge verbs (`manifest`/`setup`/`verify`/`send-test`),
  the webhook-registration route, the progress config block, and progress semantics — owned
  explicitly by task_08 (subtask 8.4), which co-ships the official-skill update with the docs
  parity pass.

## 8. Web/Docs Impact

- **Web:** `web/src/systems/bridges/` — create-wizard provider step surfaces manifest + setup
  checklist; detail panel shows verify results inline on secret cards; new `delivery.progress`
  form fields; SSE health stream untouched (L-017 noted for any new named events). Generated TS
  types refresh via `make codegen`.
- **Docs (`packages/site`):** `core/bridges/setup.mdx` rewritten CLI-first for all 8 providers;
  new `core/bridges/progress.mdx`; `index.mdx` slots table completed; `cli-reference/bridge/*`
  regenerates for new verbs; `api-reference/bridges.mdx` regenerates from OpenAPI. Discord
  README corrected. Slack scope list enumerated (verify against Slack docs during task_08).

## 9. Verification Strategy

> **Authoritative case list:** `_tests.md` — the complete numbered unit/integration/E2E
> inventory per lane (this section summarizes; that file is what executors work through).
> **QA base:** `_qa.md` — personas, journeys, scenario/charter seeds feeding tasks 09–10.

- Canonical suites (edit, don't fork — `eng-consolidate-test-suites`):
  `internal/bridges/delivery_broker_test.go` + `delivery_projection_test.go` (progress
  projection, gating, coalescing); `internal/bridgesdk/*_test.go` (accumulator, chunking —
  UTF-8/UTF-16/code-fence corpus); per-provider `provider_test.go` (rendered payload contract:
  fake `ToolCallStarted` → expected `chat.postMessage`/`chat.update` body — closes Hermes F9);
  `internal/cli/bridge_test.go` (manifest relations + vendor schema, wizard validators);
  `internal/doctor/doctor_test.go` (CategoryBridge probes); transcript-purity contract test
  (progress ∉ session history).
- Conformance: auto-discovering provider suite (task_07) modeled on
  `test_plugin_platform_interface.py`; docs↔code drift tests for slots table + scope lists.
- E2E: `make test-e2e-runtime` extends the bridge lane (contract co-ship — L-007); web changes
  via `bunx turbo run lint typecheck test --filter=./web` + `eng-ui-screenshot` for UI tasks.
- Completion gate: single `make verify` per batch; 80% coverage floor on touched packages.
- QA: task_09/10 per SD-005; `docs/qa/state.csv` rows flagged per behavior change.

## 10. Reference Catalog

- **Ours:** `internal/bridges/{delivery_broker,delivery_types,types,task_notifier,resource,
  lifecycle,diagnostics,resolver,target}.go`; `internal/bridgesdk/{runtime,webhook,batching,
  dedup,cache,errors,hostapi,peer,target_snapshots}.go`; `internal/extension/
  {host_api_bridges,bridge_delivery_notifier,manager,manifest}.go`; `internal/daemon/
  {bridges,task_event_bridge_notifier}.go`; `internal/cli/bridge.go`; `internal/api/core/
  bridges.go`; `internal/acp/types.go`; `internal/events/registry.go`; `internal/tools/
  dispatch.go`; `extensions/bridges/{slack,telegram,discord,whatsapp,teams,gchat,github,
  linear}/`; `web/src/systems/bridges/`; `packages/site/content/runtime/core/bridges/`.
- **Competitor (`.resources/hermes/`):** `gateway/run.py` (16674-17272 progress pipeline);
  `gateway/platforms/base.py` (adapter contract, 5494 truncate, 2593 format_tool_event);
  `gateway/display_config.py`; `gateway/stream_events.py` (+ the F1 lesson);
  `agent/display.py` (412-626 labels/previews); `agent/tool_executor.py`;
  `hermes_cli/{slack_cli,commands,setup_whatsapp_cloud,setup}.py`;
  `plugins/platforms/{slack,telegram,discord,whatsapp,teams,google_chat}/`;
  `tests/gateway/{test_plugin_platform_interface,test_run_progress_topics,test_send_retry,
  test_text_batching,test_pairing}.py`; `tests/hermes_cli/test_slack_cli.py`;
  `tests/gateway/relay/test_contract_doc_conformance.py`;
  `website/docs/user-guide/messaging/*.md`.
- Full per-file roles: each analysis §7 Reference Index.
