# TechSpec — Hermes-Referenced AGH Improvements (memory, tools, sessions, reliability, security, automation, usage, extensibility)

- **Status:** Approved — cross-LLM peer review READY (round 4, 2026-07-05; findings in `qa/peer-review-findings-round1..4.md`)
- **Date:** 2026-07-04
- **Scope:** `internal/{daemon,session,acp,transcript,memory,tools,toolruntime,automation,subprocess,doctor,scheduler,store/globaldb,registry,skills,mcp,bridgesdk,sandbox,config}`, new `internal/redact` + cost-estimation home (§3.6), `web/`, `packages/site`, `skills/agh/`, repo-root `SECURITY.md`
- **Companion artifacts:** `analysis/summary.md` (synthesis), `analysis/01..08_analysis_*.md` (evidence), `adrs/adr-001..010.md` (user-approved decisions), `_tasks.md` (+ task files)

## 1. Context & Goals

The task-orchestration comparison against Hermes produced the archived task-blocks spec
(`.compozy/tasks/_archived/1783210421905-fe0eaf86-task-blocks/`). This program covers the other
Hermes surfaces — memory/context, tools UX, session lifecycle, reliability, security, cron/
automation, provider usage, skills/plugins — and turns the confirmed gaps into an executable
backlog. Channels/bridges/communications are **excluded** (dedicated research later).

Goals (numbered):
1. Fix the seven confirmed defects D1–D7 (`analysis/summary.md` §1) at root cause.
2. Adopt the user-approved Hermes patterns (ADR-001..010) adapted to AGH's daemon + ACP-subprocess
   architecture — never importing Hermes' in-process mechanisms.
3. Preserve the surfaces where AGH is already ahead (summary §2 "do not regress").

Non-goals (user-declined or architecture-rejected — evidence in summary §2 and the ADRs):
command-risk classifier and external secret sources (ADR-005 §3); automation blueprints (ADR-007);
daemon-side LLM client/router; model-API fallback chains; prepaid billing; live subprocess-window
rewriting; in-process plugins/untyped hooks; JSON-file cron store; channels/comms surfaces.

**MVP boundary:** tasks 01–11 are the program — W1 correctness (01–02), W2 session & memory
(03), W3 security & reliability (04–05), W4 product layers (06–08), W5 extensibility (09–10),
W6 orchestration correctness (11); tasks 12–13 are the SD-005 QA pair that closes it. Deferred
beyond this program (tracked as verification gates in §6, never as hidden task scope): the
account-usage fetcher implementation half (gated on the L-015 reachability spike), `@`-reference
parity, progressive tool disclosure, and activation of the loop-template suggestion variant (gated
on the Loops program shipping). Channels/bridges/communications and every user-declined item above
are out of scope entirely.

## 2. Root Causes / Findings (evidence-backed)

| # | Defect / gap | Root cause | Evidence | Analysis |
|---|---|---|---|---|
| D1 | `allow_always` is a lie | Outcome handler collapses always→once; no durable store consulted | `internal/daemon/tool_approval_bridge.go:183-193,151-163,196-203` | 02 §5, 05 G4 |
| D2 | Recurring jobs silently skip after downtime | Only `skip_missed` policy; `MisfireGraceSeconds` never wired (always 0) | `internal/automation/schedule.go:785`; CHECK `internal/store/globaldb/global_db.go:2586` | 06 §6.1 |
| D3 | Runtime compaction is dead code | `runContextCompaction` + hooks + `OnPreCompress` have no production caller | `internal/session/manager_hooks.go:492`; `internal/memory/contract/types.go` | 01 §3, 03 §5.2 |
| D4 | Silent context loss on resume | `session/load` failure falls to `session/new` without replaying `events.db` | `internal/session/manager_lifecycle.go:79-103`; `internal/acp/client.go:433-508` | 03 §5.1 |
| D5 | Health detection without action | `subprocess/health.go` verdict has no consumer | `internal/subprocess/health.go` | 04 §5 |
| D6 | Overflowed tool results unrecoverable | Truncation drops content; `ArtifactRef` unused on overflow | `internal/tools/result_limit.go:294-351` | 02 §5 |
| D7 | `RequiresInteraction` dead-end | No channel for an agent to ask the user | `internal/extension/manifest.go:187` | 02 §5 |
| G2 | Secrets unredacted outside diagnostics | Redaction exists only in `internal/diagnostics/item.go` | 05 F26 | 05 §5 |
| G6 | `BackendLocal` = no isolation, undocumented | No trust-model doc | `internal/sandbox/local/provider.go:135-162` | 05 §5 |
| P1-P5 | Extensibility gaps (cache/gates/serve/catalog/snapshot) | Product-mechanics layer absent above sound mechanism | `analysis/08_analysis_skills-plugins.md` §5.1 | 08 |
| U1 | Cost never estimated | tokens×rates join missing despite both sides existing | `internal/observe/observer.go:759`; `internal/modelcatalog/types.go` | 07 §3 |
| A1 | Automation authoring absent | No suggestion layer (blueprints declined — loops templates canonical) | 0 grep hits; `.compozy/tasks/loops/_techspec.md` | 06 §5 |

## Architectural Boundaries

Composition-root discipline (SD-008, `internal/CLAUDE.md`): `internal/daemon` remains the sole
composition root; no package gains an import on `daemon/`, `api/`, or `cli/`; dependencies flow
downward only; interfaces are defined where consumed; every new HTTP/UDS endpoint lands in
`internal/api/core` as shared `BaseHandlers` methods (HTTP and UDS only choose registration/auth).

New or changed package boundaries in this program:

| Package | Role in this program | May import | Must NOT import |
|---|---|---|---|
| `internal/redact` (new) | Shared default-on secret redaction engine | stdlib only (leaf) | any AGH package |
| `internal/checkpoint` (new) | Shadow-git workspace checkpoints around AGH-native mutating tools | `internal/workspace`, stdlib + git exec (retention opts injected by daemon) | `daemon`, `api`, `cli`, `internal/tools` |
| `internal/modelcatalog` (`pricing.go`) | Cost-estimation join (tokens × catalog rates); decided home (§6.4) | existing deps (buckets passed as plain values — no new edge on `acp`) | `daemon`, `api`, `observe` |
| `internal/transcript` | Gains the no-LLM pruner; stays a projection library | existing deps | `session`, `daemon` |
| `internal/registry` | TTL index-cache as a `Source` decorator + `mcp` `PackageType` | existing deps | `skills`, MCP host internals |
| `internal/skills` | `when.*` gates evaluated at catalog build; parse snapshot | existing deps + `internal/filesnap` (already used) | `registry` network sources |
| `internal/tools` | Owns approval-grant, clarify, and checkpoint tool contracts (interfaces defined where consumed) | existing deps | `store/globaldb` (implementations injected by daemon) |
| `internal/store/globaldb` | Implements grant/suggestion/dead-entity stores | `internal/store`, domain model packages | `tools`, `daemon` |

