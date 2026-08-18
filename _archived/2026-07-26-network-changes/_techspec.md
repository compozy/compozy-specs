# TechSpec — Opt-In Agent Network Participation (Local + Live)

## Executive Summary

This spec replaces channel-string inference with one explicit participation contract: every execution (session, task run, loop run, automation target) resolves `local` or `live` once, persists an immutable snapshot with provenance, and every runtime effect — environment, prompts, tools, membership, delivery, usage — projects from that snapshot (ADR-001/006). The flagship experience is the coordinated kanban run: a workspace- or task-scoped "coordination conversations" opt-in (workspace record + `ExecutionProfile` block, ADR-009), invited contextually from the run view, with the conversation watchable in the run detail. Live activation is bounded by deterministic admission (direct/mention only, coalescing, depth) plus budgets settled from the aggregate per-turn usage the runtime already receives (ADR-008) — and every durable wake is a normal `task_runs` record claimed through `ClaimNextRun`, never a second queue (ADR-011). Mailbox mode and every offline/remote concern are out of scope (ADR-004/007); with no remote consumer, the embedded broker is deleted and local delivery becomes durable-commit-then-in-process-dispatch, structurally fixing the persist-before-publish and first-send defects (ADR-010).

Primary trade-offs: a repo-wide hard cut of the old channel fields across contract/storage/CLI/web/docs in one co-shipped release (no aliases, per zero-legacy); token-budget truth measured per turn + wall-time deadline instead of per-provider-attempt interception; and a delivery-path restructure now (mandated anyway by the two-touch rule) in exchange for deleting two P0 defects and the broker overhead.

**MVP Boundary:** the MVP is this entire spec in one public release (ADR-004): participation contract + resolver, schema hard cut, default-local everywhere, coordinated-run opt-in + invitation + run-view conversation, Live admission/budgets/usage visibility, discoverability surfaces, agent-manageable CLI/HTTP/UDS parity, docs/skill/glossary updates, and the delete-target sweep. Post-MVP (separate programs, not staged phases of this one): mailbox/external interaction (ADR-004), configurable spend limits (ADR-005), MoA/swarm strategies, prompt-wrapper micro-optimization beyond the compact wake header, and any remote transport.

## System Architecture

### Component Overview

| Component | Responsibility |
| --- | --- |
| `internal/network/participation` (new leaf package) | Contract types (`Request`, `Spec`, `Bounds`), validation, typed diagnostics, `Resolver` interface. Imports none of task/session/loop/network. |
| `internal/network` (restructured) | Resolver implementation (channel validation/derivation), conversation persistence with immutable recipient dispositions, in-process dispatcher, Live admission (enqueues wake `task_runs`; never claims) + immutable accounting ledger + budget counters, presence, usage attribution feed. Broker transport deleted (ADR-010). |
| `internal/task` | `ExecutionProfile` gains the network block; queue reservation persists the resolved run snapshot; wake runs ride `task_runs` under the `run_kind='network_wake'` generalization (ADR-011/B-009); the deleted `ClaimRun` peer path converges on the existing `ClaimCriteria.RunID` selector (B-001/B-008); ingress policy (`internal/network/tasks.go`) stays separate from worker participation. |
| `internal/session` | `CreateOpts` carries the resolved spec; start/env/lifecycle project from it; join/leave only for `live`. |
| `internal/loop` | Definition/run participation; static validation rejects network-using nodes in `local` loops; action/judge sessions inherit the loop-run snapshot. |
| `internal/automation` | Job task/loop targets carry a typed participation request; dispatch resolves per fire. |
| `internal/coordinator` + `internal/daemon` | Bootstrap decoupled from channels; coordinator participation from its own profile; composition-root wiring; typed task-status projection replaces the dead conversational observer (C-06). |
| `internal/store/globaldb` | Migration v82+ hard cut: drop old channel columns, add snapshot columns, workspace coordination table, session-keyed relation rebuild (ADR-007). |
| `internal/config` + `internal/settings` | `[network]` availability + `[network.live.*]` defaults/ceilings; participation-selecting keys deleted. |
| `internal/api/*` + `internal/cli` | Contract/OpenAPI/TS/CLI co-ship; workspace coordination + usage endpoints/verbs. |
| `web/` | Participation controls, invitation, run-detail conversation + usage, oriented empty states, settings clarification. |

Data flow (send): tool/CLI/API send → validate (participation + authority + security gates) → **one acceptance `BEGIN IMMEDIATE` transaction** (B-010): conversation row + immutable recipient dispositions (pre-message participant snapshot) + per-recipient admission (mode/trigger/idempotency/coalescing/availability/budget) + admitted wakes enqueued as `task_runs` records (`run_kind='network_wake'`, session-targeted) + ledger/sources/counters — all atomic, so there is no crash window between acceptance and schedulable work. Post-commit, dispatch is notify-only (scheduler wake); the durable truth is the enqueued run. The `NetworkWakeRunner` claims via `task.Service.ClaimNextRun`, executes the `PromptNetwork` turn, and settles (outcome + aggregate usage → ledger + budget counters + usage attribution). Non-admitted recipients: durable history only. Restart recovery is the normal `task_runs` recovery — no second queue, no network-owned recovery (ADR-011).

Data flow (coordinated run): run start → resolver (request > profile > workspace record > local) → snapshot persisted in the reservation transaction → `run` strategy creates the run conversation → coordinator/worker sessions bind the run snapshot → run detail streams conversation + usage.

## Architectural Boundaries

- `internal/network/participation` is a leaf: imported by `task`, `session`, `loop`, `automation`, `network`, `api/contract`, `daemon`; imports only stdlib + shared store/error primitives. No subordinate package imports `daemon/`, `api/*`, or `cli/`.
- `internal/daemon` remains the sole composition root: it wires the network-owned `Resolver` into task/session/loop/automation constructors (functional options), the dispatcher into session prompting, and the typed status projection. No back-pointers; interfaces defined where consumed, with the shared contract types in the leaf package.
- `internal/scheduler` keeps wake/observe/sweep only — the run-channel eligibility match is deleted; it never claims and never resolves participation (L-005).
- `internal/network` admission enqueues wake `task_runs` and settles accounting; it never claims, executes, or recovers work — `task.Service.ClaimNextRun` remains the single claim authority for every run, wake runs included (ADR-011; B-001/B-002).
- The broker was the only NATS site (`internal/CLAUDE.md` boundary rule); after ADR-010 no package links NATS. `magefiles/boundaries.go` gains the `participation` leaf rule in the same commit.
- `internal/api/core` stays the canonical handler home; HTTP/UDS only register/authenticate.

## Implementation Design

### Core Interfaces

```go
// internal/network/participation — contract (leaf package)
type Mode string
const (
    ModeLocal Mode = "local"
    ModeLive  Mode = "live"
)

type ChannelStrategy string
const (
    StrategyNamed   ChannelStrategy = "named"
    StrategyRun     ChannelStrategy = "run"
    StrategyLoopRun ChannelStrategy = "loop_run"
)

// Request is explicit caller/definition intent; pointer presence = provided.
type Request struct {
    Mode            *Mode            `json:"mode,omitempty"`
    ChannelStrategy *ChannelStrategy `json:"channel_strategy,omitempty"`
    ChannelID       *string          `json:"channel_id,omitempty"`
    Bounds          *BoundsRequest   `json:"bounds,omitempty"`
}
```

