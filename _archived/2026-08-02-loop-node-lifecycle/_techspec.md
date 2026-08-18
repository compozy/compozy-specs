# TechSpec — Loop Node Lifecycle & Failure Contract

## Executive Summary

This spec implements the PRD's node lifecycle & failure contract (Spec 1 of the loop-hardening
train) inside the existing loop plane. One classification function feeds everything: a closed
`FailureClass` enum evaluated at the coordinator boundary (with payload-declared detection and
remediation-hint propagation) drives the fixed precedence chain **retry → error route → effects →
escalation to generation succession**. Durable node-lifecycle state lands in five new tables plus
four columns on `loop_generation_outputs`; all future-time behavior (retry backoff, authored
deadlines, timer waits, escalation ladders) rides the existing 15-second scheduler cycle as
due-time rows guarded by a single per-cell epoch fence — no new timer service. The per-target breaker reuses
`internal/deadentity` with a new kind; the existing same-node consecutive-failure arithmetic is
redirected from run-death to node quarantine. Liveness is evidence-based over the session activity
metadata that already exists; the hidden 7m30s no-progress kill is deleted; confirmed session
death auto-resumes as continuation bounded by a progress-reset streak. Authored `on_*` effects
persist as same-transaction outbox deliveries and drain post-commit through an idempotent relay,
fail-open and observe-only. Every new state and verb co-ships across contract, HTTP/UDS, CLI, native tools, web,
docs, and the official skill; the run-level `stop` verb is hard-cut into `cancel` + `kill`.

Primary trade-offs accepted: scheduled fires have up to 15s slack (durable correctness over
precision — confirmed in grilling); at-least-once effect delivery (result append/ack idempotent on a deterministic per-entry
`delivery_id`; external tool execution stays at-least-once with consumer-side dedupe); one Goose migration carries all schema work — five new tables, three
row-preserving rebuilds (`loop_generation_outputs`, `loop_generations` origin CHECK,
`dead_entities` kind CHECK), and one `loop_run_events` column addition (`delivery_key` +
partial UNIQUE index).

**MVP Boundary**: everything in this spec is one MVP delivered by one task tree — schema,
classification, DSL grammar + lint, retry/timeout/deadline, error routes + absorption, effects,
per-target breaker + quarantine + requeue, liveness/death-resume/silence, cancel/kill +
parent-close, node pause + auto-pause rules, `wait` control kind + governed resume, watch-admission
dedupe, full surface co-ship (contract/HTTP/UDS/CLI/native tools/web/docs/skill), web Visual
Contract for run page + inventories, **loop editor lifecycle grammar authoring + chrome states**,
and **hero path** (catalog → run form → loop detail) per ADR-018, and the trailing qa-report +
qa-execution pair. Post-MVP (explicitly out of scope, PRD Non-Goals): idea 11 (cut), idea 12
(held), Spec 2 (fan-out/scale/rollover/cost), Spec 3 (overlap policy, ingress verification, run
inbox, time-travel verbs, **and start-binding authoring** — editor writes `start[]`, production
sidebar gains the Start lane, docs drop read-only strip language; until then Visual Contract
material only / current production Start strip retained), agent-drivable DAG RFC, node-family
state-machine refactor (idea 33), `on_resume` trigger, state-writing effects.

## System Architecture

### Component Overview

All behavior lands in existing packages; `internal/daemon` gains wiring only.

- **Failure classifier** (`internal/loop`, new `failure_class.go` + `failure_classify.go`) — pure
  function from failure evidence to `ClassifiedFailure`; payload `result_contract` evaluation;
  hint extraction from `ActionFailure.Recovery`. Consumed by the coordinator boundary and the
  attempt ledger (ADR-013).
- **Lifecycle planner** (`internal/loop`, new `coordinator_lifecycle.go` +
  `coordinator_retry.go` + `coordinator_wait.go`) — extends the finisher path: consults control
  rows, plans retries (`retrying` + `next_attempt_at` + epoch), applies the precedence chain,
  routes handled failures, plans wait parking, and computes quarantine/needs-attention outcomes.
  Feeds intents into the existing `GenerationSnapshotPayload`/completion plan (single writer
  preserved).
- **Node control service verbs** (`internal/loop`, new `service_node_control.go`) — pause /
  resume / cancel / kill / requeue at node grain plus run-level cancel/kill, following the
  `Approve`/`Pause` CAS + reactivation shape in `service_control.go`.