Authoritative-primitive exclusivity is preserved: `task_runs` stays the single durable queue and
`task.Service.ClaimNextRun` the only claim primitive — W6 hardens both in place, adding no peer
claimer and no parallel queue. Hooks keep dispatching at the call site: the compaction driver
routes through the existing `internal/hooks` session dispatcher and never tails event tables.
`mage Boundaries` rules are updated in the same commit that introduces `internal/redact` and
`internal/checkpoint`.

## 3. Target Architecture / Design

Constraining rules: composition-root discipline (SD-008), detached execution (SD-010),
extensible + agent-manageable by design (SD-011), consumer-not-redesign (SD-009), zero-legacy
(SD-002), provider auth boundary (L-015), migrations append-only (L-021), 500-line file cap.
Every new capability ships the native-tool triad (descriptor/toolset + CLI + HTTP/UDS + web/SSE)
plus `skills/agh/` docs — none of the items below is UI-only or Go-call-only.

### 3.1 Approvals & clarify (ADR-001; fixes D1, D7)
Durable approval store keyed `{workspace_id, agent_name?, tool_id, input_digest?}`, consulted
before prompting; `agh__tool_approvals_{list,revoke}`; three-mode ceiling stays sole authority
(archived tools-registry ADR-005); zero new prompt classes. **Persistence width (B-001):** a
single `allow_always`/`reject_always` selection persists at the most-specific key —
`{workspace_id, agent_name, tool_id, input_digest}` — by default; any wider grant (agent-wide or
tool-wide) is an explicit, separately-surfaced user choice via `agh__tool_approvals`/CLI/web,
never the default effect of answering one prompt. `agh__clarify` `{question, choices[≤4]}`
modeled on the `internal/acp/permission.go` pending/resolve channel, config timeout,
`RequiresInteraction` routes through it. **Timeout fallback (B-002):** clarify timeout resolves to
an explicit unanswered sentinel (`Choice=nil`, `Text=""`, `Fallback=true`) — callers MUST treat it
as a non-answer, never as a selected option. **Approval-plane boundary (round-2 N-001):** durable
grants, the three-mode ceiling integration, and clarify govern the native `agh__*` tool plane
only; the ACP subprocess agent's own fs-tool approvals (`internal/acp/permission.go`) are a
deliberately separate plane, untouched by this program. Hermes references:
`.resources/hermes/tools/approval.py` (persistence only), `.resources/hermes/tools/clarify_tool.py`
+ `clarify_gateway.py`.