```go
// Spec is the immutable resolved snapshot persisted on the owning record.
type Spec struct {
    Version         string          `json:"version"` // "network-participation/v1" (brand-neutral)
    Mode            Mode            `json:"mode"`
    WorkspaceID     string          `json:"workspace_id,omitempty"`
    ChannelStrategy ChannelStrategy `json:"channel_strategy,omitempty"`
    ChannelID       string          `json:"channel_id,omitempty"`
    Source          Source          `json:"source"`
    Bounds          Bounds          `json:"bounds,omitzero"` // resolved; live only
}

type Source string
const (
    SourceExplicitRequest       Source = "explicit_request"
    SourceTaskProfile           Source = "task_profile"
    SourceWorkspaceCoordination Source = "workspace_coordination"
    SourceLoopDefinition        Source = "loop_definition"
    SourceAutomationJob         Source = "automation_job"
    SourceBuiltInLocal          Source = "built_in_local"
)
```

```go
// Bounds are finite by contract; zero is invalid for live (never "unlimited").
// Durations are canonical strings on every wire ("5m"); parsed post-validation.
type Bounds struct {
    MaxWakes         int    `json:"max_wakes"`
    MaxWakeWallTime  string `json:"max_wake_wall_time"`
    MaxTotalWallTime string `json:"max_total_wall_time"`
    MaxInputTokens   int64  `json:"max_input_tokens"`
    MaxOutputTokens  int64  `json:"max_output_tokens"`
    MaxWakeDepth     int    `json:"max_wake_depth"`
    CoalesceWindow   string `json:"coalesce_window"`
}

// Resolver is the ONLY owner of precedence and channel derivation.
type Resolver interface {
    Resolve(ctx context.Context, in ResolveInput) (Spec, error)
}

type ResolveInput struct {
    WorkspaceID string
    Owner       OwnerRef // {Kind: session|task_run|loop_run|automation_run, ID}
    Request     *Request // per-execution explicit intent (nil = none)
    Definition  *Request // owning profile/definition intent (nil = none)
    RunID       string   // required for StrategyRun
    LoopRunID   string   // required for StrategyLoopRun
}
```

```go
// internal/store/globaldb — the SINGLE owner of the acceptance transaction
// (B-010/B-018). One BEGIN IMMEDIATE covers: conversation row, recipient
// dispositions, per-recipient admission checks against transactional state
// (network_availability row, network_wake_sources dedup, owner-level
// open-wake unique, budget counters), wake task_runs enqueue, and ledger/
// sources writes. The network layer resolves all policy inputs beforehand
// (eligibility verdicts, bounds, owner keys) from immutable data; nothing
// outside this method touches the transaction.
func (g *GlobalDB) AcceptNetworkMessage(
    ctx context.Context, req AcceptNetworkMessageRequest,
) (AcceptNetworkMessageResult, error)

type AcceptNetworkMessageRequest struct {
    Message      NetworkMessageRow     // validated envelope projection
    Dispositions []MessageDisposition  // pre-message participant snapshot decisions
    Admissions   []WakeAdmissionInput  // per recipient: owner key, resolved bounds, eligibility verdict, wake-run template
}

type AcceptNetworkMessageResult struct {
    AcceptanceSeq int64
    Admitted      []WakeReservation
    Skipped       []WakeSkip              // typed reasons: not_live, not_addressed, duplicate, depth_exceeded, coalesced, budget_exhausted, network_disabled
    Notify        []CommittedNotification // post-commit scheduler-wake/SSE signals; fired by the caller only after commit
}
```

```go
// internal/network — settlement stays a slim ledger-service contract.
type WakeSettler interface {
    // Settle applies the wake run's terminal outcome + aggregate turn usage
    // as an idempotent open→terminal compare-and-set (B-011): a duplicate
    // terminal delivery no-ops and cannot double-apply counters. Failed
    // turns are never marked delivered (C-03); missing usage keeps the
    // reserved quantities consumed and marks usage_unavailable.
    Settle(ctx context.Context, res WakeReservation, out WakeOutcome) error
}
```

```go
// WakeReservation is the durable admission evidence, keyed one-to-one to the
// canonical task run (task_run_id is UNIQUE on the ledger). One in-flight
// wake per owner — enforced by the ledger's partial unique index on
// (owner_key) WHERE state='open' — is the overshoot bound (invariant 11).
type WakeReservation struct {
    WakeID           string   // ledger id (accounting identity)
    TaskRunID        string   // canonical queue record (task_runs); UNIQUE
    OwnerKey         string   // canonical participation owner ("task_run:<id>" | "session:<id>" | "loop_run:<id>")
    EnvelopeIDs      []string // coalesced source envelopes
    ReservedWallTime string   // per-wake deadline (canonical duration)
    CoalesceUntil    string   // window cutoff: sources may attach until claim or this instant, whichever first
}

type WakeOutcome struct {
    State              string // succeeded | failed | canceled | deadline_exceeded
    ActualInputTokens  int64
    ActualOutputTokens int64
    UsageState         string // actual | usage_unavailable
}
```

```go
// internal/task — manual exact-run claim converges on the SAME primitive
// (B-001/B-008): ClaimCriteria.RunID ALREADY EXISTS at HEAD and is applied
// as `tr.id = ?` inside the authoritative ClaimNextRun selection
// (globaldb.selectClaimableRunID). No new selector is introduced.
// task.Service.ClaimRun and its API/CLI/native-tool callers are delete
// targets; manual exact claims flow through Service.ClaimNextRun →
// GlobalDB.ClaimNextRun via the existing RunID criterion with identical
// atomic selection, claim-token issuance, lease fencing, hooks, and events.
type ClaimCriteria struct {
    // ...existing fields, including:
    RunID           string // existing exact selector: claim exactly this queued run or fail typed
    RunKind         string // existing kind filter; the enum gains RunKindNetworkWake
    TargetSessionID string // NEW (B-017): claim only wake runs targeted at this session
}

// Kind-aware hard cut through the claim path (B-017): ClaimResult.Task
// becomes *Task — nil exactly when run_kind='network_wake' (kind-fenced,
// mirrors the schema CHECKs); selection replaces the task INNER JOIN with
// kind-fenced predicates (task join applies to task-anchored kinds only);
// claim skips tasks.current_run_id for wake runs; heartbeat/release/
// complete/fail operate on the run row with token fencing and skip task
// status reconciliation for wake runs; recovery requeues expired wake runs
// without task reconciliation; hooks/events carry owner_key with empty task
// correlation; list/read projections branch by kind (filter/badge). The
// affected selection/result/lifecycle files join the Delete Targets sweep.
```

```go
// internal/daemon — composition-root-owned executor for wake runs (B-009).
// It claims network-wake runs for their target session through the normal
// primitive (ClaimNextRun with run_kind + target-session criteria), executes
// the PromptNetwork turn, heartbeats with the claim token, and settles the
// terminal outcome against the wake ledger. It never touches task state.
type NetworkWakeRunner interface {
    Run(ctx context.Context) error // long-lived; detached lifetime, L-001
}
```

