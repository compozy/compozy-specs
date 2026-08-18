# Spec: Loop Graph Completion (graph-eng)

---

# Part I — Product

## Overview

CompozyOS Loops already run declared, deterministic programs with durable checkpoints, human approval gates, retries, quarantine, and full agent-operable control surfaces. A cross-industry comparison against the graph-engineering state of the art (evidence: `analysis/summary.md`) found the engine ahead of the field on durability, static guarantees, and failure semantics — and behind it in exactly three places: **what humans can do mid-run** (only approve), **how paths are chosen** (no exclusive multi-way routing), and **what operators can do with history** (read it, but not compare, re-execute, or branch from it). A fourth cluster — fan-out joins — supports only "wait for everything to succeed", which forces all-or-nothing behavior on workloads that want early exit, quorum, or racing.

This feature closes those four clusters in one coordinated change, and absorbs five verified defects in the same ground (dead configuration knobs, a silently-ignored gate route, a shared revision counter).

**Who it is for:** loop authors (humans and agents writing definitions), operators supervising runs, responders answering human requests, and agent operators managing loops through structured commands.

**Why it is valuable:** loops stop ending or wedging where a person should have been asked; quality verdicts and cheap conditions steer work down the right-cost path; batches degrade gracefully instead of failing whole; and the append-only history the engine already pays for becomes an operator tool — inspect, compare, recover, and experiment without ever mutating the past.

## Goals

- Authors can declare four human interactions — structured input, edit-before-execute, respond-as-result, and (as an operator verb) amend-before-resume — with the same durable parking, escalation ladder, and validation discipline approvals have today.
- Authors can declare exclusive multi-way routing decided by conditions and by gate verdicts, with a mandatory default making runtime no-match impossible.
- Authors can declare fan-out completion strategies (`wait_all`, `fail_fast`, `best_effort`, `race`) with an explicit, first-class partial state that never presents as complete, conditions over live progress, per-fan-out iteration names, and logical width no longer capped.
- Operators and permitted agents can diff generations and runs, re-execute a run from a chosen node, and fork a new linked run from any historical generation with overridden inputs — with the source run untouched, always.
- Every "did not run" outcome (route not taken, pruned, canceled by strategy, never materialized) carries an explicit recorded cause.
- Broken conditions degrade predictably: stop conditions fail open (the loop exits with a diagnostic), routing conditions fail closed (a routable failure), and condition cost is observable.
- Zero dead knobs: the unused progress-fingerprint field is deleted; the silently-ignored gate route selection gains real semantics; gate revision budgets count per gate.

## User Stories

Canonical catalog: [Full user stories](_user_stories.md).

- US-001..US-008 — Human interaction: the four interaction types, responder rules, escalation, responder surfaces, agent answering.
- US-009..US-011 — Routing: router with default, gate-verdict routing, route audit.
- US-012..US-018 — Fan-out: three new strategies, first-class partial, progress conditions, iteration naming, unbounded width, absence causes.
- US-019..US-023 — Time travel: generation diff, run diff, rerun-from-node, fork, capability rules.
- US-024..US-026 — Reliability: fail-open stop conditions, fail-closed routing conditions, per-gate revision budgets.

## Core Features

### 1. Human interaction beyond approval

Four author-declarable ways to bring a person into a run, all built on the same durable parking and validated-answer machinery approvals use today:

- **Structured input request** — the run parks; the responder's answer is validated against the node's declared answer shape and becomes the node's output (US-001).
- **Edit-before-execute** — a proposed action pauses for review; the responder may approve as-is, approve with edited arguments, or reject with feedback; execution uses exactly what was approved (US-002).
- **Respond-as-result** — the responder's answer replaces execution entirely, validated against the node's declared output shape (US-003).
- **Amend-before-resume** — an operator repair verb on any parked node with a declared output shape: correct the output, provenance recorded, original never erased (US-004).

Responders are human by default; authors opt in agent responders per node; self-response is always denied (US-005). Every request carries the reminder → escalation → expiry ladder with an author-declared expiry outcome (US-006). Pending requests are visible and answerable in the product surfaces, and external notification channels deliver a deep link back to answer (US-007, US-008).

### 2. Routing

A **router** node selects exactly one of its named routes by evaluating author conditions in declaration order, with a mandatory default route enforced at compile time (US-009). **Gates** gain real route selection: any gate outcome — machine, judge, or human — may select a pre-declared route, replacing today's silent no-op (US-010). Non-selected routes skip their dominated subgraphs using the existing skip semantics, and every routing decision is recorded with its cause (US-011).

### 3. Fan-out completion strategies

Joins accept a declared strategy (US-012..US-014):

- `wait_all` — today's behavior, remains the default.
- `fail_fast` — the first definitive branch failure cancels remaining branches and takes the failure path.
- `best_effort` — proceed at a declared threshold; requires the author to declare that missing branches are not mandatory coverage; completes in an explicit **partial** state carrying coverage numbers.
- `race` — first success wins; the rest cancel.

Supporting capabilities: a progress namespace for conditions over totals/successes/failures/rates (US-015); author-named iteration variables that un-shadow nested fan-outs (US-016); windowed execution that removes the hard cap on logical width while bounding concurrent work (US-017); and recorded causes for every skipped, pruned, canceled, or never-materialized branch (US-018).

### 4. Operator time travel

Three verbs over the existing append-only history (US-019..US-023):

- **Diff** — compare two generations of a run, or two runs of the same loop (including fork vs origin), at node-outcome and output granularity.
- **Rerun from a node** — open a new generation in the same run that re-executes a chosen healthy node and its dependents; everything else carries forward; operator identity and origin recorded.
- **Fork** — start a new linked run seeded from a chosen generation's outputs with optionally overridden inputs; lineage visible from both sides; the source run untouched.

All three are available to operators and to capability-gated agents, with one restriction: an agent cannot rerun or fork its own still-executing run.

### 5. Reliability corrections

- Broken stop conditions exit the loop with a diagnostic instead of blocking iteration (US-024).
- Broken routing conditions become routable authoring failures; condition cost warnings surface (US-025).
- Gate revision budgets count per gate, per lane (US-026).
- The dead progress-fingerprint field is deleted from the authoring grammar (delete target; ADR-005).

### Feature interactions

- Routing and strategies compose: a router inside a fan-out branch decides per lane; a gate verdict can route within its lane (US-010 EC-3, US-009 EC-4).
- Human interactions park branches independently inside fan-outs; strategies treat parked branches per the business rules below.
- Time travel respects everything else: rerun re-applies routing and strategy decisions afresh in the new generation; fork inherits the loop's concurrency policy like a fresh start.

## Business Rules