### 3.2 Session lifecycle (ADR-002, ADR-003; fixes D3, D4)
Resume: replay `internal/transcript`-assembled history into the fresh subprocess when
`session/load` is unavailable/fails; "context rebuilt from log" marker; pruned via §3.3's pruner.
Compaction: pressure-triggered driver (ACP usage updates) through existing pre/post-compact hooks;
attempt cap + failure cooldown + non-destructive `archived` event flag (append-only migration);
`[session.compaction]` config. **`archived` × resume-replay seam (B-004):** resume-replay reads
only non-archived events and injects the durable checkpoint-summary artifact (§3.3) in place of
archived spans — never the raw archived rows (no re-inflation) and never a bare gap (no silent
loss). **Ordering guarantee (round-3 B-301):** the pre-compact step updates the durable
checkpoint-summary to cover the span being archived BEFORE the `archived` flag lands, so a crash
at any point (including before a clean session end) leaves either the raw events or a covering
summary — never neither; summary-update-at-compaction and checkpoint-summary injection are both
owned by task 03 (its compaction subtask consumes the summary record its checkpoint subtask
maintains; injection rides task 02's replay assembly). Reducer composition is fixed: the
`archived=0` filter applies first, then the
`Prune` pass; the compact→resume integration test is owned by task 03 and includes the
crash-before-session-end window, asserting neither loss nor re-inflation. Smaller affordances: auto-title via bounded spawn (memory-extractor
idiom, `internal/daemon/memory_runtime.go`), interrupted-prompt salvage in
`manager_busy_input.go`, file-mutation verifier footer from persisted tool-call events.
SD-001 re-evaluation required (touches `manager_*.go` + `acp/client.go`).

### 3.3 Memory hardening (ADR-003 §2/§4)
`internal/transcript` gains a no-LLM pruner (1-line tool-result summaries + dedup + oversized-arg
truncation — `.resources/hermes/agent/context_compressor.py:1179`); durable checkpoint-summary
memory artifact updated at the **compaction boundary** (the pre-compact extraction updates the
summary to cover the span being archived BEFORE its events are flagged `archived` — round-3
B-301) and at session end/dream (heading schema from Hermes' `[CONTEXT SUMMARY]`),
revertible via decision WAL; dream anti-thrash/ineffective-run backoff; `agh__memory_*` atomic
`operations[]` batch + `old_text` substring match (`.resources/hermes/tools/memory_tool.py:497-602`);
prefix-cache byte-stability audit across injected prompt segments
(`.resources/hermes/agent/system_prompt.py:448-461` discipline).

### 3.4 Security posture (ADR-005; fixes G2, G6)
New `internal/redact`: default-on, tamper-resistant (flag snapshotted at process start — changing
`[redact] enabled` requires a daemon restart, and every config/CLI/UI surface says so; no live
toggle that silently no-ops), applied to agent-visible output, logs, SSE/event payloads;
provider-prefix seed table adapted from `.resources/hermes/agent/redact.py:71-113`.
**Scope boundary (B-003; extended round 2 B-002):** the heuristic engine is bounded to
free-text/agent-visible *content* fields; it never touches the structured correlation/hash
envelope (`claim_token_hash`, session/run ids, fingerprints, idempotency keys) on ANY seam — the
canonical append-only event ledger, SSE payloads, AND log records. On the logger seam
specifically: the heuristic applies only to the free-text message body, and structured slog
attribute values are redacted exclusively through the same field-aware allow-list as `RedactJSON`
— correlation/hash attributes are never candidates, so log correlation keys (mandatory per
`internal/CLAUDE.md` Observability) survive redaction intact. The existing exact
claim_token/secret-binding redaction remains the authoritative guarantee; the heuristic engine is
additive defense-in-depth, never the sole guarantee. **Persist-vs-emit ordering (round-4 N-402):**
heuristic redaction of content fields runs BEFORE durable append — the canonical `runtime.db`
ledger and the per-session `events.db` both store the redacted form; persist-raw/redact-on-egress
is forbidden (a raw persisted secret would resurface through history queries and resume replay).
The per-session `events.db` transcript is therefore IN the redaction boundary, and its
history-query egress carries a leak test. Repo-root `SECURITY.md` documents boundaries
vs heuristics, that `BackendLocal` is not isolation, and that `agh mcp serve` stdio grants local
spawners operator authority (ADR-008 §3/§4).

### 3.5 Reliability (ADR-010; fixes D5)
Lifecycle self-protection guard at the `internal/automation` creation seam (command-shaped regex,
`.resources/hermes/cron/lifecycle_guard.py:48-66`); `[memory]` observability loop + `runtime.memory`
doctor probe; first-class `draining` via HTTP/UDS/CLI (control channel, not file marker);
health verdict → doctor probe + status degradation + config-gated `needs_attention` escalation
through `internal/scheduler/starvation.go`; workspace-scoped dead-entity registry for
extensions/bridges/MCP sidecars; bridgesdk error-taxonomy split + decorrelated jitter
(first-party outbound only; two-touch consolidation if third retry-surface change).

### 3.6 Usage & cost (ADR-006; fixes U1)
Cost-estimation service in `internal/modelcatalog/pricing.go` — decided home (peer-review round 1,
N-005); promotion to a dedicated `internal/usage` package happens only via a follow-up decision if
implementation accrues >1 cohesive file (YAGNI). Strict precedence
`actual > estimated > included > unknown`, never summing agent-reported with estimated. Four-bucket
+ reasoning pricing columns through catalog/config/extension contracts; `CostStatus`/`CostSource`
on `TokenStats` (append-only migration); `subscription_included` off auth mode. Account-usage is
verification-first (L-015): prototype token reachability before any fetcher exists.

### 3.7 Automation authoring (ADR-007; fixes D2, A1)
Engine: wire `MisfireGraceSeconds`, add `run_once_on_catchup` (+ CHECK migration), self-overlap
guard, lifetime `RunLimit` distinct from `FireLimit`. Authoring: consent-first Suggestion store
(sources `catalog|usage|integration`, dedup-latch, cap 5, accept→create), payload = prefilled
plain `Job` spec only — the loop-template suggestion variant is NOT built here; it lands with the
Loops program as a hard cut (no pre-emptive inert field/contract — N-001, zero-legacy);
`agh__automation_suggestions_{list,accept,dismiss}` + CLI/HTTP/UDS + web card + 3-5-entry starter
catalog per workspace. No Blueprint resource — loop templates are canonical.

### 3.8 Workspace checkpoints (ADR-004)
Shadow-git snapshots strictly around AGH-native mutating tools; `agh__checkpoint_{list,diff,
restore}`; per-workspace store under `$AGH_HOME`; retention config; restore-snapshots-first.
**Restore blast radius (round-3 N-301):** restore is diff-scoped — it reverts only the paths
captured in the snapshot being restored, leaving unrelated working-tree changes (including
delegated-agent edits made after the snapshot) intact; paths that changed since the snapshot are
surfaced in a restore preview before applying, and the pre-rollback snapshot still captures the
full pre-restore state. Reconciled with worktree isolation (one snapshot owner per surface) —
enforced mechanically: the
checkpoint manager consults the worktree-isolation resolver before every `Snapshot`, and a native
tool operating inside an isolated worktree is skipped (the worktree mechanism owns that surface)
(N-007). External-binary dependency (N-006): shadow-git shells out to `git`; when the binary is
absent the feature degrades gracefully — checkpoints disabled, `runtime.checkpoint` doctor probe
reports "git unavailable", checkpoint tools return truthful unavailability, never a crash. This is
a recorded exception to the single-binary posture.

### 3.9 Extensibility (ADR-008, ADR-009)
TTL registry index-cache + embedded seed catalog with `official` tier (cache keys = source+query
only); `metadata.agh` `when.{platforms,environments,requires_tools,requires_capabilities}`
activation gates evaluated at catalog build (`internal/skills/registry_agent.go:27`); `agh mcp
serve` re-projecting host-API methods (no new `agh__*` IDs; token auth for non-stdio; host-API
isolation preserved); `mcp` registry package type (transport/auth/git-install manifest, mutating
tools opt-in via risk flags, flows through existing MCP host); parse snapshot keyed by `filesnap`.

### 3.10 Task-orchestration correctness (analysis/09; fixes O1–O5)
Cross-model review (Opus + GLM 5.2) of the durable orchestration kernel — the surface
`04_analysis` excluded. Five evidence-linked fixes in the derived layers (the claim primitive itself
is sound); design source is `analysis/09_analysis_task-orchestration.md` (no separate ADR — these are
correctness fixes to existing behavior, not contested designs):
- **O1** lease-expiry recovery must consume an attempt (or a durable recovery counter) so
  `max_attempts` bounds a crash-looping worker — `requeueExpiredLease` currently skips `attempt`.
- **O2** loop circuit-breaker accounting is per-node/whole-generation, not a single global counter
  zeroed by any sibling success; unbounded watch loops get a hard generation backstop.
- **O3** `ClaimRun` gains a `status='queued'` CAS so the manual claim path can't double-write
  ownership — implemented by EXTRACTING `ClaimNextRun`'s inline guarded-claim UPDATE
  (`internal/store/globaldb/global_db_task_claim.go`) into one shared store-level helper that both
  entry points invoke (one CAS implementation; never a parallel `ClaimRunCAS`; fencing strength
  structurally cannot diverge — N-009, round-2 B-001, task_11).
- **O4** action-node runs get a wall-clock timeout (dual of the existing child-loop timeout) plus a
  progress-aware heartbeat, so a wedged-but-heartbeating run fails instead of hanging the loop.
- **O5** two bounded scaling guards on the claim/wake hot path: an indexed wake-dedup lookup replacing
  the unbounded `ListTaskEvents` scan, and a per-workspace concurrent-active-run cap that defers
  *claiming*, not enqueueing — dependents remain durably enqueued in `task_runs` (the single
  queue) and drain as capacity frees; the cap never drops or rejects queued work, so no
  starvation path exists (N-002, task_11 wording). The cap check executes inside the same
  immediate transaction as the guarded-claim CAS — never a read outside the claim transaction —
  so two concurrent claims cannot both observe `count < cap` and over-admit (round-3 N-302).
Deliberate divergence: adopt Hermes' *policies* (spawn pause, activity-liveness, reject-at-cap,
recursive cancel), never its in-memory persistence — AGH's SQLite durable-lease model stays everywhere.

## Interface Contracts (final signatures)

Signatures below are final; implementation tasks fill bodies, not shapes. Existing types referenced
(`store.TokenStats`, `automation.Job`, `registry.Source`, `tools.ToolResult`,
`subprocess.HealthState`, `scheduler.EscalationActor`) keep their current definitions except where
a field is explicitly added in the Data Model section.

### Approval grants (ADR-001, D1) — contract in `internal/tools`, implementation in `internal/store/globaldb`

```go
// internal/tools/approval_grants.go
type ApprovalGrantKey struct {
	WorkspaceID string
	AgentName   string // optional: "" matches any agent in the workspace
	ToolID      ToolID
	InputDigest string // optional: "" matches any input for the tool
}

type ApprovalGrantDecision string

const (
	ApprovalGrantAllow  ApprovalGrantDecision = "allow"
	ApprovalGrantReject ApprovalGrantDecision = "reject"
)

type ApprovalGrant struct {
	ID         string
	Key        ApprovalGrantKey
	Decision   ApprovalGrantDecision
	CreatedAt  time.Time
	LastUsedAt time.Time
}

// ApprovalGrantStore is consulted by the daemon bridge BEFORE prompting and written
// when the user selects allow_always/reject_always. Lookup resolves most-specific
// key first (exact agent+digest → agent → digest → tool-wide). Writes from a prompt
// answer ALWAYS persist the most-specific key (workspace+agent+tool+input_digest);
// wider grants exist only through explicit management surfaces (B-001).
type ApprovalGrantStore interface {
	LookupApprovalGrant(ctx context.Context, key ApprovalGrantKey) (ApprovalGrant, bool, error)
	PutApprovalGrant(ctx context.Context, grant ApprovalGrant) (ApprovalGrant, error)
	ListApprovalGrants(ctx context.Context, workspaceID string) ([]ApprovalGrant, error)
	RevokeApprovalGrant(ctx context.Context, workspaceID, id string) error
}
```

Bridge change: `toolApprovalOutcome` (`internal/daemon/tool_approval_bridge.go:179`) stops
collapsing always→once — `allow_always`/`reject_always` persist a grant via this store, and the
request path consults the store before emitting any ACP prompt (hit → no prompt).

### Clarify (ADR-001, D7) — broker in `internal/daemon`, modeled on `acp.AgentProcess` pending/resolve

```go
// internal/daemon/clarify_bridge.go
type ClarifyRequest struct {
	SessionID string
	Question  string
	Choices   []string // len ≤ 4; empty slice = free-text answer
}

// ClarifyAnswer's timeout-fallback contract (B-002): on timeout the broker resolves
// with the explicit unanswered sentinel {Choice: nil, Text: "", Fallback: true}.
// It NEVER synthesizes a selection. Callers (including RequiresInteraction routing)
// MUST treat Fallback=true as "the user did not answer", never as a selected option.
type ClarifyAnswer struct {
	Choice   *int   // index into Choices; nil for free text AND for the fallback sentinel
	Text     string // free-text answer; "" for the fallback sentinel
	Fallback bool   // true only when produced by timeout fallback
}

type ClarifyPending struct {
	RequestID string
	SessionID string
	Question  string
	Choices   []string
	AskedAt   time.Time
}

// ClarifyBroker mirrors the pendingPermission channel pattern
// (internal/acp/permission.go): buffered response channel, per-boot in-memory state.
type ClarifyBroker interface {
	Ask(ctx context.Context, req ClarifyRequest) (ClarifyAnswer, error) // blocks until Resolve or timeout
	Resolve(sessionID, requestID string, answer ClarifyAnswer) error    // HTTP/UDS/CLI/web answer path
	Pending(sessionID string) []ClarifyPending
}
```

### Compaction driver (ADR-003, D3) — `internal/session`

```go
// internal/session/compaction.go — [session.compaction] config
type CompactionConfig struct {
	Enabled            bool          `toml:"enabled"`
	PressureThreshold  float64       `toml:"pressure_threshold"`   // fraction of context window (0,1]
	MaxAttemptsPerTurn int           `toml:"max_attempts_per_turn"`
	FailureCooldown    time.Duration `toml:"failure_cooldown"`
}

// maybeCompact runs on ACP usage updates; on pressure it routes through the existing
// runContextCompaction → hooks.HookContextPreCompact/PostCompact dispatch
// (internal/session/manager_hooks.go:492) and archives compacted events non-destructively.
func (m *Manager) maybeCompact(ctx context.Context, session *Session, usage acp.TokenUsage) error
```

### Transcript pruner (§3.3) — `internal/transcript`

```go
// internal/transcript/prune.go — pure function, no LLM, consumed by extractor + resume replay
type PruneOptions struct {
	MaxToolResultLines int  // default 1: 1-line tool-result summaries
	MaxArgBytes        int  // oversized tool-arg truncation threshold
	Dedup              bool // drop repeated identical tool results
}

func Prune(messages []Message, opts PruneOptions) []Message
```

### Redaction (ADR-005, G2) — `internal/redact`

```go
// internal/redact/redact.go
type Engine struct{ /* provider-prefix seed table + entropy heuristics */ }

func New(opts Options) *Engine

// RedactString is the free-text entry point. On the logger seam it applies ONLY
// to the message body — never to rendered records or structured attribute values
// (round-2 B-002).
func (e *Engine) RedactString(s string) string

// RedactJSON applies the heuristic engine ONLY to the free-text/agent-visible
// content fields named in fields; structured correlation/hash envelope fields
// (claim_token_hash, session/run ids, fingerprints, idempotency keys) are never
// candidates (B-003). The exact claim_token/secret-binding redaction elsewhere
// remains the authoritative guarantee; this engine is additive defense-in-depth.
func (e *Engine) RedactJSON(raw json.RawMessage, fields []string) json.RawMessage

// RedactLogAttrs applies the same field-aware allow-list as RedactJSON to slog
// attribute values; correlation/hash attributes are never candidates, so log
// correlation keys survive redaction intact (round-2 B-002).
func (e *Engine) RedactLogAttrs(attrs []slog.Attr) []slog.Attr

// Enabled is snapshotted once at process start (tamper-resistant); callers never
// re-read live config at redaction time. Changing [redact] enabled requires a
// daemon restart (surfaced in config/CLI/UI copy — N-008).
func Enabled() bool
```

### Cost estimation (ADR-006, U1) — `internal/modelcatalog/pricing.go` (decided home, §6.4)

```go
// internal/modelcatalog/pricing.go
type TokenBuckets struct {
	Input, Output, CacheRead, CacheWrite, Reasoning *int64
}

type CostStatus string // "actual" | "estimated" | "included" | "unknown"
type CostSource string // "agent_reported" | "catalog_config" | "models_dev" | "builtin" | "none"

type CostResult struct {
	Amount   float64
	Currency string
	Status   CostStatus
	Source   CostSource
}

// EstimateCost joins usage × catalog rates; ok=false when no rate row exists
// (caller records status=unknown, never zero).
func EstimateCost(model *Model, usage TokenBuckets) (CostResult, bool)
```

Wired into `observe.aggregateObservedUsage` (`internal/observe/observer.go:743`) so
`store.TokenStatsUpdate` carries `CostAmount` + provenance even when the agent is silent, and into
the task roll-up at `internal/task/live.go:556-586`.

### Suggestions (ADR-007, A1) — model in `internal/automation/model`, store on `GlobalDB`

```go
// internal/automation/model/suggestions.go
type SuggestionSource string // "catalog" | "usage" | "integration"
type SuggestionStatus string // "pending" | "accepted" | "dismissed"

// Payload is the prefilled plain Job spec, re-validated through the normal Job
// validation path on accept. The loop-template suggestion variant is deliberately
// absent — it lands with the Loops program as a hard cut (N-001; ADR-007).
type Suggestion struct {
	ID          string           `json:"id"`
	WorkspaceID string           `json:"workspace_id"`
	Source      SuggestionSource `json:"source"`
	DedupKey    string           `json:"dedup_key"`
	Status      SuggestionStatus `json:"status"`
	Payload     Job              `json:"payload"`
	CreatedAt   time.Time        `json:"created_at"`
	ResolvedAt  *time.Time       `json:"resolved_at,omitempty"`
}

// internal/store/globaldb (idiom: global_db_automation.go CreateJob/GetJob)
func (g *GlobalDB) CreateSuggestion(ctx context.Context, s automation.Suggestion) (automation.Suggestion, error)
func (g *GlobalDB) ListSuggestions(ctx context.Context, workspaceID string, status automation.SuggestionStatus) ([]automation.Suggestion, error)
// ResolveSuggestion is a CAS pending→{accepted,dismissed}; ErrSuggestionResolved on conflict.
func (g *GlobalDB) ResolveSuggestion(ctx context.Context, workspaceID, id string, to automation.SuggestionStatus) (automation.Suggestion, error)
```

### Checkpoints (ADR-004) — `internal/checkpoint`, tool contract in `internal/tools`

```go
// internal/checkpoint/checkpoint.go
type Checkpoint struct {
	ID          string
	WorkspaceID string
	ToolID      string // AGH-native mutating tool that triggered the snapshot
	Label       string
	CreatedAt   time.Time
	Bytes       int64
}

type RetentionPolicy struct {
	MaxCount int
	MaxBytes int64
	MaxAge   time.Duration
}

type Manager interface {
	Snapshot(ctx context.Context, workspaceID, toolID, label string) (Checkpoint, error)
	List(ctx context.Context, workspaceID string) ([]Checkpoint, error)
	Diff(ctx context.Context, workspaceID, id string) (string, error)
	// Restore snapshots the current state first (pre-rollback snapshot), then restores.
	Restore(ctx context.Context, workspaceID, id string) (Checkpoint, error)
}
```

### Reliability (ADR-010) — drain, dead entities, health consumption

```go
// internal/daemon/drain.go — driven by HTTP/UDS/CLI drain|undrain endpoints
type DrainState string // "active" | "draining"

func (d *Daemon) Drain(ctx context.Context) error   // gate new prompt/run admission
func (d *Daemon) Undrain(ctx context.Context) error

// internal/store/globaldb — dead-entity registry (workspace-scoped)
type DeadEntity struct {
	WorkspaceID string
	Kind        string // "extension" | "bridge" | "mcp_sidecar"
	EntityID    string
	Reason      string
	MarkedAt    time.Time
}

func (g *GlobalDB) MarkDeadEntity(ctx context.Context, e store.DeadEntity) error
func (g *GlobalDB) ClearDeadEntity(ctx context.Context, workspaceID, kind, entityID string) error
func (g *GlobalDB) ListDeadEntities(ctx context.Context, workspaceID string) ([]store.DeadEntity, error)
```

Health consumption reuses existing types end-to-end: `subprocess.HealthState` (existing) feeds a
`runtime.subprocess_health` doctor probe + status degradation; persistent unhealthy escalates via
the existing `scheduler.EscalationActor.MarkRunNeedsAttention` — no new escalation primitive.

### Registry cache + skill gates (ADR-009)

```go
// internal/registry/cache.go — Source decorator; cache keys = source name + query only
func NewCachedSource(inner Source, ttl time.Duration, seed []Listing) Source

// internal/skills/metadata.go — metadata.agh `when.*` gates, evaluated inside
// Registry.ForAgent/ForWorkspace (internal/skills/registry_agent.go:27) at catalog build
type ActivationGates struct {
	Platforms            []string `yaml:"platforms,omitempty"`
	Environments         []string `yaml:"environments,omitempty"`
	RequiresTools        []string `yaml:"requires_tools,omitempty"`
	RequiresCapabilities []string `yaml:"requires_capabilities,omitempty"`
}
```

## Data Model & Config Lifecycle

Rule applied throughout: ownership/matchable state gets typed columns or side tables; JSON is for
opaque payloads only — never for state a query must match on.

### Storage changes (every migration appends at the registry tail — L-021, `eng-schema-migration`)

| Store | Change | Shape | Rationale |
|---|---|---|---|
| `agh.db` — `tool_approval_grants` (new side table) | durable grants | `id PK`, `workspace_id NOT NULL`, `agent_name DEFAULT ''`, `tool_id NOT NULL`, `input_digest DEFAULT ''`, `decision CHECK IN ('allow','reject')`, `created_at`, `last_used_at`; `UNIQUE(workspace_id, agent_name, tool_id, input_digest)` | consulted on the approval hot path before every prompt → indexed columns, never JSON metadata |
| `events.db` — session events (owning store: `internal/store/sessiondb`; the L-021 append-only obligation attaches to the sessiondb migration registry — round-2 N-002) | new `archived INTEGER NOT NULL DEFAULT 0` column | flag only; rows never deleted | compaction must be non-destructive (ADR-003) |
| `agh.db` — `automation_scheduler_state` | widen CHECK to `catch_up_policy IN ('skip_missed','run_once_on_catchup')` | table-rebuild migration (SQLite CHECK), append-only | D2: `MisfireGraceSeconds` gets wired; new policy fires one synthetic run after downtime |
| `agh.db` — `automation_jobs` | new nullable `run_limit INTEGER` | lifetime budget | distinct semantics from rolling `fire_limit` window |
| `agh.db` — `token_stats` | new `cost_status TEXT`, `cost_source TEXT` | provenance columns | UI must never render `estimated` as `actual` (ADR-006) |
| model catalog rows + provider config + extension model source | `CostCacheReadPerMillion`, `CostCacheWritePerMillion`, `CostReasoningPerMillion` (`*float64`) | nullable — absence ≠ zero | four-bucket + reasoning pricing |
| `agh.db` — `automation_suggestions` (new side table) | suggestion store | `id PK`, `workspace_id`, `source CHECK IN ('catalog','usage','integration')`, `dedup_key`, `status CHECK IN ('pending','accepted','dismissed')`, `payload TEXT` (JSON), `created_at`, `resolved_at`; `UNIQUE(workspace_id, dedup_key)` | status/dedup are matchable → typed columns; the prefill payload is opaque → JSON, re-validated through Job validation on accept |
| `agh.db` — `dead_entities` (new side table) | dead-entity registry | `workspace_id`, `kind`, `entity_id`, `reason`, `marked_at`; `PK(workspace_id, kind, entity_id)` | probe loops query it cheaply; auto-clear = `DELETE` on success |
| `$AGH_HOME/checkpoints/<workspace_id>/` (filesystem) | shadow git repo per workspace | git refs are the index — no SQLite table | git already stores the snapshot DAG; a DB mirror would drift |
| `$AGH_HOME` skill parse snapshot (filesystem) | serialized parsed catalog keyed by `map[string]filesnap.Snapshot` | cache, safe to delete | cold-start performance only, never truth |
| `agh.db` — W6 orchestration storage (task 11) | O1 durable recovery counter, O2 per-node breaker counters, O4 last-progress column, O5 wake-dedup index + per-workspace active-run accounting | exact column/index shapes are owned by task 11's subtasks; every change is an append-only tail migration | the L-021 obligation is governed here even though field shapes are task-owned (N-003) |

Migration numbering (N-004): migration versions are assigned **at land time** — each
migration-bearing task appends at the current registry tail when it merges; versions are never
pre-baked into task files, and concurrent migration-bearing tasks serialize their tail appends
(L-021 append-only identity).

### Config keys (each ships struct + defaults + validation + docs + tests together — SD-011; idiom: `internal/config/automation.go`)

| Key | Default | Purpose |
|---|---|---|
| `[session.compaction] enabled` | `true` | pressure-triggered compaction driver |
| `[session.compaction] pressure_threshold` | `0.85` | context-window fraction that triggers compaction |
| `[session.compaction] max_attempts_per_turn` | `1` | Hermes attempt-cap guard |
| `[session.compaction] failure_cooldown` | `"10m"` | Hermes failure-cooldown guard |
| `[tools.clarify] timeout` | `"5m"` | clarify block timeout → deterministic fallback answer |
| `[redact] enabled` | `true` | snapshotted at process start (tamper-resistant); restart-required — CLI/UI copy states it, no live toggle (N-008) |
| `[daemon] memory_report_interval` | `"5m"` (`0` disables) | `[memory]` observability loop + doctor probe |
| `[automation.suggestions] pending_cap` | `5` | consent-first pending cap |
| `[checkpoint] retention.max_count / max_bytes / max_age` | `20` / `1GiB` / `"720h"` | shadow-git retention |
| `[registry] cache_ttl` | `"24h"` | offline index-cache TTL |
| `[tasks] action_run_timeout` | `"30m"` | O4 wall-clock bound for action-node runs |
| `[tasks] max_active_runs_per_workspace` | `16` (`0` disables) | O5 defer-at-cap for auto-enqueue |
| provider `cost_cache_read_per_million` / `cost_cache_write_per_million` / `cost_reasoning_per_million` | unset | pricing override chain (config > models.dev > builtin) |

### Side-table vs JSON decisions (explicit)

- **Approval grants → side table.** Hot-path lookup with composite key; JSON metadata would force
  full-scan matching. Decision state never lives in an event/metadata blob.
- **Suggestions → side table with JSON payload column.** `status`/`dedup_key`/`source` are
  matchable → typed columns with CHECKs; the prefill (`Job` spec or loop-template ref) is opaque
  until accept, when it re-enters the normal Job validation path.
- **`archived` → typed column on events**, not event-payload JSON — replay/pruning filters on it.
- **Cost provenance → typed columns on `token_stats`** — list views filter/badge on status.
- **Dead entities → side table** — probe suppression is a keyed lookup.
- **Checkpoints → filesystem shadow-git store, no DB table** — git refs/log are the authoritative
  index; a SQLite mirror would be a second source of truth.
- **Clarify pending state → in-memory only** (mirrors `pendingPermissions`): durable pending
  questions would resurrect ghost prompts after restart; timeout fallback covers crash windows.
- **Catch-up policy → existing typed column** with widened CHECK, never a JSON settings bag.

## Safety Invariants (numbered)

Approvals & clarify:
1. The three-mode policy (`deny-all`/`approve-reads`/`approve-all`, `internal/tools/policy.go`)
   remains the sole approval ceiling; the grant store only remembers decisions the ceiling already
   routed to a prompt — it never approves a call the ceiling denies and never adds a prompt class.
2. A single `allow_always`/`reject_always` answer persists at the most-specific grant key
   (`workspace_id + agent_name + tool_id + input_digest`); wider grants (agent-wide, tool-wide)
   exist only through explicit management surfaces, never as the default effect of one prompt
   (B-001; covered in the `internal/daemon` bridge suite).
3. Grant consult happens strictly BEFORE any ACP permission prompt; a remembered grant never
   re-prompts; a grant-store read error fails to the prompt (never auto-approve on error).
4. Clarify pending state is per-boot in-memory (mirrors `pendingPermissions`); timeout resolves to
   the explicit unanswered sentinel (`Choice=nil`, `Text=""`, `Fallback=true`) — callers MUST
   treat it as a non-answer, never a synthesized selection (B-002); clarify never mutates
   approval state.

Session lifecycle & memory:
5. AGH never rewrites the live subprocess context window; compaction acts only through
   pre/post-compact hooks and non-destructive event archiving (ADR-003).
6. At most one compaction attempt in flight per session; `max_attempts_per_turn` caps retries;
   failure arms the cooldown; archiving sets `archived=1` and never deletes rows.
7. Resume replay assembles only from the session's own `events.db` (session-scoped, cannot cross
   workspaces); every degraded resume emits the "context rebuilt from log" marker event.
