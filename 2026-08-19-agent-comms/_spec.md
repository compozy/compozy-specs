# Spec: Agent Comms — Typed Calls, Mailbox, and Subagents

---

# Part I — Product

## Overview

**Problem.** Delegation is the center of an agent operating system, and CompozyOS delegates blind. A parent that spawns a child gets back an address, not an answer: the child finishing produces no push with the result, so the parent polls, reads the child's transcript, and re-parses prose — the exact failure the market documents as an orchestration stall. What structured-return machinery exists is fragmented: five separate return paths (Loop agent nodes, Loop goals, human-request decisions, native tool outputs, task results) each validate differently, fail differently, and budget differently, and two of them can silently disagree about the same payload. Cross-session communication state is process-memory that dies on restart. Meanwhile the only shipped alternative — Compozy Network — is a group-conversation protocol whose enrollment ceremony is exactly wrong for "one parent asks one child and gets one typed answer".

**Who it is for.** Agent authors building multi-agent capabilities; the agents themselves (parents delegating, children returning); operators watching and steering delegation trees from the OS shell; integrators driving delegation headlessly; extension authors building on stable contracts.

**Why it is valuable.** This feature makes delegation a first-class, trustworthy primitive: one verb to delegate with a declared result contract, a guaranteed-valid typed result delivered in the same act as completion, an automatic parent↔child mailbox with no setup, a classic subagents experience on top of the agent definitions that already exist, and a collaboration surface in the OS shell that makes multi-agent work visible — the UI being the product's main differentiator. It closes the market gap the competition has publicly declined to close: no shipped harness offers schema-contracted results at the parent→child boundary.

## Goals

After this ships:

- A parent can delegate work and **trust the shape of what comes back**: a completed delegation carrying an invalid result is impossible — invalid results are typed failures with a repair round, never silent successes.
- **Completion and result are one delivery.** A parent never polls a child or parses its transcript to learn an outcome; being woken means having the typed result in hand.
- **Parent↔child messaging exists with zero setup** — durable across restarts, non-interrupting, receipt-bearing — because the lineage edge exists, with no conversation-channel machinery involved.
- **Subagents become real**: agents discover a described roster of specialists inside the call surface itself, fan out batches in one act, and delegate recursively to a walled depth — all on the existing agent-definition registry.
- **Every structured return in the system behaves identically** — one validation regime, one failure vocabulary, one budget model, one provenance story — across delegations, Loop nodes, tasks, decisions, and tool outputs.
- **Operators see and steer the whole thing** from a dedicated OS-shell app: delegation trees, inboxes, typed results, cost — with controls that map 1:1 to real runtime operations.
- Runaway collaboration **stops on its own**: rate limits, duplicate suppression, pending-delivery caps, depth walls, child caps, and idle ceilings are protocol properties.
- The system becomes **honest about what no longer exists**: the bare spawn surface, the inert idempotency input, and the 1:1 use of the group-decision return path are deleted, not aliased.

## User Stories

Full catalog: [_user_stories.md](_user_stories.md) — 38 stories, 6 personas.

- US-001..006 — Typed Call: delegate with contract, follow-up, batch fan-out, idempotency, cancel, await.
- US-007..011 — Typed Return: terminal return act, one repair round, completed-without-result, one budget, prose fallback.
- US-012..016 — Mailbox: child→parent, steer, receipts, loop-breakers, provenance + capability subtraction.
- US-017..020 — Lifecycle: result-carrying wake, parked revival, TTL errors, parent-terminal drain.
- US-021..024 — Subagents: described definitions, injected roster, depth-3 recursion, roster manageability.
- US-025..027 — Unification: Loop settle contracts, task result contracts, one regime everywhere.
- US-028..031 — Observability & UI: dedicated app, session detail, needs-you signals, truthful cost.
- US-032..034 — Governance: subset-only permissions, workspace isolation, secret redaction.
- US-035 — Network Bridge: publish completed calls as conversation evidence, one-directional.
- US-036..037 — Extensibility: lifecycle hooks (fail-open, sanitized), consented Host API reads.
- US-038 — Operator identity: one durable operator-caller per workspace, excluded from capacity/targeting.

## Core Features

**1. The typed call.** One verb delegates: it names a target (an agent definition to spawn, or an existing child to follow up with), carries the prompt, and optionally declares a result contract. It returns immediately with a durable call identity; the result arrives later as a wake that carries it (US-001, US-002, US-017; ADR-001, ADR-002, ADR-009). Calls support caller-chosen idempotency (US-004), cancellation with real stop propagation and superseded-evidence semantics (US-005), an explicit bounded await (US-006), and batch fan-out with per-item acceptance and hard-reject over cap (US-003; ADR-007).

**2. The typed return.** The child ends its assignment through a native return act validated at admission — completion and result recording are one atomic fact (US-007). Invalid results get exactly one repair round with the validator's errors verbatim; infrastructure failures don't consume it; a wrapper-artifact payload is unwrapped rather than failed (US-008). A contracted child finishing with nothing yields a distinct completed-without-result state (US-009). Prose-only answers are recovered by extraction with provenance (US-011). One declared budget with an explicit overflow strategy governs each result end to end (US-010).

**3. The lineage mailbox.** Parent and child can message each other because the edge exists — nothing to create or configure (US-012). Delivery is durable-first and non-interrupting (turn boundaries; new turn when idle), steers to a settling child surface as missed rather than vanish (US-013), receipts state transport truth from a closed vocabulary (US-014), and loop-breakers are protocol properties (US-015). Every peer message is provenance-stamped and structurally incapable of approving, configuring, or executing anything (US-016).

**4. Subagents on the existing registry.** Agent definitions gain a description; the call surface carries a shadow-aware roster of available specialists so selection costs zero extra turns; unknown names fail with the roster in the error (US-021, US-022; ADR-007). Recursive delegation works to a default depth of 3 — at the wall the call verb is absent from the child and the child's prompt states its literal remaining depth (US-023; ADR-008). The roster is fully manageable through structured surfaces (US-024).

**5. Contract unification.** The five existing return paths and the new call all ride one contract regime: same validation, same failure vocabulary, same budget/overflow model, same provenance and preview semantics, same entity-reference checking (US-025..027; ADR-006). Loop nodes get settle-time contract enforcement as part of the same regime; tasks gain an optional declared result contract enforced at completion.

**6. Lifecycle with parked children.** A finished child parks — addressable, context preserved, not consuming a live slot — and revives on the next call or message; TTL is the hard ceiling of addressability with typed errors past it (US-018, US-019; ADR-003). A terminating parent drains its delegation subtree deterministically; completed results survive the drain (US-020).

**7. The collaboration surface.** A dedicated OS-shell app shows delegation trees, call detail with schema-aware result rendering, inboxes with delivery outcomes, and cost — with real controls (message, cancel, revive, stop subtree) mapping 1:1 to runtime operations; session detail gains a calls panel and wake explanations; needs-you states reach the shell's attention grammar; usage stays on one truthful accounting substrate (US-028..031; ADR-004). CLI and API expose everything the app shows, with structured output and deterministic errors.

**8. The Network boundary and bridge.** The call/mailbox mechanism carries no conversation-container concepts and is mechanically disjoint from Compozy Network by construction; completed calls may be published one-directionally into Network conversations as evidence; the group-decision harvest remains Network's, while 1:1 typed returns are calls (US-035; ADR-005).

**Feature interactions.** The call is the substrate: subagents are agent definitions + the call; the mailbox delivers the call's completion and everything mid-flight; the contract regime validates what the call and every legacy path return; the lifecycle governs whom a call can address; the app renders all of it; budgets and loop-breakers bound the whole system.

## Business Rules

Lifecycle and states:

1. A call moves through a closed state vocabulary: queued → running → terminal, where terminal is exactly one of {completed, invalid-result, completed-without-result, failed, canceled, timeout, expired}. `queued` means durably admitted with activation or delivery still pending — it is a projection of the system's single work queue and delivery records, never a second queue of its own. Post-terminal transitions are forbidden; late outcomes attach as superseded evidence.
2. A completed call always carries a result satisfying its contract (or the uncontracted prose result, marked as such). No path can settle a contracted call as completed with an invalid or missing payload.
3. The completion notification and the result travel together — a completion signal without its result reference must be impossible, not discouraged.
4. A child is exactly one of running, parked, or gone (reaped/expired). Parked children revive on contact; gone children answer with typed errors distinguishing expired from never-existed.
5. Parent terminal state drains the governed subtree: outstanding calls close with the parent-terminal reason; already-completed results are preserved.