1. **Validation is universal.** Every human answer, edited argument set, substituted result, and amended output is validated against the node's declared shape before admission; on failure nothing changes and the request stays pending.
2. **Self-response is always denied.** An agent never answers, approves, edits, or responds to a request originating from a run it started (including through its spawn chain), regardless of per-node opt-in.
3. **Agent responders are per-node opt-in.** Default responder is human; the author's opt-in is explicit per node; capability gates apply before validation.
4. **Escalation is author-owned.** Every human request carries reminder → escalation → expiry with the author's declared expiry outcome (alternate route or halt); defaults match today's approval ladder. Exactly one of "answered" and "expired" wins, deterministically.
5. **Amendment is repair, not authoring.** Amend-before-resume applies only to parked/paused nodes with a declared output shape, is always provenance-recorded, and never erases the original output from history.
6. **Routers cannot fail on no-match.** A router without a default route does not compile; declaration order breaks ties; the taken route and its cause are always recorded.
7. **A broken condition never wedges a run.** Stop conditions fail open (exit with diagnostic); routing conditions fail closed (routable authoring failure); the default route is not taken on a broken condition — default covers no-match only. Authors may override per node.
8. **Partial never presents as complete.** A best_effort join that proceeds with missing branches is partial everywhere it is observable — downstream reads, run status, history — with coverage numbers attached.
9. **Quorum requires a coverage declaration.** best_effort does not compile unless the author declares missing branches are not mandatory coverage.
10. **Early exit really cancels.** fail_fast and race cancel outstanding branches (in-flight and never-materialized alike), and completed branch results are never rewritten by a cancellation.
11. **Strategies act on definitive outcomes.** Retryable failures with retries remaining do not trigger fail_fast; strategy evaluation is monotonic — an admitted completion is never retracted by late arrivals.
12. **History is append-only, everywhere.** Rerun opens a new generation with recorded parentage; fork creates a new run with two-way lineage; no verb mutates or deletes an existing generation, output, or run.
13. **Fork surface is inputs-only.** Fork carries the chosen generation's outputs and accepts overrides of declared inputs under normal input validation; individual node outputs are not editable at fork time.
14. **Time-travel permissions.** Diff is a workspace-authorized read requiring no extra capability. Rerun and fork are capability-gated for agents; an agent may not rerun or fork a run it started while that run is executing; diff is same-loop only, and every time-travel read or mutation resolves runs inside the caller's workspace only.
15. **Absence is always explained.** Every node or branch that did not run carries exactly one recorded cause (route not taken, branch pruned, canceled by strategy, never materialized), distinct from failure.
16. **Revision budgets are per gate, per lane.** Only a gate's own revise decisions consume its budget; operator-origin generations do not.
17. **Existing loop invariants are preserved.** Pinned definitions, deterministic topology (no model-decided rewiring), single-writer cells, capability gating, and redaction rules apply to every new surface.

## User Experience

Personas and their journeys (surface-level change map lives in `_uiux.md` at Stage 2):

- **Loop author** declares the new nodes and strategies in the definition; compile-time diagnostics name every mistake (missing default route, missing coverage declaration, shadowed iteration name, unreachable expiry route) with the same precision existing lint gives.
- **Responder** meets requests where they already meet approvals: the run page's needs-you surface and the pending-requests list, each request showing kind, context, and expected answer shape; external notifications deep-link straight to the answer form; answering, editing, or rejecting is one action with immediate feedback.
- **Operator** supervising a run sees parked human requests, route decisions with causes, live fan-out progress (settled/active/pending, partiality), and absence causes on not-run branches. On any historical run: pick two generations or two runs → diff; pick a node → rerun; pick a generation → fork with an input form pre-filled from the original.
- **Agent operator** does everything above through structured commands with deterministic outcomes, discoverable in the same places existing loop verbs live.

Accessibility follows the product baseline (keyboard-first operation of answer forms and diff navigation; no color-only signaling for partial/canceled states). Discoverability: new verbs appear in existing loop command groups and the run page; no new top-level surface is introduced.

## High-Level Technical Constraints

- **Build on shipped foundations.** Human interactions extend the existing durable parking/validated-admission machinery; routing reuses skip semantics; time travel reads the existing history and generalizes the existing requeue/succession machinery. No second execution engine, no parallel signal plane (ADR-001).
- **Determinism preserved.** Topology remains static and compile-validated; nothing a model emits adds, removes, or rewires nodes or routes.
- **Append-only history preserved.** No rewind-by-mutation anywhere (ADR-002).
- **Agent/operator manageability outcome:** every new capability — answering, amending, diffing, rerunning, forking, strategy inspection — is fully operable through structured command surfaces with deterministic errors, without the web UI (SD-011).
- **Extension ecosystem expectation:** existing notification channels (bridges) carry human-request notifications with deep links; extension-provided watch sources and tools are unaffected; no new extension surface is required for this feature, and third-party participation continues through the surfaces that already exist.
- **Performance from the user's seat:** parking, answering, and routing decisions feel immediate; diffs over large histories return promptly with summarized large outputs; wide fan-outs display aggregate progress without degrading the run page.
- **Privacy/security:** request payloads and notifications obey existing redaction rules; capability gates cover every new verb; provenance (who answered/edited/amended/forked) is always recorded.

## Non-Goals (Out of Scope)

- **Long-run economics** — rollover/continue-as-new, byte budgets, per-node cost rollups and spend-based stopping (loop-ideas 29/30/32): follow-up spec.
- **Node state-machine consolidation** (loop-ideas 33): internal refactor scheduled after this work.
- **Agent-drivable topology** — agents enqueuing nodes into their own run (loop-ideas 35): future RFC; this spec deliberately keeps topology static.
- **Webhook ingress presets** (loop-ideas 26): dormant until external webhooks become a watch-source kind.
- **Bridge-native answering** — responding to human requests from inside external chat tools: notifications deep-link back instead (ADR-001).
- **Restart/soft-reset stall semantics** (loop-ideas 12): stays out; the related dead field is deleted, not wired (ADR-005).
- **Execution-identity caching / selective invalidation** and **live run-topology visualization**: identified by the gap analysis as valuable, deferred to their own efforts.
- **Editing individual node outputs at fork time**: point corrections belong to amend on a live run (ADR-002).

## Open Questions

None at the product level — every fork raised during the two grill rounds was decided and recorded in ADR-001..ADR-005. Surface naming, exact grammar, and payload shapes are Stage 2 subjects and will be settled in `_dx.md`/`_uiux.md` before Part II.

---

# Part II — Technical

## Executive Summary

Every capability in this spec extends the shipped loop engine in place: human requests ride the durable wait-resume plane (a new wait kind plus a matchable side table — never a second signal plane, per v0 anti-lesson 7); routing is a new coordinator-owned control node reusing the branch-skip machinery; completion strategies rewrite the `collect` readiness barrier into a pure, strategy-aware function with real per-lane cancellation; time travel is a read (`diff`), a generalization of the shipped requeue planner (`rerun`), and a seeded start (`fork`) — all forward-only over the append-only lineage. The coordinator stays a pure planner; every new mutation commits through the existing single-transaction snapshot path with epoch CAS. Primary trade-off: windowed fan-out materialization touches the widest surface (materialization, throttling, join arithmetic) and is sequenced late; the whole spec executes after the herdr-parity program merges to main (ADR-006), so the bell surface integrates with the redesigned attention pipeline directly.

## MVP Boundary

MVP boundary: every numbered task in `_tasks.md` composes the MVP, sequenced by the Build Order below (P0 cleanup → P1 router → P2 requests → P3 review/amend → P4 strategies → P5 progress/naming → P6 windowing → P7 diff/rerun → P8 fork → P9 web run-page → P10 bell), closed by the standard trailing `qa-report` + `qa-execution` pair. Nothing in scope is post-MVP; out of scope is exactly Part I's Non-Goals list. The entire spec executes only after herdr-parity is merged to main.

## Developer Experience

- [Developer experience contract](_dx.md) — YAML grammar (`ask`, `route`, `review:`, `strategy:`, `bind_as`/`index_as`, `on_eval_error`), CLI (`loop requests|respond|diff|rerun|fork`, `node amend`, `--item` on node verbs), HTTP/UDS routes, `config.toml` (`requests.expire_after`), native tools, deterministic errors.
- [UI change map](_uiux.md) — S1–S11 across run page, diff view, fork dialog, editor, detail DAG, and the herdr-aligned bell.

## System Architecture