```go
// internal/workspace — coordination setting (written by the invitation).
type CoordinationSettings interface {
    Get(ctx context.Context, workspaceID string) (CoordinationSetting, error)
    Set(ctx context.Context, workspaceID string, enabled bool, actor string) (CoordinationSetting, error)
}

type CoordinationSetting struct {
    WorkspaceID string `json:"workspace_id"`
    Enabled     bool   `json:"enabled"`
    Revision    int64  `json:"revision"` // transactionally incremented; last-write oracle (B-014)
    UpdatedAt   string `json:"updated_at"`
    UpdatedBy   string `json:"updated_by"`
}
```

Error convention: the participation package exports sentinel diagnostics — `ErrUnavailable` (`network_participation_unavailable`), `ErrChannelUnknown`, `ErrStrategyInvalid`, `ErrStrategyChannelConflict`, `ErrAuthorityDenied`, `ErrBoundsExceedCeiling`, `ErrLoopRequiresLive`, `ErrLiveUnsupported` — matched with `errors.Is`, mapped once in `api/core` to stable machine-readable codes used identically by HTTP/UDS/CLI/native tools.

### Data Models

Migration **v82+** (append-only tail after v81 `add_goal_adoption_attempt`; destructive alpha rebuild, no legacy reads — L-008 exception documented here as Delete Targets):

| Column | Shape | Purpose |
| --- | --- | --- |
| `sessions.network_spec_json` | `TEXT NOT NULL` — the v82+ migration **backfills every retained pre-cut row with the canonical Local snapshot** (`{"version":"network-participation/v1","mode":"local","source":"built_in_local"}`) plus matching projections; an empty or undecodable value is impossible by construction (B-012) | The authoritative, lossless immutable `Spec` snapshot (version, mode, strategy, channel, source, bounds) — nothing re-derivable after restart (B-004). The same backfill rule applies to `task_runs` and `loop_runs` snapshot columns. |
| `sessions.network_mode` | `TEXT NOT NULL DEFAULT 'local'` | SQL-matchable projection of the snapshot, written in the same transaction; replaces semantic of `sessions.channel` (deleted). |
| `sessions.network_channel` | `TEXT` | Matchable projection; empty for local. |
| `sessions.network_source` | `TEXT NOT NULL DEFAULT 'built_in_local'` | Matchable provenance projection (PRD rule 7). |
| `task_runs.network_spec_json` / `.network_mode` / `.network_channel` / `.network_source` | same shapes | Immutable per-attempt snapshot + projections; replaces `task_runs.network_channel` + `task_runs.coordination_channel_id` (both deleted, with indexes). |
| `task_runs` wake generalization (explicit hard cut, B-009) | `run_kind` gains `'network_wake'`; `task_id` becomes nullable with kind fencing — `CHECK (run_kind='network_wake' OR task_id IS NOT NULL)` and `CHECK (run_kind<>'network_wake' OR task_id IS NULL)`; correlation columns `network_wake_id TEXT`, `network_target_session_id TEXT`, `network_owner_key TEXT` (destructive table rebuild in v82+) | The durable wake IS a task_run (ADR-011): enqueued by admission inside the acceptance transaction, claimed only via `ClaimNextRun` (run_kind + target-session criteria), recovered by normal run recovery — no parallel queue (L-003/L-005). Kind-scoped semantics: the open-run-per-task reservation rule, `tasks.current_run_id` projection, and task status reconciliation apply only to task-anchored kinds; `network_wake` runs never touch task/kanban state and are filtered/badged truthfully in run lists. `origin_kind` remains the untouched ingress enum. |
| `task_execution_profiles.network_mode` | `TEXT NOT NULL DEFAULT ''` | Task-scoped requested intent ('' = unset → inherit). |
| `task_execution_profiles.network_channel_strategy` / `.network_channel` | `TEXT` | Requested strategy + named channel. |
| `task_execution_profiles.network_bounds_json` | `TEXT` | Requested bound overrides. |
| `workspace_network_coordination` (new) | `workspace_id TEXT PK, enabled INTEGER NOT NULL, revision INTEGER NOT NULL, updated_at TEXT NOT NULL, updated_by TEXT NOT NULL` | The persisted workspace opt-in (ADR-009); side table because it is workspace-matchable state consulted by every coordinated-run resolution. `revision` increments inside the committing transaction — the deterministic last-write oracle (B-014). |
| `network_coordination_invitations` (new) | `scope_kind TEXT, scope_id TEXT, dismissed_at TEXT NOT NULL, dismissed_by TEXT NOT NULL, PRIMARY KEY(scope_kind, scope_id)` | Persisted invitation dismissal per workspace/task scope (B-007, US-010); reset = row delete via API/CLI; daemon truth, never browser-local. |
| `loop_runs.network_spec_json` / `.network_mode` / `.network_channel` / `.network_source` | same shapes as `task_runs` | Loop-run resolved snapshot on the existing `loop_runs` record (table named per B-004). |
| `network_thread_participants.session_id`, `network_subscriptions.session_id`, `network_direct_rooms.session_a/session_b`, `network_work` owner/target | `TEXT NOT NULL` (rebuild) | Session-keyed durable relations replacing peer-id keys (ADR-007); destructive rebuild, old peer-keyed rows reset. |
| `network_message_dispositions` (new) | `message_id TEXT, recipient_session_id TEXT, decision TEXT, decided_at TEXT, acceptance_seq INTEGER, PRIMARY KEY(message_id, recipient_session_id)` | Immutable recipient-decision evidence written in the acceptance transaction from the pre-message participant snapshot; admitted wakes are enqueued in the same commit, so dispositions are pure evidence — never a pollable pending queue and never recomputed (invariant 20; B-002/B-010). |
| `network_live_wakes` (new, accounting only) | `wake_id TEXT PK, task_run_id TEXT NOT NULL UNIQUE, owner_key TEXT NOT NULL, workspace_id, channel, root_id, depth INTEGER, state TEXT, coalesce_until TEXT, reserved_wall_ms INTEGER, actual_wall_ms INTEGER, reserved_at, settled_at, input_tokens INTEGER, output_tokens INTEGER, usage_state TEXT, reason TEXT` + partial `UNIQUE(owner_key) WHERE state='open'` | Immutable admission/usage accounting keyed one-to-one to the canonical `task_run_id`. The owner-level partial unique IS the one-in-flight overshoot bound (B-011); root/window coalescing is separate logic: while the owner's open wake exists and `now < coalesce_until` and the run is unclaimed, new eligible envelopes attach as sources; after claim or cutoff they accumulate. `Settle` is an atomic `open → terminal` compare-and-set — duplicate terminal delivery no-ops. `reserved_wall_ms` is the admission reservation; `actual_wall_ms` is settled truth (missing usage keeps reserved values consumed). Owns NO pending work, polling, claim, retry, or recovery authority (B-002). |
| `network_wake_sources` (new) | `owner_key TEXT, envelope_id TEXT, wake_id TEXT NOT NULL, PRIMARY KEY(owner_key, envelope_id)` | Same-envelope idempotency: one envelope admits at most once per owner, forever (B-003); links coalesced sources to their wake. |
| `network_participation_budgets` (new) | `owner_key TEXT PK, wakes_used INTEGER, wall_ms_used INTEGER, input_tokens_used INTEGER, output_tokens_used INTEGER, exhausted_reason TEXT, updated_at TEXT` | Budget counters mutated only inside the acceptance/settlement `BEGIN IMMEDIATE` transactions — the serialization point that prevents concurrent overdraw (B-003); settlement replaces reserved wall-time with actual and applies token usage exactly once (CAS-fenced by the ledger state). |
| `network_availability` (new) | `id INTEGER PRIMARY KEY CHECK (id = 1), enabled INTEGER NOT NULL, epoch INTEGER NOT NULL, updated_at TEXT NOT NULL, updated_by TEXT NOT NULL` | Persisted availability state (B-013): config apply/reload for `network.enabled` writes this row transactionally, and admission reads it inside the acceptance transaction — one SQLite serialization domain linearizes disable against concurrent admission; `epoch` increments on every transition so re-enable is distinguishable and cannot re-admit settled/accumulated envelopes. |