8. A compacted-then-resumed session neither re-inflates nor silently loses context: replay reads
   only `archived=0` rows and injects the durable checkpoint-summary artifact in place of archived
   spans; the checkpoint-summary is updated to cover a span BEFORE that span's events are flagged
   `archived` (compaction-boundary update), so a crash in any window — including before a clean
   session end — leaves either raw events or a covering summary, never neither; the archived
   filter applies before the `Prune` pass (B-004, round-3 B-301; the task-03-owned compact→resume
   integration test asserts all three properties, including the crash window).
9. The checkpoint-summary memory artifact is update-only (never regenerated from scratch) and
   revertible via the existing decision WAL.

Security:
10. Raw `claim_token` never crosses transport, channel, log, or memory; this is the existing
    exact, structured guarantee and it remains authoritative — the heuristic redactor is never
    the mechanism that upholds it (B-003).
11. The heuristic redactor is additive defense-in-depth bounded to free-text/agent-visible content
    fields; it never rewrites structured correlation/hash envelope fields (`claim_token_hash`,
    session/run ids, fingerprints, idempotency keys) on any seam — event ledger, SSE, and log
    records alike; on the logger seam the heuristic touches only the message body while structured
    attributes go through the field-aware allow-list; enablement is snapshotted at process start;
    content-field redaction runs BEFORE durable append (canonical ledger and per-session
    `events.db` store the redacted form — never persist-raw/redact-on-egress), so history-query
    and resume-replay egress can never surface a raw heuristic-class secret (B-003, N-008,
    round-2 B-002, round-4 N-402).