Validation and results:

6. Result validation happens at admission, inside the same act that settles the call — capture-time validation alone is insufficient.
7. Exactly one repair round per contracted return, carrying validator errors verbatim; infrastructure failures are classified separately and do not consume it; single-key wrapper artifacts are unwrapped, not failed.
8. Every result carries provenance: producing agent, session, call, time, and how it was admitted — the closed verdict set `returned | extracted | repaired`. Post-settle corrections are amendment records with their own provenance overlay, never a fourth admission verdict.
9. One declared budget per contract with an explicit overflow strategy; stored payloads are never truncated — projections are bounded previews with a full-fetch path.
10. A declared contract that cannot be enforced is a loud startup/authoring failure — silent schema drop is forbidden.

Addressing, permissions, isolation:

11. Delegation and messaging are workspace-scoped; cross-workspace attempts are hard-denied at the boundary. All list/read/stream surfaces scope to the caller's workspace.
12. Child permissions are subset-only in every category; widening — by caller or by hook — rejects with the widening atoms named. Narrowing composes down the tree.
13. Peer messages are provenance-stamped, rendered in bounded untrusted frames, and capability-subtracted: they cannot approve pending decisions, alter configuration, or execute embedded commands.
14. Payloads are scanned at admission; secret-shaped content is field-redacted on write (hash forms allowed); when redaction cannot preserve contract validity, the payload rejects with a typed reason.
15. Addressing binds to durable identities; a name that resolves to a different agent than previously reached refuses the send rather than delivering to the wrong one.

Recursion, concurrency, cost:

16. Default maximum delegation depth is 3, operator-configurable. At the wall the call verb is absent from the child's toolset and the child's prompt states its literal remaining depth.
17. Fan-out batches are hard-rejected over the configured cap with the cap named — never silently truncated. Per-item acceptance is independent.
18. Idempotency is caller-chosen: replaying a key returns the original call marked as replay; same key with different payload is a typed conflict.
19. Delivery and activation are separate facts: durable commit always precedes notification; waking an idle recipient consumes a turn on the shared accounting substrate; delivering to a busy recipient costs no extra activation. A committed completion is **never** admission-denied — no budget or ceiling may stand between a settled call and its delivery+activation in v1.
20. Loop-breakers are protocol properties: per-sender rate limits, identical-repeat suppression, pending-delivery caps, depth walls, child caps, and idle TTLs. Their engagement is observable, never silent.
21. Usage accounting for delegation rides the same owner-keyed substrate as every other activation; absent provider data renders as unavailable, never fabricated.

Boundary with Compozy Network:

22. The call/mailbox mechanism never carries conversation-container concepts (channel, surface, thread, direct room, work id). An exchange needing any of them belongs to Network.
23. The bridge is one-directional: completed calls may be published into Network conversations as evidence; Network content never creates, answers, or mutates a call.

## User Experience

**Personas and journeys** live per-story in [_user_stories.md](_user_stories.md); the primary flows:

- **Agent author**: writes an agent definition with a description → it appears in the roster → other agents delegate to it with contracts → results come back typed; the author debugs from the app's call detail, not from transcripts.
- **Parent agent**: reads the roster inside the call surface → delegates (single or batch) → continues working → is woken with the typed result → follows up with the same child, still warm, by the same verb.
- **Child agent**: receives the assignment as its first prompt with the contract stated → works → asks/reports mid-flight through the mailbox → returns the typed result as its terminal act → parks, revivable.
- **Operator**: sees the delegation tree light up in the dedicated app → drills into a call, reads the schema-aware result → messages a child to steer it, cancels a stray call, revives a parked child for one more question → dock/badge signals pull them in only when something needs them.
- **Integrator**: drives the same calls headlessly with structured output, idempotency keys, bounded awaits, and deterministic errors; sees identical state on CLI, API, and app.

**Discoverability**: the roster travels inside the call surface (zero-lookup selection); failure copy carries recovery paths (unknown name → roster; over cap → the cap; expired → spawn-fresh hint). The app's empty state teaches the feature. **Accessibility**: the app follows the OS shell's existing accessibility floor (keyboard navigation, focus management, WCAG-conformant contrast via the design system).

The screen-by-screen map (dedicated app, session detail panel, attention integration) lives in `_uiux.md` (Stage 2).

## High-Level Technical Constraints

- **Durability**: every call, result, and message survives daemon restart from the moment of acceptance; in-memory-only communication state is a defect class this feature removes.
- **Non-interruption**: no delivery may interrupt a running tool or splice into past context; boundaries only, new turn when idle.
- **Single sources of truth**: one activation/usage accounting substrate; one audit vocabulary; one contract regime. Parallel bookkeeping is forbidden.
- **Provider heterogeneity**: child agents run on heterogeneous providers; the return path must work when forced tool invocation is unavailable (extraction fallback is first-class).
- **Agent/operator manageability outcome**: every capability here — calling, awaiting, canceling, messaging, listing calls/messages/roster, reading results, reviving, configuring — must be operable by agents and operators through structured non-UI surfaces with deterministic errors, at parity with the app.
- **Extension ecosystem expectation**: extension authors can observe and participate in the call/message lifecycle through the runtime's extension surfaces (with governance re-validation after any mutation), and the contract regime is stable enough to build on.
- **Security**: workspace isolation, subset-only permissions, provenance stamping, capability subtraction, secret redaction, and bounded untrusted rendering are non-negotiable properties of every path this feature adds.
- **Truthful UI**: the app renders only what the runtime models; every control maps to a real operation.

## Non-Goals (Out of Scope)