**Side-table-vs-JSON decisions:** `network_spec_json` is the authoritative immutable snapshot (read whole, never SQL-filtered — a validated document, not a metadata bag); `network_mode`/`network_channel`/`network_source` are same-transaction typed projections because they are matched, filtered, and listed by SQL. A JSON bag of matchable or ownership state remains forbidden (L-003): budgets, wake sources, dispositions, coordination toggle, and invitation dismissals are all typed side tables with explicit keys/constraints. Automation job participation stays a typed field inside the existing job-config payload (definition payload, not matchable queue state); the automation fire record echoes the request for audit, and the resolved snapshots live on the runs the fire materializes.

### Participation Ownership Matrix (B-004)

One canonical `OwnerKey` (`kind:id`) identifies the budget/usage/event owner everywhere (ledger, budgets, admission, events, recovery):

| Execution | Owner of the persisted snapshot | Relationship |
| --- | --- | --- |
| User/plain session | `session:<id>` | Owns its own snapshot. |
| Task run | `task_run:<id>` | Owns; worker and task-role sessions **bind** this snapshot (same-execution projection — not children, no re-resolution). |
| Loop run | `loop_run:<id>` | Owns; action/judge sessions bind it (projection). One budget per loop run. |
| Automation fire | — | Materializes task/loop runs which own their snapshots; the fire record echoes the request. |
| Coordinator session | `session:<id>` (its own, from coordinator profile) | Never bound to an observed run's snapshot. |
| Spawn / review / detached | own key | **Child executions**: resolve independently, default local (invariant 4). |

"Inherit" language is reserved for projections (same execution, one owner, one budget); genuine children never inherit. Live wakes target the bound session but charge the owner's budget via `OwnerKey`.

### API Endpoints

No transport duplication: all in `api/core` `BaseHandlers`, registered by HTTP + UDS; contract → `openapi/agh.json` → generated TS via `make codegen` in the same change.

| Method/Path | Change |
| --- | --- |
| `POST /api/sessions` (+UDS) | `channel` deleted; optional `network_participation` request object; response carries `resolved_network_participation` (spec) — unknown legacy field fails validation. |
| Task create/profile/start/enqueue/fan-out | Profile carries the request block; start/enqueue/fan-out accept a per-run override; run payloads project the resolved spec; `network_channel`/`coordination_channel_id` fields deleted everywhere. |
| Loop config/run (+validate/dry-run) | Definition + run override; dry-run echoes the resolved spec; `local` loop with network-using nodes → typed `loop_requires_live` validation item. |
| Automation job payloads | `JobTaskConfig.NetworkChannel` → typed participation request on task and loop targets. |
| `GET/PUT /api/workspaces/{id}/network-coordination` (new) | Read/write the coordination setting; PUT is what the invitation's accept calls; deterministic errors (`network_participation_unavailable` when admin-disabled). GET includes invitation dismissal state for the scope. |
| `PUT /api/workspaces/{id}/network-coordination/invitation` (new) | Persist/reset invitation dismissal per scope (`{scope: workspace|task, task_id?, dismissed: bool}`) into `network_coordination_invitations` — daemon truth for US-010 dismissal (B-007). |
| `GET /api/workspaces/{id}/network/usage` (new) | Wake-ledger aggregation by run/channel/window; values labeled `actual` or `usage_unavailable`. |
| Run detail + SSE | Adds resolved participation, conversation projection reference, live bound consumption/exhaustion reason, and usage. |
| Network status/routes | Report `disabled | ready | active` (broker states deleted). |

### CLI (agent-operable, structured output)

- `agh session new --network local|live --network-channel-strategy named|run --network-channel <id>`; bound overrides only via `--network-bounds '<structured object>'` (validated against the same contract — no 13-flag surface).
- `agh task profile update ... --network ...` (same flag trio on profile/start/enqueue/fan-out overrides).
- `agh network coordination status|enable|disable --workspace <id> -o json` — the CLI twin of the invitation; `agh network coordination invitation dismiss|reset --workspace <id> [--task <id>] -o json` manages the persisted dismissal state.
- `agh network usage --workspace <id> [--run <id>] -o json`.
- `agh network status -o json` reflects availability + participation counts; `agh ch ...` addressing commands keep `--channel` (they address a channel, not participation) and return `not_participating` diagnostics for local sessions.
- Every command returns the resolved spec + source; errors are the same machine-readable codes as HTTP/UDS.

## Integration Points

None external. Providers are reached through the existing ACP session layer; Live settlement consumes the aggregate per-turn usage already surfaced by observe aggregation (`internal/observe/observer.go:673-698`). No new outbound calls, auth, or sidecars.

## Safety Invariants