12. `BackendLocal` is documented as no-isolation; nothing in this program treats it as a boundary.

Cost:
13. Provenance precedence is strict and non-summing: `actual > estimated > included > unknown`;
    agent-reported and estimated amounts are never added; `estimated` always renders with its
    status badge, never as plain `$` truth.

Automation & reliability:
14. Under `run_once_on_catchup`, at most one synthetic run fires per downtime window; the durable
    cursor advances exactly once (existing CAS discipline).
15. Self-overlap guard: a job never fires while its previous fire is still running.
16. Suggestion resolution is a CAS `pending→{accepted,dismissed}`; the dedup latch survives
    dismissal (`UNIQUE(workspace_id, dedup_key)`); the pending cap is enforced at insert.
17. The lifecycle guard rejects AGH-daemon lifecycle commands at creation time on both the CLI and
    native-tool paths — one validator, two call sites (ADR-010 §1).
18. Draining gates new prompt/run admission only; in-flight work completes; drain intent, if ever
    persisted, is stamped with a per-boot nonce.
19. Checkpoint restore snapshots the pre-restore state first; exactly one snapshot mechanism owns
    any given surface — enforced by consulting the worktree-isolation resolver before every
    `Snapshot`, skipping native tools operating inside isolated worktrees (N-007).
20. Dead-entity marks require confirmed-permanent failure; any success auto-clears the entry;
    suppression is workspace-scoped.