| Component | Responsibility | Lives in |
| --- | --- | --- |
| Request plane | Ask/review request lifecycle: open, validate, admit one answer, expire, escalate | `internal/loop` (coordinator + service + store), new files `request_*.go` / `control_ask.go` / `review_gate.go` |
| Router | `route` node evaluation, gate `{route:}` outcomes, skip propagation, route causes | `internal/loop` `control_plan_route.go`, `control_gate.go`, `gate/routing.go` |
| Strategy engine | Strategy-aware join barrier, per-lane cancellation intents, partial admission, monotonic settlement | `internal/loop` `control_readiness.go` split (`control_join.go`), `fanout_materialization.go`, `cancel_control.go` |
| Window engine | Unbounded logical width with bounded materialization; per-index exactly-once lane creation | `internal/loop` `fanout_window.go` |
| Progress projection | `nodes.<id>.progress.*` + body-scope alias, derived per evaluation from generation outputs | `internal/loop` `control_namespace_progress.go`, `dsl/refs` namespace |
| Time travel | Diff (read), rerun (generalized requeue planner), fork (seeded start + lineage) | `internal/loop` `timetravel_diff.go`, `coordinator_rerun.go`, `service_fork.go` |
| Contracts + transports | New request/diff/rerun/fork payloads; routes; CLI verbs; native tools; capabilities | `internal/api/contract`, `internal/api/core` + `httpapi`/`udsapi`, `internal/cli`, `internal/tools/builtin` |
| Web surfaces | S1–S11 components/hooks per `_uiux.md` | `web/src/systems/loops`; bell composition (S9, post-herdr) edits only the `web/src/systems/os` composition seam — `use-os-attention.ts` (adds the loops source) and the bell row model — never the session-attention contracts (`attention-summary` endpoint, badge derivation in `attention-model.ts` session logic, `session_attention_changed`) |

Data flow: authoring (DSL → linter → pinned snapshot) is unchanged; at runtime the coordinator emits request/route/strategy intents inside the existing `CoordinatorCompletionPlan`; the task layer commits them in one transaction; reads flow through the existing contract payload builders; SSE gains **eight** event kinds on the same stream (`request_opened`, `request_answered`, `request_expired`, `request_canceled`, `node_amended`, `route_taken`, `branch_pruned`, `run_forked`).

## Architectural Boundaries

- **No new Go packages.** Every engine change lands as new files inside `internal/loop` (hard 500-line cap per file, split at design time) and its existing subpackages (`dsl`, `dsl/refs`, `gate`). `internal/loop` continues to import only downward (store interfaces, task contract types); it never imports `daemon/`, `api/`, `cli/`.
- `internal/api/contract` gains the new payload types; `internal/api/core` gains the handlers (`BaseHandlers` methods); `httpapi`/`udsapi` only register — no transport-duplicated parsing.
- `internal/tools/builtin` registers the six new tools + two capabilities; `internal/daemon` wires nothing new beyond the scheduler's existing due-scan (request expiry rides `EscalateDueLoopWaitsPage`'s pattern) and boot reconcile (already generic over waits).
- `web/src/systems/loops` owns every new web surface; `web/src/systems/os` is touched only by the bell composition (S9) and only after herdr-parity's redesigned bell is on main.
- `magefiles/boundaries.go` is unchanged (no new package).

## Implementation Design

### Core Interfaces

DSL additions (`internal/loop/dsl`):

```go
// AskParams is the canonical schema for control kind ask.
type AskParams struct {
	Prompt     string         `json:"prompt"               yaml:"prompt"`
	Context    map[string]any `json:"context,omitempty"    yaml:"context,omitempty"`
	Expect     Schema         `json:"expect"               yaml:"expect"`
	Responders *ResponderSpec `json:"responders,omitempty" yaml:"responders,omitempty"`
	Expires    *WaitExpiry    `json:"expires,omitempty"    yaml:"expires,omitempty"`
}

// ResponderSpec controls who may answer; agents default to deny.
type ResponderSpec struct {
	Agents ResponderAgentPolicy `json:"agents,omitempty" yaml:"agents,omitempty"` // allow | deny
}

// ReviewSpec gates one action node behind a human decision before execution.
type ReviewSpec struct {
	Decisions  []ReviewDecision `json:"decisions,omitempty"  yaml:"decisions,omitempty"` // approve|edit|reject|respond; default [approve, reject]
	When       string           `json:"when,omitempty"       yaml:"when,omitempty"`      // CEL; empty = always
	Prompt     string           `json:"prompt,omitempty"     yaml:"prompt,omitempty"`
	Responders *ResponderSpec   `json:"responders,omitempty" yaml:"responders,omitempty"`
	OnReject   *RejectPolicy    `json:"on_reject,omitempty"  yaml:"on_reject,omitempty"` // {route}; nil = fail quality_rejection
	Expires    *WaitExpiry      `json:"expires,omitempty"    yaml:"expires,omitempty"`
}

// RouteSpec is one ordered route on a route node.
type RouteSpec struct {
	When string `json:"when" yaml:"when"` // CEL bool
	To   NodeID `json:"to"   yaml:"to"`
}

// StrategySpec declares join semantics; string shorthand accepted for kinds without fields.
type StrategySpec struct {
	Kind      StrategyKind       `json:"kind"                yaml:"kind"`      // wait_all|fail_fast|best_effort|race
	Threshold *StrategyThreshold `json:"threshold,omitempty" yaml:"threshold,omitempty"`
	Missing   MissingPolicy      `json:"missing,omitempty"   yaml:"missing,omitempty"` // acceptable — required for best_effort
}

// StrategyThreshold encodes exactly the two frozen YAML forms:
// the scalar percent string ("66%") or the object form ({count: 2}).
type StrategyThreshold struct {
	Kind    ThresholdKind // percent | count
	Percent int           // 1..100 when Kind == percent
	Count   int           // >= 1 when Kind == count
}

// UnmarshalYAML/MarshalYAML (and the JSON pair) are strict closed codecs:
// scalar "NN%" → percent; {count: N} → count; mixed, unknown, or malformed
// forms are decode errors (same strictness as RuntimeSpec.UnmarshalYAML).
// The bijective editor/DSL codec suite carries round-trip cases for both forms.
```

Node envelope additions (flat, matching existing conventions): `Routes []RouteSpec`, `Default NodeID`, `Strategy *StrategySpec`, `BindAs string` (`bind_as`), `IndexAs string` (`index_as`), `Review *ReviewSpec`, `OnEvalError EvalErrorPolicy` (`on_eval_error`: `fail | exit`). `Params` decodes `AskParams` for `kind: ask`.

`contract.stop_when` becomes a string-or-object field (same strict dual-form codec pattern as `strategy`): the string form keeps today's shape with the fail-open default; the object form `stop_when: { expr: "...", on_eval_error: fail }` carries the override — giving the contract-level predicate a real grammar for its policy instead of a node-only field:

```go
// StopWhenSpec accepts the plain expression string or {expr, on_eval_error}.
type StopWhenSpec struct {
	Expr        string          `json:"expr"                   yaml:"expr"`
	OnEvalError EvalErrorPolicy `json:"on_eval_error,omitempty" yaml:"on_eval_error,omitempty"` // default: exit (fail-open)
}
// Strict dual-form UnmarshalYAML/JSON: scalar string → {Expr: s}; object →
// exact keys; anything else is a decode error. Linter validates Expr exactly
// like the string form today.
```

Service surface (`internal/loop`, methods on the existing `Service`):