1. Participation resolves exactly once per execution lifetime, before any network side effect, and the persisted snapshot never mutates in flight (config/profile edits affect future executions only).
2. Precedence is fixed: execution request > owning definition (profile/loop/job) > workspace coordination record (coordinated task runs only) > built-in `local`; the winning source persists on the snapshot.
3. `local` creates zero network state: no channel row, membership, subscription, presence, environment variables, prompt section, tool projection, or wake — enforced by projection from the snapshot, never by call-site discipline.
4. No inheritance for children; binding for projections. Genuine child executions (spawn/review/detached/automation-materialized work) resolve independently and default `local`; a child's `live` request must fall within the parent's delegated channel authority or fails `network_participation_permission_denied`. Worker, task-role, action, and judge sessions are same-execution projections: they **bind** the owning run/loop-run snapshot (one `OwnerKey`, one budget) and never re-resolve (Participation Ownership Matrix).
5. Requesting `live` while `network.enabled=false` fails with `network_participation_unavailable` before the owning record is created; no silent downgrade, ever.
6. A `named` channel must already exist in the same workspace; `run`/`loop_run` derivation happens only inside the Resolver; every other derivation/fallback path is deleted.
7. A `local` loop whose graph references network capabilities fails validation at save/start; the runtime never upgrades participation because a tool name appeared.
8. Channel identity is `(workspace_id, channel)` in every query and cache key (fixes C-01); cross-workspace reads/writes are impossible by construction and covered by same-name negative tests.
9. Only direct messages and mentions resolved to a live participant are wake-eligible; `greet`, `whois`, `receipt`, `trace`, presence, status projections, and acknowledgements are structurally ineligible (checked before admission, not by prompt policy).
10. Acceptance is one store-owned method — `globaldb.AcceptNetworkMessage` — whose single `BEGIN IMMEDIATE` transaction performs, atomically: conversation row, dispositions, trigger eligibility application, same-envelope idempotency (`network_wake_sources` primary key), owner-level open-wake check (partial unique on `(owner_key) WHERE state='open'`), depth check, the persisted availability read (`network_availability`), budget-counter mutation, and wake `task_runs` enqueue; post-commit notifications are returned to the caller, never fired inside (B-010/B-018). A crash after commit leaves schedulable runs (normal recovery); a crash before commit leaves nothing — there is no window in between.
11. One in-flight wake per owner — enforced directly by the ledger's `UNIQUE(owner_key) WHERE state='open'` — is the stated overshoot bound: aggregate usage cannot stop an already-running turn mid-flight, so at most one turn per owner may exceed the remaining token budget, and that turn is bounded by its per-wake wall-time deadline (ADR-008; B-011). Root/window coalescing is separate attach logic under the open wake's `coalesce_until` cutoff and stops at claim time (claim-vs-coalesce rule).
12. Zero/absent bounds are a validation failure for `live` — never unlimited; bounds resolve request > profile > configured defaults, capped inclusively by configured ceilings.
13. Burst arrivals while the owner's open wake accepts sources (unclaimed and before `coalesce_until`) attach to that wake; each source envelope stays individually durable and queryable via `network_wake_sources`; arrivals after claim or cutoff accumulate as history.
14. Wake depth is enforced from persisted causation; at the cap, messages accumulate instead of waking (no ping-pong).
15. Settlement is truthful and idempotent: an atomic `open → terminal` compare-and-set on the ledger fences duplicate terminal delivery (a replay no-ops and cannot double-apply counters); terminal error/cancel outcomes are recorded (never "delivered"); actual usage and wall-time replace reserved quantities exactly once; missing usage keeps the reserved quantities consumed and is labeled `usage_unavailable` — estimates are never presented as actual (C-03; B-011).
16. Budget exhaustion flips the participation to accumulate-only with a visible reason; the same envelope can never be admitted twice — `network_wake_sources` persists across restart, exhaustion, and availability re-enable.
17. The durable wake is a `task_runs` record under the explicit substrate generalization (B-009): `run_kind='network_wake'`, `task_id` NULL with kind-fenced CHECK constraints, session-targeted claim criteria, claimed only through `task.Service.ClaimNextRun`, executed by the daemon-owned `NetworkWakeRunner` with token-fenced heartbeat and terminal transitions. Wake runs never mutate task/kanban state (`tasks.current_run_id`, task status reconciliation, and open-run-per-task rules are task-anchored-kind-only); `origin_kind` remains the untouched ingress enum. `network_live_wakes` is immutable accounting keyed one-to-one to `task_run_id` and owns no pending work, polling, claim, retry, or recovery authority; restart recovery is the normal run recovery (L-003/L-005; ADR-011).
18. `ClaimNextRun` is the only claim transition in the system: `task.Service.ClaimRun` and every API/CLI/native-tool caller are deleted; manual exact-run claims use the **existing** `ClaimCriteria.RunID` selector through the same atomic selection, claim-token issuance, lease fencing, hooks, and events — no second exact selector exists (B-001/B-008; L-004/L-005).
19. The conversation is evidence, never authority: no message mutates run status, claims, verdicts, or terminal state (L-005); the task-status projection is typed state that cannot invoke `PromptNetwork` (C-06 replacement).
20. Message acceptance is one store transaction: conversation row + `network_message_dispositions` (pre-message participant snapshot, with `acceptance_seq`) + admission outcomes + admitted wake runs commit atomically (B-010); post-commit dispatch is notify-only and never recomputes recipients — dispositions are evidence, never a pollable pending queue (fixes C-02/C-05 structurally; ADR-010/011).
21. `network_work` lifecycle authority is SQLite only; the in-memory router map is deleted (C-04).
22. Live relations (participants, subscriptions, direct rooms, work) key on session identity; process peer ids are presence metadata only; session restart preserves them (ADR-007).
23. Dispatch and wake execution detach from request lifetimes via `context.WithoutCancel` with re-attached deadlines (L-001); wake deadlines cancel provider work where supported, and non-cancelable work stays charged and observed to terminal.
24. Availability is persisted state, not an in-memory flag (B-013): config apply/reload for `network.enabled` writes the `network_availability` row (enabled + monotonic epoch) transactionally, and admission reads that row inside the acceptance transaction — one SQLite serialization domain is the shared linearization primitive. An admission commits strictly before or strictly after the disable write; post-disable admissions fail and accumulate with reason `network_disabled`; in-flight wakes receive cancel-requested (owned by the wake runner) and settle truthfully; re-enable advances the epoch and never re-admits settled or accumulated envelopes (`network_wake_sources` persists) (B-006; PRD rule 15).
25. Security non-regression — token redaction: the canonical accept path validates before any durable write or dispatch and recursively rejects raw `claim_token`/secret material in body, proof, ext, and metadata across HTTP, UDS, CLI, native tools, and the extension host; only `claim_token_hash` may appear in diagnostics, events, logs, SSE, or memory (B-005; internal/CLAUDE.md Security Invariants).
26. Security non-regression — identity classification: a verified-format `nickname@fingerprint` identity without a valid proof classifies as `rejected`, never `unverified`; the rewritten validation path preserves this with canonical-suite regressions (B-005).
27. The workspace coordination record, invitation dismissals, wake ledger, budgets, usage rows, and all conversation state are workspace-scoped; every list/read path filters by authenticated workspace before channel.
28. Invitation acceptance affects future executions only — an in-flight run's immutable snapshot cannot be enrolled — and the invitation copy states exactly that; dismissal is daemon-persisted per scope (`network_coordination_invitations`) with explicit reset; browser-local dismissal state is forbidden (B-007; US-010).

## Impact Analysis

| Component | Impact | Description / Risk | Action |
| --- | --- | --- | --- |
| `internal/network` | restructured | Broker deletion + dispatcher + admission/ledger; over-cap files (`manager.go` 2095, `delivery.go` 2012, `router.go` 1787, `validate.go` 1085) must shrink into single-responsibility files. High risk → phase 5 isolates it. | New files: resolver, dispatch, admission, budget, presence, usage; delete `transport.go`. |
| `internal/network/participation` | new leaf | Contract + validation; low risk, fully unit-testable. | New package + boundaries rule. |
| `internal/task` | modified | Profile block, reservation resolution, ingress split, hooks payload swap. Medium. | Types swap at `internal/task/types.go:410-739`; profile at `profile.go:53-141`. |
| `internal/session` | modified | CreateOpts spec, env/join gating (`manager_types.go:18-31`, `manager_start_env.go:69-71`, `manager_helpers.go`). Medium (startup path). | Projection from snapshot. |
| `internal/coordinator` + `internal/daemon` | modified | Bootstrap gate removal (`coordinator.go:100`), allowlist split (`:50-64`), overlay conditional (`:235-258`), session create (`coordinator_runtime.go:670-676`), status projection, boot wiring (`boot.go:1298`). Medium-high (autonomy path) → invariant 19 + local-coordinator journey. | Decouple + typed projection. |
| `internal/loop` / `internal/automation` | modified | Adapter channel deletion (`loop_runtime_adapters.go:49,267-315`), static validation, job config swap (`automation/model/types.go:116-120`, `dispatch.go:677,1185`). Medium. | Inherit loop-run snapshot. |
| `internal/store/globaldb` | modified | Migration v82+ hard cut + relation rebuild + workspace/wake tables. High (destructive) → fresh + reopen tests. | Append-only tail. |
| `internal/config`/`settings` | modified | New `[network.live.*]`; delete participation-selecting keys. Low. | See Config Lifecycle. |
| `internal/api/*`, `internal/cli` | modified | Contract swap + new endpoints/verbs; breaking (alpha-OK). | `make codegen` co-ship. |
| `internal/scheduler` | modified | Delete run-channel eligibility match. Low. | Local matching only. |
| `internal/situation`, `internal/tools` | modified | Sections + coordination toolset gated on participation (kills 18-descriptors-to-every-session cost). Low-medium. | Projection from snapshot. |
| `web/` | modified | Controls, invitation, conversation panel, usage, empty states, settings copy. Medium. | See Web/Docs Impact. |
| Docs/skill/glossary | modified | Bind-always doctrine deleted; official skill rewrite. | Same release. |