Orchestration (W6 — O1..O5):
21. `task_runs` remains the single durable queue and `ClaimNextRun` the only authoritative claim
    primitive; W6 hardens them in place — no peer claimer, no parallel queue.
22. Lease-expiry recovery consumes an attempt (or a durable recovery counter) so `max_attempts`
    bounds a crash-looping worker (O1).
23. Loop circuit-breaker accounting is per-node/per-generation; a sibling's success never zeroes
    another node's counter; watch loops carry a hard generation backstop (O2).
24. `ClaimRun` and `ClaimNextRun` share one store-level guarded-claim primitive — the single
    `status='queued'` CAS helper EXTRACTED from `ClaimNextRun`'s inline UPDATE, invoked by both
    entry points (never a parallel guarded method); the manual and autonomous claim paths
    structurally cannot diverge in fencing strength, and double-claim is impossible on either path
    (O3, N-009, round-2 B-001).
25. Action-node runs are bounded by wall-clock timeout plus progress-aware liveness; a wedged but
    heartbeating run terminalizes instead of hanging its loop (O4).
26. The per-workspace active-run cap defers *claiming* only — dependents remain durably enqueued
    in `task_runs` and drain as capacity frees (never dropped, never rejected — no starvation);
    the cap check runs inside the same immediate transaction as the guarded-claim CAS (a
    hard bound, not an advisory read — round-3 N-302); wake dedup uses an indexed lookup, never
    an unbounded `ListTaskEvents` scan (O5, N-002).