- **Cross-workspace delegation or messaging** — the hard denial stands; no grant system, no relaxation. (Workspace isolation is a core security posture.)
- **Cross-host / remote reach** — both shipped references gate this behind separate explicit tiers; deferring keeps v1 coherent. Delegation stays within one daemon.
- **Broadcast, groups, multicast** — reaching N children is N messages; group semantics belong to Network conversations.
- **Promoting Network content into calls** — the bridge is one-directional by rule 23; Network content is untrusted and uncontracted.
- **Contract-net, voting, escalation protocols** — future-RFC territory; explicitly out (matching Network's own MVP exclusions).
- **End-to-end encryption of payloads** — the daemon is the trust boundary in v1; provider-specific encrypted-content channels don't port across heterogeneous providers.
- **Auto-routing by task description** — no shipped harness does it; selection is explicit against an injected roster (ADR-007).
- **A second execution runtime for subagents** — subagents are agent definitions + the call primitive + existing session machinery; no parallel engine (the market's negative proof).

## Open Questions

- **Primitive naming and glossary entries** (the call verb's public name, "call"/"parked"/"result contract" as glossary terms; never "handoff") — resolved in the Stage 2 surface grill on `_dx.md`, before internals.
- **Dedicated app composition** (layout, information hierarchy, which states get dedicated views vs drill-ins) — resolved in the Stage 2 `_uiux.md` draft + grill.
- **Numeric defaults beyond depth 3** (batch cap, concurrency budget, message/pending caps, rate-limit window, default result budget, await maximum) — resolved in Part II Config Lifecycle with the existing-config audit; the rules governing them (hard-reject, observability) are fixed above.

---

# Part II — Technical

## Executive Summary

Two new packages carry the feature. **`internal/contracts`** is the unified result-contract regime — digest-keyed schema registry, one resolver, admission-time validation with a per-digest compiled cache, byte budgets with declared overflow, contract-preserving redaction, prose extraction — consumed by loops, tasks, sessions, store, and the new call domain (ADR-006, ADR-010, ADR-013). **`internal/calls`** is the call + mailbox domain: durable call records with single-writer settlement (the #438 `RequireLeaseSettlementActor` precedent generalized), a lineage-derived durable mailbox with commit-then-notify delivery at turn boundaries, protocol loop-breakers, and park/revive lifecycle with idle-TTL semantics (ADR-001..004, ADR-009). The agent-facing spawn surface is deleted; the internal spawn engine becomes the call's engine (ADR-002). Subagent invocation rides the existing AGENT.md registry plus a new `description` field with roster injection (ADR-007). The mechanism is type-level disjoint from Compozy Network and reuses its owner-key accounting vocabulary without admission ceilings in v1 (ADR-005, ADR-011). Loops adopt the contract regime but keep their own records (ADR-012). Primary trade-offs: a wide but phased blast radius across the five legacy pipelines in exchange for one regime; a new domain read model in exchange for keeping `task_runs` a pure work queue.

## MVP Boundary

MVP = everything in this spec except: admission ceilings for call wakes (ADR-011 defers), AGENT.md default output schemas (per-call `expect` only), typed mailbox payloads (text-only v1), cross-workspace/cross-host reach, broadcast, auto-routing, a unified loops+calls activity tree (ADR-012), and Network-content promotion (all recorded Non-Goals or deferred evolutions). Task decomposition happens in `cy-create-tasks`; the trailing QA pair covers report + execution with browser e2e (`_uiux.md` present).

## Developer Experience

- [Developer experience contract](_dx.md) — golden path; AGENT.md `description`; CLI `compozy call` (incl. `publish`)/`compozy message`/`compozy session stop --subtree`; HTTP/UDS `/calls` (incl. `/publish`), `/messages`, session-stop `subtree`; `[calls]` config; native tools `compozy__agent_call`, `compozy__call_return/await/cancel/result/publish`, `compozy__agent_message`, `compozy__session_stop` (+`subtree`); completion-wake text; deterministic error table.
- [UI change map](_uiux.md) — Agents app extension (Activity/call detail/inbox locations + catalog/detail enrichment), session Calls panel, attention, dock badge; component plan and artboard list.

## System Architecture

- **`internal/contracts`** (new; leaf): registry (pin/resolve), a fixed admission pipeline — **secret classification/sanitization on raw bytes first**, then validation (verdict + sanitized errors + single-key unwrap) — compiled cache, normalization (full-schema vs shorthand), extraction (fenced/brace-balanced candidate scan, newest-first), budgets + overflow, redaction walk, `x-compozy-kind` entity walk (moves here from loop), provenance types.
- **`internal/calls`** (new): `Service` owning call/message state machines; create (spawn-or-address, batch, idempotency), settle (return admission → terminal write, single writer), await (durable-backed waiter registry), cancel (propagating stop + superseded evidence), mailbox (send, receipts, loop-breakers), delivery contracts, park/revive orchestration, hook dispatch at transitions.
- **Daemon bridges** (`internal/daemon`): implement `calls.SessionInvoker` (spawn via the retained internal engine, boundary delivery via the synthetic-prompt seam, revive via manager start), `calls.PublishBridge` (one-way Network evidence publishing — `internal/calls` never imports `internal/network`), `calls.ActivationRunCanceler` (backed by the task authority), the **call-activation dispatcher** (claims `call_activation` runs and drives safe-spawn — the network-wake executor precedent), the **deadline ticker** (invokes `Service.SweepDeadlines`), and the **subtree drain**: `Service.DrainSubtree` sets the root-closing fence (`sessions.draining_at`) **before** enumerating, then idempotently stops descendants, cancels queued activation runs (CAS via `ActivationRunCanceler`), and closes open calls — exposed as `subtree` on the session-stop surface; boot recovery resumes any incomplete drain from the persisted fence. Plus native tool handlers and roster rendering into the call tool description at serve time.
- **Manual admission mapping** (operator = peer, L-004): operator-originated calls (CLI without agent identity, HTTP operator auth, web) resolve their caller to the workspace's durable **operator-caller session** — selected-or-created through the existing session primitive, role-tagged, with no model runtime attached; its completion deliveries surface as attention/app items and `call await`/`call show` reads, never model turns. `calls.caller_*` stores that session's OwnerRef; the authenticated creator is recorded separately in `calls.actor_*`. Agent-invoked paths resolve caller via `internal/agentidentity`; operator endpoints never infer agent identity from environment (glossary rule). Every surface — agent tool, CLI, HTTP/UDS, web — funnels through the same `calls.Service.Create`.
- **Store** (`internal/store/globaldb`): new schema fragment + sqlc queries + repo files for calls, messages, deliveries, contract registry, payload blobs; `tasks.expect_digest`; sessions park timestamps.
- **Session manager** (`internal/session`): park transition (settle with empty queue → stop runtime, arm idle clock), revive path, idle-TTL reaper adjustment (never reap with an open call), spawn-surface deletion.
- **API/CLI/tools/web**: full-surface completion per the house chain (contract → core handlers → HTTP + UDS → spec registry → codegen → CLI → native tools → web system → docs).
- **Data flow (happy path)**: `agent_call` → calls.Service.Create **admission transaction** (contract pin, prompt blob, call row, idempotency fence, **and one `run_kind = "call_activation"` task_run for every call that must start or revive a child** — the network-wake taskless precedent; `calls.activation_run_id` is non-null for those calls) → commit → **activation**: the post-commit fast path immediately claims that exact run through `task.Service.ClaimNextRun` and invokes the safe-spawn engine; when the governed root is at its execution budget the same run simply stays queued and is claimed as capacity frees — one durable owner for both paths, so a crash between commit and start always recovers through the queue, never through a call-table scan. A follow-up to a live child creates no activation run — its execution is a durable `call_deliveries` row drained at the child's next boundary. A claimed activation that fails settles the call `failed` with a typed spawn reason. → child works → `call_return` → calls.Service.Return (tx: sanitize, validate by digest, blob write, terminal write, delivery row) → commit → notify → boundary injection/wake with result reference → parent acts. Subprocess work never happens inside a store transaction, and queued execution lives only in `task_runs`.

## Architectural Boundaries

- `internal/contracts` imports only stdlib + the JSON-schema library + `internal/diagnostics`-class helpers. It never imports store, loop, task, session, network, or daemon.
- `internal/calls` imports `internal/contracts`, `internal/store` (repos), `internal/network/participation` (OwnerRef/OwnerKey only), `internal/hooks`. It never imports `internal/daemon`, `internal/api/*`, `internal/cli`, or `internal/network` proper (the type-level boundary, ADR-005).
- `internal/store` imports `internal/contracts` (replacing its `internal/loop` imports for ref/validation helpers — delete target).
- `internal/loop`, `internal/task`, `internal/session` import `internal/contracts`; none imports `internal/calls` (the call service consumes their interfaces, wired in `daemon/`).
- `internal/daemon` remains the sole composition root; `magefiles/boundaries.go` gains rules for both packages in the same commit.

## Implementation Design

### Core Interfaces

```go
// internal/contracts
type Contract struct {
    Digest string          // "sha256:<64hex>" over the canonicalized schema — schema identity ONLY
    Schema json.RawMessage // canonical form
}
type ByteBudget struct{ MaxBytes int; Overflow OverflowMode } // never stored on the registry row
type OverflowMode string // "store" | "reject"
type ValidationIssue struct{ Path, Message string }
type Verdict struct {
    Valid     bool
    Issues    []ValidationIssue // bounded, verbatim
    Unwrapped bool              // single-key wrapper artifact removed
}
type Registry interface {
    Pin(ctx context.Context, schema json.RawMessage) (Contract, error)  // dedup by digest; immutable rows
    Resolve(ctx context.Context, digest string) (Contract, error)
    Validate(ctx context.Context, digest string, payload json.RawMessage) (Verdict, error)
}
func ResolveBudget(override *ByteBudget, cfg CallsResultsConfig) (ByteBudget, error) // per-consumer: override capped by max_budget, else config default
func EnforceBudget(b ByteBudget, payload json.RawMessage) (BudgetOutcome, error)     // store-with-preview | reject, per the consumer's immutable snapshot
func ExtractCandidate(text string) (json.RawMessage, bool) // newest-first scan
func RedactPreservingContract(c Contract, payload json.RawMessage) (json.RawMessage, []Redaction, error)
func SanitizeText(s string) (clean string, redactions []Redaction, reject bool) // contractless path: mailbox bodies, previews, prompts
```

```go
// internal/calls — service surface (consumed by daemon handlers)
type Target struct{ Agent, SessionID string } // exactly one set
type CreateInput struct {
    WorkspaceID    string
    Caller         participation.OwnerRef
    Target         Target
    Prompt         string
    Expect         json.RawMessage // optional; pinned to a digest on accept
    IdleTTL        time.Duration   // zero → config default
    Deadline       *time.Time      // optional; no default. Public wire form on EVERY create surface (HTTP/UDS body, native tool, batch items) is `deadline_seconds` (positive integer, relative); the CLI `--deadline 5s` converts; admission validates (>0, integer) → `call_deadline_invalid`, converts to absolute `deadline_at`, and includes it in idempotency payload identity
    IdempotencyKey string
    Runtime        *session.RuntimeSpec // optional override
    Narrow         session.PermissionAtoms
    Actor          Actor // authenticated creator (operator or agent session) — audit identity, distinct from Caller ownership
}
func (s *Service) Create(ctx context.Context, in CreateInput) (CallRecord, error)
func (s *Service) CreateBatch(ctx context.Context, in []CreateInput) ([]BatchOutcome, error)
func (s *Service) Return(ctx context.Context, in ReturnInput) (Settlement, error)
func (s *Service) Cancel(ctx context.Context, callID, reason string, actor Actor) (CallRecord, error)
func (s *Service) Await(ctx context.Context, in AwaitInput) (AwaitOutcome, error)
func (s *Service) SendMessage(ctx context.Context, in MessageInput) (MessageReceipt, error)
func (s *Service) Get(ctx context.Context, workspaceID, callID string) (CallRecord, error)
func (s *Service) List(ctx context.Context, q CallListQuery) (CallPage, error)
func (s *Service) Publish(ctx context.Context, in PublishInput) (PublishReceipt, error)          // completed calls with a valid result only; one-way (ADR-005)
func (s *Service) DrainSubtree(ctx context.Context, rootSessionID string, actor Actor, reason string) (DrainReport, error)
func (s *Service) SweepDeadlines(ctx context.Context, now time.Time) (SweepReport, error)        // deadline authority (daemon ticker)
```

```go
// internal/calls — Network evidence publishing, implemented in daemon/ (calls never imports network)
type PublishInput struct {
    WorkspaceID, CallID string
    Actor               Actor
    Channel             string
    ThreadID            string // optional; channel-thread conversations only — no direct-room branch exists in v1
}
type PublishBridge interface {
    PublishResultEvidence(ctx context.Context, ev ResultEvidence) (networkMessageID string, err error)
}
```

```go
// internal/calls — activation-run revocation, implemented by the task authority (L-005)
type ActivationRunCanceler interface {
    // Compare-and-swap the linked run out of the claimable set BEFORE the call
    // terminalizes. Exactly one side wins: if cancellation wins, a later claim
    // fails; if a claim already won, the caller falls back to managed stop.
    CancelActivationRun(ctx context.Context, runID string, reason string) (CancelOutcome, error)
}
```

```go
// internal/calls — interfaces defined where consumed, implemented in daemon/
type SessionInvoker interface {
    SpawnChild(ctx context.Context, in ChildSpec) (SessionRef, error) // retained internal engine
    Revive(ctx context.Context, sessionID string) error               // parked → running
    DeliverAtBoundary(ctx context.Context, d Delivery) (DeliveryOutcome, error)
}
type SettlementActor struct{ Kind, ID string }
func RequireCallSettlementActor(rec CallRecord, actor SettlementActor) error // single-writer fence
```

### Data Models

New globaldb fragment `internal/store/globaldb/schema/definitions/73_agent_calls.sql` (side-table decisions inline; **no JSON metadata blobs** — every matchable field is a column):

| Column | Type | Purpose |
| --- | --- | --- |
| `contract_schemas.digest` | TEXT PK | canonical schema identity (ADR-013) |
| `contract_schemas.schema` | TEXT | canonical schema bytes |
| `calls.result_budget_bytes` / `.result_overflow` | INT / TEXT | **immutable per-call budget snapshot** taken at admission (per-call override capped by `max_budget`, else config default). The registry row stays schema-only — two calls sharing one schema keep independent budgets; the snapshot participates in idempotency payload identity |
| `calls.call_id` | TEXT PK | `call_…` identity |
| `calls.workspace_id` | TEXT | isolation scope (indexed) |
| `calls.caller_kind` / `.caller_id` | TEXT / TEXT | `participation.OwnerKey` halves (operator calls: the workspace operator-caller session) |
| `calls.actor_kind` / `.actor_id` | TEXT / TEXT | authenticated creator (operator vs agent session) — audit identity, never ownership |
| `calls.activation_run_id` | TEXT NULL FK | `task_runs` row (`run_kind = "call_activation"`) — **non-null for every call that starts or revives a child** (null only for live-child follow-ups, whose execution is a delivery row); the single durable activation owner for fast path, queued path, and crash recovery |
| `task_runs.run_kind` | CHECK rebuild | migration 00078 **rebuilds the `task_runs` CHECK** to admit `call_activation` as a second taskless kind (the `network_wake` branch precedent: `task_id` NULL allowed); `internal/task` gains `RunKindCallActivation` + validation, and the claim selector gains exact-kind selection with the governed-root execution budget evaluated **inside the claim transaction** |
| `call_activation_runs.run_id` | TEXT PK FK | typed side table binding the taskless run to its call: `call_id` FK, `workspace_id`, `governed_root_id` (root-cap admission key), activation inputs (spawn vs revive) — matchable state as columns, never JSON |
| `operator_caller_sessions.workspace_id` | TEXT PK | durable uniqueness fence for the operator-caller mapping: `session_id` UNIQUE FK; created inside the call-admission transaction with conflict-winner semantics — two concurrent first operator calls converge on one session; the bound session is excluded from call/message targeting, liveness caps, revival, and the reaper, and cascades on workspace deletion |
| `sessions.draining_at` | TIMESTAMP NULL | root-closing fence for subtree drain: set on the governed root **before** descendant enumeration; spawn/call admission re-validates it (reject with the parent-terminal reason), so no descendant can be admitted mid-drain; persisted → boot recovery resumes incomplete drains |
| `calls.parent_session_id` | TEXT NULL | lineage anchor when caller is a session |
| `calls.agent_name` | TEXT NULL | definition called (null on session-target follow-up) |
| `calls.child_session_id` | TEXT | the addressed/spawned child |
| `calls.depth` | INT | governed depth at creation |
| `calls.state` | TEXT CHECK | `queued`/`running`/terminals (closed set, business rule 1; `accepted` does not exist). `queued`/`running` are updated **in the same authoritative transactions** that claim/settle the linked activation run or delivery — one writer, no drift between the projection and the queue |
| `calls.verdict` | TEXT NULL | returned / extracted / repaired |
| `calls.expect_digest` | TEXT NULL FK | pinned contract |
| `calls.prompt_ref` | TEXT | content-addressed prompt (blob) |
| `calls.result_ref` / `.result_bytes` | TEXT NULL / INT | content-addressed result + size |
| `calls.failure_code` / `.failure_detail` | TEXT NULL | typed terminal cause (bounded detail) |
| `calls.repair_attempts` | INT | 0/1 — the bounded retry |
| `calls.superseded_ref` | TEXT NULL | late outcome recorded as evidence |
| `calls.idempotency_key` | TEXT NULL | UNIQUE `(workspace_id, caller_kind, caller_id, idempotency_key)` |
| `calls.batch_id` | TEXT NULL | groups one invocation's items |
| `calls.deadline_at` | TIMESTAMP NULL | opt-in only; indexed — `SweepDeadlines` selects due calls by it (invariant 3e) |
| `calls.created_at` / `.settled_at` | TIMESTAMP | lifecycle |
| `call_messages.message_id` | TEXT PK | `msg_…` |
| `call_messages.workspace_id` | TEXT | isolation scope |
| `call_messages.from_kind` / `.from_id` | TEXT | provenance (session/operator) |
| `call_messages.to_session_id` | TEXT | direct address |
| `call_messages.call_id` | TEXT NULL | conversation context |
| `call_messages.body` | TEXT | bounded text (≤ `calls.messages.max_bytes`) |
| `call_messages.dedup_hash` | TEXT | identical-repeat window fence |
| `call_deliveries.delivery_id` | TEXT PK | one row per completion/message delivery attempt stream |
| `call_deliveries.kind` / `.subject_id` | TEXT | completion \| message → call_id \| message_id |
| `call_deliveries.recipient_session_id` | TEXT | target |
| `call_deliveries.owner_key` | TEXT | shared accounting identity (ADR-011) |
| `call_deliveries.wake_event_id` | TEXT UNIQUE | durable dedupe fence (replaces in-memory LRU) |
| `call_deliveries.state` / `.reason` | TEXT | the ONE durable delivery state machine: `pending → injected \| woken \| failed` + named reason; the public receipt enum is an exhaustive projection (`pending→queued`, `injected→delivered-into-turn`, `woken→woke`, `failed→failed`) — message read models join the latest delivery row; no second delivery authority exists |
| `call_publications.call_id` + `.channel` + `.thread_id` | TEXT UNIQUE triple | publish idempotency per conversation: repeat publish returns the recorded `network_message_id` (`published:false`); a different conversation inserts anew |
| `payload_blobs.workspace_id` + `.ref` | TEXT PK pair | content-addressed payload store (calls/tasks) |
| `payload_blobs.bytes` | BLOB | exact stored payload (never truncated) |
| `tasks.expect_digest` | TEXT NULL FK | task result contract (US-026), authored via `expect` on every task create/update surface (CLI `--expect`, HTTP/UDS body, native task tools) with the digest + budget echoed in read projections |
| `task_runs.expect_digest` + `.result_budget_bytes` / `.result_overflow` | TEXT NULL / INT / TEXT | **start-time snapshot** copied from the task at run admission — an in-flight run settles against its snapshot even when the task's contract changes mid-run (US-026.EC-1); completion validation + one worker resubmission round run through the existing completion authority with sanitized errors |
| `sessions.parked_at` / `sessions.idle_expires_at` | TIMESTAMP NULL | park state; idle clock (null while a call is in flight) |

Side-table-vs-JSON: every entity above is columns/side-tables because all of it is matchable state (list filters, fences, projections). The only JSON at rest is schema bytes and content-addressed payloads — both opaque by definition. Loops keep `loop_output_blobs` (cell-scoped integrity is load-bearing); the shared ref helpers move to `internal/contracts` (delete target for the store→loop imports).

### API Endpoints

Per `_dx.md`, on both transports: `POST/GET /api/workspaces/{workspace_id}/calls` (batch responses use status `200` with a per-item array), `GET /calls/{call_id}`, `GET /calls/{call_id}/result`, `POST /calls/{call_id}/cancel`, `POST /calls/{call_id}/await`, `POST /calls/{call_id}/publish`, `POST/GET /messages`, and `POST /api/workspaces/{workspace_id}/sessions/{id}/stop` gaining `{"subtree": true, "reason": "…"}` → `200` with the structured drain report (`{stopped_children, closed_calls, preserved_results}`) and the standard typed errors. Implementation: contract types in `internal/api/contract/calls.go` (public projections; refs never exposed raw — preview + fetch route), shared handlers on `BaseHandlers` (`Calls CallsService` field) in `internal/api/core/calls.go` + `calls_interfaces.go` (query structs shared HTTP/UDS), registration in `internal/api/httpapi/calls_routes.go` + `internal/api/udsapi/calls_routes.go` (+ `WithCallsService` option), operations in `internal/api/spec/registry_calls.go`, UDS parity-test entries. Status codes and error codes exactly as the `_dx.md` table; error payloads use the standard structured execution-error shape.

## Impact Analysis

| Component | Impact Type | Description and Risk | Required Action |
| --- | --- | --- | --- |
| `internal/contracts`, `internal/calls` | new | two packages; boundary rules | create + `mage Boundaries` rules |
| Spawn agent surface | deprecated→deleted | tool/CLI/HTTP/contract removal | delete list below |
| `internal/loop` validation | modified | consumes `internal/contracts`; 3 resolvers → 1 | phased adoption; #438 tests guard |
| `internal/task` results | modified | `expect_digest` + per-contract budget replaces blanket 64 KiB | completion admission change |
| `internal/store/globaldb` | modified | new fragment 73 + queries + repos; drops loop imports | `eng-schema-migration` chain |
| `internal/session` | modified | park/revive, idle-TTL reaper, spawn surface removal | manager + reaper edits |
| Native tools | new+deleted | `call_*` family in; `session_spawn` out | descriptors/toolsets/bindings/availability |
| Web | new+modified | `agent-comms` system; Agents app locations; attention | per `_uiux.md` |
| Docs/skill | modified | site area + skill references + config docs | co-ship |

**Delete targets (no fallback, no compat shim, no placeholder):** `compozy__session_spawn` (descriptor, schemas, toolset entry, binding, availability, handler, input DTO), `internal/cli/spawn.go` + generated CLI doc page, HTTP/UDS spawn route + `AgentSpawnRequest`/`AgentSpawnPayload` + spec op + parity entries + generated TS, `SpawnOpts.IdempotencyKey` (inert input), the six `internal/store` → `internal/loop` helper imports (`OutputRefForPayload`, `ValidateWaitPayload` call sites), two of the three declared-schema resolvers (`linter_reference_schemas.go` and `service_amendments.go` variants collapse into the contracts resolver), `task.MaxResultBytes` as a blanket result ceiling, the stale `compozy exec` line in `internal/CLAUDE.md`, and the spawn sections of `packages/site/content/docs/sessions/orchestration.mdx` + `skills/compozy/references/*` spawn paragraphs. The `output_ref` sentinel/failure union is replaced by explicit result kinds in the loop refactor phase — readers stop string-sniffing (`outputValue` fallback deleted).

## Extensibility Integration Plan

- **Hooks**: new family `call` in `internal/hooks` (`events_calls.go` + `payloads_calls.go` + catalog entry): `call.created` (async), `call.settled` (async), `call.canceled` (async), `call.published` (async), `call.message_sent` (async), `call.message_delivered` (async), `call.subtree_drained` (async) — the same `family.event_name` grammar as the canonical events, one naming table shared by both. Hook payloads receive only sanitized data (post-redaction; B-007 pipeline). Spawn governance hooks (`spawn.pre_create` etc.) keep firing inside the call's spawn path — hook-then-revalidate preserved (US-032.AC-2).
- **Extension Host API**: `internal/codegen/hostapi/catalog.json` gains read methods `calls/list`, `calls/get`, `calls/result`, `messages/list` (family `calls`), with `PermissionContract` entries (area `calls:read`). Mutation methods are deliberately absent in v1 (extensions observe; agents/operators act) — revisit as evolution.
- **Skills/capabilities/registries**: no new resource kinds; the AGENT.md registry gains the `description` field (ADR-007) which flows to extension-provided agent resources unchanged.
- **Bridge SDKs / MCP sidecars**: no bridge contract changes; native tools reach ACP providers via the existing hosted-MCP projection (roster rendering happens at that seam). Checked surfaces: `internal/bridgesdk` (no delivery-target semantics touched), MCP sidecar lifecycle (unchanged).

## Agent Manageability Plan

Exactly the `_dx.md` surface: `compozy call` verbs (including `compozy call publish`) + `compozy message` verbs + `compozy session stop --subtree` (4 output formats, deterministic exit codes), HTTP/UDS parity routes, `compozy__call_*` (including `compozy__call_publish`)/`compozy__agent_message` tools plus the `subtree` argument on `compozy__session_stop`, with structured results (`structuredNetworkResult`-style redaction for message-bearing payloads), `compozy config get/set calls.*` (via `tool_surface_calls.go`), roster via `compozy__agent_list` + injected parameter description, status discovery via `compozy call list/show` and the attention summary. Every failure is a typed code from the `_dx.md` table on every surface.

## Config Lifecycle

New `[calls]` section: `internal/config/calls_config.go` (struct + `DefaultCallsConfig()` + `Validate()` with path-prefixed errors), `merge_calls.go` overlay (+ profile hook in `merge.go`/`merge_overlay.go`), `tool_surface_calls.go` (dotted paths + kinds — without it agents cannot `config set`), keys/defaults exactly per `_dx.md` (`max_depth=3`, `max_batch=8`, `max_children=5`, `max_active_per_root=32`, `idle_ttl="1h"`, `results.default_budget="256KiB"`, `results.max_budget="4MiB"`, `results.overflow="store"`, `messages.rate_limit_per_minute=30`, `messages.dedup_window="30s"`, `messages.pending_cap=50`, `messages.max_bytes="64KiB"`). **Concurrency semantics are deterministic, one meaning per key**: batch size > `max_batch` → whole-batch reject. `max_children` is the **per-parent admission wall** — a call that would exceed the caller's live-children cap **rejects** with the typed cap error, nothing spawns (US-001.EC-4, US-003.EC-4). `max_active_per_root` is the **execution-concurrency budget** — admitted calls beyond it stay `queued`, their `call_activation` runs claimed as capacity frees; the budget is evaluated inside the claim transaction. Nothing is silently dropped. These two keys supersede the hardcoded `DefaultSpawnMaxChildren`/`MaxActivePerWorkspace` Go consts (delete targets); `[roles.coordinator].max_children` remains coordinator-only. Docs: section block + index row in `packages/site/content/docs/configuration/config-toml.mdx`; `skills/compozy/references/configuration.md`. Tests: defaults, validation rejects (non-positive caps, budget over max), overlay precedence, tool-surface classification. Removed keys: none.

## Testing Approach

Strategy only (cases live in `_tests.md`): unit tests per package with `-race` and table-driven subtests (`t.Run("Should …")`), fakes only at I/O boundaries (SessionInvoker fake in `internal/calls` tests; real SQLite in store tests). Integration (`+integration`) for store transactions (settlement fences, idempotency races, delivery dedupe) and config lifecycle. E2E runtime (`make test-e2e-runtime`, `acpmock`) for the full call loop: create → child returns via tool → wake carries result → await/cancel/revive journeys, exercising the exact `_dx.md` CLI/HTTP invocations; contract co-ship rule applies (acpmock learns `call_return`). E2E web (Playwright, `make test-e2e-web`) for the Agents app locations, session Calls panel, and attention badge against a seeded daemon. Coverage floor 80% per package; the observability coverage-matrix test extends to call lifecycle events.

## Development Sequencing

### Build Order

1. `internal/contracts` + registry/cache + unit suite (pure; no behavior change). Gate: `make gate`.
2. Store layering fix: ref/digest helpers move to contracts; store drops loop imports (mechanical). Gate: full unit + boundaries.
3. Legacy pipeline adoption: loop capture/settle validation, ask/review `ValidateWaitPayload`, descriptor validation, task result budget → contracts; resolver collapse; `output_ref` explicit kinds. Gate: loop/task suites + `#438` regression tests green.
4. `internal/calls` domain + fragment 73 + sqlc/repos + settlement/idempotency/await/cancel + spawn-engine wiring. Gate: integration suite.
5. Mailbox + durable deliveries + loop-breakers + park/revive + idle-TTL reaper change. Gate: integration + runtime e2e (acpmock).
6. Surfaces: native tools (+ roster injection, `call_publish`, session-stop `subtree`), CLI (incl. `call publish`, `session stop --subtree`), HTTP/UDS + spec + codegen, the Network publish bridge, hooks family, host API methods, `[calls]` config. Gate: `make gate` + parity tests.
7. Hard cut: spawn surface deletion + `tasks.expect_digest` + docs/skill updates. Gate: full verify.
8. Web: `agent-comms` system + Agents app locations + session panel + attention + Playwright e2e. Gate: web lanes.
9. QA tail: scenarios flagged + walked; `make gate-full`.

### Technical Dependencies

None external. Internal ordering only: phases 1–2 unblock everything; phase 4 needs 1; phase 6 needs 4–5; phase 8 needs 6's codegen output.

## Monitoring and Observability

Canonical events with correlation keys (`workspace_id`, `session_id`, `parent_session_id`, `call_id`, `message_id`, `agent_name`, `spawn_depth`, `actor_kind/actor_id`): `call.created`, `call.state_changed`, `call.settled`, `call.canceled`, `call.published`, `call.message_sent`, `call.message_delivered`, `call.message_rejected(reason)` (sender-side brake rejections: rate limit, dedup, caps), `call.revived`, `call.reaped(idle_ttl)`, `call.subtree_drained`. One naming grammar — `family.event_name` with underscores — shared verbatim by the hook catalog, event descriptors, docs, and the coverage-matrix test. No delivery-skip event exists: completions are never admission-denied (ADR-011). SSE catalog events power the web system; the coverage-matrix test fails on any lifecycle path without its event. Usage rows keyed by `owner_key`.

## Technical Considerations

### Key Decisions

ADRs 001–013 carry every significant decision. Additional non-ADR choices: JSON-schema engine standardizes on the loop's current library (v6) inside `internal/contracts` — the other two engines stop being validation sources; roster injection renders at the hosted-MCP projection seam (serve-time), giving convergence without daemon restart (US-022.AC-2); `payload_blobs` is workspace+ref keyed with digest re-verification on read (the strongest existing primitive, generalized).

### Known Risks

- Loop pipeline adoption regressions — mitigated by #438's settle-validation tests plus phase-3 gates before any call code lands.
- Roster prompt-size growth — bounded rendering with count/length caps (US-022.EC-1); policy documented in the tool description itself.
- Park/revive interplay with liveness caps under fan-out — revive counts against caps at revive time (US-018.EC-1); integration tests cover the race.
- Cancellation reaching a mid-turn ACP subprocess is new ground (no reference implements it) — designed as caller-owned cancel → managed-stop path with grace (existing subprocess discipline), settled as `canceled` with superseded-evidence handling.

## Safety Invariants

1. `calls.Service` settlement methods are the **only** writers of call terminal states; `RequireCallSettlementActor` rejects any other actor (single-writer fence).
2. Result sanitization, validation, blob write, and terminal state write happen in **one transaction**; a `completed` state row without a valid `result_ref` (when contracted) cannot exist.
3. A completion delivery row is written in the same transaction as settlement; notification fires only after commit (commit-then-notify). A committed completion is never admission-denied: no budget, ceiling, or admission funnel may stand between settlement and its delivery + activation (ADR-011).
3a. **Admission and activation are separate phases.** No subprocess work happens inside a store transaction: the admission transaction commits the call (and its activation intent) first; activation runs post-commit through the daemon-owned safe-spawn engine. A committed call whose activation fails settles `failed` with a typed spawn reason — a call can never be lost between commit and start.
3b. **Queued execution lives only in `task_runs`.** Every child-starting call carries a `run_kind = "call_activation"` row claimed exclusively via `task.Service.ClaimNextRun` (fast path claims immediately; budget-delayed runs wait); the `calls` tables carry no claim/lease columns, and no component scans call rows to start work (L-003, L-005).
3c. **Terminalizing a call always fences its activation run first.** Cancel, deadline, drain, and workspace deletion CAS the linked run out of the claimable set through the task authority (`ActivationRunCanceler`) before the call closes; if a claim already won, the path falls back to managed stop. A terminal call with a claimable activation run cannot exist; boot reconciliation repairs any crash window between the two records.
3d. **Subtree drain is fence-first.** `DrainSubtree` persists the root-closing fence before enumerating descendants; spawn/call admission re-validates the fence, so no descendant is admitted mid-drain; the drain is idempotent and resumes from the fence on boot.
3e. **Deadlines have one authority.** `Service.SweepDeadlines` (daemon ticker) is the only path that terminalizes a call as `timeout`: it fences the activation run (3c) or managed-stops the running child, settles through the single settlement writer, and delivers the result-carrying terminal wake; return-vs-deadline and cancel-vs-deadline resolve to exactly one terminal outcome.
4. The wake that reaches the parent always carries the call identity + terminal state + exactly the applicable payload: a valid result reference and bounded preview for `completed`, or the typed reason/evidence reference for resultless terminals (failed, canceled, timeout, expired, invalid-result, completed-without-result). A `completed` signal without its result cannot exist on any path.
5. Post-terminal writes are rejected; late outcomes land only in `superseded_ref` — never mutate the terminal.
6. Idempotency uniqueness is a DB fence (`UNIQUE` on workspace+caller+key); concurrent duplicates resolve to exactly one call row.
7. Permission narrowing is validated before and **re-validated after** every hook mutation; widening rejects with named atoms.
8. Depth is enforced at create admission from durable lineage (never from prompt claims); at the wall the call tool is absent from the child toolset.
9. Every read/write/stream path filters by `workspace_id` at the store layer; cross-workspace targets are denied before any side effect.
10. **Sanitize-before-everything.** Secret classification and contract-preserving redaction run on the raw payload bytes as the FIRST admission stage — before schema validation, validator-error construction, hook dispatch, repair-prompt rendering, event emission, and persistence. Only sanitized/hash-form data reaches any of those stages; "errors verbatim" means verbatim from the sanitized validator output. When redaction cannot preserve contract validity, the return fails with a fixed typed error naming paths but never values. Raw claim-token-shaped values therefore never reach a stored payload, projection, log, SSE event, hook payload, or repair prompt.
11. The idle-TTL reaper never reaps a session with an open call (`state IN (queued, running)`); the idle clock arms only on park. Operator-caller sessions are excluded from reaping, liveness caps, targeting, and revival entirely.
12. `wake_event_id` dedupe is durable; redelivery after crash is idempotent.
13. Rate-limit, dedup-window, and pending-cap checks (queued-undelivered backlog - transport state, never read acknowledgment) run inside the message-accept transaction; their rejections are typed and observable.
14. `internal/contracts` performs no I/O beyond its registry store interface; validation is pure given (digest, payload).

## File References

### Repo Files

Conventions and integration seams (from the Stage 2 architecture exploration; read before implementing the matching phase):

- `internal/tools/builtin_ids.go:101` + `builtin_loop_ids.go:1` — tool-ID declaration pattern; new family gets `builtin_call_ids.go`.
- `internal/tools/builtin/descriptors.go:29,76,81` — descriptor group registration, capability gating, constructor.
- `internal/tools/builtin/sessions_orchestration.go:9,44,106-134` — the family-file shape to copy (and the spawn schemas to delete).
- `internal/tools/builtin/toolsets.go:6,46,69` — toolset membership; `ToolsetIDCoordination` as the analogue.
- `internal/tools/descriptor.go:35,113` + `schema_digest.go:44` — descriptor contract, validation, computed digests.
- `internal/daemon/native_tools.go:71,93` — provider join + bijective boot validation (registration tripwire).
- `internal/daemon/native_tool_bindings.go:5` + `native_tool_session_orchestration_bindings.go:5` — binding maps.
- `internal/daemon/native_tool_availability.go:23,62,101` — availability closures.
- `internal/daemon/native_tool_results.go:13,28` — `structuredResult` / `structuredNetworkResult` (message payloads use the redacting variant).
- `internal/daemon/native_tool_session_orchestration.go:31,74` — handler shape; `native_session_prompt_stream.go:11` — the response-discard lesson.
- `internal/cli/root.go:63,166,172,216` — command DI, output flags, family registration, structured errors.
- `internal/cli/network.go:93` + `network_usage_cmd.go:8` + `format.go:63,71,121,154` — family root, verb template, four-format bundles.
- `internal/cli/client.go:26` + `client_network_coordination.go:34,137` — DaemonClient extension pattern.
- `internal/cli/client_api_errors.go:178` + `command_exit_error.go:22` — exit-code mapping.
- `internal/api/contract/loop_requests.go:9-65` — public projection with provenance; the shape for `calls.go`.
- `internal/api/core/loop_interfaces.go:186,223` + `loops_requests.go:15` + `base_handlers.go:111,135` — shared handler/service pattern.
- `internal/api/httpapi/loops_routes.go:35` + `udsapi/loops_routes.go:38` + `udsapi/loop_options.go:13` + `udsapi/handlers_test.go:296` — dual registration + parity test.
- `internal/api/spec/spec.go:237` + `operation_registry.go:5` — OpenAPI operation registry.
- `magefiles/codegen.go:13,48` — codegen order and drift gate.
- `internal/config/network_config.go:17,53,82` + `merge_network.go:3` + `tool_surface_loops.go:13` — config section, overlay, tool-surface pattern.
- `internal/config/agent.go:17,36,213` + `agent_create.go:24,71,145` + `agent_discovery.go:14` — AGENT.md parsing, rendering, digest, shadowing (the `description` thread).
- `internal/api/contract/agent_observe_payloads.go:28` — payloads gaining `description`.
- `internal/tools/builtin/workspace.go:15,71` + `internal/daemon/native_entity_catalog_tools.go:1` — `agent_list`/`agent_create` surfaces.
- `internal/hooks/events.go:6,25,32` + `events_network.go:3` + `events_catalog.go:7` + `payloads_spawn.go:6` + `dispatch_loops_spawn.go:133` — hook family template + spawn governance hooks that keep firing.
- `internal/codegen/hostapi/catalog.json:1` + `internal/extension/contract/permissions.go:9,30` — Host API method + consent additions.
- `internal/store/globaldb/schema/definitions/60_network.sql` + `20_sessions.sql:140` — fragment style; lineage columns.
- `internal/store/globaldb/schema/migrations/00077_schema.sql` + `atlas.sum` + `sqlc.yaml` — migration chain (next: 00078).
- `internal/store/globaldb/global_db_network_accept.go:37-97` — the commit-then-notify acceptance transaction to mirror.
- `internal/store/globaldb/global_db_network_admission.go:39,150-161` — named skip reasons; owner-key accounting.
- `internal/network/participation/owner.go:9` + `types.go:34` — OwnerRef/OwnerKey (consumed, not modified).
- `internal/session/spawn.go:17-20,84,131` + `spawn_permissions.go:15,51` + `spawn_governance.go:32` — the retained internal engine (consts superseded by `[calls]`).
- `internal/session/spawn_wake.go:86-97,108-119,140-146` — redact-then-bound, the wake-reason hole being fixed, content-addressed event ids.
- `internal/session/session_wait.go:12-18,79-127` + `session_wait_registry.go:236-249` — the await skeleton (register-before-snapshot, resume grace).
- `internal/session/synthetic_prompt.go:159-177,347-365` — the boundary-delivery seam gaining durable backing.
- `internal/daemon/spawn_reaper.go:161-193,230-261` — the reaper gaining idle-clock semantics.
- `internal/loop/action_schema.go:18-58,102-149,255-294,296-434` — validation/extraction/repair moving into contracts.
- `internal/loop/action_result_validation.go:12` + `coordinator_output_validation.go:11` + `internal/task/lease_settlement_authority.go:7` — the #438 checkpoints being generalized.
- `internal/store/globaldb/global_db_loop_amendments.go:17-30,179-220` — the amendment/provenance overlay model.
- `internal/task/coordinator_control.go:11-98` + `limits.go:16-17` — the result path + blanket ceiling being replaced.
- Web (per `_uiux.md` anchors): `web/src/systems/os/lib/app-catalog.ts:34,70`, `apps/agents/agents-window.tsx:10` + location files, `web/src/systems/session/components/session-inspector.tsx:29-40`, `session-timeline-render.tsx:275-293`, `runtime-activity-notice.tsx:29`, `web/src/systems/os/lib/attention-model.ts:32,1-17`, `web/src/systems/session/lib/session-hierarchy.ts:50`, `session-badge.ts:37,166`, `web/src/lib/api-client.ts:13` + `api-contract.ts:57`, `web/src/lib/cost-provenance.ts:94`, `web/src/lib/ticketed-event-source.ts`, `web/e2e/fixtures/os-navigation.ts:16` + `test.ts:15`.
- Docs/skill: `packages/site/content/docs/meta.json`, `content/docs/network/meta.json` (folder pattern), `content/docs/sessions/orchestration.mdx:55` (rewrite target), `configuration/config-toml.mdx:99`, `lib/docs-icons.ts`, `lib/docs-navigation.ts:6`; `skills/compozy/SKILL.md:20` + `references/native-tools.md:41-364` + `references/agent-definitions.md` + new `references/agent-comms.md` (modeled on `references/network.md:29`).

### Competitor References

- `.resources/omp/docs/tools/task.md:1-176` — spawn-carries-prompt, batch shape, lifecycle end-states, derived-messaging note.
- `.resources/omp/packages/coding-agent/src/tools/yield.ts:1-140` + `task/yield-assembly.ts:1-90` — forced terminal return, `schemaOverridden` provenance (our `verdict`).
- `.resources/omp/packages/coding-agent/src/irc/bus.ts:1-150` — delivery receipts vocabulary; in-memory volatility to avoid.
- `.resources/hermes/tools/delegation_output_schema.py:1-151` — one bounded retry, errors verbatim, no schema re-paste, candidate extraction.
- `.resources/hermes/tools/async_delegation.py:1-40` — completion as a new turn when idle; never splice; self-contained payload.
- `.resources/hermes/tools/delegate_tool.py:4597-4730` — batch schema + control-plane multiplexing; the 4,793-line file is the layout anti-pattern.
- `.resources/codex/codex-rs/core/src/agent/control/residency.rs:17-232` + `spawn.rs:257-387` — LRU park/lazy reopen (our parked/revive).
- `.resources/codex/codex-rs/core/src/session/input_queue.rs:121-263` — mailbox delivery phases (defer after answer, reopen on steer).
- `.resources/codex/codex-rs/core/src/agent/role.rs:270-348` + `tools/spec_plan.rs:673-683` — roster rendered into the param description (ADR-007's mechanism).
- `.resources/pi/packages/coding-agent/examples/extensions/subagent/index.ts:287-294,473-518` — error-driven roster (anti-pattern) and exactly-one-mode enforcement.
- `.resources/herdr/src/api/wait.rs:216-248,611-640` — identity-pinned waits + typed stall errors (await semantics).

### Design and Analysis Sources

- `analysis/summary.md` — convergence map feeding ADRs 001–006.
- `analysis/01..06_analysis_*.md` — per-domain ground truth (each cited in the matching ADR).
- `docs/design/opendesign/graph-eng/` (lineage/timeline/needs-you), `loop-legibility/` (rosters), `herdr-parity/` (bell/badge grammar), `_done/worktree/` (inspectors) — visual-language references for the six `agent-comms-*.html` artboards named in `_uiux.md`.

## Assumptions and Defaults

- Config defaults exactly as `_dx.md` (`max_depth=3`, `max_batch=8`, `idle_ttl="1h"`, budget `256KiB`/`4MiB`/`store`, message caps `30/min`, `30s`, `50`, `64KiB`). No default deadline exists.
- Contract payloads are JSON objects at the root (matching every existing pipeline); non-object roots are a contract-authoring error in v1.
- Extraction fallback is ON by default for contracted calls; a strict mode (per call) disables it (US-011.EC-3).
- Messages have no read/seen state anywhere (nothing models it — truthful UI).
- Batch items share the caller's narrowing/TTL unless overridden per item; `batch_id` groups them for read models only (no batch-level state machine).
- Roster rendering caps at 32 definitions / 120 chars per description in the tool parameter; the full roster is always available via `compozy__agent_list`. The `description` authoring maximum is 500 characters, enforced at load/create with the bound named.
- Queued-message terminal policy: messages queued for a parked/busy child deliver at the next boundary or revival; target expiry or drain terminalizes them `failed` with that reason — no retry knob exists.
- Await timeout clamps to a 30-minute maximum (the session-wait ceiling); an over-max request is clamped, not rejected, and the response reflects the clamped value — identical on tool, CLI, HTTP, and UDS.
- The operator-caller session is one durable role-tagged session per workspace, created on first operator call; it holds no model runtime, never counts against agent liveness caps, and its deliveries render as attention/app items.
- Concurrency defaults: `max_children = 5` per parent (admission wall — rejects; today's spawn default preserved), `max_active_per_root = 32` per governed root (execution budget — admitted work and revivals over it queue via `call_activation` task_runs, visible as `queued`). Revival policy: reviving a parked child — by call or by message — is an activation like any other: it never rejects for capacity; over the root budget it queues its activation run and proceeds as capacity frees (the child already exists; rejection would force a context-losing re-spawn).
- Registry rows are retained indefinitely in v1 (schemas are small and deduped); blob retention follows existing session-data retention.
- The internal spawn engine keeps serving internal roles unchanged; coordinator spawn remains refused on the call path exactly as it is on spawn today.
- `compozy exec` does not exist and is not promised by this feature; headless journeys use `compozy call` + `compozy message`.

Compozy Impact Audit:

- Native tools: adds `compozy__agent_call`, `compozy__call_return`, `compozy__call_await`, `compozy__call_cancel`, `compozy__call_result`, `compozy__call_publish`, `compozy__agent_message` (new toolset `ToolsetIDCalls`, descriptors + computed schema digests + risk classes: `agent_call`/`agent_message`/`call_cancel`/`call_return`/`call_publish` mutating, `call_await`/`call_result` read); deletes `compozy__session_spawn`; modifies `compozy__agent_create`/`agent_list` schemas (+`description`) and `compozy__session_stop` (+`subtree`); availability diagnostics per dependency closures.
- Extensibility and hooks: new `call` hook family — exactly seven events: `call.created`, `call.settled`, `call.canceled`, `call.published`, `call.message_sent`, `call.message_delivered`, `call.subtree_drained` (one naming table shared with the canonical events); Host API +4 read methods with consent contracts; AGENT.md gains `description` (registry/digest impact); no bridge SDK or MCP sidecar contract changes (checked: `internal/bridgesdk`, sidecar lifecycle); config lifecycle: new `[calls]` section fully wired (structs/defaults/merge/validation/tool-surface/docs/tests).
- Workspace data isolation: every new datum (calls, messages, deliveries, blobs, registry rows, activation-run side table, operator-caller bindings) is workspace-scoped by column + store-layer filters; list/read/stream/cache paths carry `workspace_id`; cross-workspace targets denied pre-side-effect (invariant 9); operator-caller bindings and drain fences cascade on workspace deletion; SSE catalog events scoped per workspace; web queries keyed by workspace (query-key factories). Proof obligations land in `_tests.md` (isolation cases per surface).
- Official Compozy skill: `skills/compozy/SKILL.md` router row + new `references/agent-comms.md`; updates to `references/native-tools.md` (new family section + spawn removal), `references/agent-definitions.md` (`description`), `references/configuration.md` (`[calls]`), `references/runtime-operations.md` (call/message operations, spawn removal).

Web/Docs Impact: `web/` — new `systems/agent-comms/`, Agents app locations, session inspector tab + timeline rows, attention model + badge union, generated types refresh; `packages/site` — new `content/docs/agent-comms/` folder (modeled on `network/`: index, calls, mailbox, subagents, budgets-and-safety pages) + `meta.json` wiring + config reference section + regenerated CLI/API references.

## Architecture Decision Records

- [ADR-001: Typed call result lives in a first-class durable call record](adrs/adr-001.md)
- [ADR-002: One agent-facing call verb; the spawn surface is deleted](adrs/adr-002.md)
- [ADR-003: Finished children are parked and revivable; TTL is an idle ceiling](adrs/adr-003.md)
- [ADR-004: Hybrid cost semantics — durable delivery; wake consumes a turn on the shared substrate](adrs/adr-004.md)
- [ADR-005: Type-level disjunction from Compozy Network with a one-directional bridge](adrs/adr-005.md)
- [ADR-006: One contract package unifies all five structured-output pipelines](adrs/adr-006.md)
- [ADR-007: Explicit registry-name invocation, injected roster, `description` field, batch fan-out](adrs/adr-007.md)
- [ADR-008: Recursive delegation, default max depth 3, budget-based containment](adrs/adr-008.md)
- [ADR-009: Async-by-default calls; result-carrying wake; explicit bounded await](adrs/adr-009.md)
- [ADR-010: `internal/calls` + `internal/contracts` package architecture](adrs/adr-010.md)
- [ADR-011: Accounting-only call activations in v1 (no admission ceilings)](adrs/adr-011.md)
- [ADR-012: Loop nodes adopt the contract regime without call records](adrs/adr-012.md)
- [ADR-013: Digest-keyed contract registry with compiled-schema cache](adrs/adr-013.md)