## No Fallback / No Compat / No Placeholder

No aliases, dual fields, fallback reads, legacy parsing, or inferred modes survive this cut. Unknown legacy fields (`channel`, `network_channel`, `coordination_channel_id`) fail schema validation. The destructive migration resets old auto-created coordination channels and peer-keyed relation rows; there is no read-time repair of pre-cut state. Any surface that cannot ship its replacement in the same release blocks the release — there is no temporary hybrid.

## Delete Targets

- `task.Task.NetworkChannel` (`types.go:416`), `task.Run.NetworkChannel` (`:482`), `task.Run.CoordinationChannelID` (`:487`), request fields (`:707,733,739`), run-view projections (`:539,628,657,679,1019,1031`).
- Auto-derivation path: `coordinationChannelIDForQueuedRun`/`derivedRunCoordinationChannelID`/`ensureQueuedRunCoordinationChannel` (`global_db_task_aux.go:804-880`) — derivation survives only inside the Resolver for explicit `run` strategy.
- `session.CreateOpts.Channel` (`manager_types.go:31`), contract `CreateSessionRequest.channel` (`contract.go:29`) + response projection (`:101`), channel env inference (`manager_start_env.go:69-71`).
- Coordinator: `DecisionMissingChannel` gate (`coordinator.go:100-143`), unconditional network tools in `toolAllowlist` (`:50-64`), unconditional channel guidance (`:235-258`), channel-bound session create (`coordinator_runtime.go:670-676`).
- Loop unconditional channels (`loop_runtime_adapters.go:49`, `loopRuntimeSessionChannel` `:267-315`); automation `JobTaskConfig.NetworkChannel` (`model/types.go:120`) + forwarding (`dispatch.go:677,1185`).
- Harness `ChannelBound` inference (`harness_context.go:171-172,321-326,535`); task-role fictional `default` channel; review/detached/spawn channel inheritance; hooks dual fields (`payloads.go:956-957`) + fallback (`manager_run_hooks.go:180-187`).
- Scheduler run-channel eligibility match; `network_task_status_observer.go` conversational say path (replaced by typed projection).
- Embedded broker, full dependency sweep (N-001): `internal/network/transport.go`, every direct NATS import, subscription ownership, publish/broadcast/flush machinery in `internal/network/manager.go` (`:286-296` transport start; `:1166-1191`-adjacent publish path via router), broker-dependent tests, the NATS module dependency in `go.mod` (via `go get`, never hand-edit), broker config keys, and protocol/site pages describing broker subjects (ADR-010); capability-scored auto-activation (`activation_top_k` routing) and digest batching machinery (`delivery.go` digest paths); per-delivery tutorial guidance blocks + `network_delivery_guidance_state` table; heartbeat greets persisted as conversation timeline (`manager.go:868-881` behavior — presence stays operational state).
- Manual peer claimer (B-001/B-008): `task.Service.ClaimRun` (`internal/task/manager_run_claim.go:119`) and every caller — API handler (`internal/api/core/tasks.go:1524`), contract fields, CLI verb wiring, native-tool descriptor paths — converged on the **existing** `ClaimCriteria.RunID` exact selector through `ClaimNextRun`; no adapter retains the old mutation and no second exact selector is introduced.
- Task-anchored claim assumptions (B-017): the task `INNER JOIN` selection predicates (`global_db_task_claim_select.go`), mandatory-`Task` claim results and task-state reconciliation branches in claim/heartbeat/complete/fail/recovery (`global_db_task_claim*.go`, `internal/task/lease_manager.go`) are replaced by the kind-fenced contracts — task-anchored kinds keep identical behavior; `network_wake` runs skip task projections entirely.
- Config keys: `network.default_channel`, `network.port`, `network.max_payload`, `network.activation_top_k`, `network.digest_flush_interval`, `network.digest_max_envelopes`, `network.response_guidance_max_bytes`, `network.delivery_structured_body_max_bytes`; bundle `BindPrimaryChannelAsDefault` + effective-default channel resolution (`api/core/bundles.go:177-195`).
- SQLite: `tasks.network_channel` + index (`global_db.go:81`), `task_runs.network_channel` (`:677`) + `coordination_channel_id` (`:711`) + indexes (`:726,910`), `sessions.channel`, peer-keyed relation columns/indexes, guidance-state table.
- Tests freezing old behavior (auto-channel expectations in task/store suites), glossary "Bind always, speak when useful" doctrine, autonomy site pages' every-run-channel language, generated contracts (regenerated).

## Testing Approach

Unit + integration in canonical suites (extend, never duplicate — `eng-consolidate-test-suites`): participation validation/resolution table-driven in the new leaf package; store/reservation/coordinator/session/loop/automation invariants in their existing manager/integration suites; delivery/admission in network suites with fake clocks and a stub prompt runner at the I/O boundary; migration fresh-DB + reopen-after-restart per L-008. Runtime E2E (`make test-e2e-runtime`) covers the local-default and coordinated-run journeys against `acpmock` (runtime-contract co-ship, L-007); web E2E covers invitation → enable → conversation. Every concrete case with IDs lives in `_tests.md`. Scoped lanes during development; one `make verify` at completion.

## Development Sequencing

### Build Order

1. **Contract + resolver** — `internal/network/participation` types/validation/diagnostics; resolver implementation + boundaries rule. No dependencies.
2. **Schema hard cut** — migration v82+ (columns, workspace table, wake ledger, relation rebuild) + store read/write swaps; C-01 workspace qualification fixed here.
3. **Owner wiring** — task profile/reservation, session create, loop definition/run, automation dispatch resolve-and-persist; all derivation/inheritance/fallback delete targets in those layers removed (compiler enumerates consumers).
4. **Autonomy decoupling** — coordinator bootstrap/allowlist/overlay, scheduler match removal, task-role identity, typed status projection (C-06), synthetic wake untouched; `ClaimRun` deleted and manual exact claims converged on the existing `ClaimCriteria.RunID` (B-001/B-008); `task_runs` substrate generalization for wake runs lands here (kind, nullable task_id CHECKs, projections fencing; B-009).
5. **Delivery restructure** — broker deletion (full NATS dependency sweep incl. manager call sites, `go.mod`, tests; N-001), acceptance transaction with persisted recipient dispositions (C-02/C-05), single work authority (C-04), session-keyed relations (ADR-007), file splits.
6. **Live admission + usage** — admission inside the acceptance transaction (idempotency/owner-level in-flight constraints, persisted availability state, budget counters) enqueuing wake `task_runs` + `NetworkWakeRunner` executor (ADR-011/B-009/B-010), CAS settlement (C-03 outcome; B-011), usage attribution + query; adversarial multi-connection concurrency tests incl. distinct roots, claim-vs-coalesce races, duplicate settlement, restart replay (B-003/B-011).
7. **Projection gating** — env, prompts, situation sections, coordination toolset availability, compact wake header.
8. **Public surfaces** — contract/OpenAPI/TS/CLI/web co-ship: participation controls, coordination endpoints/verbs, invitation, run-detail conversation/usage, empty states, settings; codegen.
9. **Docs/skill/sweep** — glossary/autonomy/site/official-skill rewrites, QA tracker rows, repository-wide delete-target search gate, final `make verify`.