- **Effect relay** (`internal/loop`, new `effects_dispatch.go` + `effect_context.go`; daemon
  wiring `internal/daemon/loop_effect_relay.go`) — drains the dedicated `loop_effect_outbox`
  (typed delivery rows written in the same transaction as the trigger's state change; ADR-015);
  executes `emit` and `tool` entries in isolation, idempotent on `delivery_id`. The relay never
  fires hooks and never derives work from `loop_run_events` — hooks keep dispatching at their
  owning call sites (B-001).
- **Due-scan** (daemon, new `internal/daemon/scheduler_loop_due_scan.go`) — joins the existing
  loop coordinator backstop: pages rows whose `next_attempt_at` / `resume_at` /
  `next_escalation_at` / deadline anchor are due, enqueues coordinator wakes, drops stale-epoch
  schedules (ADR-012).
- **Liveness prober** (daemon, evolved `internal/daemon/loop_action_liveness.go` →
  evidence-only) — polls `session.Status()` activity metadata, records evidence, and
  raises/clears the silence attention flag through the idempotent attention CAS. It mutates
  nothing else: confirmed death is handed to the single atomic authority `ResumeDeadNode`
  (below), which owns the death-vs-cancel race (round-2 B-007; ADR-016).
- **Target health** (`internal/deadentity`, new kind + `internal/loop` seam
  `target_health.go`) — `BeforeProbe`/`RecordFailure`/`RecordSuccess` keyed
  `(workspace, loop_target, family:target)`. Policy is daemon-global (`[loops.breaker]`),
  applied to a **separately constructed loop-target service instance** sharing the store and
  event sink but NOT the threshold/interval of the existing shared instance — bridge, MCP, and
  sidecar health are provably unaffected by loop breaker settings (round-3 B-003). Never
  run-pinned (round-2 B-003). Open/half-open marks are durable rows; pre-threshold streaks are
  in-memory and reset on restart — accepted, documented, and tested (round-2 B-002) (ADR-014).
- **Stores** (`internal/store/globaldb`) — new tables `loop_node_controls`,
  `loop_node_attempts`, `loop_node_waits`, `loop_admission_claims`, `loop_effect_outbox`;
  extended `loop_generation_outputs`; new event kinds; sqlc queries (ADR-011).
- **Surfaces** — contract payload extensions + node-verb routes + inventory route, CLI verb
  group `compozy loop node …` + `compozy loop nodes`, native tools (8 new, 1 deleted), web loops
  system, site docs, official skill.

Data flow (one failing attempt): task run fails → coordinator boundary classifies →
attempt-ledger intent + outputs cell mutation (retrying/next_attempt_at/epoch, or terminal
disposition) + effect-outbox rows ride the completion plan → `CompleteCoordinatorAndEnqueueNext`
commits (BEGIN IMMEDIATE, claim-fenced) → lifecycle events + deliveries durable → post-commit:
wakes + effect relay drain → when due, the backstop due-scan enqueues the retry run
(epoch-checked) → exhaustion enters route → effects → escalation, exactly once per step, each
step recorded.

### Architectural Boundaries

- `internal/loop` owns classification, precedence, planning, verbs, and effect resolution;
  `internal/loop/dsl` owns grammar; `internal/loop/gate` and `internal/loop/goal` are untouched
  except where named (goal keeps its turn loop, its `RetrySpec` consumption, and its checkpoint
  resume — generic retry excludes `goal` nodes by lint).
- `internal/store/globaldb` owns schema + queries; loop consumes via reader/intent seams — no SQL
  outside the store package. `internal/deadentity` stays the breaker authority; loop calls its
  service, never reimplements streaks.
- `internal/task` is untouched except the coordinator-plan contract already owns
  (`GenerationSnapshot` stays opaque to it). No parallel queue; retry runs are ordinary
  deterministic task runs (L-003).
- `internal/api/contract` → `internal/api/core` → `httpapi`/`udsapi` co-ship; CLI consumes
  contract types only; native tools bind in `internal/daemon/native_loop_tools.go`.
- No package imports `daemon/`; new stores/services inject through existing loop constructor
  options at the composition root. Import graph unchanged — no `magefiles/boundaries.go` update
  (no new package).
- 500-line cap plan. **Frozen (extend by new files only):** `linter_references.go` (494),
  `control_plan.go` (488), `goal/route.go` (453), `linter.go` (449), plus effectively-frozen
  `coordinator_succession.go` (430) and `service.go` (424). **New files:** `failure_class.go`,
  `failure_classify.go`, `failure_result_contract.go`, `coordinator_lifecycle.go`,
  `coordinator_retry.go`, `coordinator_wait.go`, `service_node_control.go`,
  `service_run_cancel.go`, `effects_dispatch.go`, `effect_context.go`, `target_health.go`,
  `node_controls.go`, `node_attempts.go`, `node_waits.go`, `admission_claims.go`,
  `linter_lifecycle.go`, `linter_wait.go`, `dsl/lifecycle.go`, `dsl/effects.go`, `dsl/wait.go`;
  store `global_db_loop_node_controls.go`, `global_db_loop_node_attempts.go`,
  `global_db_loop_node_waits.go`, `global_db_loop_admission_claims.go`; daemon
  `loop_effect_relay.go`, `scheduler_loop_due_scan.go`; CLI `loop_node.go`, `loop_nodes.go`;
  contract `loop_nodes.go`. Files with headroom that may grow within cap:
  `coordinator_outputs.go` (status mapping), `coordinator_generation_reattempt.go` (rerun-set
  exclusion), `action_failure.go`, `dsl/node_params.go`, `dsl/graph.go`, `generation_intent.go`
  (origin values), `coordinator_hooks.go` (payload fields).

## Implementation Design

### DSL Grammar (`compozy.loop/v1`)

Node envelope additions (grill decisions; ADR-010):

```yaml
- id: fetch_data
  kind: http_call
  timeout: 30s                 # attempt-scoped ctx deadline (existing field, unchanged meaning)
  deadline: 10m                # NEW: total budget across attempts + backoff, from first schedule
  retry:                       # promoted to all action-class nodes (goal keeps its own semantics)
    max_attempts: 3
    backoff: { base: 1s, max: 30s }
    non_retryable: [tool_invalid_input]     # FailureClass names or machine codes
  result_contract:             # NEW, optional payload-failure shape (ADR-013.3)
    failure_field: err
    message_field: err.message
  on_error:                    # object, always: flow XOR + effects
    route: fallback_fetch      #   forward-only edge; XOR allow_fail: true
    effects:
      - tool: compozy__network_send
        with: { channel: ops, body: "fetch failed: {{ effect.failure.cause }}" }
  on_retry:                    # the other six triggers: plain effect lists
    - emit: { kind: fetch_retrying }
  on_success: []               # on_pause / on_timeout / on_cancel / on_quarantine identical
```

New control kind `wait` (ADR-017) and gate expiry:

```yaml
- id: wait_for_ack
  class: control
  kind: wait
  params:
    for: 24h                   # XOR until: "{{ inputs.deadline }}" XOR event: {…subscription…}
    expect: { type: object, required: [approved] }
    ahead_arrival: consume_on_entry          # | reject
    expires:                   # optional; same block valid on human gates
      after: 72h
      escalate:
        - tool: compozy__network_send
          with: { channel: ops, body: "still waiting: {{ effect.wait.link }}" }
      route: timed_out_path    # omitted => needs-attention on expiry
```

Contract-level terminal triggers: `contract.on_done | on_noop | on_blocked | on_failed |
on_exhausted | on_stalled | on_canceled`, each a plain effect list, firing exactly once per run
(`on_canceled` covers the run-level cancel/kill terminal — round-2 B-001). `run-loop` nodes gain
`on_parent_close: terminate | cancel | abandon` (default `terminate`).

New lint rules (`linter_lifecycle.go`, `linter_wait.go`; codes in `types.go`), split by severity
(N-003). **Blocking errors:** `CodeErrorRouteBackward` (route must be a forward edge),
`CodeErrorRouteConflict` (`route` + `allow_fail` together), `CodeRetryOnGoalNode` (generic retry
fields invalid on `goal` — goal keeps `max_attempts` + `on_failure` only; `deadline` also invalid
on goal), `CodeTimeoutExceedsDeadline` (`timeout > deadline`), `CodeDurationInvalid` (unparseable
duration — never a silent default), `CodeResultContractInvalid`, `CodeEffectShapeInvalid` (entry
must be exactly one of emit/tool), `CodeWaitShapeInvalid` (exactly one of for/until/event;
`expires` requires `after`), `CodeWatchIdentityRequired` (watch source cannot yield stable event
identity — ADR-017.5), `CodeAutopauseRuleInvalid` (config-side, compile + field check).
**Warnings:** `CodeErrorRouteDead` (route on an infallible node), `CodeEffectToolUnknown`
(unknown tool at authoring; runtime failure is recorded per PRD US-011 EC-2),
`CodeWaitExpiryWithoutPath` (expiry with neither escalate nor route), plus the agent-retry
large-cap warning (US-009 EC-2). Expression compilation reuses `refs.ConditionCompiler` (cost
limit from config; 80% cost-warn diagnostic added via `OptTrackCost`).

Runtime `event_key` validation (N-004): a `watch-source` `PollResponse.event_key` is normalized
(trim, NFC) and bounded (≤ 256 bytes); an empty, oversized, or non-UTF-8 key at poll time fails
closed **before** admission with a recorded `watch_identity_invalid` diagnostic — the run never
starts undeduplicated; the canonical watch-source adapter suite owns the case.

### Core Interfaces

```go
// internal/loop/failure_class.go — the classification contract (ADR-013)
type FailureClass string

const (
    FailureTransport         FailureClass = "transport"
    FailurePayloadDeclared   FailureClass = "payload_declared"
    FailureQualityRejection  FailureClass = "quality_rejection"
    FailureAuthoring         FailureClass = "authoring"
    FailureCancellation      FailureClass = "cancellation"
    FailureAttemptTimeout    FailureClass = "attempt_timeout"
    FailureBudgetExhausted   FailureClass = "budget_exhausted"
    FailureTargetUnavailable FailureClass = "target_unavailable"
)

type ClassifiedFailure struct {
    Class      FailureClass
    Code       string         // machine code (ToolError code / ReasonCode), sanitized
    Cause      string         // operator-safe text, sanitized + bounded once
    Hint       string         // remediation hint (ActionFailure.Recovery slot)
    Target     string         // resolved target identity when target-bound
    RetryAfter *time.Duration // honored within node bounds
}

// Pure; evaluated at the coordinator boundary. Order: cancellation → payload contract →
// authoring → timeout/budget anchors → breaker verdict → transport → payload_declared default.
func ClassifyNodeFailure(ev FailureEvidence) ClassifiedFailure
```

```go
// internal/loop/dsl/lifecycle.go — grammar additions (ADR-010)
type BackoffSpec struct {
    Base string `json:"base,omitempty" yaml:"base,omitempty"` // duration; default from config
    Max  string `json:"max,omitempty"  yaml:"max,omitempty"`
}

// RetrySpec (dsl/node_params.go) gains:
//   Backoff      *BackoffSpec `json:"backoff,omitempty"`
//   NonRetryable []string     `json:"non_retryable,omitempty"`
// Node (dsl/graph.go) gains:
//   Deadline       string          `json:"deadline,omitempty"`
//   ResultContract *ResultContract `json:"result_contract,omitempty"`
//   OnError        *ErrorPolicy    `json:"on_error,omitempty"`
//   Triggers       TriggerEffects  `json:"-" yaml:",inline"`   // six effect-list keys
//   OnParentClose  string          `json:"on_parent_close,omitempty"` // run-loop only

type ErrorPolicy struct {
    Route     NodeID       `json:"route,omitempty"`
    AllowFail bool         `json:"allow_fail,omitempty"`
    Effects   []EffectSpec `json:"effects,omitempty"`
}

type EffectSpec struct { // exactly one of Emit / Tool (lint-enforced)
    Emit *EmitSpec      `json:"emit,omitempty"`
    Tool string         `json:"tool,omitempty"`
    With map[string]any `json:"with,omitempty"`
}

type EmitSpec struct {
    Kind    string         `json:"kind"`
    Payload map[string]any `json:"payload,omitempty"`
}
```

```go
// internal/loop/node_controls.go — durable control state (ADR-011); mutations are service-level
// CAS verbs (the Approve/Pause pattern), reads feed the planner.
type NodeControl struct {
    LoopRunID, NodeID string
    Paused            bool
    PauseProvenance   *ControlProvenance // actor kind/id, reason, rule id, at
    CancelState       CancelState        // "" | requested | delivering | draining | canceled
    CancelProvenance  *ControlProvenance // actor, reason, requested at; grace from authored bound
    Quarantined       bool
    QuarantineEntry   json.RawMessage    // sanitized: chain, attempts, hints, target, input ref
    AttentionFlag     string             // "" | silence | resume_exhausted | dependency_quarantined
    LastEvidenceAt    *time.Time
    DeathResumeStreak int
    Revision          int64              // verb CAS only — never the stale-work fence (B-004)
}

// Cancellation state machine (B-005), one authority for manual verb, liveness delivery,
// coordinator completion, inventories, and restart:
//   "" → requested (verb; bumps every affected cell epoch — pending schedules die)
//   requested → delivering (liveness/prompt-cancel delivery observed)
//   delivering → draining (in-flight attempt draining; grace overrun stays visible here)
//   requested|delivering|draining → canceled (attempt closed; on_cancel effects enqueued)
// kill bypasses the machine: immediate close, no node-trigger effects. All transitions are
// idempotent; repeat verbs answer with current state + winner provenance.

type NodeControlStore interface {
    ListNodeControls(ctx context.Context, workspaceID, runID string) ([]NodeControl, error)
    PauseNode(ctx context.Context, m NodeControlMutation) (NodeControl, error)   // CAS on revision
    ResumeNode(ctx context.Context, m NodeResumeMutation) (NodeControl, error)   // plain|reset|immediate
    RequeueNode(ctx context.Context, m NodeRequeueMutation) (NodeControl, error) // clears quarantine
    SetAttention(ctx context.Context, m AttentionMutation) (NodeControl, error)
}
```

```go
// internal/loop/node_waits.go — governed resume (ADR-017)
type WaitClaimState string // waiting | claimed | resumed | intervention_required

type NodeWait struct {
    LoopRunID string; Generation int64; NodeID string; ItemIndex int
    Kind      string     // timer | event | approval_escalation
    ResumeAt  *time.Time // timers
    NextEscalationAt *time.Time
    ClaimState WaitClaimState
    AdmissionFailures int
    ExpectRef  string    // schema blob ref
    CreatedAt  time.Time
}

type DeathResume interface {
    // ResumeDeadNode is ONE BEGIN IMMEDIATE transaction owning the death-vs-cancel race
    // (round-2 B-007; hardened round-3 B-006): validate the cell holds the expected live
    // status AND no run- or node-scope cancellation is pending (a raced cancel wins with a
    // deterministic no-op answer) -> transition the loop_generation_outputs cell to its
    // continuation state with a bumped epoch -> append the `resumed` attempt-ledger
    // disposition + provenance -> increment the death-resume streak -> rotate the managed
    // binding -> append the lifecycle event -> reserve exactly one deterministic continuation
    // task_run. Losers leave the cell AND the ledger unchanged; replay returns the existing
    // reservation; a continuation that disagrees with durable node truth is unrepresentable.
    ResumeDeadNode(ctx context.Context, m DeathResumeMutation) (DeathResumeResult, error)
}

type WaitAdmission interface {
    // ResumeWait is ONE BEGIN IMMEDIATE transaction (the ReactivateLoopCoordinator pattern):
    // validate payload against expect → win or observe the claim → transition wait/output/
    // control state → append provenance + lifecycle event → reserve the deterministic
    // coordinator task_run — all before commit. Replay returns the existing winner; a
    // claimed-without-enqueue state is unrepresentable (B-003). Losers receive the winner's
    // provenance; admission failures increment durably and flip to intervention_required.
    ResumeWait(ctx context.Context, m WaitResumeMutation) (WaitResumeResult, error)
}
```

```go
// internal/loop/effects_dispatch.go — post-commit outbox relay (ADR-015). Delivery rows are
// created by the same transaction that commits the trigger's state change; the relay only
// drains them. DeliveryID = sha256(loop_run_id, source_event_id, trigger, entry_index) —
// deterministic across crash/replay; emitted custom_event/effect_results appends are
// idempotent on it (B-002).
type EffectDispatcher interface {
    // Boot-started and cycle-driven: pages ALL pending loop_effect_outbox rows (workspace-safe
    // join through loop_runs), executes entries in isolation, and acks each in one transaction —
    // append result events idempotently (INSERT OR IGNORE on loop_run_events.delivery_key =
    // "<delivery_id>:<kind>") + mark the row delivered/failed + attempts++. The post-commit
    // nudge is a latency hint only; the daemon-singleton relay sequentially pages, so stranded
    // rows are always rediscovered (round-2 B-004). Tool execution is at-least-once by
    // contract — delivery_id rides the tool correlation for consumer-side dedupe.
    DrainPendingLoopEffects(ctx context.Context, page EffectDrainPage) (EffectDrainReport, error)
}

// internal/loop/target_health.go — deadentity seam (ADR-014)
type TargetHealth interface {
    BeforeProbe(ctx context.Context, key store.DeadEntityKey) (deadentity.ProbeDecision, error)
    RecordFailure(ctx context.Context, key store.DeadEntityKey, class deadentity.FailureClass, reason string) error
    RecordSuccess(ctx context.Context, key store.DeadEntityKey) error
}
```

Error handling follows package conventions: store errors wrapped `%w`; verbs answer invalid
states with `ReasonError` codes naming the actual state and allowed transitions (US-026 AC-2);
every new path emits its canonical event with correlation keys.

### Data Models

Declarative source `internal/store/globaldb/schema/definitions/50_loops.sql`; next gap-free Goose
migration appended; `atlas.sum` + sqlc via `make codegen` (`eng-schema-migration`).

```sql
-- current control state per author-node; spans generations (ADR-011.1)
CREATE TABLE loop_node_controls (
    loop_run_id          TEXT    NOT NULL REFERENCES loop_runs(id) ON DELETE CASCADE,
    node_id              TEXT    NOT NULL,
    paused               INTEGER NOT NULL DEFAULT 0,
    pause_actor_kind     TEXT, pause_actor_id TEXT, pause_reason TEXT, pause_rule_id TEXT,
    pause_requested_at   TIMESTAMP,
    quarantined          INTEGER NOT NULL DEFAULT 0,
    quarantine_entry_json TEXT CHECK (quarantine_entry_json IS NULL OR json_valid(quarantine_entry_json)),
    quarantined_at       TIMESTAMP,
    attention_flag       TEXT    NOT NULL DEFAULT ''
        CHECK (attention_flag IN ('', 'silence', 'resume_exhausted', 'dependency_quarantined',
                                  'wait_intervention', 'expired_wait')),
    attention_reason     TEXT    NOT NULL DEFAULT '',
    cancel_state         TEXT    NOT NULL DEFAULT ''
        CHECK (cancel_state IN ('', 'requested', 'delivering', 'draining', 'canceled')),
    cancel_actor_kind    TEXT, cancel_actor_id TEXT, cancel_reason TEXT,
    cancel_requested_at  TIMESTAMP,
    last_evidence_at     TIMESTAMP,
    death_resume_streak  INTEGER NOT NULL DEFAULT 0 CHECK (death_resume_streak >= 0),
    revision             INTEGER NOT NULL DEFAULT 0,   -- verb CAS; never the schedule fence
    updated_at           TIMESTAMP NOT NULL,
    PRIMARY KEY (loop_run_id, node_id)
);
CREATE INDEX idx_loop_node_controls_quarantined ON loop_node_controls(quarantined) WHERE quarantined = 1;
CREATE INDEX idx_loop_node_controls_attention   ON loop_node_controls(attention_flag) WHERE attention_flag != '';

-- append-only attempt ledger (ADR-011.2); the quarantine entry's evidence
CREATE TABLE loop_node_attempts (
    loop_run_id   TEXT    NOT NULL REFERENCES loop_runs(id) ON DELETE CASCADE,
    generation    INTEGER NOT NULL CHECK (generation >= 1),
    node_id       TEXT    NOT NULL,
    item_index    INTEGER NOT NULL DEFAULT 0,
    attempt       INTEGER NOT NULL CHECK (attempt >= 1),
    failure_class TEXT CHECK (failure_class IS NULL OR failure_class IN
        ('transport','payload_declared','quality_rejection','authoring','cancellation',
         'attempt_timeout','budget_exhausted','target_unavailable')),
    failure_code  TEXT NOT NULL DEFAULT '',
    cause         TEXT NOT NULL DEFAULT '',     -- sanitized + bounded once
    hint          TEXT NOT NULL DEFAULT '',
    target        TEXT NOT NULL DEFAULT '',
    disposition   TEXT NOT NULL CHECK (disposition IN
        ('succeeded','retried','routed','absorbed','escalated','quarantined','canceled','resumed')),
    started_at    TIMESTAMP NOT NULL,
    ended_at      TIMESTAMP,
    next_attempt_at TIMESTAMP,                  -- recorded when disposition = retried
    PRIMARY KEY (loop_run_id, generation, node_id, item_index, attempt)
);

-- durable waits, one row per pause point (ADR-017)
CREATE TABLE loop_node_waits (
    loop_run_id        TEXT    NOT NULL REFERENCES loop_runs(id) ON DELETE CASCADE,
    generation         INTEGER NOT NULL CHECK (generation >= 1),
    node_id            TEXT    NOT NULL,
    item_index         INTEGER NOT NULL DEFAULT 0,
    kind               TEXT    NOT NULL CHECK (kind IN ('timer','event','approval_escalation')),
    resume_at          TIMESTAMP,
    next_escalation_at TIMESTAMP,
    escalation_cursor  INTEGER NOT NULL DEFAULT 0,
    claim_state        TEXT    NOT NULL DEFAULT 'waiting'
        CHECK (claim_state IN ('waiting','claimed','resumed','intervention_required')),
    claimed_by_kind    TEXT, claimed_by_id TEXT, claimed_at TIMESTAMP,
    admission_failures INTEGER NOT NULL DEFAULT 0,
    expect_json        TEXT CHECK (expect_json IS NULL OR json_valid(expect_json)),
    ahead_payload_json TEXT CHECK (ahead_payload_json IS NULL OR json_valid(ahead_payload_json)),
    issued_epoch       INTEGER NOT NULL DEFAULT 0,   -- cell epoch this schedule was issued under
    created_at         TIMESTAMP NOT NULL,
    PRIMARY KEY (loop_run_id, generation, node_id, item_index)
);
CREATE INDEX idx_loop_node_waits_due     ON loop_node_waits(resume_at)          WHERE claim_state = 'waiting' AND resume_at IS NOT NULL;
CREATE INDEX idx_loop_node_waits_ladder  ON loop_node_waits(next_escalation_at) WHERE claim_state = 'waiting' AND next_escalation_at IS NOT NULL;
CREATE INDEX idx_loop_node_waits_state   ON loop_node_waits(claim_state);

-- admission dedupe tombstones (ADR-017.5); inserted in the loop-start transaction
CREATE TABLE loop_admission_claims (
    workspace_id      TEXT NOT NULL,
    loop_name         TEXT NOT NULL,
    source_key        TEXT NOT NULL,
    event_key         TEXT NOT NULL,
    loop_run_id       TEXT NOT NULL,
    claimed_at        TIMESTAMP NOT NULL,
    expires_at        TIMESTAMP NOT NULL,
    suppressed_count  INTEGER NOT NULL DEFAULT 0,
    last_suppressed_at TIMESTAMP,
    PRIMARY KEY (workspace_id, loop_name, source_key, event_key)
);

-- authored effect deliveries; rows written in the SAME transaction as the trigger's state
-- change; the relay only drains (ADR-015; B-001/B-002)
CREATE TABLE loop_effect_outbox (
    loop_run_id     TEXT    NOT NULL REFERENCES loop_runs(id) ON DELETE CASCADE,
    delivery_id     TEXT    NOT NULL,   -- sha256(loop_run_id, source_event_id, trigger, entry_index)
    source_event_id TEXT    NOT NULL,   -- loop_run_events.id that recorded the trigger state
    trigger         TEXT    NOT NULL,   -- on_error | on_retry | … | contract terminal trigger
    generation      INTEGER NOT NULL,
    node_id         TEXT    NOT NULL DEFAULT '',
    item_index      INTEGER NOT NULL DEFAULT 0,
    entry_index     INTEGER NOT NULL CHECK (entry_index >= 0),
    entry_json      TEXT    NOT NULL CHECK (json_valid(entry_json)),  -- FULLY rendered, sanitized execution payload; the relay executes these immutable bytes (N-001 r2). Render failure inside the owning tx persists an error-as-data form {render_error, diagnostic} with the same delivery_id; the relay acks it failed without execution — the trigger's state transition always commits and sibling entries deliver (round-3 B-004)
    state           TEXT    NOT NULL DEFAULT 'pending'
        CHECK (state IN ('pending','delivered','failed')),
    attempts        INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMP NOT NULL,
    delivered_at    TIMESTAMP,
    PRIMARY KEY (loop_run_id, delivery_id)
);
CREATE INDEX idx_loop_effect_outbox_pending ON loop_effect_outbox(state) WHERE state = 'pending';

-- loop_generation_outputs: +4 columns and a status CHECK (table rebuild; ADR-011.3)
-- attempt            INTEGER NOT NULL DEFAULT 1 CHECK (attempt >= 1)
-- next_attempt_at    TIMESTAMP
-- first_scheduled_at TIMESTAMP
-- epoch              INTEGER NOT NULL DEFAULT 0   -- THE authoritative cell fence (B-004)
-- status CHECK: ('pending','enqueued','running','retrying','waiting','paused','awaiting_child',
--                'control_pending','awaiting_goal','succeeded','failed','canceled','quarantined')
```

**One migration** performs everything above: five CREATEs (`loop_node_controls`,
`loop_node_attempts`, `loop_node_waits`, `loop_admission_claims`, `loop_effect_outbox`); three
row-preserving rebuilds (`loop_generation_outputs` columns + status CHECK; `loop_generations`
origin CHECK gains `'requeue'`; `dead_entities` kind CHECK gains `'loop_target'` — declarative
source `41_reliability.sql`, round-2 B-002); and two column additions — `ALTER TABLE
loop_run_events ADD COLUMN delivery_key TEXT` + `CREATE UNIQUE INDEX
uq_loop_run_events_delivery ON loop_run_events (loop_run_id, delivery_key) WHERE delivery_key
IS NOT NULL` (round-2 B-004), and `ALTER TABLE loop_runs ADD COLUMN cancel_requested INTEGER
NOT NULL DEFAULT 0` + `ADD COLUMN cancel_kind TEXT NOT NULL DEFAULT '' CHECK (cancel_kind IN
('', 'cancel', 'kill'))` (round-3 B-001) — a single next-version Goose file (N-002).

**The stale-work fence is one number: `loop_generation_outputs.epoch`** (B-004). Every scheduled
future action (retry `next_attempt_at`, wait `resume_at`/`next_escalation_at`, deadline expiry)
persists the epoch it was issued under (`loop_node_waits.issued_epoch`; retries compare against
their own row's epoch). Every lifecycle mutation that supersedes scheduled work — pause, resume,
cancel request, kill, quarantine, requeue, route — increments the epoch of every affected cell
**in the same transaction** that mutates control/wait state; the due-scan and both output writers
compare issued epoch against cell epoch and drop mismatches with a `stale_schedule_dropped`
diagnostic. `loop_node_controls.revision` fences verb CAS only; it never gates schedules.

**Run-level cancel/kill contract (round-2 B-001; hardened round-3 B-001).** `canceled` becomes the seventh terminal
outcome across the whole domain: `dsl.TerminalState`, runtime `Status`, contract
`LoopRunStatus`, the transition table (new causes `operator_cancel`, `operator_kill`), terminal
events, web, and docs — and the contract trigger set gains `on_canceled`. `loop_runs` gains
`cancel_requested INTEGER NOT NULL DEFAULT 0` and `cancel_kind TEXT NOT NULL DEFAULT ''
CHECK (cancel_kind IN ('', 'cancel', 'kill'))`, reusing the existing `control_actor_*` columns
for provenance (the `pause_requested` pattern). Authority: the service verb is the single request authority — ONE `BEGIN IMMEDIATE` service
transaction records the run request AND projects it onto every affected live cell (sets each
live node's `cancel_state = requested` with provenance and bumps its epoch — pending schedules
die in the same commit), then wakes the coordinator; a live node can never miss a run-scope
request because delivery and `ResumeDeadNode` read the same control rows the projection wrote.
`ResumeDeadNode` rejects when run-scope OR node-scope cancellation is pending — a death observed
in the request window can never reserve a continuation. Both cancel and kill carry the existing
Goal stop cleanup (`GoalRunStopStore` prompt-lease revocation, binding close, session cleanup —
`service_control.go:62` precedent) inside their transactions before terminalizing. `kill`
transitions the run terminal `canceled` (reason `operator_kill`) in that same transaction and
fires interrupts; `cancel` on a run with live nodes lets the drain finish — node machines walk
to `canceled` and the boundary closes the run `canceled` via the completion plan; `cancel` on a
run with no active node (queued / watching / needs-approval / paused) transitions terminal
directly with deterministic answers. Covered states: queued, parked, multi-node, Goal-bearing,
and cancel-vs-death races. Contract terminal effects select `on_canceled`; node-trigger
suppression on kill is unchanged (SI-8).

Field rationale:

- `loop_node_controls` — per-node cross-generation state: pause/cancel/quarantine/attention are
  what succession, due-scans, verbs, liveness delivery, and inventories match on; `revision`
  fences verb CAS (the schedule fence is the cell epoch); `cancel_state` + provenance are the
  single durable authority for the cancellation machine (B-005); `death_resume_streak` +
  `last_evidence_at` implement the progress-reset streak (grill).
- `loop_node_attempts` — attempt visibility (US-008 AC-3) and the quarantine entry's evidence
  chain (US-024 AC-1) as queryable rows; `disposition` records each precedence-chain outcome
  (Business Rule 1 "each step's outcome is recorded").
- `loop_node_waits` — the inventory row (identity, reason, age — US-022 AC-1) plus the
  single-claim CAS cell; due/ladder partial indexes bound the due-scan.
- `loop_admission_claims` — durable tombstones with loud counters (US-025); PK is the suppression
  key; same-transaction insert makes the duplicate race unrepresentable; `expires_at` captures
  the admission-time horizon so a later config reload cannot shorten or extend an existing claim.
- `loop_generation_outputs.attempt/next_attempt_at/first_scheduled_at/epoch` — retry scheduling
  at the execution grain the coordinator already reads; `first_scheduled_at` anchors `deadline`;
  `epoch` is the one authoritative stale-work fence (B-004).
- `loop_effect_outbox` — the authored-delivery ledger: `delivery_id` is the stable at-least-once
  identity (deterministic digest — crash/replay reproduces it, B-002); `source_event_id` ties the
  delivery to the durable trigger event; `state`/`attempts` make the relay idempotent and its
  backlog inspectable.

Side-table-vs-JSON: pause/cancel/quarantine/attention/wait/claim state = **typed columns**
(inventories, verbs, and the due-scan filter on them — matchable); quarantine entry chain, wait
expectation, and rendered effect entries = **JSON columns** (opaque, read-whole, sanitized once);
breaker state = **`internal/deadentity` rows** (existing authority, no new table); attempt
evidence and effect deliveries = **side tables** (queried by entries, relay, UI, and future eval
work).
Origin enum extension: `loop_generations.origin` CHECK gains `'requeue'` (table rebuild;
`generation_intent.go` gains `OriginRequeue`) — the sole new generation-creating path in this spec
(pause/resume, waits, death-resume, and error routes act within a generation).

### API Endpoints

Deleted (hard cut, grill decision): `POST /api/workspaces/:ws/loop-runs/:run_id/stop` (HTTP +
UDS), CLI `compozy loop stop`, native `compozy__loop_stop`.

New/changed routes (both `httpapi` and `udsapi`, shared `BaseHandlers`):

```
POST /api/workspaces/:ws/loop-runs/:run_id/cancel            # cooperative; idempotent
POST /api/workspaces/:ws/loop-runs/:run_id/kill              # immediate; no node-trigger effects (contract on_canceled still fires once)
POST /api/workspaces/:ws/loop-runs/:run_id/nodes/:node_id/pause    {drain|cancel, reason?}
POST /api/workspaces/:ws/loop-runs/:run_id/nodes/:node_id/resume   {mode: plain|reset_attempts|immediate, payload?}
POST /api/workspaces/:ws/loop-runs/:run_id/nodes/:node_id/cancel
POST /api/workspaces/:ws/loop-runs/:run_id/nodes/:node_id/kill
POST /api/workspaces/:ws/loop-runs/:run_id/nodes/:node_id/requeue
GET  /api/workspaces/:ws/loop-nodes?state=waiting|quarantined|attention|retrying
     &loop=&run_id=&cursor=&limit=                            # workspace inventory, stable order
```

`resume` with `payload` is the manual wait-resume path (validated against `expect`, single-claim);
node verbs answer invalid states with deterministic `ReasonError`s (`node_not_paused`,
`node_not_quarantined`, `run_terminal`, `already_decided{winner}`), concurrent verbs resolve by
CAS with the winner's provenance in the loser's answer (US-026 EC-1).

Payload extensions (contract → OpenAPI → TS via `make codegen`):

- `LoopGenerationOutput` +`attempt`, `next_attempt_at?`, `failure_class?`, `disposition?`.
- `LoopRunResponse` +`node_controls []LoopNodeControlPayload` (paused/quarantined/attention with
  provenance), +`waits []LoopNodeWaitPayload` (identity, kind, age, claim state).
- New `LoopNodeInventoryResponse{items, next_cursor}` for the inventory route.
- SSE: new event kinds (below) ride the existing `LoopRunEventPayload`; kinds registered in the
  contract `LoopRunEventKind` enum.

New loop-run event kinds (append in-tx; validated in `loopRunEventKindValid`):
`node_retry_scheduled`, `node_paused`, `node_resumed`, `node_canceled`, `node_killed`,
`node_quarantined`, `node_requeued`, `node_wait_started`, `node_wait_resumed`,
`node_attention_flagged`, `node_attention_cleared`, `effect_results`, `custom_event`,
`duplicate_suppressed`, `target_breaker_transition`. Existing kinds unchanged.

Hooks: no new hook events; additive payload fields only — `LoopNodeTerminalPayload` gains
`{failure_class, disposition, attempt, target}`; matchers unchanged. Extensions observe the new
lifecycle through loop-run events (SSE + watch-events mapping) per PRD's extension expectation.

## Impact Analysis

| Component | Impact Type | Description and Risk | Required Action |
| --- | --- | --- | --- |
| `internal/loop` coordinator boundary | modified | Precedence chain, classification, retry planning, wait parking, quarantine — core semantics; high risk | New lifecycle/retry/wait planner files; regression matrix in `_tests.md` |
| `internal/loop/dsl` + linter | modified | 4 envelope fields, `wait` kind, effect grammar, ~13 lint codes; medium | New `dsl/lifecycle.go`, `dsl/effects.go`, `dsl/wait.go`, `linter_lifecycle.go`, `linter_wait.go` |
| `internal/store/globaldb` | modified | 5 new tables + 3 rebuilds (outputs, generations origin, dead_entities kind) + `loop_run_events.delivery_key` + 15 event kinds + queries; medium | One Goose migration + sqlc/Atlas via `make codegen`; extend canonical migration suites |
| `internal/deadentity` | modified | New `DeadEntityKindLoopTarget`; low | Kind constant + validation + docs |
| `internal/daemon` | modified | Effect relay, due-scan, liveness rewrite, session-supervision override at bind, native tool bindings; medium | New relay/due-scan files; evolve `loop_action_liveness.go`; delete 7m30s kill |
| `internal/session` seam | none (read-only) | Liveness derives from existing `Status()`/activity metadata; checked `prompt_activity*.go` — no session-package change | Bind-time supervision config override only |
| `internal/task` | none | Plans/queue untouched; retry runs are ordinary deterministic runs — checked `coordinator.go` plan contract | — |
| `internal/api/contract` + core + routes | modified | 8 node/run verbs + inventory route + payload fields + `stop` deletion; medium | Handlers in `internal/api/core/loops*.go`; both route files; `make codegen` |
| `internal/cli` | modified | `loop cancel/kill`, `loop node …` group, `loop nodes`; `stop` deleted; low | New `loop_node.go`, `loop_nodes.go`; generated CLI docs regen |
| Native tools | modified | +8 tools, −1 (`loop_stop`); digests refresh; kill tools Destructive; low | `internal/tools/builtin/loops.go` + `toolmeta` + daemon bindings |
| `internal/config` | modified | `[loops.defaults.*]` lifecycle keys + tool-surface entries; low | See Config Lifecycle |
| `web/src/systems/loops` | modified | Reducer kinds, story rows, node controls UI, inventories, attempt visibility; medium | See Web/Docs Impact |
| `packages/site` + `skills/compozy` | modified | Failure-handling chapter, grammar/config/CLI docs, skill loops section; low | See Web/Docs Impact |
| `internal/loop/goal` | none | Turn loop, goal retry, checkpoints untouched — checked `goal/executor.go`, `goal/turns.go`; generic retry lint-excluded on goal nodes | — |

## Extensibility Integration Plan

- **Hooks**: additive payload fields on `loop.node.terminal` (`failure_class`, `disposition`,
  `attempt`, `target`); the 7-event taxonomy, matchers, and deny surfaces are unchanged
  (`internal/hooks/events.go:141-147` intact). Pre-hooks still cannot mutate lifecycle state.
- **Watch sources (`loop.watch_source`)**: `PollResponse` gains required `event_key` (stable
  event identity) — a breaking provide-surface change, greenfield hard cut: extension protocol
  docs, `compozysdk.WatchSource` scaffold template, and `internal/extension/watch_source.go`
  dispatch update together; a described source that cannot yield identity fails definition
  validation (`CodeWatchIdentityRequired`).
- **Tools as effects**: reaction `tool:` entries call the standard tool surface — extension
  tools, MCP tools, and native tools are all valid effect targets under the run's workspace scope
  (grill decision); no new tool kind.
- **Bundles, registries, bridge SDKs, MCP sidecars**: no impact — loop execution stays off those
  surfaces (checked: no loop resource kind in `internal/bundles`/`internal/registry`; bridges are
  reachable only as tool effects through the tool surface).
- **Official Compozy skill** (`skills/compozy/references/loops.md`): failure contract, verbs,
  inventories, and event kinds sections updated (below).

## Agent Manageability Plan

- CLI: `compozy loop cancel|kill --run-id`, `compozy loop node pause|resume|cancel|kill|requeue
  --run-id --node [--item]`, `compozy loop nodes --state waiting|quarantined|attention|retrying
  [--loop] [--run-id]` — all with `-o json|jsonl`, stable field names, paginated stable-ordered
  lists.
- HTTP/UDS parity via shared `BaseHandlers` (routes above); deterministic `ReasonError` codes
  name actual state + allowed transitions; concurrent verbs answer with winner provenance.
- Native tools: `compozy__loop_cancel`, `compozy__loop_kill` (Destructive),
  `compozy__loop_node_pause`, `compozy__loop_node_resume`, `compozy__loop_node_cancel`,
  `compozy__loop_node_kill` (Destructive), `compozy__loop_node_requeue`, `compozy__loop_nodes`
  (Read). No new capability gates (grill decision; `loops.approve` stays the only gate).
  `compozy__loop_status` output schema extends with node lifecycle fields (digest refresh; the
  generation object's `additionalProperties:false` is updated in the same change).
- Discoverability: `compozy loop validate` surfaces every new lint code; `compozy loop inspect`
  renders effective family defaults with sources (config resolution, US-027 AC-3); attempt/retry
  state visible in `loop status`.

## Config Lifecycle

New keys under both `[loops.defaults.delivery]` and `[loops.defaults.watch]` (grill defaults v2;
struct + defaults + validation in `internal/config/loops.go`, overlay in `merge_loops.go`,
tool-surface entries in `tool_surface_loops.go`, tests per `TestToolConfigPathPolicy`, manual
docs in `config-toml.mdx`):

```toml
[loops.defaults.delivery.retry]        # mechanical families only; agent/sub-loop fixed at 0
max_attempts = 3                       # 0 disables; cap 10
backoff_base = "1s"
backoff_max  = "30s"
[loops.defaults.delivery.liveness]
silence_window = "30m"                 # 0 disables silence evaluation
[loops.defaults.delivery.resume]
death_streak_limit = 3                 # consecutive resumes without post-resume evidence
[loops.defaults.delivery.predicates]
cost_limit = 10000                     # CEL runtime cost; warn at 80%
[loops.defaults.delivery.waits]
admission_attempts       = 3
admission_retry_interval = "60s"
[loops.defaults.delivery.admission]
tombstone_horizon = "168h"
[[loops.defaults.delivery.autopause]]  # ordered; first match wins; NOT agent-mutable
match  = "class == 'transport' && attempt >= 2"
action = "pause"
```

Breaker policy is deliberately NOT per-kind and NOT run-pinned (round-2 B-003) — target health
is shared per `(workspace, family, target)`, so its policy has one owner:

```toml
[loops.breaker]                        # daemon-global; applied at service construction
threshold      = 5                     # consecutive transport failures per target
probe_interval = "60s"                 # half-open probe cadence
```

Changes take effect on daemon config reload, never mid-stream per run; `compozy loop inspect`
reports them under a `breaker (global)` source label.

Validation: durations parse + positive where required; `max_attempts` 0..10; monotonic pairs
(`backoff_base <= backoff_max`) per the `internal/config/autonomy.go` cross-field pattern;
autopause `match` compiles against the rule namespace at write. Agent-mutable additions: per
kind (×2 delivery/watch) `retry.max_attempts`, `retry.backoff_base`, `retry.backoff_max`,
`liveness.silence_window`, `resume.death_streak_limit`, `predicates.cost_limit`,
`waits.admission_attempts`, `waits.admission_retry_interval`, `admission.tombstone_horizon`
(18 paths); plus global `loops.breaker.threshold`, `loops.breaker.probe_interval` (2 paths). `autopause` follows
`runtime_rules` (append-merge, not agent-mutable). Resolution node > loop (`loop_config`) >
config default, captured at admission; running runs keep their resolution (US-027 EC-1). Session
supervision keys are untouched — loop-bound sessions override `inactivity_warning_after`/
`inactivity_timeout` to 0 at bind time in code, not via config.

## Web/Docs Impact

- **Visual Contract authority (ADR-018):** `docs/design/opendesign/loops/` artboards +
  `DESIGN-BACKLOG.md` + `DESIGN-LESSONS.md` (+ promoted L-034 / design-system directives) are
  normative for web lifecycle UI. Ownership split: run page / controls / quarantine /
  inventories → web task for run UI; editor lifecycle grammar + chrome states → web editor task;
  catalog / run form / loop detail → web hero-path task. Start-binding allowlist rows on the
  editor artboard are **held** (Spec 3 ship gate) — authorized difference vs the prototype.
- `web/src/systems/loops/` **run UI**: `lib/loop-events.ts` reducer + `UNRETAINED_KINDS` for the
  15 new kinds; `lib/loop-run-story-rows.ts` row builders (retry, pause, quarantine, wait,
  attention, effect results, suppression); `lib/loop-run-page-view.ts` + `loop-run-progress.ts`
  (parked states excluded from progress denominators); run-page: node control actions
  (pause/resume/cancel/kill/requeue) in `loop-run-controls.tsx` + node rows, attempt/next-attempt
  visibility, quarantine entry sheet, waiting/attention badges on `loop-run-needs-you-card.tsx`;
  new inventory views (waiting/quarantine/attention lists with age sort + empty states) under
  `components/runs/`; canonical timeline + collapse anatomy per run-detail artboards;
  `mocks/fixtures.ts` + `handlers.ts` + stories for every new state; generated types via
  `make codegen`.
- `web/src/systems/loops/` **editor**: `components/editor/` gains truthful authoring of Spec 1
  DSL keys — reliability envelope (`deadline`/`retry`/`result_contract`/`on_error` with route
  XOR allow_fail), six node `on_*` and seven contract terminals as effect lists (emit/tool
  one-of), `wait` inspector, `on_parent_close`, lint dock (errors gate Publish; warnings never;
  zero counts omit), built-in read-only + fork chrome, publish-rejected 422 strip. Round-trip
  via existing PATCH/publish surfaces; SD-007 daemon/linter truth. Start lane write path is
  out of scope (ADR-018).
- `web/src/systems/loops/` **hero path**: catalog (`loops` list — built-in/custom, status filter
  incl. `canceled`, Rows|Cards, empty/clear-filters), run form (arrive-and-use, limits folded,
  Dry/Start, required-input gate, this form = `http`, no `stop`), loop detail (Contract,
  reliability tiles, DAG, recent runs with `canceled` pill). Reuse shared status
  pills/formatters from the run UI task.
- Verify every Visual Contract row with durable `eng-ui-screenshot` evidence bundles before
  completion (SD-007: every control bound to daemon truth).
- `packages/site/content/docs/loops/`: new `failure-handling.mdx` (precedence chain, classes,
  effects, verbs, inventories — added to `meta.json`); `dsl-reference.mdx` (+`retry`, `deadline`,
  `result_contract`, `on_*`, `wait`, `on_parent_close`); `reference-grammar.mdx` (+`effect.*`
  context root); `running.mdx` (+cancel/kill, node verbs, inventories); `configure.mdx` +
  `configuration/config-toml.mdx` (+lifecycle keys, manual edit); `extensions.mdx`
  (+`event_key` requirement); CLI docs regenerate via `make codegen`. Do **not** document
  Start-lane authoring as shipping until Spec 3 adjacency.
- `skills/compozy/references/loops.md`: tool/CLI table (+8/−1), failure contract section, event
  kinds, inventories, verbs.
- **QA impact**: add `untested` content-addressed scenarios — transient-retry heals (US-008),
  error-route fallback walk (US-005), pause-repair-resume live run (US-019), quarantine +
  requeue (US-024), duplicate watch delivery suppressed (US-025), cancel vs kill on a running
  agent node (US-018), approval link via effect (US-012), **editor-authoring-walk (US-028)**,
  **catalog-runform-walk (US-029)**; reset any existing scenario touching `loop stop` or
  run-stop semantics to `untested`. Walk all flagged scenarios per `qa-execution` before
  completion.

## Safety Invariants

1. The precedence chain is fixed and identical across node families: automatic retry (if
   eligible) → error route (if declared) → failure effects → escalation to succession; each
   step's outcome is recorded as an attempt-ledger disposition (PRD Rule 1).
2. Absorption requires declaration; `route` XOR `allow_fail` is lint-enforced; error routes are
   forward-only edges (PRD Rules 2–3).
3. Bounds are bounds: every attempt, routed/absorbed failure, requeue, and resume counts toward
   `iteration_cap`, budgets, and `max_revisions`; no lifecycle path bypasses run bounds
   (restates loops-paper-adoption SI-10; briefing constraint on SI-7 re-verified per target).
4. Only `transport` and `attempt_timeout` classes are ever auto-retried. Agent and sub-loop
   families gain retry exclusively from an explicit node-level `retry:` declaration in the DSL —
   family defaults, `loop_config`, and `[loops.defaults.*]` can never enable it (the resolver
   skips those layers for expensive families); once declared, the planner schedules those
   attempts like any other, with checkpoint continuation, the large-cap lint warning, and run
   bounds applying (PRD Rules 10–11; US-009; round-2 B-006). `non_retryable` narrows, never
   widens.
5. Single-writer preserved: attempt state, wait rows, provenance rows, and events mutate only
   inside the claim-fenced coordinator completion transaction or the service-level CAS verbs;
   the effect dispatcher and liveness prober mutate no run state (L-005).
6. One stale-work fence: `loop_generation_outputs.epoch` is the sole schedule authority. Every
   scheduled future action (retry, timer, deadline, ladder step) persists the epoch it was
   issued under; every superseding lifecycle mutation increments the epoch of every affected
   cell in the same transaction that mutates control/wait state; a due fire whose issued epoch
   differs drops with a `stale_schedule_dropped` diagnostic — no ghost timers. Verb CAS uses
   `loop_node_controls.revision` and never gates schedules (PRD Rule 15; B-004).
7. Commit-gated observation: no event, effect delivery, or hook is observable before its state
   commits; effect deliveries are at-least-once, idempotent on the deterministic `delivery_id`
   persisted in `loop_effect_outbox` by the same transaction; replayed history equals live
   emissions; observer failures never touch the run. Hooks dispatch only at owning call sites —
   the relay never fires hooks and never derives work from event tables (PRD Rule 23;
   B-001/B-002).
8. Effects are fail-open, observe-only, isolated per entry; tool effects execute daemon-side in
   the run's workspace scope and never widen tool policy; approval-requiring tools fail the
   effect deterministically. Kill suppresses all node-trigger effects (`on_*`); contract
   terminal effects ride terminal truth and fire once for the resulting outcome regardless of
   how the run reached it, kill included (PRD Rules 20–22 as refined by B-006; US-013 AC-3).
9. No default duration limits anywhere; loop-bound sessions run with session-supervision
   warning/timeout disabled; `timeout`/`deadline` are opt-in, validated (`timeout <= deadline`),
   and their clocks suspend while parked (PRD Rule 14).
10. Death is deterministic (process/transport confirmed); ambiguity resolves to the silence
    path; resume is continuation with provenance, bounded by a 3-streak that resets on any
    post-resume evidence; death while parked never resumes. All death handling flows through
    the single atomic `ResumeDeadNode` transaction — a raced cancel wins deterministically and
    two continuations for one death are unrepresentable (PRD Rules 16–19; round-2 B-007).
11. Parked states (`paused`, `waiting`, `awaiting-approval`, `quarantined`) are classified by one
    function that uniformly excludes them from stall/no-progress arithmetic, rerun sets, and
    due-scans, and suspends node clocks and the run wall-clock budget; token spend always counts
    (PRD Rule 27).
12. Wait resume is one atomic transaction: payload validation, claim win/observe, wait/output/
    control transition, provenance + lifecycle event, and the deterministic coordinator
    `task_run` reservation commit together (the `ReactivateLoopCoordinator` pattern) — a
    claimed-without-enqueue state is unrepresentable; replay returns the existing winner; losers
    receive the winner's provenance; bounded admission failures flip to `intervention_required`
    — never silent limbo (PRD Rules 24–25; B-003).
13. The target breaker counts only `transport`-class failures; semantic and handled failures
    record success; an open target fails bound attempts fast with class/reason
    `target_unavailable` into the normal chain — never the run-terminal `circuit_breaker` code
    (ADR-014).
14. Quarantine never terminates the run; a required-but-quarantined (or rule-paused) producer
    surfaces needs-attention naming the dependency; requeue re-enters via succession with origin
    `requeue` under all bounds; every generation-creating path still writes exactly one
    `loop_generations` row (PRD Rules 29–30).
15. Watch admission claims its suppression key in the same transaction as run start; duplicates
    answer structurally and append a `duplicate_suppressed` diagnostic; tombstones persist ≥ the
    configured horizon; a source without stable `event_key` fails validation (PRD Rules 32–33).
16. All causes, hints, quarantine entries, wait payloads, and effect contexts pass the single
    sanitization boundary (secret redaction + size bounds + truncation markers) once, before any
    row, event, surface, or namespace reads them (loops-paper-adoption SI-11 extended).
17. Cancel is cooperative and idempotent; cancel/kill/timeout collapse into the existing single
    cancellation path (SD-001); kill is immediate and explicit — no silent cancel→kill
    escalation; parent-close propagation is strictly parent → child. Run-level cancel/kill is
    owned by the service verb + `loop_runs.cancel_requested`/`cancel_kind` and lands the
    `canceled` terminal (causes `operator_cancel`/`operator_kill`); node-level cancel is owned
    by the `loop_node_controls.cancel_state` machine — one authority per scope, no duplicate
    primitive (PRD Rule 5; ADR-016; round-2 B-001).

## Delete Targets

No fallback, no compat shim, no placeholder; old behavior dies in the same change.

- `internal/daemon/loop_action_liveness.go` duration-kill machinery:
  `loopActionNoProgressTimeout` (7m30s), `loopActionReasonNodeTimeout`,
  `loopActionReasonNoProgress` and the lease-fail paths they feed — replaced by evidence-only
  liveness + authored `timeout`/`deadline` (ADR-016.1).
- Run `stop` verb, all surfaces: `POST /loop-runs/:run_id/stop` (httpapi + udsapi routes +
  `BaseHandlers.StopLoopRun`), `compozy loop stop` (CLI command + client + generated docs),
  `compozy__loop_stop` (descriptor, binding, toolmeta digest, skill table) — replaced by
  `cancel` + `kill`.
- `explicitDependencyBlocker` magic-string sniffing
  (`coordinator_terminal_helpers.go:249`) — replaced by classification-driven blocked mapping
  (`dependency_missing`/`credential_missing`/`resource_unreachable` become classified causes).
- The same-node arm of the run-terminal breaker (`perNodeFailureLimitReached` producing
  `stalled/circuit_breaker`) — redirected to node quarantine; the uncapped-watch-loop arm keeps
  the run terminal (ADR-014.3).
- Goal-only gating of `RetrySpec` consumption comments/lints that forbid generic retry
  (`linter_goal.go` scope narrows to goal-specific members; `CodeRetryMaxUnsupported` for the
  retired `retry.max` key stays).
- Session-supervision defaults as applied to loop-bound sessions (bind-time override to 0 —
  behavior deletion, keys untouched).
- Pre-change loop-run history is NOT deleted — this migration is additive except the three
  table rebuilds (`loop_generation_outputs`, `loop_generations` origin CHECK, `dead_entities`
  kind CHECK), all row-preserving.

## Testing Approach

Strategy only — every concrete case lives in `_tests.md`. Unit tests own classification (pure
truth tables per class and per family), payload-contract detection (v0 rule matrix), precedence
ordering, backoff/epoch arithmetic, lint codes, wait-resume atomicity, and the parked-state classifier —
all as pure state→label tests, no LLM. Store tests extend the canonical
fresh/reopen/ahead/integrity/equivalence migration suites for the five schema changes and add
two-writer race coverage on `loop_generation_outputs` (epoch fencing). Integration tests drive the
coordinator against real SQLite: retry→route→effects→escalation walks, quarantine/requeue,
pause/resume variants, wait park/claim/restart, dedupe claims, breaker open/half-open via
deadentity fakes at the probe seam only. E2E-runtime (Go harness + `acpmock`) walks the QA-flagged
journeys (transient blip heals, error-route fallback, live pause repair, death-resume
continuation, cancel vs kill, duplicate suppression) with deterministic terminal assertions —
never `done|exhausted` alternatives; the effect relay suite induces crash-between-commit-and-
dispatch and asserts at-least-once with stable identity; secret-redaction regressions assert
absence across rows, SSE, CLI, native tools, and `previous.*`. Gates: `make gate` per lane,
`make gate-full` at close; contract changes co-ship `make codegen-check`; E2E mocks/matchers ship
with the contract change (L-007).

## Development Sequencing

### Build Order

1. Schema: the single migration above (5 tables, 3 rebuilds, events column) + sqlc/Atlas
   (`make codegen`) — no behavior.
2. Classification: `FailureClass`, classifier, `result_contract`, hint propagation, attempt
   ledger writes at the boundary (pure + store tests green).
3. DSL + linter: envelope fields, `wait` kind grammar, effect grammar, all lint codes, CEL
   cost-warn.
4. Retry + scheduling: retrying status, backoff, cell epochs, due-scan, deadline anchors,
   opportunistic timers.
5. Routes + absorption: precedence integration, skip cascade reuse, handled dispositions;
   delete magic-string blockers.
6. Effects: pending-outbox relay + idempotent ack + emit/tool execution + pre-bound context +
   `effect_results`; new
   event kinds live end to end.
7. Target health + quarantine: deadentity kind, admission probes, quarantine redirect, requeue
   origin, needs-attention.
8. Liveness: evidence prober, silence flags, death-resume streak, supervision bind override;
   delete the 7m30s kill.
9. Cancel ≠ kill: run + node verbs, parent-close policy, typed sub-loop failure boundary;
   delete `stop` everywhere.
10. Pause + waits: node pause verbs, auto-pause rules, `wait` runtime, governed resume,
    escalation ladders, admission dedupe in the start transaction.
11. Surface co-ship: contract/OpenAPI/TS, routes, CLI, native tools + digests, `make codegen`.
12. Web run UI: lifecycle states, controls, quarantine sheet, inventories + stories +
    screenshot evidence.
13. Web loop editor: Spec 1 lifecycle grammar authoring + chrome states (US-028) + screenshot
    evidence; Start-binding held.
14. Web hero path: catalog, run form, loop detail Visual Contract (US-029) + screenshot
    evidence (reuses status pills/formatters from step 12).
15. Site docs + config-toml + official skill.
16. QA tail: qa-report planning + qa-execution walks (includes editor + hero scenarios).

### Technical Dependencies

None external. Internal: 2 before 4/5/7 (classification feeds retry/routes/breaker); 4 before 10
(epoch/due-scan shared); 6 before 10 (ladders are effects); 5–10 before 11; 11 before 12–15;
12 before 14 (shared formatters/pills); 12–15 before 16. Schema first so every step tests
against the final shape.

## Monitoring and Observability

- Canonical loop events extended with the 15 new kinds, all durable-append-then-broadcast, all
  carrying `loop_run_id`, `generation`, `node_id`, `item_index`, `attempt`, `failure_class`,
  `disposition`, `target`, `epoch`, `delivery_id` as applicable.
- `slog` fields on new paths: the correlation set above plus `wait_kind`, `claim_state`,
  `rule_id`, `streak`, `suppressed_count`, `breaker_state`.
- Coverage-matrix test extended: every new lifecycle path fails the build if its canonical event
  is missing; deadentity transitions for `loop_target` emit through the existing event store.

## Technical Considerations

### Key Decisions

Recorded as ADR-010..017 (grammar; persistence; scheduling; classification; target
health/quarantine; effect dispatch; liveness/cancel; parked states/dedupe). Grill decisions
2026-08-01: `on_*` direct keys with `effects` naming, single shape; effect tools daemon-side in
run scope; only kill Destructive, no new capability gates; `stop` hard-cut to cancel/kill;
defaults v2 (no duration limits; streak-based death-resume; 7m30s kill deleted).

### Known Risks

- **Semantic breadth**: the precedence chain touches every failure path — mitigated by the
  per-family conformance matrix in `_tests.md` and phased build order (classification first).
- **Two-writer epoch discipline** on `loop_generation_outputs` — mitigated by CAS-on-epoch in
  both writers and induced-race tests.
- **Effect volume** on retry-heavy runs — bounded by isolation, payload caps, drain paging;
  `on_retry` is opt-in per node.
- **15s scheduling slack** under-delivers sub-second backoff expectations — accepted in grilling;
  opportunistic timers recover precision on a healthy daemon.
- **`event_key` breaking change** for extension watch sources — greenfield cut, scaffold +
  protocol docs updated in the same change; validation makes the gap loud at authoring time.

## Architecture Decision Records

- [ADR-001..009](adrs/) — PRD-phase decisions (scope slate; escalate-by-default; family-
  asymmetric retry; long-running-first; indefinite visible waits; authorable reactions; target
  health & quarantine; cancel ≠ kill; admission dedupe).
- [ADR-010: DSL failure-vocabulary surface](adrs/adr-010.md) — direct `on_*`, `effects`,
  promoted retry/timeout + `deadline`.
- [ADR-011: Node-lifecycle persistence](adrs/adr-011.md) — controls row, attempt ledger, wait
  rows, claims; outputs columns.
- [ADR-012: Scheduling of future work](adrs/adr-012.md) — 15s due-scan, cell epochs, coordinator-
  mediated retries.
- [ADR-013: Failure classification](adrs/adr-013.md) — closed class enum, result contracts,
  hints, predicate policy mechanics.
- [ADR-014: Target health via deadentity + quarantine redirect](adrs/adr-014.md).
- [ADR-015: Effect dispatch](adrs/adr-015.md) — same-tx effect outbox + idempotent drain; daemon-side tool
  identity.
- [ADR-016: Liveness by evidence](adrs/adr-016.md) — death-resume, silence flags, cancel/kill,
  parent-close.
- [ADR-017: Parked states](adrs/adr-017.md) — `wait` kind, governed resume, pause + auto-pause,
  loud dedupe.
- [ADR-018: Web Visual Contract expansion](adrs/adr-018.md) — editor lifecycle authoring + hero
  path in Spec 1 MVP; start-binding authoring held for Spec 3.

## Compozy Impact Audit

- **Native tools**: +`compozy__loop_cancel`, `compozy__loop_kill`, `compozy__loop_node_pause`,
  `compozy__loop_node_resume`, `compozy__loop_node_cancel`, `compozy__loop_node_kill`,
  `compozy__loop_node_requeue`, `compozy__loop_nodes`; −`compozy__loop_stop`; kill tools carry
  the Destructive risk flag; `compozy__loop_status` output schema + digests refresh; no new
  capability gates (`loops.approve` unchanged).
- **Extensibility and hooks**: additive `loop.node.terminal` payload fields; watch-source
  `PollResponse.event_key` required (breaking, hard cut, SDK scaffold + protocol docs updated);
  effects consume the standard tool surface (extension/MCP/native tools reachable); no
  bundle/registry/bridge-SDK/MCP-sidecar changes (checked — no loop resource kind exists);
  config lifecycle enumerated above.
- **Workspace data isolation**: all new tables are loop-run-scoped via `loop_run_id` FK CASCADE
  except `loop_admission_claims` (workspace-keyed PK) and deadentity rows (workspace-scoped by
  construction); the inventory route filters by workspace path param; SSE events carry
  `workspace_id` as today; tests assert node/wait/claim queries are unreachable across
  workspaces.
- **Official Compozy skill**: `skills/compozy/references/loops.md` updated (verbs table,
  failure contract, event kinds, inventories) in the docs step.

## Assumptions / Defaults

- PRD + `_user_stories.md` (2026-08-01, web delta 2026-08-02 for US-028/US-029) are the scope
  authority; grill decisions of 2026-08-01 bind grammar, identity, gating, verb rename, and the
  defaults table (v2); ADR-018 binds web Visual Contract expansion and start-binding hold.
- Defaults: mechanical retry 3 attempts / 1s base / 30s cap / decorrelated jitter / retry-after
  honored; agent+sub-loop retry 0; death-resume streak 3 (progress-reset); silence window 30m
  (0 disables; transport presence = life); breaker 5 consecutive transport failures / 60s
  half-open probe; quarantine at same-node failure across 2 consecutive generations; predicate
  cost limit 10000 with 80% warn; wait admission 3×60s; tombstone horizon 7d; 15s scheduling
  granularity accepted.
- `goal` nodes are excluded from generic retry/timeout promotion (their machinery is richer and
  untouched); `deadline` is **invalid** on goal nodes — goal keeps only its existing segment
  `timeout` semantics, and `CodeRetryOnGoalNode` rejects the generic fields (N-001; UT-173).
- The existing single-shot schema re-prompt in `action_runagent.go:62-79` stays as-is (idea 11
  is cut; no third feedback loop is built).
- Effect `emit` kinds are author-namespaced strings bounded by the event payload cap; they do not
  enter the closed hook-event taxonomy.
- Conversation in BR-PT; all artifacts (this spec, ADRs, tests, code, docs) in English (SD-003).