```go
// Requests
func (s *Service) ListRequests(ctx context.Context, workspaceID string, q RequestQuery) (RequestPage, error)
func (s *Service) GetRequest(ctx context.Context, workspaceID string, ref RequestRef) (RequestDetail, error) // full redacted context + schemas + previews + provenance
func (s *Service) Respond(ctx context.Context, in RespondInput) (RespondResult, error)
func (s *Service) AmendNodeOutput(ctx context.Context, in AmendInput) (LoopMutationResult, error)

// Time travel
func (s *Service) DiffRun(ctx context.Context, workspaceID string, q DiffQuery) (DiffResult, error)
func (s *Service) RerunFromNode(ctx context.Context, in RerunInput) (RerunResult, error)
func (s *Service) ForkRun(ctx context.Context, in ForkInput) (StartResult, error)

type RespondInput struct {
	WorkspaceID string
	RunID       string
	NodeID      string
	ItemIndex   int
	Decision    RequestDecision // "" for ask (implicit respond) | approve|edit|reject|respond
	Payload     json.RawMessage
	Note        string
	Actor       task.ActorContext // trusted, transport-boundary-resolved — never caller-supplied
}

type ForkInput struct {
	WorkspaceID string
	RunID       string
	Generation  int64
	Inputs      map[string]any // overrides; validated like a fresh start
	Reason      string
	RequestID   string // idempotency key; "" derives the canonical request digest
	Actor       task.ActorContext
}
// RerunInput carries the same RequestID/Actor pair. AmendInput carries the
// trusted Actor only — amend has no idempotency key; its concurrency contract
// is the amendment-seq insert CAS. There is no
// ControlActor type anywhere in this design: every mutation input consumes the
// trusted task.ActorContext resolved by the existing transport boundary (the
// same resolver the approve path uses), and the one daemon adapter feeds it to
// ResponderPolicy. Operation-specific rule: respond/approve/amend deny the
// initiator chain always; rerun/fork deny it only while the source run is
// executing (an agent may time-travel its own *terminal* runs).

// ResponderPolicy is the trust boundary for self-operation rules. It is
// implemented in daemon/ (trusted task.ActorContext resolution + durable
// session spawn lineage) and injected into loop.Service; internal/loop never
// derives identity from actor-ref formatting or caller-supplied fields.
// One policy instance serves approve, respond, amend, rerun, and fork.
type ResponderPolicy interface {
	// DeniesSelfOperation reports whether actor belongs to the initiator
	// chain (starter or transitively spawned child) of runID, scoped to
	// workspaceID. Missing or stale lineage fails closed (denied, error nil).
	DeniesSelfOperation(ctx context.Context, workspaceID, runID string, actor task.ActorContext) (bool, error)
}
```

Coordinator additions stay pure: `control_plan_route.go` evaluates routes and returns skip/cause intents; `control_join.go` computes `JoinSettlement{Outcome, Admit, CancelLanes []LaneRef, Coverage}` as a pure function of branch cell states + `StrategySpec`; `coordinator_rerun.go` computes the rerun set exactly as `coordinator_requeue.go` does today, with origin `operator_rerun`.

### Data Models

New side table `loop_requests` (loops stream, `50_loops.sql` fragment + next Goose migration). The row keeps a hard **private-execution vs public-projection split**: `*_ref` columns are content-addressed store-internal payloads (exact, unredacted — same trust plane as `loop_output_blobs`; consumed only by the execution path), while `*_preview_json` columns are redacted + bounded and are the only inline values transports return. `GET /loop-requests` and run-detail embeds return previews plus fetchable redacted-full refs; SSE/events carry previews only; the raw `proposed_ref` payload never crosses an API, event, log, or UI boundary.

```
loop_requests
  workspace_id         TEXT NOT NULL            -- workspace scope, matches every loop table
  loop_run_id          TEXT NOT NULL            -- FK loop_runs
  generation           INTEGER NOT NULL         -- generation that opened the request
  node_id              TEXT NOT NULL
  item_index           INTEGER NOT NULL DEFAULT 0
  kind                 TEXT NOT NULL CHECK (kind IN ('ask','review'))
  state                TEXT NOT NULL CHECK (state IN ('pending','answered','expired','canceled'))
  prompt               TEXT NOT NULL            -- rendered at open, redacted, bounded (public)
  context_preview_json TEXT NOT NULL DEFAULT '{}' -- redacted + bounded inline preview (public)
  context_ref          TEXT                     -- content-addressed redacted FULL context (public via fetch)
  answer_schema_json   TEXT                     -- ask: the answer shape (public)
  edit_schema_json     TEXT                     -- review+edit: the action input shape snapshot (public)
  respond_schema_json  TEXT                     -- review+respond / ask: the output shape snapshot (public)
  decisions_json       TEXT NOT NULL            -- allowlisted decision set for this request (public)
  proposed_ref         TEXT                     -- review only: EXACT resolved-params snapshot (private, execution-only)
  proposed_preview_json TEXT                    -- review only: redacted + bounded preview (public)
  answered_decision    TEXT                     -- approve|edit|reject|respond when answered
  answered_payload_ref TEXT                     -- content-addressed admitted payload (private; validated echo returns preview)
  answered_note        TEXT
  actor_kind           TEXT                     -- provenance on resolution (answer or cancel)
  actor_id             TEXT
  opened_at            TEXT NOT NULL
  resolved_at          TEXT                     -- set on answered|expired|canceled
  expires_at           TEXT
  PRIMARY KEY (loop_run_id, generation, node_id, item_index)
  INDEX idx_loop_requests_pending (workspace_id, state, expires_at)
```

Per-decision validation uses the matching schema column — `edit` validates against `edit_schema_json`, `respond` against `respond_schema_json`, ask answers against `answer_schema_json`; a decision whose schema column is NULL is not in `decisions_json` (lint guarantees the pairing). **Cancellation is part of the atomic lifecycle**: every path that terminally removes a pending request's cell — node/run cancel or kill, route-skip of the dominated subgraph, strategy cancellation, prune — claims the wait and transitions the request row to `canceled` in the same store transaction, recording the canceling actor/cause and appending a `request_canceled` event. Exactly one of answer / expiry / cancel wins, decided by store transaction order.

New side table `loop_node_amendments` (append-only amendment provenance):

```
loop_node_amendments
  workspace_id   TEXT NOT NULL
  loop_run_id    TEXT NOT NULL
  generation     INTEGER NOT NULL
  node_id        TEXT NOT NULL
  item_index     INTEGER NOT NULL DEFAULT 0
  amendment_seq  INTEGER NOT NULL             -- 1..N per cell, monotonic
  original_ref   TEXT NOT NULL                -- output_ref before this amendment
  amended_ref    TEXT NOT NULL                -- output_ref after this amendment
  actor_kind     TEXT NOT NULL
  actor_id       TEXT NOT NULL
  reason         TEXT
  created_at     TEXT NOT NULL
  PRIMARY KEY (loop_run_id, generation, node_id, item_index, amendment_seq)
```

Amend eligibility: the cell must carry a settled output (`output_ref` present) **and** be paused/parked (or the run paused); a cell with no output rejects with `amend_no_output`. **Amendments are an append-only overlay — `loop_generation_outputs` stays immutable after settlement** (Part I Business Rule 12 holds literally): an amend inserts exactly one `loop_node_amendments` row and touches no output row. The cell's **effective output** — what namespace reads, resume, and downstream consumption see — is the newest amendment's `amended_ref` when any exists, else the recorded `output_ref`; history and diff expose both the recorded and the effective value. Concurrency control is the insert itself: `amendment_seq` is a CAS (insert at `latest+1`; a duplicate seq loses and re-reads). `amendments[]` returns full redacted original/amended values inline when within the payload bound, else a size + content-hash summary — there is no raw ref-read surface in v1, stated explicitly. Amend never retroactively re-runs consumers — pairing with `loop rerun --from-node` is the documented recovery combo.

**Blob durability roots:** every new ref column is a first-class root for the shipped orphan sweep — the sweep's root set extends to `loop_requests.context_ref`/`proposed_ref`/`answered_payload_ref` and `loop_node_amendments.original_ref`/`amended_ref` (alongside generation outputs and goal tables), so pending and resolved requests, every amendment original, and fork seeds stay readable for the life of the run while true orphans are still reclaimed at the existing transition/terminal boundaries. Canonical sweep suites extend with retention + reclamation cases for each new root.