### Technical Dependencies

- Migration tail coordination: v82+ numbers claimed after whatever is at tail when implementation starts (currently v81); one worktree owns registry edits.
- Product-line merge alignment (ADR-004): brand-neutral persisted identifiers here; the merge's broker-subject rename item is superseded by ADR-010 — flag to that program.
- Design-phase input for invitation/empty-state composition (design system flow) before phase 8 web work.

## Monitoring and Observability

Canonical events with correlation keys (`workspace_id`, `session_id`, `run_id`, `owner_key`, `channel`, `wake_id`, `root_id`, `source`, `acceptance_seq`, `actor_kind`, `actor_id`, and `claim_token_hash` where a task-run correlation exists — never the raw token; N-007): `network.participation.resolved` (mode/source/channel), `network.coordination.setting_changed` (enabled/actor), `network.coordination.invitation_dismissed|reset`, `network.wake.admitted|coalesced|skipped|exhausted|settled` (reason, depth, reserved/actual usage, `usage_state`), `network.message.accepted|dispatched` (with `acceptance_seq`), `network.projection.task_status` (zero-activation proof), availability transitions. Coverage-matrix test extends to these events and asserts redaction (raw `claim_token` absent from every emitted payload). Usage meter reads are served from the wake ledger — no metric label carries high-cardinality user content.

## Technical Considerations

### Key Decisions

- Aggregate-usage settlement + wall-time deadline over provider-attempt interception (ADR-008): give up mid-turn token precision, gain provider-universal Live and a shippable release; the one-in-flight-wake-per-owner rule bounds the overshoot to a single deadline-limited turn (invariant 11).
- Durable wakes ride `task_runs` via `ClaimNextRun` (ADR-011) over a network-owned reservation queue: give up a self-contained network scheduler, gain the single-queue/single-claimer law, free restart recovery, and manual/autonomous convergence.
- Broker deletion over lazy-start (ADR-010): give up a pre-built remote hop nobody consumes, gain structural fixes for C-02/C-05 and the performance goal.
- Session-keyed relations over a new address identity (ADR-007): restart continuity with zero new identity concepts; offline addressing deferred with mailbox.
- Leaf contract package over network-owned types: keeps import flow downward while every domain shares one shape.
- Coordination toolset projected by participation instead of daemon dependency: deletes the 18-descriptor tax on every local session.

### Known Risks

- Delivery restructure regressions → phase-isolated, invariant tests 20-21, and the existing collaboration E2E journey must pass under the new dispatcher.
- Wake deadline without provider cancel support → charged-to-terminal semantics (invariant 23); documented per provider.
- Coordinated-run conversation UX depends on design-phase quality (invitation/empty states) → phase 8 gates on `eng-design`/`ui-craft` + screenshot evidence.
- Hard-cut breadth (contract + storage + web in one release) → build order keeps each phase `make verify`-green internally; the release gate is the delete-target sweep + full verify.
- Two programs editing shared docs (glossary/autonomy pages) → this spec's phase 9 owns them; merge program rebases on top (dependency flagged in ADR-004).

## Extensibility Integration Plan

- **Extensions/bundles:** bundle `BindPrimaryChannelAsDefault` and effective-default channel projection are deleted; bundles may still declare channel resources. Extension/bundle definitions that start work carry the participation request field; a declared `live` requirement surfaces at activation for explicit confirmation (US-020) — never auto-applied. Extension host network API returns `not_participating` diagnostics for local executions.
- **Hooks (N-006):** typed dispatch at the owning call sites — never tailed from event tables (durable events stay observability-only). Two events: `network.participation.pre_resolve` dispatched inside the Resolver before the snapshot persists (payload: requested `Request`, owner ref, workspace; hooks may deny or narrow, never widen without an explicitly granted capability) and `network.participation.resolved` dispatched after the owning record commits (payload: immutable `Spec` + `OwnerKey`; read-only). `TaskRunContext` flat channel fields are replaced by the one resolved-participation object; matchers may match mode/channel/source.

```go
// internal/hooks — participation payloads (dispatch at call site, L-005/CLAUDE.md)
type NetworkParticipationPreResolve struct {
    Owner     participation.OwnerRef  `json:"owner"`
    Workspace string                  `json:"workspace_id"`
    Requested *participation.Request  `json:"requested,omitempty"`
}
type NetworkParticipationResolved struct {
    Owner    participation.OwnerRef `json:"owner"`
    OwnerKey string                 `json:"owner_key"`
    Spec     participation.Spec     `json:"spec"`
}
```
- **Skills/capabilities:** capability catalog/read surfaces unchanged; capability-triggered auto-activation deleted (ADR-008). Official skill rewrite below.
- **Tools/resources:** coordination toolset (18 descriptors per the current source inventory — regenerate the count from the canonical registry, never hand-write it; N-002) projected only to participating sessions; descriptors/schemas/digests regenerate; availability diagnostics name the required mode. No new native tool ID — existing session/task/loop management tools carry the participation fields.
- **Bridge SDKs / MCP sidecars:** no contract change beyond regenerated types; a bridge route named "channel" remains not-participation (PRD Non-Goals).
- **Extension participation requirements (N-008/B-016/B-019):** the extension manifest gains a typed `network_participation` requirement block (`{required: bool, mode: "live", channel_scopes: [...]}`). Bundle activations are **typed resources** at HEAD (`bundle.activation` in the generic `resource_records.spec_json`; a legacy activation table is forbidden by the canonical suite), so the confirmation persists as final typed fields on `ActivationResourceSpec`/`Activation` — `NetworkRequirementDigest`, `NetworkRequirementConfirmedBy`, `NetworkRequirementConfirmedAt` — keyed by the activation resource's existing identity (extension/bundle/profile/scope/workspace) and fenced by the resource's optimistic version (expected-version CAS). No SQL columns and no migration item exist for this datum. The operator's decision travels as an explicit field on the existing preview/activate/update contracts (HTTP/UDS via the shared core handler) and CLI (`--confirm-network-requirement`), closing contract → handler → transports → CLI → persistence → tests in one pass. The digest covers the block's canonical encoding; an update that changes it clears the confirmation fields in the same versioned resource update and re-surfaces the prompt (IT-034). Declining leaves the activation failed with the requirement named — no partial state.
- **Protocol docs:** RFC/protocol pages updated: envelope semantics unchanged, transport now in-process, broker subjects retired until the external-interaction program.

## Agent Manageability Plan