## 4. Delete Targets (zero-legacy)

| Removed | Replaced by |
|---|---|
| Inert-scaffolding status of `runContextCompaction`/`OnPreCompress` (test-only callers as the sole consumers) | Production compaction driver + checkpoint-summary flow (ADR-003) — "no scaffolding without a production caller" |
| `allow_always`→`allow_once` silent-downgrade branch in `toolApprovalOutcome` | Durable-store consult + persist path (ADR-001) |
| Dead `MisfireGraceSeconds` (unwired field) | Wired grace + `run_once_on_catchup` policy (+ CHECK migration) |
| Drop-on-overflow behavior in `result_limit.go` | Artifact offload + preview + page-back pointer |
| `automation` CHECK constraint value-set at `global_db.go:2586` (as-is) | Migrated CHECK accepting the new catch-up policy value (append-only tail migration) |

No public rename in this program; every contract change co-ships OpenAPI/TS codegen in the same
task (no dual fields, no aliases).

## 5. Implementation Steps (waves)

| Wave | Tasks | Gate |
|---|---|---|
| W1 Correctness (D1, D2, D4, D5, D6, D7 + pruner dep) | 01-02 | tests-first on each defect's owning suite; scoped `make lint` + package `-race` tests |
| W2 Session & memory (D3 + hardening + UX affordances) | 03 | SD-001 supervisor re-evaluation note in each session task |
| W3 Security & reliability posture | 04-05 | doctor/status assertions; redaction leak tests |
| W4 Product layers (cost, suggestions, checkpoints) | 06-08 | append-only migration checks; verification-first account-usage half of task 06 |
| W5 Extensibility | 09-10 | offline-mode registry tests; MCP façade isolation tests |
| W6 Orchestration correctness (O1-O5; loop/lease/scheduler) | 11 | tests-first per defect; store CAS + `-race` on `internal/task`, `internal/loop`, `internal/store/globaldb` |
| W7 QA pair (SD-005) | 12-13 | `eng-qa-bootstrap` fresh lab; `make verify` completion gate |

Dependency edges live in `_tasks.md`; the graph is acyclic and keeps W1 free of cross-wave deps.

## 6. Open Decisions

All drastic calls were resolved in the 2026-07-04 user decision round and recorded as ADRs
(adr-001..010). Remaining open items are verification gates, not contested designs:

1. **Account-usage token reachability (gates task 06's account-usage implementation half)** — prototype under
   L-015 constraints; if unreachable, ship truthful `included/unknown` only (ADR-006 §5).
2. **`@`-reference parity (01 R8)** — confirm bundled ACP agents' inline-reference support before
   any AGH-side injector is specced; deliberately NOT in this task suite.
3. **Progressive-disclosure deferral (02 R5)** — measure what the subprocess actually receives
   before speccing an activation gate; deliberately NOT in this task suite.
4. **Cost-service package home** — RESOLVED (peer-review round 1, N-005): committed to
   `internal/modelcatalog/pricing.go` throughout (Boundaries table + Interface Contracts anchor
   it); promotion to `internal/usage` requires a follow-up decision and only if implementation
   accrues >1 cohesive file (§3.6).

## 7. AGH Impact Audit

- **Native tools:** new toolsets `agh__tool_approvals_*`, `agh__clarify`,
  `agh__automation_suggestions_*`, `agh__checkpoint_*`; extended `agh__memory_*` (batch schema →
  digest updates); `agh mcp serve` re-projects host API WITHOUT minting native IDs (ADR-008 §2);
  result-payload contract gains `ArtifactRef` on overflow. Descriptors, schema digests, risk
  flags, capability gates, and availability diagnostics co-ship per toolset task.
- **Extensibility and hooks:** compaction wires existing `context.pre_compact`/`post_compact`
  hooks; extension model source gains pricing buckets; registry gains cache + `mcp` package type;
  skills gain `when.*` gates + parse snapshot; extension host API re-projected by MCP serve;
  clarify exposed to extension tools. Config lifecycle: `[session.compaction]`, approval-store
  retention, memory-monitor interval, drain, suggestion cap, checkpoint retention,
  `cost_*_per_million`, registry cache TTL — each key ships structs+defaults+validation+docs+tests
  together (SD-011).
- **Workspace data isolation:** approval grants, suggestions, checkpoints, dead-entity sets are
  workspace-scoped (keys/tables carry `workspace_id`; leak-path checks in each task's tests);
  resume replay and compaction are session-scoped and must not cross workspaces; cost aggregates
  stay session-scoped with workspace roll-up filters; account usage is operator/global-scoped and
  must not leak spend into workspace session lists; registry/skill caches are global catalog data
  keyed source+query only.
- **Official AGH skill:** `skills/agh/` updates required for: clarify + approval management,
  suggestions, checkpoints, drain verb, `agh mcp serve`/`install`, cost/usage CLI output, memory
  batch semantics, `when.*` skill gates.

## 8. Web/Docs Impact

- **web/:** approval management view (remembered grants list/revoke); clarify answer UI (SSE
  question event → choice submit); suggestions card (accept/dismiss); checkpoint timeline
  (list/diff/restore); cost display on session/task views (status-aware — `estimated` badge,
  `included` label, never fake `$` for subscriptions); doctor/status surfaces for draining +
  memory probe + dead entities; automation job form gains catch-up policy + run-limit fields.
  Truthful-UI rule: render only daemon-backed controls (SD-007).
- **packages/site:** config reference (every key in §7), CLI reference regen, MCP serve guide,
  suggestions/checkpoints/cost docs, `SECURITY.md`-derived trust-model page.

## 9. Test Strategy

Per test-placement rules: each defect fix lands in the existing canonical suite of its owning
package (approval bridge → `internal/daemon` bridge tests, including assertions that a prompt
answer persists the most-specific grant key by default — B-001; clarify broker → timeout-fallback
test asserting the unanswered sentinel is produced and treated as a non-answer — B-002; automation
policies → `internal/automation` schedule tests; resume/compaction → `internal/session` manager
suites + `internal/acp` client tests; result offload → `internal/tools` result-limit tests;
redaction → new `internal/redact` suite with a secret-leak test, a
does-not-mangle-correlation-keys test over structured envelope fields — B-003 — and a log-seam
test asserting `claim_token_hash`/`session_id`/`run_id` attributes survive redaction intact in an
emitted log record — round-2 B-002; cost → `internal/modelcatalog`/`internal/observe` suites;
registry/skills → their package suites). Observability co-ship (round-2 N-004): every new
lifecycle transition this program introduces — grant put/revoke, clarify ask/resolve, compaction
fire, drain/undrain, checkpoint snapshot/restore, suggestion accept/dismiss, dead-entity
mark/clear — emits its canonical event with the mandatory correlation keys and extends the
observability coverage-matrix test in the same task. Migrations: fresh-DB + upgrade/reopen +
recorded-prefix tests (L-021). Integration:
`make test-integration` covers approval persistence round-trip, resume replay against a mock ACP
agent (contract co-ship, L-007), compact→resume asserting neither context loss nor re-inflation
across the `archived` seam — owned by task 03 and covering the crash-before-session-end window
(after compaction archives a span and the daemon dies pre-session-end, resume reconstructs the
archived span's facts from the injected summary; B-004, round-3 B-301) — suggestion accept→job
creation, drain admission gating, MCP serve session listing. E2E:
`make test-e2e-runtime` for resume/compaction/drain lanes; `make test-e2e-web` for clarify +
approvals + suggestions + cost UI. UI changes verified with `eng-ui-screenshot` before completion.
Completion gate: single full `make verify` per batch (not per micro-task).

## 10. Reference Catalog

- **Approvals/clarify:** `internal/daemon/tool_approval_bridge.go`, `internal/tools/policy.go`,
  `internal/acp/permission.go` · Hermes `tools/approval.py`, `tools/clarify_tool.py`,
  `tools/clarify_gateway.py`
- **Sessions/memory:** `internal/session/manager_lifecycle.go`, `manager_hooks.go`,
  `manager_busy_input.go`, `internal/acp/client.go`, `internal/transcript/`, `internal/memory/`
  · Hermes `agent/context_compressor.py`, `agent/memory_provider.py`, `tools/memory_tool.py`,
  `agent/system_prompt.py`
- **Reliability:** `internal/subprocess/health.go`, `internal/doctor/`, `internal/scheduler/
  starvation.go`, `internal/bridgesdk/errors.go`, `internal/retry/` · Hermes
  `gateway/{restart_loop_guard,drain_control,dead_targets,memory_monitor}.py`,
  `cron/lifecycle_guard.py`
- **Security:** `internal/diagnostics/item.go`, `internal/sandbox/local/provider.go`,
  `internal/vault/` · Hermes `agent/redact.py`, `SECURITY.md`
- **Automation:** `internal/automation/` (`schedule.go`, `model/types.go`),
  `internal/store/globaldb/global_db.go` · Hermes `cron/{scheduler,jobs,suggestions,
  suggestion_catalog}.py`, `tools/cronjob_tools.py`
- **Usage:** `internal/modelcatalog/`, `internal/observe/observer.go`, `internal/task/live.go`,
  `internal/store/types.go`, `internal/config/provider.go` · Hermes `agent/account_usage.py`
- **Extensibility:** `internal/registry/`, `internal/skills/`, `internal/mcp/`,
  `internal/extension/`, `internal/filesnap/` · Hermes `plugins/`, `skills/index-cache/`,
  `optional-mcps/`, `mcp_serve.py`
- Full per-file roles: Reference Index tables in `analysis/01..08_analysis_*.md`.