New side table `loop_timetravel_ops` (idempotent operation ledger for rerun/fork):

```
loop_timetravel_ops
  workspace_id     TEXT NOT NULL
  op_id            TEXT NOT NULL               -- daemon-generated
  kind             TEXT NOT NULL CHECK (kind IN ('rerun','fork'))
  idempotency_key  TEXT NOT NULL DEFAULT ''    -- caller request_id; '' = fresh operation, no replay identity
  request_digest   TEXT NOT NULL               -- canonical hash of the validated input (key-reuse validation)
  source_run_id    TEXT NOT NULL
  source_generation INTEGER
  from_node        TEXT                        -- rerun only
  item_index       INTEGER                     -- rerun only
  actor_kind       TEXT NOT NULL
  actor_id         TEXT NOT NULL
  reason           TEXT
  result_run_id    TEXT NOT NULL               -- rerun: same run; fork: the new run
  result_generation INTEGER                    -- rerun only
  created_at       TEXT NOT NULL
  PRIMARY KEY (workspace_id, op_id)
  UNIQUE (workspace_id, idempotency_key) WHERE idempotency_key != ''   -- partial: keys only
```

**Idempotency identifies caller intent, never content**: an omitted `request_id` is a **fresh operation** — every acknowledged keyless call performs its effect and writes its own ledger row (two intentional identical reruns or forks are two operations). The replay guarantee exists only for an explicit key: a matching `(idempotency_key, request_digest)` returns the prior result; the same key with a different digest rejects `timetravel_key_reuse`. Rapid duplicate protection without a key comes from the shipped guards (`rerun_busy` while a generation is in flight; the loop's concurrency policy for forks). **Fork extends the canonical start path** — the same atomic start transaction that applies concurrency policy, creates generation records, reserves the single coordinator `task_run`, and dispatches `loop.started` at the owning call site — with a validated seed + lineage. No peer start path, no second queue (L-003/L-004/L-005).

- **Side-table, not JSON**: request state is matchable (inventory by state, expiry scans, bell aggregates count) — JSON blobs are forbidden for matchable state. The pending wait itself stays in `loop_node_waits` (new `wait_kind = 'request'` joins the existing CHECK) so scheduler expiry/escalation and boot reconcile reuse the shipped machinery; `loop_requests` carries the request-shaped read model and the answer record.
- `loop_runs` gains `forked_from_run_id TEXT` + `forked_from_generation INTEGER` (nullable pair, CHECK both-or-neither) — columns, not JSON, because lineage is queried both directions (`forks[]` is a projection query on these columns). A fork's source must live in the caller's workspace; cross-workspace source resolution fails as unknown-run before any read.
- `loop_runs` gains `completion_state TEXT NOT NULL DEFAULT 'complete' CHECK (completion_state IN ('complete','partial'))` — the run-boundary partiality projection (Business Rule 8). Set at terminal commit: `partial` when any `best_effort` join on the terminal path settled partial. Propagated through run list/detail payloads, CLI `loop status`/`loop runs` output, native-tool output schemas, SSE `status_changed` payloads, and the web outcome card — one canonical field, every surface.
- **Fork seed semantics (no dependence on deferred execution-identity):** the fork's baseline is **durably represented as the child run's generation 1 — a pre-settled seed generation**. The canonical start transaction writes, in the child run: one `loop_generations` row `(generation=1, parent_generation=0, origin='fork_seed')`; one `loop_generation_outputs` row per source cell, pre-settled `succeeded` with the source `output_ref`; the referenced `loop_output_blobs` rows copied by content address into the child run; and `best_generation=1` (+ the source's best score when finite). No seed cell executes. The first **executing** generation is 2 (`origin='initial'`, `parent_generation=1` — intent validation allows `initial` to parent the seed), so the shipped same-run history reader hydrates `previous.*`/`best.*` for it unmodified — no cross-run read path exists or is needed. Overridden inputs therefore can never coexist with stale derived outputs (nothing is carried as current work), and a no-override fork is a fresh full-body attempt from the same baseline. Missing source blob → the start transaction aborts whole (deterministic rejection, no partial child).
- `loop_generations.origin` gains two values, each in its owning phase's migration: `operator_rerun` (P7) and `fork_seed` (P8). The briefing's other candidates are dead by decision (restart held with `hash_fields` deleted; `operator_reset` impossible in the forward-only model; `inbox_update` answers settle cells without opening generations) — speculative enum values are the anti-lesson-1 failure mode.
- `loop_run_events` kind CHECK adds `request_opened`, `request_answered`, `request_expired`, `request_canceled`, `node_amended`, `route_taken`, `branch_pruned`, `run_forked` (all bounded/redacted like existing kinds; request events carry previews only).
- Collect cells reuse `loop_generation_outputs` with a new status value `partial` (status CHECK extends) and a structured `output_ref` payload `{total, succeeded, failed, canceled, coverage_rate, partial}`.
- Gate revision counters move to `loop_node_controls` (`gate_revisions_json` — per-lane counters keyed by item index) — control-plane state on the existing per-node control row; not matchable across runs, so a JSON column on the owning row is correct.
- Config: `LoopRequestsDefaultConfig{ExpireAfter string \`toml:"expire_after"\`}` nested under `LoopDefaultConfig` as `Requests`; `fan_out_width`/`max_fan_out` ceiling validation drops the 64 cap (positive bound only).

### API Endpoints

Per `_dx.md`, all as `BaseHandlers` methods registered by both transports: `ListLoopRequests` (GET `/loop-requests`), `GetLoopRequest` (GET `.../nodes/:node_id/request` — the full request record: full redacted context, per-decision schemas, previews, state, provenance; the workspace-authorized read that makes `context_ref` retrievable end to end across CLI/HTTP/UDS/native tools), `RespondLoopNode` (POST `.../nodes/:node_id/respond`), `AmendLoopNode` (POST `.../nodes/:node_id/amend`), `DiffLoopRun` (GET `.../diff`), `RerunLoopRun` (POST `.../rerun`), `ForkLoopRun` (POST `.../fork`). Run detail additionally embeds `amendments[]` — the typed public projection of `loop_node_amendments` (node, item, sequence, redacted original/amended previews + refs, actor, reason, timestamp) — consumed by `loop status`, HTTP/UDS run detail, native `compozy__loop_status` output, the diff view (amended cells marked), and S2/S5. Existing node verbs accept `item_index`; rerun/fork bodies accept optional `request_id`. Error mapping: `request_validation_failed` → 422 with field details (reuses the schema-validation error shape from wait admission), `request_already_answered` → 409 carrying the recorded decision, `request_expired` → 410, `request_canceled` → 410, `respond_not_permitted`/`respond_self_denied`/`timetravel_self_denied` → 403, `rerun_busy`/`timetravel_key_reuse` → 409, `rerun_node_unsettled`/`amend_no_output` → 422, `fork_generation_unknown` → 404, `diff_cross_loop` → 422. Requests filtering: `state=pending` maps to the `pending` row state; `state=resolved` is the closed union `answered | expired | canceled`. Ordering is stable and pinned: pending sorts by `expires_at ASC NULLS LAST, opened_at ASC, rowid ASC`; resolved sorts by `resolved_at DESC, rowid DESC`; cursors encode the sort key + rowid and stay stable across the union. OpenAPI + generated TS + generated CLI docs co-ship (`make codegen`).

## Impact Analysis

Compozy Impact Audit:

- Native tools: seven new tools (`compozy__loop_requests`, `compozy__loop_request`, `compozy__loop_respond`, `compozy__loop_node_amend`, `compozy__loop_diff`, `compozy__loop_rerun`, `compozy__loop_fork`) in toolset `compozy__loops` with descriptors, I/O schemas, and risk classes per `_dx.md`; existing node-verb tools gain `item_index` (schema digests change; per-run pinning unaffected); two new capabilities `loops.respond` + `loops.timetravel` beside `loops.approve`.
- Extensibility and hooks: no new hook events, no extension manifest/provide-surface change, no bridge SDK change (notify = existing effects + deep link); MCP sidecars, registries, marketplace schemas unchanged — checked and listed in the Extensibility Integration Plan; `skills/compozy/references/loops.md` and site loop docs update in the same phases; `config.toml` lifecycle limited to `loops.defaults.<kind>.requests.expire_after` + the `fan_out_width` ceiling removal.
- Workspace data isolation: every new datum is **workspace-scoped** — `loop_requests`, `loop_node_amendments`, `loop_timetravel_ops`, and the `loop_runs` fork-lineage/completion columns all key `workspace_id`, and every list/read/write/join predicates on it. Fork resolves its source strictly inside the caller's workspace (cross-workspace source = unknown run, fail-closed before any read); lineage projections (`forked_from`/`forks[]`) validate ownership before following run ids; diff resolves both sides in one workspace; SSE remains per-run under workspace-scoped routes; web query keys stay workspace-scoped (existing `loopsKeys` pattern + a workspace-scoped `requestsByWorkspace` key); bell composition consumes per-workspace `aggregates.pending` with per-workspace error isolation; native tools bind workspace exactly like existing loop tools. Cross-workspace negative tests for all seven operations (request detail included) + lineage + SSE + bell counts are IT-016.
- Official Compozy skill: `skills/compozy/references/loops.md` updates required (tool table rows, grammar prose for `ask`/`route`/`review`/`strategy`/`bind_as`/`on_eval_error`, request/respond/amend semantics, time-travel semantics + idempotency, capability list, `completion_state`).

| Component | Impact Type | Description and Risk | Required Action |
| --- | --- | --- | --- |
| `internal/loop/dsl` + linter | modified | New grammar + ~10 lint codes; medium risk (grammar is contract) | Lint tables + fixtures; editor codec parity |
| `internal/loop` coordinator | modified | Route/join/window/rerun planning; high risk (core engine) | Pure-function tests per planner; race suite |
| `internal/store/globaldb` | modified | New table + columns + CHECK rebuilds; medium risk | `eng-schema-migration` full suite (fresh/reopen/ahead/integrity/equivalence) |
| `internal/api/*`, `internal/cli`, `internal/tools` | modified | Seven operations ×3 surfaces + capabilities; low risk (additive) | Parity tests + codegen |
| `internal/config` | modified | `requests.expire_after`; ceiling removal; low risk | Config lifecycle suite |
| `web/src/systems/loops` | modified/new | S1–S8, S10–S11 components/hooks | MSW fixtures + Playwright |
| `web/src/systems/os` | modified | Bell composition (post-herdr) | Integrates with merged bell |
| `skills/compozy` + `packages/site` | modified | Docs for every new surface | Tool table + loops docs pages + CLI pages |

**Delete Targets** (no fallback, no compat shim, no placeholder — exact artifact paths; the implementing task greps each symbol to zero):

- `hash_fields` (B-1): `internal/loop/dsl/contract.go` (`NoProgress.HashFields`), `internal/api/contract/loops.go` (the `NoProgress` mirror field), every fixture/test referencing `hash_fields` under `internal/loop/**` and `internal/api/**`, the `no_progress` example line in `packages/site/content/docs/loops/dsl-reference.mdx` (contract block), any `hash_fields` mention in `skills/compozy/references/loops.md`, and regenerated OpenAPI/TS (`openapi/compozy.json`, `web/src/generated/compozy-openapi.d.ts` via `make codegen`). Lint reports `unknown_parameter`.
- Gate route action string `branch` (B-4): `internal/loop/gate/types.go` (`RouteBranch`, verdict `Branch` field plumbing), `internal/loop/gate/routing.go` (legality rows + branch-name requirement), `internal/loop/control_gate.go` (the no-op case), gate fixtures under `internal/loop/**`, the `on_result` route vocabulary rows in `packages/site/content/docs/loops/dsl-reference.mdx` and `skills/compozy/references/loops.md`, and the editor's on_result option list in `web/src/systems/loops` — replaced by the `on_result` `{route:}` object form (lint `route_action_removed`).
- The generation-derived gate revision computation (`GateInput.Revision = generation-1`) — replaced by per-gate per-lane counters (B-5).
- The 64-width ceilings: `LoopMaxFanoutWidth` enforcement in lint (`fan_out_ceiling_exceeded` as a cap-64 rule), runtime `fan_out_width_exceeded` at 64, and the config validation cap — replaced by window-bounded materialization with author-declared bounds.
- Dead code activation, not deletion: `ApplyPredicateFailurePolicy` gains its three call sites and the `on_eval_error` grammar (B-2/B-3) — the currently-unreachable behavior "broken `stop_when` blocks succession" is removed.

## Extensibility Integration Plan

- **Native tools**: seven new `compozy__loop_*` tools in toolset `compozy__loops` (descriptors, input/output schemas, risk classes per `_dx.md` — `compozy__loop_request` is a read like `compozy__loop_requests`); two new capabilities `loops.respond`, `loops.timetravel` registered beside `loops.approve`. Tool schema digests change for node verbs gaining `item_index` — digest-pinned runs are unaffected (pinning is per-run).
- **Hooks**: no new hook events; the existing seven `loop.*` hooks fire unchanged. Request lifecycle is observable via run events + effects, not hooks (hooks stay typed dispatch at transitions that already exist).
- **Effects/bridges**: author-declared `expires.escalate` effects carry bridge notifications for requests (existing `emit`/`tool` grammar; deep link = the run's `web_url`); no bridge SDK change (ADR-001 keeps bridge-native answering out).
- **Extensions/MCP/registries**: unaffected — checked: extension manifests (no new provide surface), `loop.watch_source` contract untouched, marketplace/registry schemas untouched.
- **Official Compozy skill**: `skills/compozy/references/loops.md` updates are mandatory — tool table rows for the six verbs, grammar prose for `ask`/`route`/`review`/`strategy`/`bind_as`/`on_eval_error`, request/respond semantics, time-travel semantics, capability list.
- **Protocol docs**: Compozy Network untouched (loop execution never travels the wire — checked `network_participation` surfaces).

## Agent Manageability Plan

Every capability is agent-operable without the web UI, extending the canonical triple table (`running.mdx`):

| Capability | CLI | HTTP/UDS | Native tool |
| --- | --- | --- | --- |
| List requests | `compozy loop requests` | `GET /loop-requests` | `compozy__loop_requests` |
| One request in full | `compozy loop request` | `GET .../nodes/:id/request` | `compozy__loop_request` |
| Answer/decide | `compozy loop respond` | `POST .../nodes/:id/respond` | `compozy__loop_respond` (gated `loops.respond`) |
| Amend output | `compozy loop node amend` | `POST .../nodes/:id/amend` | `compozy__loop_node_amend` (gated `loops.respond`) |
| Diff | `compozy loop diff` | `GET .../diff` | `compozy__loop_diff` |
| Rerun | `compozy loop rerun` | `POST .../rerun` | `compozy__loop_rerun` (gated `loops.timetravel`) |
| Fork | `compozy loop fork` | `POST .../fork` | `compozy__loop_fork` (gated `loops.timetravel`) |

All outputs structured (`-o json|jsonl|toon`); deterministic error codes per `_dx.md` Errors; discovery via `compozy__tool_info`; self-denial rules enforced daemon-side regardless of surface. Status/config discovery: requests embed in run detail; lineage in `loop status`; new lint codes in `loop validate`.

## Config Lifecycle

- **Added**: `loops.defaults.delivery.requests.expire_after` + `loops.defaults.watch.requests.expire_after` (string duration, default `""` = no expiry). Struct + defaults + overlay merge + clone + validation (must parse as positive duration when set) + example in config docs + site config page + config round-trip tests — same change.
- **Changed**: `loops.defaults.<kind>.fan_out_width` validation drops the `<= 64` ceiling (nonnegative only); docs updated; the ceiling test becomes a bound test.
- **Checked, unaffected**: `loops.breaker.*`, `loops.inputs.*`, `[roles]`, `[attention]` (herdr-owned), session/provider config — no key added, renamed, or removed there.

## Testing Approach

Strategy only — concrete cases live in [`_tests.md`](_tests.md). Unit: table-driven pure-planner tests (route selection, join settlement per strategy, window advance, rerun-set computation, request admission decisions, revision counters), linter tables for every new code, CEL namespace tables for `progress.*`/`bind_as` — fakes only at store/clock boundaries, `t.Run("Should …")` + `t.Parallel`, `-race`. Integration (`+integration`): migration suites (fresh/reopen/ahead/integrity/equivalence) for the new table/columns/CHECKs; store CAS races (concurrent respond, amend, rerun idempotency); scheduler expiry exactly-once; HTTP=UDS parity for the six verbs. E2E runtime (Go + acpmock): ask → respond → resume; review → edit → execute-once; fail_fast cancels lanes; best_effort partial propagates; rerun opens `operator_rerun`; fork starts a linked run. E2E web (Playwright + MSW): request answer form, diff view, fork dialog, editor round-trip of new grammar. `make codegen-check` guards contract drift; 80% per-package floor; `make gate` per phase, `make gate-full` at close.

## Development Sequencing

### Build Order

**Per-phase co-ship rule (public surfaces):** a phase that touches any public surface (P1–P8) is complete only when its contract types, HTTP **and** UDS registration, CLI verb, native tool, `make codegen` output, official-skill rows, and site-doc rows land in the same PR as the behavior — no "docs later", no CLI-without-UDS interim. The Tail phase is final verification and assembly only, never the first home of a public artifact.

- **P0 — Cleanup batch** (behavior-scoped, no new surface): delete `hash_fields` everywhere; wire `ApplyPredicateFailurePolicy` + `on_eval_error` grammar + cost surfacing; per-gate/lane revision counters. Gate: scoped `go test -race` + lint tables.
- **P1 — Router**: `route` node (lint + coordinator + events `route_taken`) + gate `{route:}` object form + `branch` action deletion. Gate: planner tables + linter fixtures.
- **P2 — Requests core**: `ask` grammar, `loop_requests` migration, wait kind `request`, the full request read/write set — list, **detail (full-context read)**, respond — across all three surfaces, escalation/expiry, self-denial, capabilities. Gate: migration suite + admission race tests + seven-operation parity.
- **P3 — Per-lane addressing, review + amend** (strict internal order): **first** the complete per-lane mutation primitive — `ItemIndex` through the cancellation/pause/resume/kill mutations, store identity, post-commit session delivery, and every operator/native surface (`--item`/`item_index`) with tests — then the `review:` block with pre-execution gating and edited-params execution, then the `amend` verb. Nothing in P3 exposes an `--item` surface before the primitive behind it is complete. Gate: per-lane mutation suite + e2e runtime review flow.
- **P4 — Strategies** (consumes P3's per-lane primitive): `strategy:` grammar, join settlement, partial status + collect output + `completion_state`, `branch_pruned` events. Gate: settlement tables + cancellation e2e.
- **P5 — Progress + naming**: `progress.*` projection + alias, `bind_as`/`index_as` scoping. Gate: namespace tables + linter fixtures.
- **P6 — Windowing**: window engine, ceiling removals (lint/runtime/config), wide-run truthful counts. Gate: window race suite + 500-lane integration run.
- **P7 — Diff + rerun**: diff read service; P7's own migration (`loop_timetravel_ops` + the `operator_rerun` origin rebuild — never riding another phase's migration); rerun planner + guards + idempotency. Gate: diff golden tests + rerun e2e.
- **P8 — Fork**: lineage columns, seeded start pinning the source snapshot, input validation, `run_forked` event. Gate: fork e2e + lineage projections.
- **P9 — Web run-page**: S1–S8, S10–S11 per `_uiux.md`, MSW fixtures, Playwright. Gate: `make gate` web lanes + screenshot bundles per the design pass.
- **P10 — Bell** (S9): compose loop-request rows + `aggregates.pending` into the merged herdr bell. Gate: Playwright bell journey.
- **Tail** (verification and assembly only — no first-time source mutation): `make gate-full`, docs link-check, cross-phase consistency sweep, then the `qa-report`/`qa-execution` pair. Every site-doc page and `skills/compozy` row lands inside its owning P1–P10 phase per the co-ship rule; the Tail only verifies they did.

**Migration ownership (append-only Goose identity — every schema mutation has exactly one owning phase, in execution order; no phase ever edits, reuses, or conditionally rides another phase's migration):**

| Phase | Owning migration contents |
| --- | --- |
| P0 | `loop_node_controls.gate_revisions_json` column |
| P1 | events CHECK rebuild + `route_taken` |
| P2 | `loop_requests` table; wait-kind CHECK + `request`; events CHECK + `request_opened`/`request_answered`/`request_expired`/`request_canceled` |
| P3 | `loop_node_amendments` table; events CHECK + `node_amended` |
| P4 | `loop_runs.completion_state`; outputs status CHECK + `partial`; events CHECK + `branch_pruned` |
| P7 | `loop_timetravel_ops` table; origins CHECK + `operator_rerun` |
| P8 | `loop_runs` fork-lineage columns; origins CHECK + `fork_seed`; events CHECK + `run_forked` |

Each migration updates the declarative `50_loops.sql` fragment, refreshes `atlas.sum` + sqlc output via `make codegen`, and extends the canonical fresh/reopen/ahead/integrity/equivalence suites in the same phase. P5/P6/P9/P10 are schema-free.

### Technical Dependencies

herdr-parity merged to main (global precondition, ADR-006). No other external dependencies; every phase depends only on earlier phases.

## Monitoring and Observability

New run events (`request_opened|request_answered|request_expired|request_canceled|node_amended|route_taken|branch_pruned|run_forked`) join the durable per-run ledger + SSE with the existing redaction/16 KiB bounds; `route_taken` carries `{route, cause, matched_when|default}`; `branch_pruned` aggregates at high width (bounded payload, queryable detail). Structured logs on every new service path carry the standard correlation keys (`workspace_id`, `run_id`, `node_id`, `item_index`, `actor_kind`, `actor_id`). Counters worth watching in dev: pending-request age (expiry scan lag), join settlement latency at width, fork seed size.

## Technical Considerations

### Key Decisions

- **Two origin values, two phase-owned rebuilds** — `operator_rerun` (P7's migration) and `fork_seed` (P8's migration), each its own next-gap-free append-only rebuild; the briefing's other candidates are dead by decision (rationale in Data Models). Trade-off: a future restart/inbox origin pays its own rebuild; acceptable against shipping dead enum values or branch-timing-dependent migration identity.
- **Requests reuse the wait plane** — `wait_kind = 'request'` + side table, not a new parking mechanism: scheduler, boot reconcile, escalation, and epoch fencing come free (anti-lesson 7).
- **Review gates at enqueue, not inside the action** — the coordinator holds the `NodeRun` until admission, so execution consumes exactly the approved params snapshot; no mid-execution interception.
- **Fork pins the source snapshot** — the forked run reuses the source run's executed-definition digest (not the live catalog), preserving replay meaning; a definition upgrade is a fresh start, not a fork.
- **Fork seeds baseline, never current state** — the child's generation 1 is the pre-settled seed; **generation 2 is the first executing generation** and hydrates `previous.*`/`best.*` from the seed via the unmodified same-run history reader. This makes fork sound without the deferred execution-identity feature (no stale-output/overridden-input coexistence). Selective carry-forward on fork is explicitly future work gated on execution identity.
- **Fork start transition (exact, through the canonical start path)** — the one start transaction inserts the child `loop_runs` row already at `generation = 1` (the seed rows are written settled in the same transaction), writes the `(1, 0, fork_seed)` generation record, reserves the coordinator for **generation 2** with intent `(2, 1, initial)`, and applies concurrency policy exactly like a fresh start — a `queue`d fork writes its seed at insert and gets its coordinator reservation on promotion, and boot reconcile sees a normal run at generation 1 with a pending generation-2 coordinator. Best projection respects the shipped both-or-neither CHECK: `best_generation = 1` + `best_score = <source best_score>` only when the source carries a finite best score; otherwise both stay NULL and only `previous.*` hydrates. Fork-of-fork repeats the same procedure against the fork's own generations.
- **`progress.*` scope rule** — the qualified form `nodes.<fanout>.progress.*` is valid wherever that node id is referenceable (standard namespace visibility, any downstream condition); the bare `progress.*` alias is valid only inside that fan-out's body. One rule, stated once, mirrored by the linter and the DX contract.
- **Diff is computed daemon-side** — one implementation serves CLI/HTTP/web identically; content-level diff only where shapes are comparable, else size/hash summary.

### Known Risks

- Join-settlement × retry interaction (does `fail_fast` fire on retryable failure?) — settled: only definitive failure (retries exhausted or non-retryable) triggers strategies; encoded in invariants and tests.
- Window engine racing lane terminals — mitigated by epoch fencing per lane + pure window-advance function with race suite.
- Editor codec drift for new grammar — the bijective codec test table extends with every new field (P1/P4/P5 gates).
- Request payload sprawl in `context_json` — bounded + redacted at open (16 KiB, same as events); truncation marker per `_uiux.md`.

## Safety Invariants

1. Exactly one answer admits per request: admission claims the parked cell via epoch CAS; losers receive `request_already_answered` with the recorded decision; the store transaction is the tiebreak between answer and expiry.
2. Self-operation denial (respond, approve, amend, rerun, fork) is decided by the one daemon-injected `ResponderPolicy` — trusted `task.ActorContext` identity + durable spawn lineage, workspace-scoped — before validation, on every surface, regardless of capability; missing or stale lineage fails closed.
3. A reviewed action executes at most once, with exactly the admitted params snapshot (original `proposed_ref` or the validated edit); the snapshot is immutable after admission, and its raw payload never crosses an API, event, log, or UI boundary — transports see previews only.
4. Join settlement is monotonic: once an outcome (met / failed / won) commits for a collect cell, later lane terminals cannot change it; late results append `late_arrival` events only.
5. Early-exit settlement delivers per-lane cancellation post-commit to every outstanding lane; a lane that completed before delivery keeps its result — completed cells are never rewritten to canceled.
6. Window materialization creates each `(generation, node, item)` lane exactly once under epoch fencing; never-materialized lanes settle `canceled` with a never-started cause and no task run.
7. Fork reads the source run read-only in one snapshot transaction, resolves the source strictly inside the caller's workspace, seeds only the `previous.*`/`best.*` baseline (never current cells), and goes through the canonical start transaction — concurrency policy, generation records, the single coordinator `task_run` reservation, and `loop.started` dispatch at the owning call site. Lineage columns are written only at fork creation; the source's `forks[]` is a projection query, never a source-row mutation.
8. Every rerun and fork records exactly one `loop_timetravel_ops` row in the same transaction as its effect. An explicit key is a replay identity (matching key + digest returns the prior result; key reuse with a different digest rejects `timetravel_key_reuse`); an omitted key is a fresh operation with no replay identity. A rerun's parent is the latest settled generation with parked cells excluded from the rerun set.
9. Amend is an append-only overlay: each amendment inserts one immutable `loop_node_amendments` row (original ref, amended ref, actor, reason, timestamp) with `amendment_seq` as the insert CAS; settled `loop_generation_outputs` rows are never mutated; the effective output is resolved as newest-amendment-else-recorded at every read; every amendment ref is an orphan-sweep root; amend requires a settled output on a paused/parked cell and never re-runs consumers implicitly.
10. Route selection is deterministic: declaration order is the tiebreak, `default` fires only on no-match, and a broken condition is an authoring failure — never a silent default.
11. Gate revision budgets count per `(gate, item_index)` from persisted counters; operator-origin generations do not consume authored budgets.
12. Request expiry fires exactly once across the timer path and the boot/scan backstops (shared idempotency key), and every expiry outcome (route/halt) is recorded with cause `wait_expired`.
13. Request cancellation is atomic with its cause: any path that terminally removes a pending request's cell (node/run cancel or kill, route skip, strategy cancel, prune) claims the wait and transitions the request to `canceled` in the same store transaction; exactly one of answer / expiry / cancel wins, decided by transaction order, and `aggregates.pending` reflects it in the same commit.

## Assumptions and Defaults

- The entire spec executes after herdr-parity merges to main (user directive; ADR-006).
- `review:` applies to every action kind; for `run-agent`/`goal` the reviewable proposal is the resolved prompt+params; `respond` requires a declared output shape (lint).
- Ask answers admit as the node's output with origin provenance; they do not open generations.
- `strategy` on the fan-out node governs its whole scope (its collect); default `wait_all` preserves today's semantics bit-for-bit.
- `progress.*` totals include unsettled lanes; rates are over `total`; zero-lane collections define rates as 0.
- `bind_as`/`index_as` default to `item`/`index`; declared names must be unique per nesting chain and not reserved roots.
- Fork seeds the `previous.*`/`best.*` baseline from every output of the chosen generation (carried-forward cells included) as the child's pre-settled generation 1 and executes the full body as **generation 2**; blob retention is guaranteed by the sweep-root rules (missing baseline blob → deterministic fork rejection, whole transaction aborts).
- Request context/prompt render at open time against the same namespace as effects, then freeze: previews are redacted + bounded inline, the full redacted context is a fetchable content-addressed ref, and the exact executable snapshot (`proposed_ref`) stays on the execution plane.
- Amend targets settled outputs on paused/parked cells; a cell with no output rejects `amend_no_output`; amend + `loop rerun --from-node` is the documented repair pair for already-consumed outputs.
- `requests.expire_after` seeds only requests with no authored `expires`; `""` means park indefinitely.
- Diff pagination: node-level rows are bounded per response with cursors; content diffs cap at the event payload bound.
- New capabilities are grantable exactly like `loops.approve` (same capability plumbing); operators are ungated.
- Web bell composition assumes the herdr row model as merged; adaptation at integration is a P10 concern, not a contract change.

## Architecture Decision Records

- [ADR-001: Human interaction model — four declarable interactions on the wait-resume plane](adrs/adr-001.md)
- [ADR-002: Operator time travel — read-only diff, in-run rerun, cross-run fork](adrs/adr-002.md)
- [ADR-003: Router node and gate-verdict routing, with a mandatory default route](adrs/adr-003.md)
- [ADR-004: Fan-out v2 — completion strategies, first-class partial, progress, naming, windowed width](adrs/adr-004.md)
- [ADR-005: Bug absorption — hash_fields deletion, predicate failure defaults, gate revision semantics](adrs/adr-005.md)
- [ADR-006: Bell integration rides the herdr-parity attention pipeline](adrs/adr-006.md)