- CLI verbs (all `-o json`): `session new/task profile/start/enqueue/fan-out --network*`, `network coordination status|enable|disable`, `network usage`, `network status`, `ch *` with typed `not_participating` errors.
- HTTP/UDS parity: participation objects on every execution create/read; workspace coordination GET/PUT; usage GET; run detail/SSE resolved spec + bounds consumption. No CLI-only or UI-only state.
- Deterministic errors: single code set (`network_participation_unavailable`, `network_channel_unknown`, `network_strategy_invalid`, `network_strategy_channel_conflict`, `network_participation_permission_denied`, `network_bounds_exceed_ceiling`, `loop_requires_live`, `network_live_unsupported`, `not_participating`) mapped once in `api/core`.
- Discovery: `agh network status` + situation surface expose availability, participation counts, and (for the owning session) its own resolved spec; a local agent asking for network tools receives the diagnostic naming what would enable them (US-019).

## Config Lifecycle

- **Added:** `[network.live.defaults]` and `[network.live.limits]`. Defaults fill omitted fields only after an execution explicitly selected `live`; limits are inclusive administrative ceilings; a default outside its limit fails load/reload validation. Initial values locked here (N-005) — administratively configurable within the validation rules:

| Key | Default | Ceiling (`limits`) |
| --- | --- | --- |
| `max_wakes` | 8 | 64 |
| `max_wake_wall_time` | "5m" | "15m" |
| `max_total_wall_time` | "30m" | "2h" |
| `max_input_tokens` | 200000 | 1000000 |
| `max_output_tokens` | 50000 | 200000 |
| `max_wake_depth` | 3 | 5 |
| `coalesce_window` | "500ms" | range "100ms"–"5s" (`min/max_coalesce_window`) |
- **Kept:** `network.enabled` (availability kill switch; never selects participation) — applied through the persisted `network_availability` row so config apply/reload and admission share one serialization domain (B-013; invariant 24); `network.greet_interval` (presence cadence — operational state only).
- **Removed:** `network.default_channel`, `network.port`, `network.max_payload`, `network.activation_top_k`, `network.digest_flush_interval`, `network.digest_max_envelopes`, `network.response_guidance_max_bytes`, `network.delivery_structured_body_max_bytes` — strict validation rejects them post-cut.
- Structs, defaults, merge/overlay, validation, `agh config` tool-surface paths (`tool_surface.go:39-54,239-260,732`), examples, generated site config reference, and loader tests move in the same change. Workspace coordination is deliberately **not** a config key (ADR-009) — it is workspace data with API/CLI lifecycle.

## Web/Docs Impact

- **web/:** session create dialog gains the participation control (Local default); task Orchestration tab gains the profile network block; fan-out card and automation task-run step replace raw channel inputs; run detail gains participation chip + conversation panel + bounds/usage; the kanban invitation component (trigger per PRD rule 11) writes the coordination PUT; network area empty states rebuilt oriented (US-015); settings network page reframed as availability + live defaults ("does not opt executions in") + usage; generated types regenerate. All through `eng-design` + `ui-craft` + `impeccable`, verified with `eng-ui-screenshot`.
- **packages/site:** session/task/loop/automation/network concept + CLI references regenerate; configuration reference updated for the new/removed keys; autonomy pages (`coordination-channels.mdx`, `index.mdx`, `coordinator.mdx`) rewritten — "bind always" doctrine deleted; network concepts updated for Local/Live, admission, budgets, usage.
- **Official skill (`skills/agh/`):** `references/network.md` rewritten for modes/admission/bounds/usage + conditional tooling; `tasks-and-orchestration.md` for local coordinator + separate ingress; `native-tools.md` for gated toolset.
- **Glossary/COPY:** Coordination Channel entry replaced (opt-in semantics); mode naming pass through `COPY.md` (PRD open question 1).
- **QA tracker:** `docs/qa/state.csv` — new rows (participation controls, coordination toggle+invitation, run conversation, usage views, network empty states) as `untested`; reset affected session/task/loop/automation/network rows (flag, don't retest).

## AGH Impact Audit

- **Native tools:** high impact — coordination toolset gated by participation; task/loop/session management tool schemas gain participation fields; descriptors/digests/availability diagnostics regenerate; no new tool ID (checked: `tools/builtin/network.go`, `toolsets.go:25`, `builtin_ids.go:373`).
- **Extensibility and hooks:** high impact — hook payload swap, bundle default-channel deletion, extension confirmation flow, host API diagnostics (checked: `hooks/payloads.go:949-957`, `manager_run_hooks.go:136-187`, `api/core/bundles.go:177-195`, extension host network API).
- **Workspace data isolation:** high impact — coordination record, wake ledger, usage, conversations all workspace-scoped; `(workspace_id, channel)` qualification everywhere (C-01); same-name cross-workspace negative tests (checked: store network tables, cache keys, SSE scoping).
- **Official AGH skill:** required — rewritten as listed in Web/Docs Impact.

## Assumptions / Defaults

- This spec implements the amended PRD (Local + Live; mailbox and external interaction out per ADR-004/007; spend caps out per ADR-005).
- Lands in current `agh` vocabulary; persisted identifiers are brand-neutral (`network-participation/v1`, source atoms, error codes carry no brand) so the product-line rename sweeps code/tool IDs/envs mechanically without touching stored state. Release ordering vs the merge is roadmap-owned; ADR-004's no-window rule binds both.
- Session identity is the durable Live relation key (ADR-007); deleting a session explicitly retires its direct rooms/subscriptions.
- Invitation acceptance scope (current vs future runs, PRD open question 2) is a design-phase decision; both resolve to the same PUT + truthful copy.
- The Config Lifecycle table is the single normative owner of initial bound defaults/ceilings (B-015); operators may reconfigure within the validated ceilings. Their existence/visibility/exhaustion semantics are contract (invariants 10-16).
- `agh exec`-style headless surfaces inherit session semantics unchanged (participation default local).

## Architecture Decision Records

- [ADR-001: Network Participation Is Explicit, Per-Execution, and Default-Local](adrs/adr-001.md) — two planes, smallest-unit scope, no inheritance.
- [ADR-002: Coordinated Task Runs Are the Flagship, Enrolled by Persisted Opt-In + Invitation](adrs/adr-002.md) — the aha moment and enrollment model.
- [ADR-003: Discoverability — Disclosed, Not Hidden](adrs/adr-003.md) — three on-ramps, no in-session nudges.
- [ADR-004: One Complete Release — Local and Live Together; Mailbox Removed](adrs/adr-004.md) — release packaging; mailbox non-goal.
- [ADR-005: Usage Visible at Release; Spend Limits Deferred](adrs/adr-005.md) — truthful meter now.
- [ADR-006: One Participation Contract — Resolve Once, Immutable Snapshot, Project Everywhere](adrs/adr-006.md) — the core technical shape.
- [ADR-007: Live-Only Identity — Session-Keyed Relations, No Offline Address Layer](adrs/adr-007.md) — C-07 descoped structurally.
- [ADR-008: Live Activation — Deterministic Admission + Aggregate-Usage Budgets](adrs/adr-008.md) — practical accounting floor.
- [ADR-009: Coordination Setting Home — Workspace Record + Execution Profile](adrs/adr-009.md) — persisted opt-in storage and precedence.
- [ADR-010: In-Process Delivery Replaces the Embedded Broker](adrs/adr-010.md) — transport deletion and structural bug fixes.
- [ADR-011: Durable Network Wakes Ride task_runs Through ClaimNextRun](adrs/adr-011.md) — no second queue, no peer claimer; ledger is accounting only (peer-review round 1, B-002).
